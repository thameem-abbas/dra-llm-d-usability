# DRA Webhook Best Practices: Operational Patterns

## Introduction

DRA mutating admission webhooks often need to create ResourceClaimTemplates or other cluster objects during pod admission, then patch the pod spec to reference them. Several operational challenges emerge at scale: orphaned resources, concurrent mutation races, extended resource migration, and configuration validation. This document describes four patterns that address these problems, drawn from building a GPU-NIC pairing webhook that now supports Ethernet and InfiniBand fabrics, extended resource interception, and offline dry-run simulation.

## Webhook Admission Flow

```
    Pod CREATE request (composite.dra.io/gpu-nic-pair: "N")
    # previously: dra.llm-d.io/gpu-nic-pair (webhook interception via synthetic resource)
         │
         ▼
    ┌──────────────┐
    │  API Server   │
    │  (admission)  │
    └──────┬───────┘
           │ MutatingWebhookConfiguration
           ▼
    ┌──────────────────────────────────────────────┐
    │  DRA Admission Webhook                        │
    │                                               │
    │  1. Debounce batch (3s quiet window)          │
    │  2. Sort by pair count (largest first)        │
    │  3. For each pod in batch:                    │
    │     a. Scan ResourceSlices (all nodes)        │
    │     b. Build per-node availability map        │
    │        keyed by (node, rail, numaZone)        │
    │     c. Filter by pod affinity/anti-affinity   │
    │     d. Select NUMA zone (bin-pack policy)     │
    │     e. Select rails (no collisions)           │
    │     f. Create ResourceClaimTemplates with     │
    │        CEL selectors + MatchAttribute on      │
    │        pcieRoot + opaque driver params        │
    │     g. Record pending reservations (2m TTL)   │
    │  4. Patch pod spec with claim references      │
    └──────────────────┬───────────────────────────┘
                       │
                       ▼
    ┌──────────────────────────┐
    │  Scheduler                │
    │  (sees patched pod with   │
    │   DRA claims, binds to    │
    │   devices on nodes)       │
    └──────────────────────────┘
```

## Pattern 1: Orphan Cleanup Reconciler

### Problem

When a webhook creates a ResourceClaimTemplate during admission, the pod does not exist yet. The template cannot carry an ownerReference to the pod, so Kubernetes garbage collection will not clean it up. Templates become orphaned when a pod is deleted before claims bind, when pod creation fails after template creation, or when a stale template from a prior deployment collides with a new one.

Left unchecked, orphaned templates accumulate indefinitely, consuming etcd storage and confusing operators inspecting cluster state.

### The Pattern: Detect, Annotate, Grace Period, Reap, Prune

**Step 1 -- Detect.** A reconciler loop (we use a 5-minute interval) lists all templates carrying a managed-by label (`app.kubernetes.io/managed-by: dra-gpu-nic-webhook`). It cross-references these against templates actually referenced by active pods. Any unreferenced template is an orphan candidate.

Label-based detection is deliberate. It scopes the reconciler to only the resources it owns and avoids expensive full-namespace scans.

**Step 2 -- Annotate.** On first detection, the reconciler adds `dra.llm-d.io/orphaned-at: <RFC3339 timestamp>` to the template. This makes orphan status observable: operators can query for annotated resources without the reconciler needing to auto-delete anything.

**Step 3 -- Grace Period.** A configurable delay (default 10 minutes) must elapse between annotation and deletion. This prevents a race where a template appears orphaned during the window between template creation and pod scheduling. Without a grace period, a slow scheduler could cause the reconciler to delete a template that a pod is about to use.

**Step 4 -- Reap.** If `autoReap` is enabled and the grace period has elapsed, the reconciler deletes the resource. Every deletion is recorded in a persistent state file (`/data/reconciler-state.json`) that survives restarts. This provides an audit trail of what was deleted and when.

**Step 5 -- Prune.** Old reap records are cleaned up after a configurable retention period (default 7 days) to prevent unbounded state file growth.

The reconciler also handles orphaned ResourceClaims by checking whether the owning pod (from ownerReferences) still exists. It uses the `resource.kubernetes.io/pod-claim-name` annotation -- set by Kubernetes when creating a claim from a template -- to identify webhook-originated claims.

**Why persistent state?** Atomic JSON writes ensure the reconciler remembers detection timestamps across restarts. Without persistence, a restart resets the grace period clock, and a rapidly restarting reconciler would never delete anything.

## Pattern 2: Priority Queue for Batch Mutation

### Problem

When a Deployment creates N replicas simultaneously, N admission requests hit the webhook concurrently. Without coordination, multiple pods grab the same network rail (rail collision), NUMA packing decisions lack a global view (leading to imbalance), and ResourceClaimTemplate creation races produce conflicts.

### The Pattern: Debounce, Sort, Serialize

**Step 1 -- Debounce.** Incoming admission requests enter a queue. A 3-second timer resets on each new request. After 3 seconds of quiet, the batch is processed. This collects requests from a single rollout into one batch without adding latency to isolated pod creations.

