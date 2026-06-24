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

**Device layer**: The [composite-dra-driver](https://github.com/openshift-psap/composite-dra-driver) is the current solution — a native DRA driver that watches underlying GPU and NIC ResourceSlices, pairs them by PCIe root, and publishes composite ResourceSlices. The scheduler allocates from these natively; no admission-time decisions. Users request `composite.dra/gpu-nic-pair: "N"` (or `nvidia.com/gpu: N` via ExtendedResource on K8s 1.36+). Deployed as DaemonSet via Helm chart. The previous approach — [dra-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook) — had admission-scheduling races and orphan resource accumulation inherent to any mutating webhook making allocation decisions outside the scheduler.

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
| KEP-5732 (Topology-Aware Scheduling) | Alpha v1.36, milestoned v1.37 | Multi-pod placement (rack/pool level) — complementary to our intra-node device pairing, DRA integration deferred to beta. kannon92 invited us to present at Monday calls. |
| KEP-5491 (List Type Attributes) | Alpha v1.36, second alpha v1.37 | Forward-compatible with our pcieRoot/numaNode MatchAttribute constraints. **everpeace confirmed** (May 2026): `.includes()` is universal CEL pattern, scalar-as-singleton-set is scheduler-side only, migration path will be documented in KEP. |
| ~~KEP-4816 (Prioritized Alternatives)~~ | GA v1.36 | ~~Not applicable — solves per-pod fallback, not coordinated cross-pod rail assignment~~ |
| ~~KEP-5055 (Device Taints)~~ | Beta v1.36 | ~~Not applicable — taints mark health/maintenance, our preflight checks allocation state not device health~~ |
| ~~KEP-5007 (Binding Conditions)~~ | Beta v1.36 | ~~Not applicable — post-allocation device readiness, our preflight is pre-allocation capacity checking~~ |
| ~~KEP-5517 (Node Allocatable Resources)~~ | Alpha v1.36 | ~~Misidentified — solves scheduler double-counting of CPU/memory between DRA and pod.spec.resources, not a device availability query API~~ |
| KEP-5004 (Extended Resources Bridge) | Beta v1.36 | Simple DRA path — does NOT cover our multi-device topology use case. Our webhook's `/mutate-ext` provides a similar bridge for `nvidia.com/gpu` → DRA before Kubernetes 1.35+ `DRAExtendedResource` feature gate. |
| KEP-5729 (ResourceClaim Support for Workloads) | Alpha v1.36 | Shared claim per PodGroup. **Per-replica unique claims confirmed out of scope** by KEP author nojnhuh (May 2026). johnbelamaric reviewing. |
| KEP-6012 (CompositePodGroup) | Open, targeting Alpha v1.37 (WIP) | Hierarchical scheduling grouping — ResourceClaim scoping delegated to KEP-5729, no direct DRA integration in this KEP |
| KEP-5488 (Multi-host DRA UX) | **Closed** (auto-closed by lifecycle/rotten bot, April 18, 2026) | Was the unified solution for workload-level DRA. Dead, no replacement. |
| KEP-6080 (Derived Attributes) | Provisional, Alpha v1.37 — kad LGTM, pohly reviewing | Eliminates cross-driver attribute name standardization requirement — `derivedAttributes` synthesize virtual grouping keys via CEL. Directly solves our Pattern 3 rigidity (both drivers must publish identical `pcieRoot` attribute). |
| [#6048](https://github.com/kubernetes/enhancements/issues/6048) (Consume From Resource Claim) | Open, **pushed to v1.38** | Proposes reservation mechanism. Our first-pod-as-leader concern validated by wojtek-t — reservations should bind to workload intent, not runtime. Use case doc needed first. [wg-device-management #54](https://github.com/kubernetes-sigs/wg-device-management/issues/54) documents the per-replica distribution use case that #6048 may eventually generalize to cover. |
| [#6006](https://github.com/kubernetes/enhancements/issues/6006) (Health-Aware Topology) | Open | Performance-based device placement (cooling, network runs, burn-in variance). KubeCon in-person discussion April 2026. |
| LWS #444 (ResourceClaimTemplate) | Open, stalled | Template model for LWS — deferred to Workload API. Edwinhr716 confirmed direction (March 2026). |
| JobSet #762 (Job-level ResourceClaimTemplate) | Open, stale (April 2026) | Same gap for JobSet — deferred to Workload API, kannon92 keeping open. |

### New DRA KEPs (March–June 2026)

Recent upstream activity:

| Issue | Title | Relevance |
|-------|-------|-----------|
| [wg-device-management #54](https://github.com/kubernetes-sigs/wg-device-management/issues/54) | Cross-driver device composition via shadow claims | **Our issue** — documents composite driver pattern, asks for community feedback on shadow claims, virtual ResourceSlices, cross-pool exclusion |
| [#6048](https://github.com/kubernetes/enhancements/issues/6048) | Consume From Resource Claim | Pool model — reservation-based claim consumption. wg-device-management #54 documents the per-replica distribution need that #6048 may eventually cover. |
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
