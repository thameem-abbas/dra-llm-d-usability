# NIC DRA Drivers and GPU-NIC Topology

This is the fourth and final document in a reading guide about Kubernetes Dynamic Resource Allocation (DRA). If you have not already read the first three documents, start with `01-what-is-dra.md` (concepts), `02-how-dra-works.md` (object model and lifecycle), and `03-gpu-dra-drivers.md` (GPU drivers). This document covers NIC DRA drivers, the GPU-NIC topology pairing problem, and the composite DRA driver that makes cross-driver topology pairing transparent to workload authors.

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

## Composite DRA Driver: Simplifying GPU-NIC Pairing

The claims above work for InfiniBand — `matchAttribute` on `pcieRoot` is all that's needed, and DRANET handles the rest. RoCE introduces a harder problem: each rail requires distinct network configuration (routing table, gateway, MTU, policy routes) that depends on which specific NIC gets allocated. DRA's opaque driver parameters are set at claim creation time, before the scheduler picks a device. There is no mechanism to parameterize config based on the actual allocation outcome.

The [composite DRA driver](https://github.com/openshift-psap/composite-dra-driver) solves this by operating as a native DRA driver between the underlying drivers and the scheduler.

### What It Does

The composite driver runs as a DaemonSet on each node. It:

1. **Watches** underlying ResourceSlices from `gpu.nvidia.com` and `dra.net`
2. **Pairs** devices by `matchAttribute` on `pcieRoot` (config-driven, not hardcoded)
3. **Publishes** composite ResourceSlices — 8 GPU-NIC pair devices per 8-GPU node
4. **Prepares** allocated devices by resolving per-rail network config and delegating to underlying drivers via gRPC shadow claims

The scheduler sees composite devices as normal allocatable resources. Users request them with standard resource syntax:

```yaml
resources:
  requests:
    composite.dra.io/gpu-nic-pair: "4"
```

On Kubernetes 1.36+ with the `DRAExtendedResource` feature gate, this maps directly to the composite driver via a DeviceClass. On older clusters, a temporary webhook (included in the Helm chart) creates the appropriate ResourceClaimTemplate.

### Shadow Claims and Per-Device Config

When kubelet calls the composite driver to prepare an allocated device, the driver:

1. Looks up which underlying GPU and NIC make up the composite pair
2. Resolves per-rail network configuration based on the actual NIC allocated (the RailConfigResolver matches NIC IP to rail, generates routing table, gateway, MTU, policy routes)
3. Creates shadow ResourceClaims for each underlying device — pre-filled with allocation results and opaque driver parameters
4. Delegates device setup to the underlying drivers via gRPC (`gpu.nvidia.com` for GPU visibility, `dra.net` for NIC namespace + routing)

Shadow claims have `ownerReferences` to the composite claim — Kubernetes garbage collection handles cleanup automatically.

This solves the RoCE chicken-and-egg: per-device config is resolved at prepare time from the actual allocated device, not guessed at claim creation time.

### Configuration

The composite driver is config-driven. Pairing rules are defined in a ConfigMap:

```yaml
sources:
  - name: gpu
    driver: "gpu.nvidia.com"
    forwardAttributes:
      - domain: "resource.kubernetes.io"
        attributes: [pcieRoot]
  - name: nic
    driver: "dra.net"
    forwardAttributes:
      - domain: "dra.net"
        attributes: [rdma, ipv4]
    filter:
      cel: 'device.attributes["dra.net"].rdma == true'

compositions:
  - name: "gpu-nic-pair"
    members: [{source: gpu, count: 1}, {source: nic, count: 1}]
    constraints:
      - type: matchAttribute
        attribute: "resource.kubernetes.io/pcieRoot"
```

Adding a new underlying driver is a YAML change — zero driver-specific code.

### What It Solves vs What Remains

| Solved by composite driver | Still open |
|---|---|
| Per-device driver configuration (per-rail routing) | Per-replica unique claims (no workload-layer guarantee for distinct pairs across LWS replicas) |
| Admission-scheduling race (scheduler allocates natively) | Device-level bin-packing (no packing policy — anti-affinity is the workaround) |
| Orphan resource cleanup (ownerReferences cascade GC) | Cross-pool device exclusion ([#28](https://github.com/openshift-psap/composite-dra-driver/issues/28)) |

The remaining gaps are tracked in [wg-device-management #54](https://github.com/kubernetes-sigs/wg-device-management/issues/54) and [dra-roce-challenges.md](../dra-roce-challenges.md).

## What DRA Cannot Do Yet

Two gaps remain between what the DRA scheduler supports and what topology-aware workloads need:

- **Device-level bin-packing** — The scheduler has no packing policy for device allocation. Small requests can scatter across a node, fragmenting it so larger requests can't be served. Pod anti-affinity between decode and prefill pods is the current workaround. [KEP-5732](https://github.com/kubernetes/enhancements/issues/5732) (Topology-Aware Scheduling, alpha v1.36, DRA integration deferred to v1.37+ beta) and Kueue TAS are the upstream paths.

- **Per-replica unique claims** — LWS/StatefulSet/JobSet use a single pod template. Each replica gets identical ResourceClaimTemplates. The composite driver mitigates this in practice (500ms synthesizer cycles mean successive replicas see updated availability), but there is no formal scheduler-level guarantee. [KEP-5729](https://github.com/kubernetes/enhancements/issues/5729) confirmed per-replica unique claims out of scope. [#6048](https://github.com/kubernetes/enhancements/issues/6048) (Consume From Resource Claim) may eventually generalize to cover this.

## Summary

This guide has covered DRA from fundamentals through practical drivers:

1. **What Is DRA** — The shift from opaque device counts to attribute-based, scheduler-driven allocation.
2. **How It Works** — ResourceSlices, ResourceClaims, CEL selectors, constraints, and the allocation lifecycle.
3. **GPU Drivers** — NVIDIA, AMD, and Intel drivers that publish GPU attributes for DRA matching.
4. **NIC Drivers and Topology** — DRANET, GPU-NIC pairing via matchAttribute, and the composite DRA driver that makes multi-driver topology pairing transparent to workload authors.

For deeper dives into the patterns and techniques mentioned here, see the companion documents in this repository:

- [Composite DRA Driver](https://github.com/openshift-psap/composite-dra-driver) — Source code, architecture docs, Helm chart
- [DRA + llm-d: RoCE Challenges](../dra-roce-challenges.md) — Full gap analysis with upstream KEP tracking
- [NRI Plugin Explainer](../auxiliary-items/nri-plugin-explainer.md) — What DRANET's NRI plugin does and timeout configuration
- [CEL Device Selector Patterns](../cel-selector-patterns.md) — Three production-grade CEL patterns for rail selection, device pinning, and cross-driver topology
- [DRA Webhook Best Practices](../dra-webhook-best-practices.md) — Operational patterns from the previous webhook approach (orphan cleanup, batch mutation, extended resource interception)
