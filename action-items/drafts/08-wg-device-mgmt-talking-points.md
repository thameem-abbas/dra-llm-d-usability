# Draft: WG-Device-Management Talking Points

**Target:** WG-Device-Management meeting
**Action:** User presents
**Status:** Ready for review

---

## Topic

"Workload API + DRA Integration: Who owns this gap?"

## Context to Establish (1 min)

We run LLM inference on LWS with DRA-based GPU-NIC pairing. Hit two layers of gaps:
- **Device layer:** Webhook reimplements scheduler topology logic (filed on KEP-5732)
- **Workload layer:** LWS has no ResourceClaim support, and the upstream path is circular

## The Circular Dependency — Partially Resolved (2 min)

Lay out the situation factually, then ask about remaining gaps:

1. **LWS #444** (ResourceClaimTemplate for LWS) → stalled, deferred to "Workload API integration"
2. **JobSet #762** (same gap for JobSet) → stalled, also "waiting on Workload API"
3. **KEP #4671** (Gang Scheduling / Workload API) → explicitly excludes ResourceClaims
4. **KEP #5488** (Multi-host DRA UX — the bridge) → closed by triage bot, April 2026
5. **KEP #5729** (ResourceClaim Support for Workloads) → **Alpha v1.36, active.** Adds ResourceClaimTemplates to PodGroup. Solves shared-claim-per-group.
6. **KEP #6012** (CompositePodGroup) → hierarchical grouping, delegates ResourceClaim work to KEP-5729

KEP-5729 breaks the circular dependency for the template model. But our use case needs per-replica unique claims (each pod gets its own GPU-NIC pair), not a shared claim across the group. The pool model also remains unaddressed.

## Update Since Draft (May–June 2026)

- **KEP-5729 scoping confirmed**: nojnhuh (KEP author) confirmed per-replica unique claims is "a different problem." Shared claims per PodGroup only.
- **[#6048](https://github.com/kubernetes/enhancements/issues/6048) opened**: johnbelamaric proposed "Consume From Resource Claim" — reservation mechanism where first pod allocates pool, subsequent pods consume subsets. Directly addresses pool model.
- **Composite driver built and open sourced**: [openshift-psap/composite-dra-driver](https://github.com/openshift-psap/composite-dra-driver). The "proxy driver" concept is implemented and running — not theoretical.
- **[wg-device-management #54](https://github.com/kubernetes-sigs/wg-device-management/issues/54) opened** (June 2026): Documents the cross-driver device composition pattern, asks for community feedback on design questions below.

## What We Built

The composite driver solves the device layer gaps by operating as a native DRA driver:
- Watches underlying driver ResourceSlices per node, pairs devices by `matchAttribute` (pcieRoot), publishes composite ResourceSlices
- At Prepare time: RailConfigResolver resolves per-NIC routing config, creates shadow ResourceClaims pre-filled with allocation results + opaque params, delegates to underlying drivers via gRPC
- Deployed as DaemonSet via Helm chart; per-node independence, no cross-node coordination; BoltDB state persistence

Solves: per-device driver configuration, admission-scheduling race, orphan cleanup. Remaining: per-replica unique claims guarantee, bin-packing policy.

## Open Design Questions (wg-device-management #54)

Community input requested:

1. **Shadow claims pattern**: Should "delegating drivers" be formalized in the DRA API/ecosystem, or kept as an implementation detail? We use shadow ResourceClaims (pre-filled allocation, owned by composite claim) to delegate to underlying drivers.
2. **Virtual ResourceSlice publishing**: Should there be a formal concept of "derived" or "composed" ResourceSlices? Currently composite driver publishes standard ResourceSlices with no signal that they're synthesized.
3. **Cross-pool device exclusion**: Same physical device can appear in multiple composition pools. DRA has no cross-pool mutual exclusion — the scheduler can double-allocate. (a) Pairer-side partitioning: static assignment to one pool, no race, wastes capacity. (b) Continuous recomputation: re-watch slices, drop composed devices whose members are allocated, small race window. Is this a gap the DRA API should address, or does it belong at the driver layer? ([composite-dra-driver #28](https://github.com/openshift-psap/composite-dra-driver/issues/28))

## Questions for the Meeting (updated)

1. Does KEP-5729 satisfy LWS #444 and JobSet #762, or do those issues need additional work?
2. **Per-replica unique claims out of scope for KEP-5729.** Is #6048 the right vehicle for formal per-replica distribution, or a separate KEP?
3. **#6048 (Consume From)** — can a reservation span drivers (GPU from `gpu.nvidia.com` + NIC from `dra.net` with `matchAttribute` on `pcieRoot`)?
4. **Composite device (we built it)** — feedback on the three design questions above. Should any of these become KEPs?
5. Should LWS/JobSet adopt KEP-5729 now (alpha) or wait for beta?

## Links to Share

- Composite DRA driver: https://github.com/openshift-psap/composite-dra-driver
- wg-device-management issue: https://github.com/kubernetes-sigs/wg-device-management/issues/54
- LWS gap analysis: https://github.com/thameem-abbas/dra-llm-d-usability/blob/main/lws-resourceclaim-gap.md
- Full usability repo: https://github.com/thameem-abbas/dra-llm-d-usability
