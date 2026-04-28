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

**No LWS issue and no upstream KEP covers this.** All existing work follows the template model.

| | Template Model | Pool Model |
|---|---|---|
| Claim lifecycle | Created per replica, 1:1 | Pre-existing, reused |
| Scaling up | New claim from template | Claim assigned from pool |
| Scaling down | Claim deleted with replica | Claim returned to pool |
| Resource exhaustion | Unbounded (DRA scheduler decides) | Bounded by pool size |
| Use case | Per-replica GPU/NIC pairs | Shared hardware, licensed resources, IMEX channels |

## Upstream Landscape

| KEP / Issue | Status | ResourceClaim Relevance |
|-------------|--------|------------------------|
| **LWS #444** (ResourceClaimTemplate) | Open, unassigned, stalled | Direct — template model for LWS. Deferred to Workload API. |
| **JobSet #762** (Job-level ResourceClaimTemplate) | Open, rotting | Same gap for JobSet. Also deferred to Workload API. |
| **KEP #5729** (ResourceClaim Support for Workloads) | **Alpha v1.36** | Adds ResourceClaimTemplates to PodGroup — shared claim per group, not per replica. Template model only. Directly targets LWS/JobSet gap. |
| **KEP #5488** (Multi-host DRA UX) | **Closed** (triage bot, April 2026) | Was the closest unified solution. KEP-5729 now covers the ResourceClaim lifecycle portion. |
| **KEP #4671** (Gang Scheduling / Workload API) | Alpha v1.35-1.36 | **None** — explicitly excludes ResourceClaims. Scheduling only. |
| **KEP #6012** (CompositePodGroup) | Open, targeting Alpha v1.37 (WIP) | Hierarchical scheduling grouping — delegates ResourceClaim scoping to KEP-5729. |
| **k/k #132192** (Workload-aware scheduling) | Open, active | Umbrella initiative. No concrete ResourceClaim design. |

### The Circular Dependency (Partially Resolved)

1. LWS #444 deferred → "waiting on Workload API for ResourceClaim support"
2. Workload API (KEP #4671) → explicitly excludes ResourceClaims
3. KEP #5488 (the bridge) → dead
4. **KEP #5729** (ResourceClaim Support for Workloads) → **Alpha v1.36, active.** Adds ResourceClaimTemplates to PodGroup. Solves shared-claim-per-group (template model). Does NOT solve per-replica unique claims or pool model.
5. KEP #6012 (CompositePodGroup) → hierarchical grouping, delegates ResourceClaim work to KEP-5729

**Update**: KEP-5729 breaks the circular dependency for the template model. LWS/JobSet can use PodGroup-level ResourceClaimTemplates instead of implementing claim lifecycle themselves. However, our use case needs per-replica unique claims (each pod gets its own GPU-NIC pair), not a shared claim across the group. The pool model remains unaddressed.

## Connection to DRA Admission Webhook

Our webhook creates ResourceClaimTemplates at pod admission time — one claim per GPU-NIC pair, per pod. This is the **template model at pod granularity**. The missing pieces:

1. **Replica-level claim sharing**: LWS leader + workers in one replica should share claims. Currently each pod gets independent claims.
2. **Cross-replica coordination**: In a multi-replica LWS, the webhook has no visibility into what other replicas have claimed. The allocator's pending reservation TTL is a fragile workaround.
3. **Pool-model allocation**: For workloads with fixed hardware pools (IMEX channels, licensed accelerators), the webhook's template-based approach doesn't fit — claims should be drawn from a pre-existing pool, not created on demand.

KEP-5732 (Topology-Aware Scheduling) addresses device-level topology in the scheduler, but the workload-level gap (how LWS/JobSet distribute claims across replicas) sits above it. Both gaps need solving for LLM inference workloads.

## Key Contacts

- **LWS**: @Edwinhr716 (maintainer, confirmed Workload API direction)
- **WG-Device-Management**: @johnbelamaric (noted each controller still needs API changes regardless of Workload integration)
- **JobSet**: @kannon92 (confirmed "waiting on Workload API")
- **CompositePodGroup**: watch [KEP PR #6017](https://github.com/kubernetes/enhancements/pull/6017)
