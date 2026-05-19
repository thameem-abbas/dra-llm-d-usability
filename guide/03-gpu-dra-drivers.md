# GPU DRA Drivers

## Why GPU Drivers Needed DRA

GPUs are not interchangeable. A machine learning training job may require a specific GPU architecture, a minimum amount of video memory, or placement in a particular NUMA zone. Video transcoding workloads may need Intel Quick Sync or NVIDIA NVENC support. Inference services may demand a specific CUDA compute capability.

Device plugins reduced all of this diversity to a single counter. When a cluster advertises `nvidia.com/gpu: 8`, workloads see eight identical resources. They cannot distinguish between an H100 with 80 GB of memory and a T4 with 16 GB. They cannot request a GPU on NUMA node 0 or verify that a GPU sits on the same PCIe root complex as a high-speed network interface.

DRA solves this problem by publishing GPU attributes into ResourceSlices. The scheduler matches ResourceClaim selectors against these attributes. A workload can request "a GPU with at least 80 GB of memory" or "an H100" or "a GPU on NUMA zone 0 with PCIe root address pci-0000:00:00.0". The scheduler evaluates these constraints and allocates a matching device.

GPU DRA drivers publish these attributes. They enumerate devices, measure their capabilities, discover their topology, and write ResourceSlices to the API server. When a claim arrives, they prepare the device for use and return CDI (Container Device Interface) specifications that tell the container runtime how to inject the GPU into the pod.

## NVIDIA k8s-dra-driver-gpu

### Overview

The NVIDIA GPU DRA driver is the most widely deployed GPU DRA driver in the Kubernetes ecosystem. Originally developed and maintained by NVIDIA, it was donated to the Cloud Native Computing Foundation at KubeCon Europe in March 2026. It now lives under `kubernetes-sigs/k8s-dra-driver-gpu` as the reference implementation for GPU DRA.

The driver manages two resource types: **GPUs** and **ComputeDomains**.

GPUs represent physical GPU devices. Each GPU is published with a full set of attributes: model name, memory capacity, compute capability, architecture generation, UUID, PCIe address, and NUMA zone. Workloads select GPUs by filtering on these attributes.

ComputeDomains represent logical partitions of GPUs. A ComputeDomain can be a MIG (Multi-Instance GPU) slice, an MPS (Multi-Process Service) share, or a time-sliced GPU partition. Workloads that do not need exclusive access to a full GPU can request a ComputeDomain instead, enabling fine-grained sharing.

The NVIDIA GPU DRA driver replaces the older `nvidia/k8s-device-plugin`, which used the device plugin API.

### What It Publishes

The driver publishes attributes in the `gpu.nvidia.com` domain. Key attributes include:

- `uuid` — unique device identifier, for example `GPU-abc12345-6789-0def-1234-56789abcdef0`
- `productName` — model name, for example `NVIDIA H100 80GB HBM3`
- `memory` — video memory capacity in bytes
- `architecture` — GPU architecture generation, for example `Hopper` or `Blackwell`
- `cudaComputeCapability` — CUDA compute capability version, for example `9.0`
- `numaNode` — NUMA zone the GPU is attached to

The driver also publishes topology attributes in the `resource.kubernetes.io` domain:

- `pcieRoot` — PCIe root complex address, for example `pci-0000:00:00.0`

This attribute enables cross-driver topology matching. A workload can request a GPU and a network interface on the same PCIe root complex by filtering on `pcieRoot` in both claims.

### Minimal Claim Examples

Request a GPU with at least 80 GB of memory:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: large-gpu
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["gpu.nvidia.com"].memory >= 85899345920
```

Request a GPU by product name:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: h100-gpu
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["gpu.nvidia.com"].productName == "NVIDIA H100 80GB HBM3"
```

Request a GPU on NUMA node 0:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: numa0-gpu
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["gpu.nvidia.com"].numaNode == 0
```

### Prerequisites

The NVIDIA GPU DRA driver requires:

- Kubernetes v1.34.2 or later (DRA GA release)
- Container runtime with CDI support (containerd v1.7+, CRI-O v1.28+)
- NVIDIA Driver v580 or later on each GPU node
- NVIDIA GPU Operator v25.3.0 or later (optional, but recommended for automated deployment)

The NVIDIA GPU Operator can deploy the DRA driver automatically. It replaces the device plugin with the DRA driver and configures the runtime for CDI.

## AMD GPU DRA Driver

### Overview

The AMD GPU DRA driver is part of the AMD ROCm ecosystem. It targets AMD Instinct data center GPUs, including the MI300X and MI250 series, as well as AMD Radeon consumer GPUs. The driver is currently in beta. API surface and attribute schemas may change between releases.

The driver is part of the `ROCm/k8s-device-plugin` project. When deployed in DRA mode, it enumerates AMD GPUs, publishes their attributes, and prepares them for pod injection.

### What It Publishes

The driver publishes attributes in the `gpu.amd.com` domain. Key attributes include:

- `uuid` — unique device identifier
- `productName` — model name, for example `AMD Instinct MI300X`
- `memory` — video memory capacity in bytes
- `computeUnits` — number of compute units on the GPU
- `targetGfxVersion` — GFX ISA version, for example `gfx942` for MI300X
- `firmwareVersion` — device firmware version

The driver also publishes the `resource.kubernetes.io/pcieRoot` topology attribute, enabling GPU-NIC topology pairing.

### Minimal Claim Example

Request an AMD GPU by GFX version (MI300X):

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: mi300x-gpu
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.amd.com
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["gpu.amd.com"].targetGfxVersion == "gfx942"
```

