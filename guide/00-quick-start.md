# DRA GPU-NIC Pairing: Quick Start for Application Users

This page covers everything you need to request GPU-NIC pairs for your workloads. The DRA admission webhook handles topology, rail selection, and NUMA placement automatically. You configure your pod spec; the webhook and scheduler do the rest.

## Requesting GPU-NIC Pairs

Add a resource request to your pod (or pod template in a Deployment, Job, etc.):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-inference-pod
  namespace: my-namespace    # namespace must have webhook label (see below)
spec:
  containers:
    - name: inference
      image: my-image
      resources:
        requests:
          dra.llm-d.io/gpu-nic-pair: "4"
```

That's it. The webhook intercepts this request and generates the DRA objects (ResourceClaimTemplates, CEL selectors, matchAttribute constraints) needed to allocate 4 GPU-NIC pairs with correct PCIe topology. Your pod will get 4 GPUs, each paired with an RDMA NIC on the same PCIe root complex.

### Common pair counts

| Pairs | What you get |
|-------|-------------|
| `"1"` | 1 GPU + 1 NIC on the same PCIe root |
| `"2"` | 2 pairs, packed into one NUMA zone |
| `"4"` | 4 pairs, fills one NUMA zone |
| `"8"` | 8 pairs, entire node (both NUMA zones) |

The webhook handles NUMA bin-packing: small requests pack into one zone to leave the other zone available for larger requests.

## Namespace Label

Your namespace must be labeled for the webhook to process pods:

```
dra.llm-d.io/webhook-enabled: "true"
```

Pods in unlabeled namespaces are not intercepted.

## Extended Resource Mode

If your workload uses standard GPU resource requests (`nvidia.com/gpu`), the webhook can intercept those too. This is useful during migration from device plugins to DRA:

```yaml
resources:
  requests:
    nvidia.com/gpu: "2"
```

The `/mutate-ext` endpoint converts this to DRA ResourceClaims automatically. This works in all non-system namespaces without the label requirement. Each container gets claim references only for the GPUs it requested.

**Note:** A pod cannot request both `dra.llm-d.io/gpu-nic-pair` and `nvidia.com/gpu`. Use one or the other.

## What You Get

When the webhook processes your pod, each GPU-NIC pair is guaranteed:

- **PCIe root affinity** — GPU and NIC share the same PCIe root complex for optimal RDMA bandwidth
- **Rail assignment** — each NIC lands on a distinct network rail (subnet), preventing bandwidth contention
- **NUMA locality** — pairs are packed into the fewest NUMA zones possible

You don't need to understand ResourceClaims, CEL selectors, or matchAttribute constraints to use this. The webhook generates all of it from your pair count.

## Verifying Your Allocation

After your pod is running, check what was allocated:

```bash
# See the ResourceClaimTemplates created for your pod
kubectl get resourceclaimtemplates -n my-namespace -l app.kubernetes.io/managed-by=dra-gpu-nic-webhook

# See the bound ResourceClaims
kubectl get resourceclaims -n my-namespace

# Check a specific claim's allocation details
kubectl get resourceclaim <claim-name> -n my-namespace -o yaml
```

The claim's `status.allocation` shows which specific GPU and NIC devices were allocated and on which node.

## Troubleshooting

**Pod stuck in Pending:**
- Check that the namespace has the `dra.llm-d.io/webhook-enabled: "true"` label
- Check webhook logs: `kubectl logs -n <webhook-namespace> -l app=dra-gpu-nic-webhook`
- Verify enough GPU-NIC pairs are available: check ResourceSlices for node capacity

**Pod scheduled but NIC performance is poor:**
- Verify PCIe pairing: check the ResourceClaim allocation to confirm GPU and NIC share a pcieRoot value
- Check CRI-O NRI plugin timeout: multi-VF RDMA operations require `nri_plugin_request_timeout: 60s` on GPU worker nodes. Without this, the NRI plugin may crash during device setup.

## Further Reading

For details on what happens under the hood:

- [What Is DRA?](01-what-is-dra.md) — fundamentals of Dynamic Resource Allocation
- [How DRA Works](02-how-dra-works.md) — object model, scheduler lifecycle, CEL selectors
- [GPU DRA Drivers](03-gpu-dra-drivers.md) — NVIDIA, AMD, Intel driver details
- [NIC DRA Drivers and Topology](04-nic-dra-drivers.md) — DRANET, GPU-NIC pairing mechanics, the webhook in depth
