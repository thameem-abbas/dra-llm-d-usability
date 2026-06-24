# Use Case: GPU-NIC Topology-Aware Allocation via DRA Admission Webhook

> **Note:** This document was written against the webhook approach. The synthetic resource name `dra.llm-d.io/gpu-nic-pair` has since been replaced by `composite.dra.io/gpu-nic-pair` (composite DRA driver), which eliminates the need for a mutating webhook. The scheduler-level gaps described here remain accurate and are the motivation for the composite driver.

## Summary

We've built a Kubernetes mutating admission webhook that converts a synthetic resource request (`dra.llm-d.io/gpu-nic-pair: "N"`) into full DRA objects -- ResourceClaimTemplates with PCIe MatchAttribute constraints, NUMA co-location constraints, CEL-based rail selectors, and opaque driver parameters. The webhook exists because DRA's scheduler cannot express multi-device, cross-driver topology constraints. Every component -- NUMA-aware packing, PCIe root affinity, rail-aware NIC selection, pod affinity filtering, priority queue serialization, and pending reservation tracking -- is scheduler logic forced into an admission webhook. KEP-5732's Placement primitives would eliminate it.

## Problem Statement

Large-scale LLM inference (prefill/decode disaggregation) requires GPU-NIC pairs where the GPU and NIC share a PCIe root complex for RDMA. Beyond PCIe affinity, these deployments need:

- **NUMA locality**: Pairs within a single NUMA zone to avoid cross-socket memory traffic. An 8-GPU B200 node has two NUMA zones with 4 GPU-NIC pairs each; small requests must pack onto one zone, leaving the other zone available for large requests.
- **Network rail isolation**: Each NIC must land on a distinct parallel network fabric. Rail collisions cause bandwidth contention and break collective communication.
- **Cross-driver coordination**: GPUs come from `gpu.nvidia.com`, NICs from `dra.net`. These drivers have no awareness of each other.

DRA's current MatchAttribute handles pairwise attribute matching within a single claim, but the scheduler has no concept of NUMA-aware bin-packing, rail exclusivity, or cross-claim topology grouping.

## What We Built

The webhook intercepts pod CREATE requests via two endpoints (`/mutate` for labeled namespaces with full topology features, `/mutate-ext` for cluster-wide extended resource interception) and runs a full allocation pipeline before the scheduler sees the pod:

**Cluster-wide resource scanning.** The allocator lists ResourceSlices, extracts device attributes (IPv4 addresses, NUMA zones), and builds a per-node availability map of NIC slots keyed by `(node, railIndex, numaZone)`.

**NUMA-aware packing.** NIC slots are grouped by NUMA zone. Small requests pack onto the most-utilized zone; large requests target the most-available zone. This bin-packing logic (`selectSlots()`) is 50 lines of scheduling policy that DRA cannot express.

**PCIe root affinity.** Each GPU-NIC pair uses `MatchAttribute` on `resource.kubernetes.io/pcieRoot`. This is the one piece that maps cleanly to existing DRA primitives.

**Rail-aware network allocation.** NIC IPv4 prefixes map to rail indices. Selected rails are encoded as CEL selectors. Per-rail policy routing tables are injected as opaque driver parameters. The allocator prevents rail collisions across concurrent requests.

**Pod affinity/anti-affinity filtering.** Re-implements 200 lines of scheduler predicate logic (`nodeSelector`, `nodeAffinity`, `podAntiAffinity`, `podAffinity`), including tracking webhook-pinned but not-yet-scheduled pods.

**Priority queue with debounce batching.** Concurrent admissions are debounced, sorted by pair count descending, and processed serially so large requests get first pick.

**Pending reservation tracking.** An in-memory `"node:rail"` reservation map with 2-minute TTL bridges admission and scheduling. If the scheduler takes longer, reservations expire and double-booking can occur.

## Why This Belongs in the Scheduler

Each webhook mechanism duplicates a scheduler concern:

| Webhook mechanism | Scheduler concern it duplicates |
|---|---|
| ResourceSlice scanning + availability map | Scheduler's device accounting |
| NUMA zone bin-packing (`selectSlots`) | Topology-aware placement policy |
| Rail exclusivity enforcement | Multi-resource anti-collocation |
| Pod affinity/anti-affinity re-evaluation | Core scheduler predicate logic |
| Priority queue serialization | Scheduler's serial pod processing |
| 2-minute TTL pending reservations | Scheduler's atomic bind cycle |

The webhook cannot observe actual DRA allocation outcomes -- it guesses based on ResourceSlice state at admission time, which may be stale.

## Topology Dimensions We Need

### Node Topology Overview

