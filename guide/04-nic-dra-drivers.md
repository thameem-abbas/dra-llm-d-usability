# NIC DRA Drivers and GPU-NIC Topology

This is the fourth and final document in a reading guide about Kubernetes Dynamic Resource Allocation (DRA). If you have not already read the first three documents, start with `01-what-is-dra.md` (concepts), `02-how-dra-works.md` (object model and lifecycle), and `03-gpu-dra-drivers.md` (GPU drivers). This document covers NIC DRA drivers, the GPU-NIC topology pairing problem, and a real-world webhook use case that illustrates where the DRA scheduler currently hits its limits.

## Why NICs Need DRA

High-performance networking in Kubernetes depends on selecting the right network interface card (NIC) for the workload. Distributed training and inference jobs often require RDMA (Remote Direct Memory Access) support, specific IP subnets for network rail isolation, or NICs on the same PCIe root complex as their paired GPUs. These requirements cannot be expressed with the traditional device plugin model, which only reports a count: "this node has 8 NICs."

Device plugins provide no filtering mechanism. A pod requests `nvidia.com/rdma: 1` and gets any NIC from the pool. There is no way to request a NIC with RDMA support on a specific subnet, or a NIC sharing a PCIe root complex with a particular GPU. DRA solves this by enabling attribute-based NIC selection. NICs become DRA devices with attributes like RDMA capability, interface name, IPv4 address, link speed, PCIe address, and NUMA zone. Workloads use CEL selectors to match the right NIC, and constraints ensure GPU-NIC pairs share the same topology.

## DRANET (kubernetes-sigs/dranet)

### Overview

DRANET is the primary NIC DRA driver in the Kubernetes ecosystem. It is hosted in the kubernetes-sigs/dranet repository (also mirrored at google/dranet) and publishes physical and virtual network interfaces as DRA devices with rich attributes. DRANET is designed for high-performance networking workloads, especially those requiring RDMA. It works alongside existing CNI plugins — it does not replace CNI, but augments it by exposing network interfaces through the DRA API.

DRANET communicates with the kubelet through the DRA gRPC API and with the container runtime through NRI (Node Resource Interface). When a pod requests a NIC, DRANET allocates the device and the runtime injects the interface into the pod's network namespace.

### What It Publishes

DRANET publishes devices in the `dranet` device class with attributes in the `dra.net` domain. Key attributes include:

- `name` — interface name (e.g., ens2f0np0)
- `ipv4` — IPv4 address of the interface
- `ipv6` — IPv6 address
- `rdma` — boolean indicating RDMA support
- `driver` — kernel driver name
- `speed` — link speed in Mbps
- `mtu` — maximum transmission unit

DRANET also publishes topology attributes:

- `pciRoot` in the `resource.kubernetes.io` domain — the PCIe root complex the NIC is attached to. This attribute enables cross-driver pairing with GPUs.
- `numaNode` in the `dra.net` domain — the NUMA zone the NIC is in.

### Minimal Claim Examples

A basic ResourceClaim requesting an RDMA-capable NIC looks like this:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: rdma-nic
spec:
  devices:
    requests:
      - name: nic
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true
```

This claim requests one NIC with RDMA support. The scheduler finds a node with available RDMA NICs, the driver allocates the device, and the pod gets access.

For workloads that need a NIC on a specific subnet (rail-specific selection), use a prefix match:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: rail0-nic
spec:
  devices:
    requests:
      - name: nic
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true &&
                  device.attributes["dra.net"].ipv4.startsWith("10.0.")
```

This claim requests an RDMA NIC on the 10.0.x.x subnet. In a multi-rail network topology, each rail has a distinct subnet. This selector ensures the NIC is on rail 0.

## GPU-NIC Topology Pairing

### The Problem

GPU performance in distributed workloads depends heavily on the data path between the GPU and the network. When a GPU needs to send data over the network (for example, during an RDMA write or read operation), the shortest path is through the PCIe root complex they share. If the GPU and NIC sit under different PCIe roots, data must cross the CPU's interconnect, adding latency and reducing bandwidth.

In multi-GPU nodes with 8 GPUs and 8 NICs, correct pairing is critical. Benchmark data from the DRANET project shows up to 59.6% bandwidth improvement for all_gather operations and 58.1% for all_reduce operations when GPUs and NICs are correctly paired on the same PCIe root complex.

### Node Topology

