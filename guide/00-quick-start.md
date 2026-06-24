# DRA GPU-NIC Pairing: Quick Start for Application Users

This page covers everything you need to request GPU-NIC pairs for your workloads. The [composite DRA driver](https://github.com/openshift-psap/composite-dra-driver) handles topology pairing, rail configuration, and driver delegation automatically. You configure your pod spec; the composite driver and scheduler do the rest.

## Requesting GPU-NIC Pairs

Add a resource request to your pod (or pod template in a Deployment, Job, LWS, etc.):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-inference-pod
spec:
  containers:
    - name: inference
      image: my-image
      resources:
        requests:
          composite.dra.io/gpu-nic-pair: "4"
```

The composite driver publishes pre-paired GPU-NIC devices in ResourceSlices. The scheduler allocates from these natively — no webhook, no admission-time decisions. Your pod gets 4 GPUs, each paired with an RDMA NIC on the same PCIe root complex, with per-rail network configuration resolved automatically.

### How it works under the hood

1. The composite driver watches GPU and NIC ResourceSlices on each node
2. It pairs devices by `matchAttribute` on `resource.kubernetes.io/pcieRoot`
3. It publishes composite ResourceSlices (8 GPU-NIC pairs per 8-GPU node)
4. The scheduler allocates composite devices to your pod
5. At prepare time, the driver resolves per-rail routing config and delegates to the underlying GPU and NIC drivers

### Common pair counts

| Pairs | What you get |
|-------|-------------|
| `"1"` | 1 GPU + 1 NIC on the same PCIe root |
| `"2"` | 2 pairs |
| `"4"` | 4 pairs (fills one NUMA zone on typical 8-GPU nodes) |
| `"8"` | 8 pairs (entire node) |

## How the Resource Request Maps to DRA

**Kubernetes 1.36+** with the `DRAExtendedResource` feature gate: `composite.dra.io/gpu-nic-pair` maps directly to the composite DRA driver via a DeviceClass. The scheduler handles it natively — no webhook needed.

**Kubernetes < 1.36**: A mutating webhook (included in the composite driver Helm chart) intercepts the synthetic resource, creates the appropriate ResourceClaimTemplate, and patches the pod. This is a temporary bridge.

**A pod cannot request both `composite.dra.io/gpu-nic-pair` and `nvidia.com/gpu`.** Use one or the other. The composite driver already includes the GPU in each pair.

## What You Get

Each GPU-NIC pair is guaranteed:

- **PCIe root affinity** — GPU and NIC share the same PCIe root complex for optimal RDMA bandwidth
- **Per-rail network config** — routing tables, gateways, MTU, and policy routes are resolved per-NIC at prepare time based on which rail the NIC is on
- **Rail diversity** — DRA naturally allocates distinct NICs, so each NIC lands on a different rail

You don't need to understand ResourceClaims, CEL selectors, or matchAttribute constraints. The composite driver generates and manages all DRA objects internally.

## Installation

The composite driver deploys as a DaemonSet via Helm chart:

```bash
helm install composite-dra-driver oci://ghcr.io/openshift-psap/composite-dra-driver/charts/composite-dra-driver \
  --namespace composite-dra-driver --create-namespace
```

Prerequisites:
- NVIDIA GPU DRA driver (`gpu.nvidia.com`) running on GPU nodes
- DRANET (`dra.net`) or DRA SR-IOV driver running on GPU nodes
- Kubernetes v1.34+ (DRA GA)
- Container runtime with CDI support (containerd v1.7+, CRI-O v1.28+)
- CRI-O NRI plugin timeout increased to 30s+ on GPU worker nodes (see [NRI Plugin Explainer](../auxiliary-items/nri-plugin-explainer.md))

## Verifying Your Allocation

After your pod is running:

```bash
# See composite ResourceSlices on a node
kubectl get resourceslices -l driver=composite.dra.io

# See ResourceClaims for your pod
kubectl get resourceclaims -n <namespace>

# Check a specific claim's allocation details
kubectl get resourceclaim <claim-name> -n <namespace> -o yaml

# See shadow claims created by the composite driver
kubectl get resourceclaims -n <namespace> -l app.kubernetes.io/managed-by=composite-dra-driver
```

The composite claim's `status.allocation` shows which composite devices were allocated. Shadow claims show the underlying GPU and NIC allocations with per-device opaque parameters.

## Troubleshooting

**Pod stuck in Pending:**
- Check that composite ResourceSlices exist: `kubectl get resourceslices -l driver=composite.dra.io`
- Check composite driver logs: `kubectl logs -n composite-dra-driver -l app=composite-dra-driver`
- Verify underlying drivers are publishing ResourceSlices: `kubectl get resourceslices` (should see slices from `gpu.nvidia.com` and `dra.net`)
- On K8s < 1.36: verify the webhook is running and the `DRAExtendedResource` feature gate is not expected

**Pod scheduled but NIC not configured correctly:**
- Check shadow claim status: look for `NetworkDeviceReady` and `RDMALinkReady` conditions
- Verify CRI-O NRI plugin timeout: multi-VF RDMA operations require `nri_plugin_request_timeout: 30s` on GPU worker nodes. Without this, the NRI plugin may crash during device setup. See [NRI Plugin Explainer](../auxiliary-items/nri-plugin-explainer.md).
- Check DRANET logs for NIC netns move errors

**Composite driver not publishing ResourceSlices:**
- Verify underlying drivers are running and publishing their own ResourceSlices
- Check composite driver ConfigMap for correct driver names and constraint config
- Check composite driver logs for pairing errors

## Further Reading

For details on what happens under the hood:

- [What Is DRA?](01-what-is-dra.md) — fundamentals of Dynamic Resource Allocation
- [How DRA Works](02-how-dra-works.md) — object model, scheduler lifecycle, CEL selectors
- [GPU DRA Drivers](03-gpu-dra-drivers.md) — NVIDIA, AMD, Intel driver details
- [NIC DRA Drivers and Topology](04-nic-dra-drivers.md) — DRANET, GPU-NIC pairing mechanics, composite driver architecture
- [NRI Plugin Explainer](../auxiliary-items/nri-plugin-explainer.md) — what DRANET's NRI plugin does and the timeout configuration
- [DRA + llm-d: RoCE Challenges](../dra-roce-challenges.md) — remaining gaps and upstream KEP tracking
