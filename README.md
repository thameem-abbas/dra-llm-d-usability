# DRA Usability: Upstream Engagement for llm-d

Analysis of gaps between Kubernetes DRA (Dynamic Resource Allocation) and LLM inference requirements — both at the device level (GPU-NIC topology) and the workload level (LWS/JobSet claim distribution). Tracks upstream KEP engagement and contributions.

## Context

Two layers of DRA gaps affect LLM inference workloads:

**Device layer**: The [dra-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook) converts a synthetic resource request (`dra.llm-d.io/gpu-nic-pair: "N"`) into full DRA objects — ResourceClaimTemplates with CEL selectors, PCIe MatchAttribute constraints, NUMA co-location, and rail-aware NIC allocation. Much of this is scheduler-level work done in an admission webhook because upstream DRA doesn't support topology-aware multi-device allocation yet.

**Workload layer**: LeaderWorkerSet (LWS) — the workload construct for prefill/decode disaggregation — has no ResourceClaim support. The upstream path to workload-level DRA integration is fragmented and stalled, with the closest unified solution (KEP #5488) closed with no replacement.

This repo documents both layers of gaps, proposes upstream contributions, and tracks progress.

## Gap Analysis

| Document | Description |
|----------|-------------|
| [kep-5732-use-case.md](kep-5732-use-case.md) | Use case for KEP-5732 (Topology-Aware Scheduling) — our webhook IS the motivating example |
| [kep-5491-feedback.md](kep-5491-feedback.md) | Consumer feedback on KEP-5491 (List-Typed Attributes) — scalar-to-list migration concerns for pcieRoot/numaNode |
| [cel-selector-patterns.md](cel-selector-patterns.md) | Three reusable CEL selector patterns: rail-specific RDMA, explicit device pinning, cross-driver topology alignment |
| [dra-webhook-best-practices.md](dra-webhook-best-practices.md) | Two operational patterns: orphan cleanup reconciler and priority queue batch mutation |
| [lws-resourceclaim-gap.md](lws-resourceclaim-gap.md) | LWS has no ResourceClaim support — template model stalled, pool model not covered anywhere upstream |

## Upstream Gaps Summary

### Device Layer (DRA Admission Webhook)

Features this webhook implements that upstream DRA doesn't support (yet):

1. **Rail-aware network topology allocation** — no concept of network rails or multi-subnet RDMA fabrics in DRA
2. **Batch mutation with priority ordering** — no batch-aware admission for DRA resources
3. **Cluster-level pending reservation tracking** — no coordination mechanism for admission webhooks
4. **Orphan resource cleanup** — no built-in GC for webhook-created DRA objects without ownerReferences
5. **Explicit device pairing mode** — no "admin declares exact device sets" pattern in DRA

### Workload Layer (LWS / JobSet)

6. **Replica-level ResourceClaim sharing** — LWS leader + workers in a replica can't share claims ([LWS #444](https://github.com/kubernetes-sigs/lws/issues/444) stalled)
7. **Finite-pool claim distribution** — no mechanism to distribute pre-existing claims across replicas and reclaim on scale-down (not covered anywhere)
8. **Cross-replica claim coordination** — no workload-level view of which replicas hold which claims
9. **Workload API + DRA integration** — KEP #5488 (unified solution) closed, but KEP-5729 (ResourceClaim Support for Workloads) is now alpha in v1.36. Covers shared claims per PodGroup but not per-replica unique claims.

## KEP Alignment (as of Kubernetes v1.36, April 2026)

| KEP | Status | Relationship |
|-----|--------|-------------|
| KEP-4381 (DRA Core) | GA v1.34 | Foundation — webhook uses v1 API with `Exactly` sub-field |
| KEP-5732 (Topology-Aware Scheduling) | Alpha v1.36 | Multi-pod placement (rack/pool level) — complementary to our intra-node device pairing, DRA integration deferred to beta |
| KEP-5491 (List Type Attributes) | Alpha v1.36 | Forward-compatible with our pcieRoot/numaNode MatchAttribute constraints — enables matching against devices with multi-valued topology attributes |
| ~~KEP-4816 (Prioritized Alternatives)~~ | GA v1.36 | ~~Not applicable — solves per-pod fallback, not coordinated cross-pod rail assignment~~ |
| ~~KEP-5055 (Device Taints)~~ | Beta v1.36 | ~~Not applicable — taints mark health/maintenance, our preflight checks allocation state not device health~~ |
| ~~KEP-5007 (Binding Conditions)~~ | Beta v1.36 | ~~Not applicable — post-allocation device readiness, our preflight is pre-allocation capacity checking~~ |
| ~~KEP-5517 (Node Allocatable Resources)~~ | Alpha v1.36 | ~~Misidentified — solves scheduler double-counting of CPU/memory between DRA and pod.spec.resources, not a device availability query API~~ |
| KEP-5004 (Extended Resources Bridge) | Beta v1.36 | Simple DRA path — does NOT cover our multi-device topology use case |
| KEP-5729 (ResourceClaim Support for Workloads) | Alpha v1.36 | Directly addresses LWS/JobSet ResourceClaim gap — shared claim per PodGroup, but not per-replica unique claims |
| KEP-6012 (CompositePodGroup) | Open, targeting Alpha v1.37 (WIP) | Hierarchical scheduling grouping — ResourceClaim scoping delegated to KEP-5729, no direct DRA integration in this KEP |
| KEP-5488 (Multi-host DRA UX) | **Closed** (April 2026) | Was the unified solution for workload-level DRA. Dead, no replacement. |
| LWS #444 (ResourceClaimTemplate) | Open, stalled | Template model for LWS — deferred to Workload API that has no ResourceClaim story |
| JobSet #762 (Job-level ResourceClaimTemplate) | Open, rotting | Same gap for JobSet — also deferred to Workload API |

## Action Items

See [action-items/](action-items/) for tracked upstream engagement tasks.