A typical high-performance GPU node has a topology like this:

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
    │  Pairing:  matchAttribute on resource.kubernetes.io/pcieRoot│
    └─────────────────────────────────────────────────────────────┘

    Request sizes:
    • 2-pair request → pack into one NUMA zone
    • 4-pair request → fill one NUMA zone entirely
    • 8-pair request → entire node
    • Rail isolation: each NIC in a request on a distinct rail subnet
```

This node has two NUMA zones. Each NUMA zone has one PCIe root complex with four GPU-NIC pairs. Each NIC is on a distinct network rail (identified by subnet). A 2-pair request should pack into one NUMA zone for best performance. A 4-pair request fills an entire NUMA zone. An 8-pair request spans the entire node.

### How DRA Handles It

Both the GPU driver (such as the NVIDIA DRA driver) and DRANET publish a `pcieRoot` attribute in the `resource.kubernetes.io` domain. This common domain is the key to cross-driver pairing. A single ResourceClaim can request one GPU and one NIC with a `matchAttribute` constraint that ensures both devices share the same `pcieRoot` value.

Here is a minimal example:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-nic-pair
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
      - name: nic
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true &&
                  device.attributes["dra.net"].ipv4.startsWith("10.0.")
    constraints:
      - requests: ["gpu", "nic"]
        matchAttribute: resource.kubernetes.io/pcieRoot
```

This claim requests one GPU and one RDMA NIC on rail 0 (10.0.x.x subnet). The constraint ensures the GPU and NIC share the same PCIe root complex. The scheduler finds a node with both devices available under the same root, and the driver allocates them as a pair.

For workloads that need multiple GPU-NIC pairs with NUMA co-location, the claim includes both PCIe constraints (for each pair) and a NUMA constraint (grouping all NICs, and by transitive topology, all GPUs):

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: two-gpu-nic-pairs
spec:
  devices:
    requests:
      - name: gpu-0
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
      - name: nic-0
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true &&
                  device.attributes["dra.net"].ipv4.startsWith("10.0.")
      - name: gpu-1
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
      - name: nic-1
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true &&
                  device.attributes["dra.net"].ipv4.startsWith("10.1.")
    constraints:
      - requests: ["gpu-0", "nic-0"]
        matchAttribute: resource.kubernetes.io/pcieRoot
      - requests: ["gpu-1", "nic-1"]
        matchAttribute: resource.kubernetes.io/pcieRoot
      - requests: ["nic-0", "nic-1"]
        matchAttribute: dra.net/numaNode
```

This claim requests two GPU-NIC pairs. The first two constraints ensure each pair shares a PCIe root. The third constraint ensures both NICs (and by transitive topology, both GPUs) are in the same NUMA zone. The scheduler allocates devices that satisfy all constraints.

## Real-World Example: The DRA Admission Webhook

This section frames an admission webhook as a teaching example. The webhook handles LLM inference workloads that need multiple GPU-NIC pairs with topology constraints. It shows where the DRA scheduler currently hits its limits and what real-world topology demands look like.

### What It Does

The DRA admission webhook is a mutating admission webhook built for LLM inference workloads. A user requests GPU-NIC pairs with a simple annotation:

```yaml
resources:
  requests:
    composite.dra.io/gpu-nic-pair: "4"   # previously: dra.llm-d.io/gpu-nic-pairs annotation (webhook approach)
