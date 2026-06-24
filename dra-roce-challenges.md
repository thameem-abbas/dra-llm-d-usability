# DRA + llm-d: Minimum Challenges for RoCE Support

## Overview

With the shared device plugin model (pre-DRA), InfiniBand worked without topology-aware allocation because all NICs on the node were visible to every pod. UCX automatically selected the PCIe-adjacent NIC for each GPU, and RDMA traffic flowed through optimal paths without explicit pairing. DRA changes this — only specifically allocated NICs are visible to a pod, so the allocation must pair each GPU with a PCIe-adjacent NIC or UCX cannot select the right one. DRA solves this for InfiniBand natively — Kubernetes/OpenShift's built-in `matchAttribute` constraint on `resource.kubernetes.io/pcieRoot` handles GPU-NIC pairing without any webhook or custom controller, just a DRA network driver (dranet or dra-sr-iov).

RoCE (RDMA over Converged Ethernet) requires everything InfiniBand does, plus per-rail network configuration that depends on which specific NIC gets allocated — a chicken-and-egg problem that exposes fundamental gaps in DRA's scheduling model. This document captures the minimum set of challenges that must be solved before claiming DRA support for llm-d inference workloads on RoCE fabrics.

Each challenge describes the problem, its impact on llm-d, the current solution ([composite-dra-driver](https://github.com/openshift-psap/composite-dra-driver) — now open source), and potential solutions — both upstream KEPs and alternative approaches.

[dra-rail-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook) was the previous solution but had bugs and race conditions from performing scheduling decisions outside the Kubernetes scheduler. The composite driver replaces it by operating as a native DRA driver — allocation stays in the scheduler. The upstream conversation on remaining gaps is tracked in [wg-device-management #54](https://github.com/kubernetes-sigs/wg-device-management/issues/54).
---

## Challenge 1: Per-Replica Unique Claims `Blocker`

**Problem**: llm-d uses LeaderWorkerSet (LWS) for prefill/decode disaggregation. Each replica needs distinct GPU-NIC pairs with topology-specific constraints (specific PCIe root, specific rail, specific NUMA zone). KEP-5729 (ResourceClaim Support for Workloads, Alpha v1.36) only covers shared claims per PodGroup — one claim shared across all replicas. Per-replica unique claims are confirmed out of scope by the KEP author (nojnhuh, May 2026).

**Impact**: Without per-replica claim support, there is no workload-level mechanism to distribute distinct GPU-NIC pairs across LWS replicas. Each pod's admission is independent, with no workload-level view of which replicas hold which devices.

One potential path is having LWS/workload layers accept pre-created concrete ResourceClaims and distribute them across replicas. This would also solve Challenge 2 (per-device driver configuration) since opaque parameters can be set correctly at claim creation time when the target device is already known. However, this approach risks embedding node/NUMA topology awareness into the workload layer — scheduling concerns that don't belong there.

**Previous Workaround (webhook)**: The webhook created ResourceClaimTemplates at individual pod admission time. Each pod got its own claims with topology-specific CEL selectors. No cross-replica coordination — pending reservation TTL (2 minutes) was the only mechanism to avoid allocation conflicts between replicas.

**Potential Solutions**:
- **[#6048](https://github.com/kubernetes/enhancements/issues/6048)** (Consume From Resource Claim): Proposes a reservation mechanism originating from per-node resource partitioning use cases (CPU/memory). The initial scope is cluster and node-level resource reservations — pods draw from a pre-allocated pool on a node. Does not currently address per-replica unique claim distribution with topology-specific constraints across workload replicas, but will likely eventually generalize to cover this. wojtek-t validated that reservations should bind to workload intent, not runtime representation, but acknowledged this is a harder generalization. Pushed to v1.38 with "use cases doc first" as next step.
- **Composite device (implemented)**: Pre-allocated set of devices (GPU + NIC) exposed as a single meta-device via the composite DRA driver. The driver publishes composite device shapes in ResourceSlices — the scheduler sees them as first-class allocatable units. The synthesizer refreshes the composite pool at 500ms frequency, so in practice LWS replicas receive distinct pairs (successive scheduling decisions see updated availability). However, there is no formal workload-layer guarantee — the scheduler has no mechanism to *reserve* a distinct composite device per replica before scheduling begins. This is the remaining gap.
- **[wg-device-management #54](https://github.com/kubernetes-sigs/wg-device-management/issues/54)**: The upstream conversation documenting the cross-driver device composition pattern and requesting community input on the remaining design questions (shadow claims formalization, virtual ResourceSlice concept, cross-pool exclusion). The per-replica distribution gap is explicitly called out.

---

## Challenge 2: Per-Device Driver Configuration `Blocker`

**Problem**: GPUs and NICs are managed by independent DRA drivers (`gpu.nvidia.com` and `dra.net`). DRA can pair devices across drivers using `matchAttribute` on `resource.kubernetes.io/pcieRoot` (this is how InfiniBand works natively), and DRA naturally allocates distinct NICs — so each NIC already lands on a different rail.

The core RoCE problem is that each rail requires distinct network configuration (routing table ID, gateway, MTU, policy routes) that depends on which specific NIC gets allocated. DRA's opaque driver parameters are set at claim creation time, before scheduling. There is no mechanism to parameterize driver config based on the actual device the scheduler picks. Without a composite driver, any external controller must know the allocation outcome in advance to generate the correct per-NIC configuration — but that outcome is determined by the scheduler after the claim is created.

**Impact**: Without per-device driver parameterization, RoCE NICs are allocated but not correctly configured. Each rail needs its own routing table and policy routes for RDMA traffic to flow correctly. Misconfigured network paths break GPU Direct RDMA entirely. NCCL collective operations (all_reduce, all_gather) degrade by up to 59.6% with incorrect topology (measured on 8-GPU H100 nodes).

**Previous Workaround (webhook)**: The admission webhook intercepted pod creation and pre-computed the allocation — scanning ResourceSlices to determine which NICs were available, generating CEL selectors to pin each NIC to a specific rail, and embedding per-rail opaque driver parameters in each claim.

This problem applies to both dranet and dra-sr-iov drivers. Without a composite driver, dra-sr-iov would also require an external controller unless all network configuration (routing tables, policy routes, MTU) is pre-configured on the host-level VFs and an out-of-tree CNI plugin copies those settings into the pod. Without that host-level pre-configuration, the same chicken-and-egg applies — the driver needs per-VF config that depends on which VF gets allocated.

**Potential Solutions**:
- **Composite DRA driver** (deployment-ready): Watches underlying GPU and NIC ResourceSlices, computes valid pairings via config-driven constraints (PCIe root, rail, NUMA), and publishes composite ResourceSlices with per-rail configuration already resolved. The scheduler allocates composite devices natively. On `PrepareResourceClaims`, the RailConfigResolver matches each allocated NIC to its rail, generates per-rail opaque driver parameters (routing table, gateway, MTU, policy routes), and embeds them in shadow ResourceClaims before delegating to underlying drivers via gRPC. Deployed as a DaemonSet via Helm chart. **Solves this challenge.**

Note: [KEP-6080](https://github.com/kubernetes/enhancements/issues/6080) (Derived Attributes, provisional, Alpha v1.37) addresses cross-driver attribute naming flexibility but does not solve the per-device configuration problem.

---

## Challenge 3: Device-Level Bin-Packing `Required`

**Problem**: DRA's scheduler has no bin-packing policy for device allocation. When a small request (e.g., 2 GPU-NIC pairs on an 8-GPU node) arrives, the scheduler may scatter the allocation across the node instead of packing it tightly, fragmenting the node so that subsequent larger requests cannot be served. The core issue is node-level bin-packing — keeping allocations compact so remaining capacity stays contiguous and usable.

NUMA-aware packing is the next level: an 8-GPU node (e.g., B200) typically has two NUMA zones with 4 GPU-NIC pairs each. Ideally, a 2-pair request packs entirely into one NUMA zone, leaving the other zone fully available for a 4-pair request. While DRA's `matchAttribute` can theoretically constrain devices to the same NUMA zone, doing so in practice leads to combinatorial explosion — every possible NUMA grouping generates constraint combinations that scale poorly.

**Impact**: Without bin-packing, small requests scatter across nodes instead of packing tightly. For example, with 2 nodes (8 pairs each) and several 1-pair requests: if those land across both nodes, a subsequent 8-pair request (which needs an entire node) cannot be served — even though total free capacity exists across the cluster. Within a node, cross-NUMA allocation is suboptimal but still functional. The real damage is at node level, where fragmentation blocks full-node allocations common in large TP (tensor parallel) workloads.

**Current Workaround**: Pod anti-affinity between decode and prefill pods keeps them on separate nodes, preventing small prefill pods from fragmenting nodes needed for large decode allocations. With the composite driver, anti-affinity works correctly — the scheduler evaluates anti-affinity and allocates from composite ResourceSlices in a single pass, no conflicting admission-time decisions. This is coarse-grained — it doesn't handle fragmentation between workloads of the same type or mixed sizes, and NUMA-aware bin-packing within a node is not addressed. We are not actively solving this challenge beyond anti-affinity now that we are back to being able to use it.

**Potential Solutions**:
- **Kueue Topology Aware Scheduling / Gang Scheduling**: Kueue already has topology-aware scheduling and gang scheduling capabilities that could address device-level bin-packing. Should work with the composite driver without issues since the composite driver operates similar to the standard device plugin interfaces. Needs validation.
- **[KEP-5732](https://github.com/kubernetes/enhancements/issues/5732)** (Topology-Aware Scheduling): Adds multi-pod placement policies to the scheduler. Alpha in v1.36, but DRA integration explicitly deferred to beta (v1.37+). kannon92 (KEP author) invited us to present at wg-workload-aware-scheduling Tuesday/Thursday (next on Jun 18) calls.

---

## Challenge 4: Admission-Scheduling Race `Required`

**Problem**: When device allocation decisions are made outside the scheduler (e.g., in an admission webhook), concurrent pod creates race for the same NUMA zone or devices. The external controller runs before the scheduler — it cannot know which devices the scheduler will actually allocate, only which appear available in ResourceSlices at that moment. This creates a race between what the controller reserves and what the scheduler binds. This problem is inherent to any approach that makes allocation decisions outside the scheduler.

**Impact**: Without serialization, two pods may both target the same NUMA zone or devices. The controller creates correct ResourceClaimTemplates for each pod in isolation, but the combined allocation conflicts. The scheduler resolves one pod successfully; the other fails and must be retried. Worse, failing workers in a LeaderWorkerSet can cause cascading resource exhaustion — failed pods hold claims that block replacement pods from scheduling, and the situation is irrecoverable without deleting the entire workload and recreating it.

**Previous Workaround (webhook)**: The webhook implemented a priority queue with 3-second debounce (batch pods, process largest-first), sorted by pair count descending for better NUMA packing, and maintained in-memory pending reservations with 2-minute TTL. These reservations bridged the gap between admission and scheduling. Reservations were persisted — only lost if the webhook crashed mid-allocation.

**Potential Solutions**:
- **Composite DRA driver** (deployment-ready): Eliminates this problem entirely. The scheduler handles allocation natively from composite ResourceSlices — no admission-time allocation decisions, no reservation tracking, no races. **Solves this challenge.**

---

## Challenge 5: Orphan Resource Cleanup `Desired`

**Problem**: The previous webhook approach created ResourceClaimTemplates before the pod existed — `ownerReferences` could not point to a not-yet-persisted pod. If pod creation failed after template creation, templates became orphaned and accumulated in etcd.

**Impact**: Orphaned ResourceClaimTemplates consumed etcd storage and confused operators inspecting cluster state.

**Resolved by composite driver**: Shadow claims use `ownerReferences` pointing to the composite claim. Kubernetes garbage collector handles cleanup automatically. A 5-minute orphan reconciler catches any stragglers. This problem no longer exists with the composite driver approach. This could potentially be improved further.

---

## Summary

### Fundamental Gaps — problems that exist regardless of approach

| # | Challenge | Priority | Native DRA | Composite Driver | Upstream KEP | Status |
|---|-----------|----------|:---:|:---:|:---:|--------|
| 1 | Per-replica unique claims | Blocker | No | — | #6048 (future) | Unsolved — #6048 may eventually generalize to cover this |
| 2 | Per-device driver configuration | Blocker | No | **Solved** | — | Composite driver deployment-ready |
| 3 | Device-level bin-packing | Required | No | Partial (topology only) | KEP-5732, Kueue TAS | Unsolved — KEP-5732 alpha v1.36 but DRA device support deferred to v1.37+ beta |

### Webhook-Induced — problems that existed because the previous solution used an admission webhook

These are eliminated by moving device allocation into the scheduler via the composite DRA driver.

| # | Challenge | Priority | Composite Driver | Notes |
|---|-----------|----------|:---:|-------|
| 4 | Admission-scheduling race | Required | **Solved** | Composite driver eliminates — scheduler allocates natively |
| 5 | Orphan resource cleanup | Desired | **Solved** | OwnerReferences cascade GC + 5-min orphan reconciler |

**Native DRA** column shows what Kubernetes/OpenShift can do today without any webhook or custom controller. PCIe root matching (`matchAttribute` on `resource.kubernetes.io/pcieRoot`) and rail diversity (distinct NIC allocation) work natively — this is how InfiniBand support is achieved. The remaining gaps are what make RoCE support require additional machinery.

Challenge 1 is the primary focus — the workload-level gap that prevents LWS replicas from getting formally guaranteed distinct GPU-NIC pairs. In practice the composite driver's 500ms synthesizer cycles mean distinct pairs are assigned, but there is no formal scheduler-level guarantee. [wg-device-management #54](https://github.com/kubernetes-sigs/wg-device-management/issues/54) is the upstream conversation tracking this gap. Challenge 2 is the core RoCE-specific gap where driver config can't be parameterized per device — solved by the composite driver. Challenge 3 affects multi-tenant GPU capacity utilization.

**Current solution path:** The composite DRA driver paired with an ExtendedResource annotation (e.g., `nvidia.com/gpu: N`) reproduces most of the previous webhook experience while being fully transparent to the scheduler. The composite driver publishes pre-paired composite devices in ResourceSlices. The ExtendedResource annotation lets users request them using familiar resource syntax without knowing DRA internals. The scheduler allocates natively from composite ResourceSlices, eliminating challenges 2, 4, and 5. However, NUMA-aware bin-packing (challenge 3) is lost compared to the previous webhook — the scheduler has no device-level packing policy, so this gap remains.

---

## Composite DRA Driver

The [composite-dra-driver](https://github.com/openshift-psap/composite-dra-driver) is the current solution for GPU-NIC pairing on RoCE fabrics. It replaces the previous admission webhook approach by operating as a native DRA driver — the scheduler sees composite devices as first-class allocatable resources.

### What It Does

The composite driver sits between the underlying device drivers (NVIDIA GPU driver, dranet/dra-sr-iov) and the Kubernetes scheduler. It watches ResourceSlices published by each underlying driver, pairs devices using config-driven constraints (PCIe root matching, CEL filters), and publishes the paired devices as new composite ResourceSlices. The scheduler allocates from these composite slices natively — no out-of-band allocation decisions, no node pinning, no races.

When kubelet calls the composite driver to prepare an allocated device, the driver:
1. Looks up which underlying GPU and NIC make up the composite pair
2. Resolves per-rail network configuration based on the actual NIC allocated (routing table, gateway, MTU, policy routes)
3. Creates shadow ResourceClaims for each underlying device — pre-filled with allocation results and opaque driver parameters
4. Delegates device setup to the underlying drivers via gRPC (nvidia for GPU visibility, dranet for NIC namespace + routing)

Shadow claims have `ownerReferences` to the composite claim, so Kubernetes garbage collection handles cleanup automatically.

### User Experience

Users request GPU-NIC pairs the same way they request GPUs today:

```yaml
resources:
  requests:
    composite.dra.io/gpu-nic-pair: "4"
```

On Kubernetes 1.36+ with the `DRAExtendedResource` feature gate, this maps directly to the composite DRA driver via a DeviceClass — no webhook needed, the scheduler handles it natively. On older clusters, a temporary webhook (included in the Helm chart) intercepts the synthetic resource, creates the appropriate ResourceClaimTemplate, and patches the pod. 

### Architecture

- **Deployment**: DaemonSet via Helm chart (one pod per node, `system-node-critical` priority)
- **Composition**: Config-driven — any combination of DRA drivers can be composed, not hardcoded to GPU+NIC
- **Pairing**: Generic `matchAttribute` algorithm with optional CEL filters
- **State**: BoltDB persistence at `/var/lib/composite-dra/state.db` for crash recovery
- **HA**: Per-node independence — no cross-node coordination, no split-brain
- **Synthesis frequency**: ResourceSlice changes trigger re-pairing with 500ms debounce

### What It Solves

| Challenge | Status |
|-----------|--------|
| Per-device driver configuration (Challenge 2) | **Solved** — RailConfigResolver generates per-rail config at prepare time based on actual allocated device |
| Admission-scheduling race (Challenge 4) | **Solved** — scheduler handles allocation natively, no races |
| Orphan resource cleanup (Challenge 5) | **Solved** — ownerReferences cascade GC + orphan reconciler |

### What It Does Not Solve

| Challenge | Why |
|-----------|-----|
| Per-replica unique claims (Challenge 1) | Each pod's allocation is independent — no workload-level distribution mechanism. The synthesizer updates available devices (500ms frequency) so replicas get distinct pairs in practice, but there is no formal guarantee at the workload layer. |
| Device-level bin-packing (Challenge 3) | The composite driver surfaces NUMA zone as a device attribute but has no packing policy. The scheduler can match devices by topology but doesn't optimize for compaction. Anti-affinity between decode and prefill pods is the current workaround. |
| Cross-pool device exclusion ([#28](https://github.com/openshift-psap/composite-dra-driver/issues/28)) | The same physical device can appear in multiple composition pools (e.g., a GPU in both a `gpu-nic-pair` pool and a `gpu-cpu-nic-triple` pool). DRA has no cross-pool mutual exclusion — the scheduler can double-allocate the same physical device across pools. Not a blocker for single-composition deployments (e.g., gpu-nic-pair only), but an open design question raised in [wg-device-management #54](https://github.com/kubernetes-sigs/wg-device-management/issues/54). Two approaches under consideration: (a) pairer-side partitioning — statically assign each device to one pool, no race window but wastes devices; (b) continuous recomputation — re-watch underlying slices and remove composed devices whose members are already allocated, small race window but efficient. Neither implemented yet. |
| NUMA affinity is hard constraint | `matchAttribute` on NUMA zone is an exact match — all devices must be on the same zone. There is no best-effort NUMA preference; requests that can't be satisfied within a single zone fail rather than falling back to cross-NUMA. |
| No consumable capacity | Discrete device model only. CPU/memory grouped mode not supported — you can't compose a "GPU + 4 vCPUs + 16GB RAM" composite device. |

---

## Appendix: Composite DRA Driver Flow

```
                         CONTINUOUS SYNTHESIS (per node)
                         ==============================

    gpu.nvidia.com                        dra.net
    ResourceSlices                        ResourceSlices
    ┌──────────────┐                      ┌──────────────┐
    │ GPU-0 (0x00) │                      │ NIC-0 (0x00) │
    │ GPU-1 (0x00) │                      │ NIC-1 (0x00) │
    │ GPU-2 (0x00) │                      │ NIC-2 (0x00) │
    │ GPU-3 (0x00) │                      │ NIC-3 (0x00) │
    │ GPU-4 (0x80) │                      │ NIC-4 (0x80) │
    │ GPU-5 (0x80) │                      │ NIC-5 (0x80) │
    │ GPU-6 (0x80) │                      │ NIC-6 (0x80) │
    │ GPU-7 (0x80) │                      │ NIC-7 (0x80) │
    └──────┬───────┘                      └──────┬───────┘
           │          (pcieRoot values)          │
           └──────────────┬──────────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Composite Driver     │
              │  Watcher → Pairer →   │
              │  Publisher            │
              │                       │
              │  Pairs by pcieRoot:   │
              │  GPU-0+NIC-0 (0x00)   │
              │  GPU-1+NIC-1 (0x00)   │
              │  ...                  │
              │  GPU-7+NIC-7 (0x80)   │
              └───────────┬───────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │  Composite            │
              │  ResourceSlices       │
              │                       │
              │  8 GPU-NIC pairs      │
              │  (native DRA objects) │
              └───────────────────────┘


                         POD LIFECYCLE
                         ============

    User creates pod:
    ┌─────────────────────────────────────┐
    │ resources:                          │
    │   requests:                         │
    │     composite.dra.io/gpu-nic-pair: "4" │
    └──────────────────┬──────────────────┘
                       │
                       ▼
    ┌──────────────────────────────────────┐
    │ Scheduler                            │
    │                                      │
    │ Sees 8 composite devices available   │
    │ Allocates 4 pairs to this pod        │
    │ (native DRA allocation — no webhook) │
    └──────────────────┬───────────────────┘
                       │
                       ▼
    ┌──────────────────────────────────────┐
    │ Kubelet calls composite driver       │
    │ PrepareResourceClaims                │
    │                                      │
    │ For each allocated pair:             │
    │                                      │
    │  1. Look up members                  │
    │     gpu-0--nic-0 → GPU-0 + NIC-0    │
    │                                      │
    │  2. Resolve per-rail config          │
    │     NIC-0 IP 10.0.x.x → Rail 0      │
    │     → routing table, gateway, MTU    │
    │                                      │
    │  3. Create shadow claims             │
    │     ┌─────────────────────────────┐  │
    │     │ Shadow GPU claim            │  │
    │     │ (pre-filled allocation)     │  │
    │     └─────────────────────────────┘  │
    │     ┌─────────────────────────────┐  │
    │     │ Shadow NIC claim            │  │
    │     │ (pre-filled allocation +    │  │
    │     │  opaque rail config)        │  │
    │     └─────────────────────────────┘  │
    │                                      │
    │  4. gRPC → nvidia driver (GPU setup) │
    │     gRPC → dranet driver (NIC setup) │
    │                                      │
    │  Result: GPU visible + NIC in pod    │
    │  netns with correct routing          │
    └──────────────────────────────────────┘
```
