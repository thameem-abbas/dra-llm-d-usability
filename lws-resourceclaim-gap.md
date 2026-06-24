# LWS ResourceClaim Support — Gap Analysis

## Summary

LeaderWorkerSet (LWS) has **no ResourceClaim support**. The only effort ([LWS #444](https://github.com/kubernetes-sigs/lws/issues/444)) is stalled and deferred to a Workload API integration that doesn't exist yet. The upstream path to workload-level DRA integration is fragmented — the closest unified solution ([KEP #5488](https://github.com/kubernetes/enhancements/issues/5488)) was closed by triage bot in April 2026 with no replacement filed.

This matters for LLM inference because LWS is the workload construct for prefill/decode disaggregation, and our [dra-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook) creates ResourceClaimTemplates per-pod — but there is no upstream mechanism to coordinate claims across replicas or distribute a finite pool of claims to LWS workers.

## The Gap: Two Missing Models

### Template Model (LWS #444 — stalled)

Create a new ResourceClaim per replica from a template, shared across all pods (leader + workers) in that replica. Naming: `{lws-name}-{replica-index}-{template-name}`.

- LWS #444 proposed this but is **unassigned**, deferred to Workload API
- JobSet has the same problem ([JobSet #762](https://github.com/kubernetes-sigs/jobset/issues/762)) — also stalled, also "waiting on Workload API"
- Both issues keep getting rescued from stale bots, indicating intent but no progress

### Pool Model (not covered anywhere)

A fixed number of pre-existing ResourceClaims (e.g., 4 IMEX channels, 8 TPU slices, N licensed accelerators) distributed across LWS replicas. Claims reclaimed and recycled on scale-down.

[Issue #6048](https://github.com/kubernetes/enhancements/issues/6048) (DRA: Consume From Resource Claim) proposes a reservation mechanism that may address this gap — a first pod allocates a pool of resources, and subsequent pods consume subsets from that reservation. Early stage (opened April 30, 2026, no KEP PR yet).

| | Template Model | Pool Model |
|---|---|---|
| Claim lifecycle | Created per replica, 1:1 | Pre-existing, reused |
| Scaling up | New claim from template | Claim assigned from pool |
| Scaling down | Claim deleted with replica | Claim returned to pool |
| Resource exhaustion | Unbounded (DRA scheduler decides) | Bounded by pool size |
| Use case | Per-replica GPU/NIC pairs | Shared hardware, licensed resources, IMEX channels |

### Claim Distribution Models

```
    TEMPLATE MODEL (LWS #444 / KEP-5729)
    ──────────────────────────────────────
    LWS creates new claim per replica from template

    LWS Replicas:    Replica-0       Replica-1       Replica-2
                        │               │               │
                     ┌──▼──┐         ┌──▼──┐         ┌──▼──┐
                     │Claim │         │Claim │         │Claim │
                     │ new  │         │ new  │         │ new  │
                     └──────┘         └──────┘         └──────┘
    Lifecycle:     created/deleted   created/deleted   created/deleted
                   with replica      with replica      with replica


    POOL MODEL (gap — partially addressed by #6048)
    ──────────────────────────────────────────────────
    Pre-existing claims distributed to replicas, reclaimed on scale-down

                   ┌──────────────────────────┐
                   │   Claim Pool (fixed)      │
                   │  ┌───┐ ┌───┐ ┌───┐ ┌───┐ │
                   │  │ A │ │ B │ │ C │ │ D │ │
                   │  └─┬─┘ └─┬─┘ └─┬─┘ └───┘ │  D = unassigned
                   └────┼─────┼─────┼──────────┘
                        │     │     │
                        ▼     ▼     ▼
    LWS Replicas:   Rep-0   Rep-1   Rep-2    (Rep-3 blocked: pool exhausted)


    CONSUME-FROM MODEL (#6048 — proposed)
    ──────────────────────────────────────
    First pod allocates pool claim; subsequent pods consume subsets

                   ┌────────────────────┐
                   │  Reservation       │
                   │  (pool of devices) │
                   │  allocated by      │
                   │  first pod         │
                   └─────────┬──────────┘
                      ┌──────┼──────┐
                      ▼      ▼      ▼
                   ┌────┐ ┌────┐ ┌────┐
                   │Pod0│ │Pod1│ │Pod2│
                   │ 2  │ │ 2  │ │ 2  │  each consumes
                   │GPUs│ │GPUs│ │GPUs│  subset from pool
                   └────┘ └────┘ └────┘
    PodSpec: reservationName: "my-reservation"
```

## Upstream Landscape

| KEP / Issue | Status | ResourceClaim Relevance |
|-------------|--------|------------------------|
| **LWS #444** (ResourceClaimTemplate) | Open, stalled | Template model for LWS. Deferred to Workload API. Edwinhr716 removed stale label March 2026, confirmed direction. |
| **JobSet #762** (Job-level ResourceClaimTemplate) | Open, stale (April 2026) | Same gap for JobSet. kannon92 keeping open but waiting on Workload API. |
| **KEP #5729** (ResourceClaim Support for Workloads) | **Alpha v1.36** | Adds ResourceClaimTemplates to PodGroup — shared claim per group, not per replica. Template model only. Directly targets LWS/JobSet gap. |
| **KEP #5488** (Multi-host DRA UX) | **Closed** (triage bot, April 2026) | Was the closest unified solution. KEP-5729 now covers the ResourceClaim lifecycle portion. |
| **KEP #4671** (Gang Scheduling / Workload API) | Alpha v1.36, docs merged | **None** — explicitly excludes ResourceClaims. Scheduling only. |
| **KEP #6012** (CompositePodGroup) | Open, targeting Alpha v1.37 (WIP) | Hierarchical scheduling grouping — delegates ResourceClaim scoping to KEP-5729. |
| **[#6048](https://github.com/kubernetes/enhancements/issues/6048)** (Consume From Resource Claim) | Open, April 2026 | Reservation mechanism — first pod allocates pool, subsequent pods consume subsets. Directly addresses pool model. |
| **k/k #132192** (Workload-aware scheduling) | Open, active | Umbrella initiative. No concrete ResourceClaim design. |

### The Circular Dependency (Partially Resolved)

1. LWS #444 deferred → "waiting on Workload API for ResourceClaim support"
2. Workload API (KEP #4671) → explicitly excludes ResourceClaims
3. KEP #5488 (the bridge) → dead
4. **KEP #5729** (ResourceClaim Support for Workloads) → **Alpha v1.36, active.** Adds ResourceClaimTemplates to PodGroup. Solves shared-claim-per-group (template model). Does NOT solve per-replica unique claims or pool model.
5. KEP #6012 (CompositePodGroup) → hierarchical grouping, delegates ResourceClaim work to KEP-5729

**Update (April 2026):** KEP-5729 breaks the circular dependency for the template model. LWS/JobSet can use PodGroup-level ResourceClaimTemplates instead of implementing claim lifecycle themselves.

**Update (May 2026):** KEP-5729 author nojnhuh confirmed per-replica unique claims is "a different problem than what this KEP is addressing." johnbelamaric said he would think about it. Our use case (each pod gets its own GPU-NIC pair) is confirmed out of scope for KEP-5729. The pool model is partially addressed by [#6048](https://github.com/kubernetes/enhancements/issues/6048) (Consume From Resource Claim), which proposes a reservation-based approach.

## Connection to DRA Admission Webhook

Our webhook creates ResourceClaimTemplates at pod admission time — one claim per GPU-NIC pair, per pod. This is the **template model at pod granularity**. The missing pieces:

1. **Replica-level claim sharing**: LWS leader + workers in one replica should share claims. Currently each pod gets independent claims.
2. **Cross-replica coordination**: In a multi-replica LWS, the webhook has no visibility into what other replicas have claimed. The allocator's pending reservation TTL is a fragile workaround.
3. **Pool-model allocation**: For workloads with fixed hardware pools (IMEX channels, licensed accelerators), the webhook's template-based approach doesn't fit — claims should be drawn from a pre-existing pool, not created on demand.

KEP-5732 (Topology-Aware Scheduling) addresses device-level topology in the scheduler, but the workload-level gap (how LWS/JobSet distribute claims across replicas) sits above it. Both gaps need solving for LLM inference workloads.

## Design Concepts from johnbelamaric (May 2026)

Several approaches discussed for addressing the cross-driver topology and pool model gaps. These represent potential paths to v1.37 KEP proposals:

**Composite Device.** A pre-allocated set of devices (GPU + NIC + CPU) exposed as a single "meta-device." A proxy driver combines resources from multiple DRA drivers (gpu.nvidia.com, dra.net, CPU driver) and publishes valid topologies as a new device type in a new ResourceSlice. Eliminates cross-driver coordination at claim time.

**Provisionable Devices.** ResourceSlice defines a device _shape_ (e.g., "GPU-NIC pair on same PCIe root"), and devices are provisioned from underlying resources at request time. Combined with Consume-From, the provisioning process is complex claims resolved transparently.

**Consume From (#6048).** First pod allocates a reservation (pool of resources). Subsequent pods specify a reservation name in PodSpec and consume subsets. Could use claim-based approach or separate Reservation object (initially a list of claims). See [#6048](https://github.com/kubernetes/enhancements/issues/6048).

**Relationship to our gaps:**
- Composite Device → eliminates the cross-driver MatchAttribute complexity our webhook manages
- Provisionable Devices → could replace our ResourceSlice scanning + template generation
- Consume-From → directly addresses the pool model gap for LWS claim distribution
- johnbelamaric wants to get something into the v1.37 KEP pipeline

## Key Contacts

- **LWS**: @Edwinhr716 (maintainer, confirmed Workload API direction)
- **WG-Device-Management**: @johnbelamaric (noted each controller still needs API changes regardless of Workload integration)
- **JobSet**: @kannon92 (confirmed "waiting on Workload API")
- **CompositePodGroup**: watch [KEP PR #6017](https://github.com/kubernetes/enhancements/pull/6017)