**Step 2 -- Sort by resource demand.** The batch is sorted by GPU-NIC pair count, descending. Larger requests (8-pair) get first pick of rails and NUMA zones. Smaller requests (1-pair) fill remaining capacity. This produces tighter packing than FIFO ordering.

**Step 3 -- Serialize mutations.** Each `Mutate()` call within the batch sees the templates and allocations from prior requests. This prevents rail collisions, NUMA zone oversubscription, and template name conflicts. The serialization is batch-internal only; requests outside the batch are not blocked.

**Step 4 -- Blocking response.** Each admission request blocks on a result channel until its mutation completes. Since Kubernetes admission has a default 10-second timeout, the entire batch must finish within that window. The debounce window (3s) plus serialized mutations must fit comfortably under this ceiling.

**Bridging admission and scheduling.** The allocator maintains pending reservations as a `map[string]time.Time` keyed by `"<node>:<rail>"` with a 2-minute TTL. This prevents double-allocation during the gap between admission-time decisions and actual scheduler binding. Expired reservations are cleaned up on the next allocation pass.

## When You Need These

**Orphan cleanup signals:** Growing ResourceClaimTemplate count in webhook-managed namespaces. `kubectl get resourceclaimtemplates -l app.kubernetes.io/managed-by=<your-webhook>` returns resources with no referencing pod. Stale claims blocking device capacity.

**Batch mutation signals:** Rail collisions or NUMA imbalance appearing only during concurrent rollouts (not single-pod creation). Template creation conflicts under load. Non-deterministic device placement across replicas of the same Deployment.

## Pattern 3: Extended Resource Interception

The webhook now supports a `/mutate-ext` endpoint that intercepts standard extended resources (e.g., `nvidia.com/gpu`) and converts them to DRA ResourceClaims cluster-wide. This bridges the gap for clusters migrating from device plugins to DRA before Kubernetes 1.35+ `DRAExtendedResource` feature gate.

**Key differences from `/mutate`:**
- **Scope**: `/mutate-ext` operates on all non-system namespaces (no label required). `/mutate` requires `dra.llm-d.io/webhook-enabled: "true"`.
- **Failure policy**: `/mutate-ext` uses `Ignore` (pods pass through if webhook is down). `/mutate` uses `Fail`.
- **Mutual exclusivity**: A pod cannot request both `composite.dra.io/gpu-nic-pair` (previously `dra.llm-d.io/gpu-nic-pair`) and an intercepted resource.
- **Per-container binding**: Each container gets claim references only for GPUs it requested.

## Pattern 4: Offline Dry-Run Simulation

The webhook includes a `dryrun` CLI tool for validating allocation config without touching the live cluster:

1. **Capture**: `dryrun capture` dumps ResourceSlices + Nodes from a live cluster to JSON
2. **Simulate**: `dryrun simulate` runs the webhook mutation pipeline against captured state

This is especially valuable for InfiniBand deployments where `ibRails` PCIe address mappings must match actual hardware topology.

## Operational Requirements

**CRI-O NRI Plugin Timeout.** Multi-VF RDMA NIC operations require increasing `nri_plugin_request_timeout` to 60s on every GPU worker node. Without this, the NRI plugin crashes during DRA device setup. This is mandatory for production.

**InfiniBand Configuration.** InfiniBand clusters require explicit `ibRails` config mapping GPU PCIe addresses to NIC PCIe addresses (e.g., from Azure's `ndv5-topo.xml`). Auto-detection via `transportMode: auto` handles fabric type but not PCIe topology.

**Kustomize Overlays.** The webhook ships with a base deploy + overlay architecture. AKS ND96isr_H100_v5 overlay is provided (`deploy/overlays/aks-ndv5/`). Custom overlays needed for other hardware.

## Limitations and Future Work

Both patterns exist because the webhook is performing scheduler-level work at admission time. This is inherently limited: the webhook sees a point-in-time snapshot of cluster state and must make allocation decisions without the scheduler's full constraint solver.

KEP-5732 (Topology-Aware Scheduling) proposes moving topology-aware allocation logic into the scheduler natively. If adopted, the batch mutation pattern would become unnecessary -- the scheduler already serializes and optimizes placement decisions. The debounce-sort-serialize pattern would collapse into standard scheduler behavior.

Orphan cleanup, however, remains relevant for any webhook that creates DRA objects during admission. Even with improved scheduling, the fundamental problem persists: the pod does not exist when the webhook creates its supporting resources, so ownerReferences cannot be set at creation time.

No upstream KEP currently provides a purpose-built API for querying per-device availability with attribute-level filtering. KEP-5517 (Node Allocatable Resources) solves scheduler-internal double-counting of CPU/memory between DRA and pod.spec.resources but is not a device availability query API. Direct ResourceSlice scanning remains the only mechanism for building per-node, per-NUMA availability maps needed by preflight checkers.

**Explicit pairing mode** (`pairingMode: explicit`) is experimental with 4 tracked issues (#9-#12 in the webhook repo). Use `pairingMode: auto` for production.
