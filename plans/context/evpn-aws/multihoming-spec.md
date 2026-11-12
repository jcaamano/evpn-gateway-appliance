# Appliance-side all-active multihoming — implementation plan

Requirement: [CORENET-7502](https://redhat.atlassian.net/browse/CORENET-7502)
(all-active multihoming), a child of
[CORENET-7498](https://redhat.atlassian.net/browse/CORENET-7498). This note is a
**proposal**, not an approved topology; the Jira acceptance criteria are the
requirement, and an item here is a decision only once
[`kickoff-decisions.md`](kickoff-decisions.md) records an owner and date. It
expands [`networking-spec.md` item 6](networking-spec.md) and includes the
data-plane rearchitecture (see
[§3](#3-data-plane-rearchitecture-svd--vlan-aware-bridge)), which this plan owns —
multihoming is its main driver, not CORENET-7499.

Baseline: the imported design (`docs/design.md` §7, the single-paragraph "EVPN
All-Active Multi-Homing", added by
[PR #17](https://github.com/openshift/evpn-gateway-appliance/pull/17)) and the
single-homed roles under `ansible/roles/`. The design treats this as "LACP bond + shared Ethernet Segment,
FRR supports it natively"; this plan records what that actually requires on the
appliance.

## 1. Scope

**In scope — appliance side, single relay:**

- Two on-prem appliances in EVPN all-active multihoming toward one customer-edge
  (CE) LACP bundle (ESI-LAG).
- Appliance data plane: VLAN-aware bridge (SVD target; per-VNI VXLAN access ports
  interim) and bonding.
- Appliance control plane: FRR Ethernet Segment config, Type-1/Type-4 routes, DF
  election, split-horizon.
- Appliance BGP peering to a **single** relay, including how the pair exchanges the
  Ethernet-Segment routes.
- Ansible role/variable changes and multihoming health checks.

**Out of scope (deferred):**

- Relay HA / second relay ([CORENET-7503](https://redhat.atlassian.net/browse/CORENET-7503));
  this plan assumes one relay. The relay-side and cloud-side `allowas-in` symmetry
  ([§5](#5-bgp-peering-to-a-single-relay)) is noted but not specified here.
- CE / NSX-T edge configuration. The CE is a **dependency** ([§7](#7-dependencies-and-assumptions)),
  not an appliance deliverable.
- WireGuard data-plane redundancy across multiple relays. WireGuard is lab-only
  (CORENET-7498); production transports (DX/IPSec) keep the relay out of the data
  path, so appliance MH is independent of tunnel-side ECMP.
- Single-active redundancy mode (not pursued; MH-2 is settled as all-active).

## 2. Open decisions

Record owner, date and evidence in [`kickoff-decisions.md`](kickoff-decisions.md)
before building the affected workstream.

**Settled direction (pending a formal owner/date record): MH-2 — the redundancy
mode is all-active.** Grounds: CORENET-7502 asks for it, FRR's EVPN-MH is
all-active-centric (single-active support is weak), and both appliances should
carry traffic. A further reason: all-active keeps the appliance HA layer
**decoupled from edge failover** — an edge active/standby switch is just a
MAC-relearn on the appliances, with no appliance HA state change or DF switch to
follow; single-active would instead couple the two layers through non-DF
link-state signaling (fragile, and weak in FRR). The consequence is that a
**bare-metal CE that runs LACP becomes a
hard prerequisite**, not a fork — single-active is not pursued as a fallback (it
would not rescue an Edge-VM CE anyway; see [§7](#7-dependencies-and-assumptions)).
Note: the edge being active/standby does not by itself *force* all-active (both
modes could coexist with it); the grounds above are what settle it.

| # | Decision | Proposal | Owner |
| --- | --- | --- | --- |
| MH-1 | Ethernet-Segment route exchange between the two appliances: direct A–B iBGP vs. no iBGP + `allowas-in` on the appliance↔relay sessions | **Direct A–B iBGP** (canonical, no loop-check relaxation, decouples DF from relay availability). See [§5](#5-bgp-peering-to-a-single-relay) | Networking / Architecture |
| MH-3 | `es-sys-mac` / `es-id` allocation scheme and where it is tracked | Locally-administered MACs (`02:…`), one per Ethernet Segment, declared in `group_vars`; `es-id` mirrors the last octet ([§4](#4-ethernet-segment-identity)) | Networking |
| MH-4 | Appliance ASN model: shared ASN for A and B (drives MH-1) vs. per-appliance ASN | Shared ASN (matches `group_vars/all.yml` `onprem_asn: 65044`); this is what makes MH-1 a real choice | Networking |
| MH-5 | Data-plane model realizing the VLAN-aware bridge: full SVD vs. per-VNI VXLAN access ports | **SVD is the recommended target but currently undoable** (no nmstate/NM support); use **per-VNI VXLAN access ports** on the `vlan_filtering` bridge until SVD is supported ([§3](#3-data-plane-rearchitecture-svd--vlan-aware-bridge)) | Networking |

## 3. Data-plane rearchitecture (SVD / VLAN-aware bridge)

**This is the largest structural change and a hard prerequisite.** FRR/Linux
EVPN multihoming requires a **VLAN-aware bridge** (`vlan_filtering=1`). The
single-homed design (`docs/design.md` §3) uses **per-VNI bridges** (`br-vni10100`)
fed by VLAN sub-interfaces, which does **not** support the MH feature set, so
multihoming forces the VLAN-aware model. This data-plane rearchitecture is **owned
by this plan** — multihoming is its main driver; it does **not** overlap with or
depend on CORENET-7499.

A VLAN-aware bridge can be populated two ways (MH-5):

- **SVD (single VXLAN device)** — one VXLAN device for all VNIs (collect-metadata,
  `bridge vlan … tunnel_info`). The **recommended target**, but **currently
  undoable**: nmstate/NM have no support for the single VXLAN device and its
  VLAN→VNI tunnel mapping.
- **Per-VNI VXLAN access ports** — keep one VXLAN device per VNI (as today) but
  enslave each to the single VLAN-aware bridge as an untagged/PVID access port. The
  **doable approach now** and a smaller delta from the current design. The one thing
  to get right is creating/attaching each VXLAN device appropriately for the
  VLAN-aware bridge (PVID-untagged per VLAN, `nolearning`, and `neigh_suppress` for
  ARP/ND suppression — the port-flag persistence gap in
  [`networking-spec.md` item 5](networking-spec.md), needing a keyfile workaround).

Per appliance:

- One VLAN-aware bridge carrying all stretched VLANs as tagged trunk ports.
- VXLAN devices per the MH-5 model: per-VNI VXLAN access ports now; a single
  collect-metadata device once nmstate/NM support SVD.
- The CE-facing uplink(s) are **802.3ad bonds**, enslaved to the bridge as tagged
  trunks. A single-member bond is still a bond (it must run LACP and carry the ES).

Confirm the FRR version in the bootc payload supports the required EVPN-MH
primitives before relying on this ([payload evidence](source-evidence.md#9-payload-platform-and-ocp-test-inputs)).

## 4. Ethernet-Segment identity

Each CE LACP bundle is one Ethernet Segment. Both appliances present the **same
LACP system identity** on their member link(s) so the CE sees one partner:

- Linux bond: `mode 802.3ad`, `lacp_rate fast`, and `ad_actor_system` set to the
  **same** MAC on A and B for that segment.
- FRR on the bond: `evpn mh es-id <n>` + `evpn mh es-sys-mac <MAC>`, with
  `es-sys-mac` **equal to** the bond's `ad_actor_system`.

Matching rules:

- `ad_actor_system` and `es-sys-mac` are **identical across A and B** for a given
  segment, and **unique per segment**.
- `es-sys-mac` equals `ad_actor_system` on each appliance.
- The value is a **logical identifier**, not an interface hardware address — do not
  assign it with `ip link set … address`. Use a **locally-administered** MAC
  (`02:…`, even first octet) so it cannot collide with vendor/hypervisor MACs;
  vary the last octet per segment and track the allocation (MH-3).

DF election and Type-1/Type-4 generation are **automatic** once the ES is
configured; no per-route-type knob. `evpn mh es-df-pref` is an optional
preference override. No peer-link or inter-appliance BFD is required — the pair
coordinates through BGP, not a shared link.

## 5. BGP peering to a single relay

The relay stays as today: eBGP, `next-hop-unchanged` (must apply to Type-1/Type-4
as well as Type-2/3, so DF election reads the originating PE). `retain
route-target all` is **not** required on a relay with no local VNIs — it only
filters when the box imports RTs locally (confirmed against a VNI-less FRR relay).

The pair must exchange **Type-4** (ES discovery + DF) and the local **Type-1**.
Because A and B share an ASN (MH-4), routing those through the eBGP relay trips the
AS-path loop check (the relay re-advertises `65044` back toward `65044`). Two ways
to resolve it (MH-1):

- **Direct A–B iBGP (recommended).** One iBGP session between the pair carries
  Type-4/Type-1; the relay is untouched; no loop-check relaxation; DF keeps working
  even if the relay is down. The relay still carries the appliances' Type-1 EAD to
  the cloud for remote aliasing — a cross-ASN path with no loop.
- **No iBGP + `allowas-in`.** Add `allowas-in 1` on each appliance↔relay EVPN
  session so A and B accept each other's same-ASN routes via the relay. This works
  but relaxes loop protection and ties DF-route exchange to relay availability
  (a single relay then becomes a DF SPOF as well). Note the same relaxation would
  be needed cloud-side for same-ASN OCP↔OCP exchange — deferred with relay HA.

Enable **BFD** on the appliance↔relay sessions so failover is sub-second rather
than the BGP hold-timer.

## 6. Implementation workstreams

1. **Data plane.** Rework `evpn_onprem_appliance` to the VLAN-aware bridge with
   per-VNI VXLAN access ports (MH-5; SVD once nmstate/NM support it); add bond
   creation (`ad_actor_system`). Build it as NM keyfiles, not ephemeral `ip link` —
   this plan owns that work.
2. **FRR EVPN-MH.** Add ES stanzas to the on-prem FRR template (`es-id`,
   `es-sys-mac`, optional `es-df-pref`); ensure Type-1/Type-4 advertisement and
   split-horizon are active.
3. **Inventory/variables.** Add a second appliance to `onprem_appliances`; add an
   Ethernet-Segment block (`es_id`, `es_sys_mac`, bond members) per segment;
   decide MH-3 allocation.
4. **BGP.** Implement MH-1 (A–B iBGP session, or `allowas-in`); add BFD; verify the
   relay template preserves next-hops for Type-1/4.
5. **Health checks.** Extend `evpn_health_check` ([§8](#8-validation-and-health-checks)).
6. **Validation.** Molecule/emulated lane proving two appliances, DF election and
   failover (align with [CORENET-7508 e2e](pipeline-spec.md#3-integration-gates-and-test-infrastructure)).

## 7. Dependencies and assumptions

- **CE must be able to form the ESI-LAG (hard prerequisite).** With MH-2 settled as
  all-active, the CE must run LACP into one bundle spanning both appliances. For an
  NSX-T edge this means a **bare-metal edge** (an Edge VM cannot: its vNICs
  terminate on the ESXi vSwitch, not on the appliances, so no ESI-LAG can form — and
  that adjacency gap also rules out single-active as an ESI, so there is no in-band
  fallback). **Risk:** if the CE turns out to be an Edge VM, or a bare-metal edge
  that will not run LACP, appliance multihoming as specified is not achievable and
  the HA approach must be revisited. The appliance cannot satisfy this on its own.
- **nmstate/NM support** for the chosen data-plane model (MH-5): SVD is currently
  unsupported by nmstate/NM, and per-VNI VXLAN access ports still hit the
  `neigh_suppress`/port-flag persistence gap in
  [`networking-spec.md` item 5](networking-spec.md) (keyfile workaround). This plan
  owns closing these — no CORENET-7499 dependency.
- **Shared appliance ASN** (MH-4) — the premise behind MH-1.
- NSX-T edge bridge HA is active/standby at the edge-node layer, so "all-active"
  applies below the active edge; failover between edges is MAC-mobility/RARP-driven
  and gated by the (slower) NSX layer. Document expected disruption rather than
  promising hitless failover.

## 8. Validation and health checks

Extend `evpn_health_check` (keep lab-only assertions gated, per
[agent-guide output rules](agent-guide.md)):

- Each appliance: bond aggregated (LACP `Collecting/Distributing`, partner MAC
  learned); Ethernet Segment up; Type-4 peer present; Type-1 EAD advertised.
- DF elected per VNI; the two appliances split DF duty across VLANs.
- No duplicate BUM delivered to the CE (duplicate-frame check per VLAN).
- Failover: kill appliance A, assert traffic converges to B via mass-withdraw and
  LACP member-down; measure disruption.
- Negative: with the ES misconfigured (mismatched `es-sys-mac`), assert the loop is
  caught rather than silently storming.

## 9. Deferred (tracked elsewhere)

- Relay HA / second relay and cloud-side ECMP — CORENET-7503.
- WireGuard multi-relay data-plane redundancy — lab-only.
- Single-active mode — not pursued (MH-2 settled as all-active).
- Serial HA-peer upgrade and failed-upgrade rollback —
  [CORENET-7510](https://redhat.atlassian.net/browse/CORENET-7510).

Run [`tools/check-all.sh`](tools/check-all.sh) before committing.