```
                          Node (8 GPU-NIC pairs, e.g. B200)
    ┌─────────────────────────────────────────────────────────────┐
    │                                                             │
    │   NUMA Zone 0                      NUMA Zone 1              │
    │  ┌───────────────────────┐       ┌───────────────────────┐  │
    │  │                       │       │                       │  │
    │  │  PCIe Root 0x00       │       │  PCIe Root 0x80       │  │
    │  │  ┌─────┐  ┌─────┐    │       │  ┌─────┐  ┌─────┐    │  │
    │  │  │GPU-0│  │NIC-0│    │       │  │GPU-4│  │NIC-4│    │  │
    │  │  │     │──│Rail0│    │       │  │     │──│Rail0│    │  │
    │  │  └─────┘  └─────┘    │       │  └─────┘  └─────┘    │  │
    │  │  ┌─────┐  ┌─────┐    │       │  ┌─────┐  ┌─────┐    │  │
    │  │  │GPU-1│  │NIC-1│    │       │  │GPU-5│  │NIC-5│    │  │
    │  │  │     │──│Rail1│    │       │  │     │──│Rail1│    │  │
    │  │  └─────┘  └─────┘    │       │  └─────┘  └─────┘    │  │
    │  │  ┌─────┐  ┌─────┐    │       │  ┌─────┐  ┌─────┐    │  │
    │  │  │GPU-2│  │NIC-2│    │       │  │GPU-6│  │NIC-6│    │  │
    │  │  │     │──│Rail2│    │       │  │     │──│Rail2│    │  │
    │  │  └─────┘  └─────┘    │       │  └─────┘  └─────┘    │  │
    │  │  ┌─────┐  ┌─────┐    │       │  ┌─────┐  ┌─────┐    │  │
    │  │  │GPU-3│  │NIC-3│    │       │  │GPU-7│  │NIC-7│    │  │
    │  │  │     │──│Rail3│    │       │  │     │──│Rail3│    │  │
    │  │  └─────┘  └─────┘    │       │  └─────┘  └─────┘    │  │
    │  │                       │       │                       │  │
    │  └───────────────────────┘       └───────────────────────┘  │
    │                                                             │
    │  Network Rails:  Rail0 = 10.0.x.x    Rail1 = 10.1.x.x      │
    │                  Rail2 = 10.2.x.x    Rail3 = 10.3.x.x      │
    │                                                             │
    │  Drivers:  GPU → gpu.nvidia.com    NIC → dra.net            │
    │  Pairing:  MatchAttribute on resource.kubernetes.io/pcieRoot│
    └─────────────────────────────────────────────────────────────┘

    Constraint examples:
    • 2-pair request → pack into NUMA Zone 0 (bin-pack smallest-first)
    • 4-pair request → fill one NUMA zone entirely
    • 8-pair request → whole node
    • Rail isolation: each NIC in a request on a distinct rail subnet
```

Three topology dimensions govern GPU-NIC pair placement:

1. **PCIe root complex** -- pairs a GPU and NIC on the same PCIe tree. Expressible via MatchAttribute within a single claim; cross-claim PCIe affinity would require Placement-level constraints.

2. **NUMA zone** -- groups pairs for memory locality with bin-packing policy. Maps to KEP-5732's Placement concept with topology key `dra.net/numaNode` and a packing strategy.

3. **Network rail/fabric** -- ensures each NIC lands on a distinct parallel network. Maps to a PodGroup-level uniqueness constraint on a device attribute.

KEP-5732's Placement primitives could express all three, provided constraints span device classes (GPU from `gpu.nvidia.com`, NIC from `dra.net`).

**KEP-5732 alpha status (v1.36).** Kubernetes v1.36 shipped the alpha with `SchedulingConstraints` (containing `TopologyConstraints` and `DRAConstraints`) on the PodGroup API, plus new scheduler plugins: TopologyPlacementPlugin, PlacementBinPackingPlugin, and PlacementPodCountScorerPlugin. However, DRA-aware topology scheduling was explicitly deferred to beta -- the DRATestPlugin PR was closed without merge. The scheduler cannot yet combine topology placement with DRA device constraints, which is exactly the gap our webhook fills.

**Related: [#6006](https://github.com/kubernetes/enhancements/issues/6006) (Health-Aware Topology).** Recent KubeCon in-person discussion (April 2026) exploring performance-based device placement — accounting for cooling differences, network run lengths, and burn-in variance. Relevant for environments where not all GPU-NIC pairs on a node perform identically.

## Request

We are asking for native scheduler support for multi-device, cross-driver topology constraints so that admission webhooks do not have to reimplement scheduling. Specifically:

- **NUMA-aware bin-packing**: A placement policy that groups device requests by a topology attribute (e.g., `numaNode`) with a packing strategy (most-utilized-first for small requests, most-available-first for large).
- **Attribute uniqueness constraints**: A way to express "each device in this set must have a distinct value for attribute X" (rail isolation).
- **Cross-driver MatchAttribute**: MatchAttribute constraints that span device classes from different drivers within a Placement group.
- **Beyond KEP-5004's scope**: KEP-5004 (Extended Resources Bridge, beta in v1.36) allows `resources.requests` syntax to trigger DRA claims via `extendedResourceName` in DeviceClass. However, it does not support CEL selectors, matchAttribute constraints, or multi-device pairing -- reinforcing that complex topology allocation still requires native Placement primitives.

Our webhook has grown to ~6000+ lines of Go (from ~1500 originally) as we added InfiniBand support (auto-detection + explicit `ibRails` config), extended resource interception (`nvidia.com/gpu` → DRA claims via `/mutate-ext`), AKS kustomize overlays, and an offline dry-run simulator. The core topology allocation logic would reduce to a ResourceClaimTemplate with the right Placement annotations. The code is open source at [openshift-psap/dra-rail-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook) and targets multi-node GPU clusters (B200, H100) with 8 GPU-NIC pairs per node across Ethernet and InfiniBand fabrics.
