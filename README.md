# DRA Usability: Upstream Engagement for llm-d

Analysis of gaps between Kubernetes DRA (Dynamic Resource Allocation) and LLM inference requirements — both at the device level (GPU-NIC topology) and the workload level (LWS/JobSet claim distribution). Tracks upstream KEP engagement and contributions.

## Reading Guide

**Application users** — just need to configure your workloads? Start here:

- [Quick Start: GPU-NIC Pairing](guide/00-quick-start.md)

**Want to understand DRA in depth?** Read the full guide:

1. [What Is Dynamic Resource Allocation?](guide/01-what-is-dra.md)
2. [How DRA Works Under the Hood](guide/02-how-dra-works.md)
3. [GPU DRA Drivers](guide/03-gpu-dra-drivers.md)
4. [NIC DRA Drivers and GPU-NIC Topology](guide/04-nic-dra-drivers.md)

## Context

Two layers of DRA gaps affect LLM inference workloads:

**Device layer**: The [dra-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook) converts a synthetic resource request (`dra.llm-d.io/gpu-nic-pair: "N"`) into full DRA objects — ResourceClaimTemplates with CEL selectors, PCIe MatchAttribute constraints, NUMA co-location, and rail-aware NIC allocation. The webhook now also supports extended resource interception (`nvidia.com/gpu` → DRA claims via `/mutate-ext`), InfiniBand fabrics (auto-detection + explicit `ibRails` config), AKS kustomize overlays, and an offline dry-run simulator. Much of this is scheduler-level work done in an admission webhook because upstream DRA doesn't support topology-aware multi-device allocation yet.

