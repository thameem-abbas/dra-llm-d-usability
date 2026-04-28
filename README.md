# DRA Usability: Upstream Engagement for llm-d GPU-NIC Pairing

Analysis of gaps between Kubernetes DRA (Dynamic Resource Allocation) and production GPU-NIC pairing requirements for LLM inference workloads. Tracks upstream KEP engagement and contributions.

## Context

The [dra-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook) converts a synthetic resource request (`dra.llm-d.io/gpu-nic-pair: "N"`) into full DRA objects — ResourceClaimTemplates with CEL selectors, PCIe MatchAttribute constraints, NUMA co-location, and rail-aware NIC allocation. Much of this is scheduler-level work done in an admission webhook because upstream DRA doesn't support topology-aware multi-device allocation yet.

This repo documents the gaps, proposes upstream contributions, and tracks progress.

## Gap Analysis

| Document | Description |
|----------|-------------|
| [kep-5732-use-case.md](kep-5732-use-case.md) | Production use case for KEP-5732 (Topology-Aware Scheduling) — our webhook IS the motivating example |
| [kep-5491-feedback.md](kep-5491-feedback.md) | Consumer feedback on KEP-5491 (List-Typed Attributes) — scalar-to-list migration concerns for pcieRoot/numaNode |
| [cel-selector-patterns.md](cel-selector-patterns.md) | Three reusable CEL selector patterns: rail-specific RDMA, explicit device pinning, cross-driver topology alignment |
| [dra-webhook-best-practices.md](dra-webhook-best-practices.md) | Two operational patterns: orphan cleanup reconciler and priority queue batch mutation |

## Upstream Gaps Summary

Features this webhook implements that upstream DRA doesn't support (yet):

1. **Rail-aware network topology allocation** — no concept of network rails or multi-subnet RDMA fabrics in DRA
2. **Batch mutation with priority ordering** — no batch-aware admission for DRA resources
3. **Cluster-level pending reservation tracking** — no coordination mechanism for admission webhooks
4. **Orphan resource cleanup** — no built-in GC for webhook-created DRA objects without ownerReferences
5. **Explicit device pairing mode** — no "admin declares exact device sets" pattern in DRA

## KEP Alignment (as of Kubernetes v1.36, April 2026)

| KEP | Status | Relationship |
|-----|--------|-------------|
| KEP-4381 (DRA Core) | GA v1.34 | Foundation — webhook uses v1 API with `Exactly` sub-field |
| KEP-5732 (Topology-Aware Scheduling) | Alpha v1.36 | DRA integration deferred to beta — our webhook fills this gap |
| KEP-5491 (List Type Attributes) | Alpha v1.36 | Would improve our pcieRoot/numaNode MatchAttribute constraints |
| KEP-4816 (Prioritized Alternatives) | GA v1.36 | Could replace custom rail fallback logic |
| KEP-5055 (Device Taints) | Beta v1.36 | Could simplify preflight availability checking |
| KEP-5007 (Binding Conditions) | Beta v1.36 | Could replace preflight "graceful degradation" pattern |
| KEP-5517 (ResourcePoolStatusRequest) | Alpha v1.36 | Could replace direct ResourceSlice scanning in preflight |
| KEP-5004 (Extended Resources Bridge) | Beta v1.36 | Simple DRA path — does NOT cover our multi-device topology use case |

## Action Items

See [action-items/](action-items/) for tracked upstream engagement tasks.