Request an AMD GPU with at least 64 GB of memory:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: large-amd-gpu
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.amd.com
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["gpu.amd.com"].memory >= 68719476736
```

### Current Limitations

The AMD GPU DRA driver is in beta. Attribute names and schemas may change between releases. The driver does not yet publish a ComputeDomain equivalent. GPU partitioning and sharing are not exposed through DRA. The ecosystem of examples and community tooling is smaller than the NVIDIA ecosystem.

## Intel GPU Resource Driver

### Overview

The Intel GPU resource driver is part of the `intel/intel-resource-drivers-for-kubernetes` project. It supports Intel data center GPUs, including the Data Center GPU Flex series, Max series (Ponte Vecchio), and Arc series. It also supports integrated GPUs (iGPUs) found in Intel Xeon and Core processors.

The driver supports SR-IOV virtual functions. A single physical GPU can be partitioned into multiple virtual functions, each assigned to a different pod. This enables GPU sharing with hardware-level isolation.

### What It Publishes

The driver publishes attributes in the `gpu.intel.com` domain. Key attributes include:

- `uuid` — unique device identifier
- `productName` — model name
- `memory` — video memory capacity in bytes
- `deviceType` — `integrated` or `discrete`
- `pciAddress` — PCIe bus address

The driver also publishes the `resource.kubernetes.io/pcieRoot` topology attribute for discrete GPUs.

### Minimal Claim Example

Request an Intel discrete GPU:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: intel-gpu
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.intel.com
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["gpu.intel.com"].deviceType == "discrete"
```

Request an Intel GPU with at least 32 GB of memory:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: large-intel-gpu
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.intel.com
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["gpu.intel.com"].memory >= 34359738368
```

### Use Cases

Intel GPUs are widely used for workloads that benefit from Intel-specific hardware acceleration:

**Media transcoding** — Intel iGPUs and Flex GPUs include Quick Sync Video engines optimized for video encoding and decoding. These are commonly deployed at scale for video processing pipelines.

**AI inference** — Intel Max series GPUs and Gaudi accelerators (which use a separate DRA driver but the same framework) target inference workloads. They provide high throughput for transformer models and computer vision tasks.

**GPU sharing** — SR-IOV virtual function support allows multiple pods to share a single physical GPU. Each pod receives an isolated virtual function with dedicated memory and compute resources.

## Comparing GPU DRA Drivers

|                          | NVIDIA                               | AMD                       | Intel                          |
|--------------------------|--------------------------------------|---------------------------|--------------------------------|
| **Attribute domain**     | `gpu.nvidia.com`                    | `gpu.amd.com`            | `gpu.intel.com`               |
| **Status**               | GA (CNCF)                           | Beta                      | GA                             |
| **GPU partitioning**     | Yes (MIG, MPS, time-slicing via ComputeDomains) | No | SR-IOV VFs                    |
| **Topology attributes**  | pcieRoot, numaNode                  | pcieRoot                  | pcieRoot                       |
| **Data center GPUs**     | H100, B200, A100                    | MI300X, MI250             | Max, Flex                      |
| **Install method**       | GPU Operator / Helm                 | Helm                      | Helm                           |

All three drivers publish the `resource.kubernetes.io/pcieRoot` attribute. This enables GPU-NIC topology pairing, which ensures that a GPU and its network interface sit on the same PCIe root complex. This is critical for RDMA workloads, where cross-root-complex traffic introduces latency and reduces bandwidth.

## What's Next

With GPU drivers covered, the next document explores NIC DRA drivers and the GPU-NIC topology pairing problem — how to ensure a GPU and its network interface sit on the same PCIe root complex for optimal RDMA performance.

Link: [NIC DRA Drivers and GPU-NIC Topology](04-nic-dra-drivers.md)