**Workload layer**: LeaderWorkerSet (LWS) — the workload construct for prefill/decode disaggregation — has no ResourceClaim support. The upstream path to workload-level DRA integration is fragmented and stalled, with the closest unified solution (KEP #5488) closed with no replacement.

This repo documents both layers of gaps, proposes upstream contributions, and tracks progress.

## Gap Analysis

| Document | Description |
|----------|-------------|
| [kep-5732-use-case.md](kep-5732-use-case.md) | Use case for KEP-5732 (Topology-Aware Scheduling) — our webhook IS the motivating example |
| [kep-5491-feedback.md](kep-5491-feedback.md) | Consumer feedback on KEP-5491 (List-Typed Attributes) — scalar-to-list migration concerns for pcieRoot/numaNode |
| [cel-selector-patterns.md](cel-selector-patterns.md) | Three reusable CEL selector patterns: rail-specific RDMA, explicit device pinning, cross-driver topology alignment |
| [dra-webhook-best-practices.md](dra-webhook-best-practices.md) | Four operational patterns: orphan cleanup, batch mutation, extended resource interception, offline dry-run simulation |
| [lws-resourceclaim-gap.md](lws-resourceclaim-gap.md) | LWS has no ResourceClaim support — template model stalled, pool model not covered anywhere upstream |

## Upstream Gaps Summary

### Device Layer (DRA Admission Webhook)

Features this webhook implements that upstream DRA doesn't support (yet):

1. **Rail-aware network topology allocation** — no concept of network rails or multi-subnet RDMA fabrics in DRA. Webhook supports both Ethernet and InfiniBand fabrics with auto-detection.
2. **Extended resource interception** — `/mutate-ext` endpoint converts `nvidia.com/gpu` requests to DRA ResourceClaims cluster-wide, bridging device plugin → DRA migration before Kubernetes 1.35+ `DRAExtendedResource` feature gate
3. **Batch mutation with priority ordering** — no batch-aware admission for DRA resources
4. **Cluster-level pending reservation tracking** — no coordination mechanism for admission webhooks
5. **Orphan resource cleanup** — no built-in GC for webhook-created DRA objects without ownerReferences
6. **Explicit device pairing mode** — admin declares exact device-to-device mappings per node pool (experimental, 4 tracked issues)
7. **Offline dry-run simulation** — capture cluster state + simulate allocation without touching live cluster. No upstream equivalent for DRA configuration validation.

### Workload Layer (LWS / JobSet)

6. **Replica-level ResourceClaim sharing** — LWS leader + workers in a replica can't share claims ([LWS #444](https://github.com/kubernetes-sigs/lws/issues/444) stalled)
7. **Finite-pool claim distribution** — no mechanism to distribute pre-existing claims across replicas and reclaim on scale-down (not covered anywhere)
8. **Cross-replica claim coordination** — no workload-level view of which replicas hold which claims
9. **Workload API + DRA integration** — KEP #5488 (unified solution) auto-closed. KEP-5729 (ResourceClaim Support for Workloads) is alpha in v1.36, covers shared claims per PodGroup. Per-replica unique claims confirmed out of scope by KEP author (May 2026). [#6048](https://github.com/kubernetes/enhancements/issues/6048) (Consume From Resource Claim) proposes a reservation mechanism that may address the pool model gap.

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
| KEP-5004 (Extended Resources Bridge) | Beta v1.36 | Simple DRA path — does NOT cover our multi-device topology use case. Our webhook's `/mutate-ext` provides a similar bridge for `nvidia.com/gpu` → DRA before Kubernetes 1.35+ `DRAExtendedResource` feature gate. |
| KEP-5729 (ResourceClaim Support for Workloads) | Alpha v1.36 | Shared claim per PodGroup. **Per-replica unique claims confirmed out of scope** by KEP author nojnhuh (May 2026). johnbelamaric reviewing. |
| KEP-6012 (CompositePodGroup) | Open, targeting Alpha v1.37 (WIP) | Hierarchical scheduling grouping — ResourceClaim scoping delegated to KEP-5729, no direct DRA integration in this KEP |
| KEP-5488 (Multi-host DRA UX) | **Closed** (auto-closed by lifecycle/rotten bot, April 18, 2026) | Was the unified solution for workload-level DRA. Dead, no replacement. |
| KEP-6080 (Derived Attributes) | Provisional, targeting Alpha v1.37 | Eliminates cross-driver attribute name standardization requirement — `derivedAttributes` synthesize virtual grouping keys via CEL. Directly solves our Pattern 3 rigidity (both drivers must publish identical `pcieRoot` attribute). |
| [#6048](https://github.com/kubernetes/enhancements/issues/6048) (Consume From Resource Claim) | Open, April 2026 | Proposes reservation mechanism — first pod allocates pool, subsequent pods consume subsets. Directly addresses pool model gap. |
| [#6006](https://github.com/kubernetes/enhancements/issues/6006) (Health-Aware Topology) | Open | Performance-based device placement (cooling, network runs, burn-in variance). KubeCon in-person discussion April 2026. |
| LWS #444 (ResourceClaimTemplate) | Open, stalled | Template model for LWS — deferred to Workload API. Edwinhr716 confirmed direction (March 2026). |
| JobSet #762 (Job-level ResourceClaimTemplate) | Open, stale (April 2026) | Same gap for JobSet — deferred to Workload API, kannon92 keeping open. |

### New DRA KEPs (March–May 2026)

Recent upstream activity — 8 new DRA-related issues opened in the last two months:

| Issue | Title | Relevance |
|-------|-------|-----------|
| [#6048](https://github.com/kubernetes/enhancements/issues/6048) | Consume From Resource Claim | Pool model — reservation-based claim consumption |
| [#6080](https://github.com/kubernetes/enhancements/issues/6080) | DRA: Derived Attributes | Virtual grouping keys via CEL — eliminates cross-driver attribute standardization requirement. [PR #6081](https://github.com/kubernetes/enhancements/pull/6081). johnbelamaric tagged us. |
| [#5978](https://github.com/kubernetes/enhancements/issues/5978) | ClusterResourceClaimTemplate | Cluster-scoped templates — may simplify pool model |
| [#5963](https://github.com/kubernetes/enhancements/issues/5963) | Device Compatibility Groups | Cross-driver device grouping — related to GPU-NIC pairing |
| [#5961](https://github.com/kubernetes/enhancements/issues/5961) | Device Profiles: Mutually Exclusive Allocations | Exclusive device sets — related to rail isolation |
| [#5981](https://github.com/kubernetes/enhancements/issues/5981) | Sharing Affinity for Consumable Capacity | Shared resource affinity |
| [#5941](https://github.com/kubernetes/enhancements/issues/5941) | Shared Consumable Capacity | Shared consumable resources |
| [#5993](https://github.com/kubernetes/enhancements/issues/5993) | Context Locked Effective Capacity | Capacity locking semantics |
| [#5945](https://github.com/kubernetes/enhancements/issues/5945) | Optional Node Lifecycle | Optional node lifecycle for devices |

### KEP Dependency Graph

```
    DEVICE LAYER                         WORKLOAD LAYER
    (intra-node topology)                (claim distribution)

    ┌──────────────┐                     ┌──────────────────┐
    │ KEP-4381     │                     │ KEP-4671         │
    │ DRA Core     │                     │ Gang Scheduling  │
    │ (GA v1.34)   │                     │ (Alpha v1.36)    │
    └──────┬───────┘                     │ NO ResourceClaim │
           │                             └────────┬─────────┘
           │ builds on                            │
           ▼                                      ▼
    ┌──────────────┐  DRA integration    ┌──────────────────┐
    │ KEP-5732     │  deferred to beta   │ KEP-5729         │
    │ Topology     │◄─ ─ ─ ─ ─ ─ ─ ─ ─ ─│ ResourceClaim    │
    │ Scheduling   │                     │ for Workloads    │
    │ (Alpha v1.36)│                     │ (Alpha v1.36)    │
    └──────────────┘                     │ CONFIRMED: no    │
           │                             │ per-replica      │
           │                             └────────┬─────────┘
    ┌──────▼──────┐                               │
    │ KEP-5491    │                      ┌────────▼─────────┐
    │ List Attrs  │◄─ ─ ─ ─ ┐           │ KEP-6012         │
    │ (Alpha 1.36)│         │           │ CompositePodGroup│
    └─────────────┘         │           │ (WIP, v1.37)     │
                    ┌───────┴────────┐  └──────────────────┘
                    │ KEP-6080       │
                    │ Derived Attrs  │─ ─ ─ ▶ solves cross-driver
                    │ (Prov, v1.37)  │        matchAttribute rigidity
                    └────────────────┘
    ┌─────────────┐
    │ #6048       │─ ─ ─ ─ ─ ▶ addresses pool model gap
    │ Consume From│
    │ (Open)      │
    └─────────────┘

    ╳ KEP-5488 (Multi-host DRA UX) — CLOSED April 18, 2026

    Legend:  ──▶ depends on    ─ ─▶ relates to    ╳ dead
```

## Action Items

See [action-items/](action-items/) for tracked upstream engagement tasks.
