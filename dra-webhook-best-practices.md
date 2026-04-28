# DRA Webhook Best Practices: Orphan Cleanup and Batch Mutation

## Introduction

DRA mutating admission webhooks often need to create ResourceClaimTemplates or other cluster objects during pod admission, then patch the pod spec to reference them. Two operational challenges emerge at scale: orphaned resources that accumulate when pods fail or get deleted, and concurrent mutation races when multiple pods are admitted simultaneously. This document describes two battle-tested patterns that address these problems, drawn from production experience with GPU-NIC pairing workloads.

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

## Limitations and Future Work

Both patterns exist because the webhook is performing scheduler-level work at admission time. This is inherently limited: the webhook sees a point-in-time snapshot of cluster state and must make allocation decisions without the scheduler's full constraint solver.

KEP-5732 (Topology-Aware Scheduling) proposes moving topology-aware allocation logic into the scheduler natively. If adopted, the batch mutation pattern would become unnecessary -- the scheduler already serializes and optimizes placement decisions. The debounce-sort-serialize pattern would collapse into standard scheduler behavior.

Orphan cleanup, however, remains relevant for any webhook that creates DRA objects during admission. Even with improved scheduling, the fundamental problem persists: the pod does not exist when the webhook creates its supporting resources, so ownerReferences cannot be set at creation time.

No upstream KEP currently provides a purpose-built API for querying per-device availability with attribute-level filtering. KEP-5517 (Node Allocatable Resources) solves scheduler-internal double-counting of CPU/memory between DRA and pod.spec.resources but is not a device availability query API. Direct ResourceSlice scanning remains the only mechanism for building per-node, per-NUMA availability maps needed by preflight checkers.
