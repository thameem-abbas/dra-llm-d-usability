# Consumer Feedback: DRA Admission Webhook Using pcieRoot/numaNode MatchAttribute Constraints

## Our Use Case

We operate a mutating admission webhook for LLM inference workloads that converts a synthetic resource request (`dra.llm-d.io/gpu-nic-pair: "N"`) into full DRA ResourceClaimTemplates pairing NVIDIA GPUs with RDMA NICs. Each GPU-NIC pair is constrained via `matchAttribute` on `resource.kubernetes.io/pcieRoot` (string) to ensure PCIe root affinity, and optionally via `matchAttribute` on `dra.net/numaNode` (int) for NUMA co-location across NICs. Beyond claim generation, the webhook also reads ResourceSlices directly to perform preflight availability checking and cluster-level allocation decisions before admitting pods.

## What KEP-5491 Improves For Us

The intersection semantics for `matchAttribute` directly benefit our GPU-NIC pairing constraints with zero code changes. Today, both the GPU driver (`gpu.nvidia.com`) and the NIC driver (`dra.net`) publish `resource.kubernetes.io/pcieRoot` as a scalar string (e.g., `"0000:00"`). Our constraint `matchAttribute: resource.kubernetes.io/pcieRoot` on `[gpu-0, nic-0]` works because both values are identical scalars.

Under KEP-5491, if a future device (e.g., a CPU or accelerator) publishes `pcieRoot` as a list (because it spans multiple PCIe root complexes), the intersection semantics mean our existing constraint still matches correctly -- a GPU with scalar `"0000:00"` intersects with a CPU listing `["0000:00", "0000:80"]`. This is exactly the right behavior and requires no changes on our side.

## Migration Concerns

Our webhook also scans ResourceSlices directly for preflight and allocation logic. This code assumes scalar attribute extraction:

- `getPCIeRoot()` reads `attr.StringValue` and ignores `StringValues` entirely ([preflight.go:222-228](../../internal/webhook/preflight.go))
- NUMA zone grouping uses `int` as a map key from `attr.IntValue` ([allocator.go:500-502](../../internal/webhook/allocator.go))
- GPU-NIC matching does a boolean map lookup `gpuRoots[nic.pcieRoot]` on a single string ([preflight.go:172](../../internal/webhook/preflight.go))

If a driver migrates `pcieRoot` from `StringValue` to `StringValues`, our preflight silently returns empty strings and breaks pair matching. Three concrete requests:

1. **Scalar-to-list migration pattern**: Document the recommended Go pattern for consumers reading attributes that may be scalar or list. Should we check `StringValues` first and fall back to `StringValue`? Or should the API guarantee that `StringValue` is populated as a convenience when the list has exactly one element?

2. **Driver migration guidance**: When a DRA driver wants to adopt list-typed attributes for a field that was previously scalar, should it publish both `StringValue` (for backward compatibility) and `StringValues`? Or is there a versioning mechanism so consumers know which field to read? The KEP mentions scalars are treated as singleton sets for `matchAttribute` -- does this mean the API server normalizes them, or is this purely scheduler-side logic?

3. **CEL future-proofing with `.includes()`**: Should all new CEL selectors use `.includes()` even when the attribute is currently scalar? If so, this should be called out as a best practice. Our selectors currently use `==` for string comparisons (e.g., `device.attributes["dra.net"].rdma == true`). The KEP notes that `==` will fail at CEL compile time if a driver changes an attribute to list type. Recommending `.includes()` universally would prevent this breakage.

## NUMA Note

Our webhook uses `dra.net/numaNode` (int) for NUMA co-location constraints across NICs. SIG-Node discussions have trended toward `resource.kubernetes.io/pcieRoot` as the preferred topology attribute over NUMA, since PCIe root affinity directly captures the data-path relationship that matters for GPU-NIC pairing.

Is `dra.net/numaNode` expected to become list-typed? Under Intel Sub-NUMA Clustering (SNC), one NUMA zone maps to a portion of a socket, so the attribute itself likely remains a single integer -- the issue is that NUMA is the wrong abstraction level for device pairing, not that NUMA needs lists. We plan to migrate our constraints from `numaNode` to `pcieRoot`-only, but clarity on the long-term status of NUMA-based attributes in the DRA ecosystem would help us and other consumers prioritize that work.