```

The webhook intercepts the pod CREATE request and generates full DRA objects: ResourceClaimTemplates with CEL selectors, matchAttribute constraints, and opaque driver parameters. The user sees a simple interface, but the webhook translates it into a complex DRA allocation request.

### Why It Exists

The DRA scheduler handles pairwise `matchAttribute` constraints (same pcieRoot) well. But large-scale LLM inference needs more:

- **NUMA-aware bin-packing**: A 2-pair request should pack into one NUMA zone. A 4-pair request fills a zone. The scheduler has no packing policy for this.
- **Rail isolation**: Each NIC must be on a distinct network rail (subnet). There is no "attribute uniqueness" constraint in DRA.
- **Cross-driver topology at scale**: Allocating 8 GPU-NIC pairs with PCIe pairing, NUMA co-location, and rail isolation simultaneously exceeds what claim-level constraints can express today.

The webhook exists because the scheduler cannot enforce these policies yet.

### The Allocation Pipeline

The webhook runs a mini-scheduler at admission time:

1. **Debounce** — Batch concurrent pod creates with a 3-second quiet window. This prevents race conditions when multiple pods are created simultaneously.
2. **Sort** — Largest pair count first for better packing. This greedy heuristic improves NUMA zone utilization.
3. **Scan ResourceSlices** — Build a per-node availability map keyed by (node, rail, NUMA zone). This is the webhook's view of cluster state.
4. **Bin-pack** — Select NUMA zone and rails using a best-fit algorithm. Try to pack small requests into one NUMA zone.
5. **Create ResourceClaimTemplates** — Generate CEL selectors and matchAttribute constraints for each pair.
6. **Track pending reservations** — Maintain an in-memory map with a 2-minute TTL. This bridges the gap between admission (when the webhook runs) and scheduling (when the scheduler runs).

### What It Teaches

Each webhook mechanism duplicates a scheduler concern. This table shows the overlap:

| Webhook Mechanism | Scheduler Concern It Duplicates |
|---|---|
| ResourceSlice scanning + availability map | Scheduler's device accounting |
| NUMA zone bin-packing | Topology-aware placement policy |
| Rail exclusivity enforcement | Attribute uniqueness constraints |
| Pod affinity/anti-affinity re-evaluation | Core scheduler predicate logic |
| Priority queue serialization | Scheduler's serial pod processing |
| 2-minute TTL pending reservations | Scheduler's atomic bind cycle |

The webhook works. It is in production on multi-node H100 and B200 clusters. But it sees a point-in-time snapshot of cluster state, while the scheduler has the full picture. Every feature in this webhook is a capability the DRA scheduler could provide natively.

Consider the NUMA bin-packing logic. The webhook scans ResourceSlices, counts available GPU-NIC pairs per NUMA zone, and picks the zone with the fewest free slots that can still fit the request. This is a bin-packing heuristic. The scheduler already has topology-aware placement logic for CPU and memory. It could apply the same logic to DRA devices.

Consider rail isolation. The webhook generates CEL selectors with distinct subnet prefixes for each NIC in a claim. This ensures no two NICs in the allocation share a rail. But this is fragile. If a driver publishes NICs with overlapping subnets, or if the subnet scheme changes, the webhook breaks. An attribute uniqueness constraint in the DRA API would be simpler: "each device in this allocation must have a distinct value for attribute X."

The webhook's pending reservation map is the most brittle mechanism. When the webhook approves a pod, it reserves devices in memory with a 2-minute TTL. If the scheduler binds the pod within 2 minutes, the reservation is honored. If not, the reservation expires and the devices become available again. This works, but it is a race. The scheduler's atomic bind cycle eliminates the race. The scheduler reserves devices when it schedules the pod and commits them when it binds the pod. No TTL is needed.

## What DRA Cannot Do Yet

Three gaps remain between what the DRA scheduler supports and what topology-aware workloads need:

- **NUMA-aware bin-packing** — There is no placement policy to group device requests by topology attribute with a packing strategy. The scheduler can match devices by attribute, but it does not optimize for packing efficiency.
- **Attribute uniqueness constraints** — There is no way to express "each device in this allocation must have a distinct value for attribute X" (rail isolation). Users work around this by generating CEL selectors with distinct prefixes, but this is fragile.
- **Cross-driver topology in the scheduler** — The `matchAttribute` constraint works within claims, but the scheduler's topology-aware scheduling does not yet integrate with DRA device constraints. The scheduler knows about CPU topology (NUMA zones, sockets) and can place pods accordingly. It does not yet use device topology attributes (like NUMA zone or PCIe root) in its placement decisions.

These gaps are why the admission webhook exists. As the scheduler gains these capabilities, the webhook's allocation logic can be progressively retired.

## Summary

This guide has covered DRA from fundamentals through practical drivers:

1. **What Is DRA** — The shift from opaque device counts to attribute-based, scheduler-driven allocation.
2. **How It Works** — ResourceSlices, ResourceClaims, CEL selectors, constraints, and the allocation lifecycle.
3. **GPU Drivers** — NVIDIA, AMD, and Intel drivers that publish GPU attributes for DRA matching.
4. **NIC Drivers and Topology** — DRANET, GPU-NIC pairing via matchAttribute, and a real-world webhook showing where the scheduler's limits are.

For deeper dives into the patterns and techniques mentioned here, see the companion documents in this repository:

- [CEL Device Selector Patterns](../cel-selector-patterns.md) — Three production-grade CEL patterns for rail selection, device pinning, and cross-driver topology.
- [DRA Webhook Best Practices](../dra-webhook-best-practices.md) — Operational patterns for orphan cleanup, batch mutation, extended resource interception, and offline dry-run simulation.
