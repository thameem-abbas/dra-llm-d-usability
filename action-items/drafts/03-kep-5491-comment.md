# Draft: Comment on kubernetes/enhancements#5491

**Target:** https://github.com/kubernetes/enhancements/issues/5491
**Action:** `gh` can post directly
**Status:** Ready for review

---

## Comment Body

### Consumer Feedback: DRA Admission Webhook Using pcieRoot/numaNode MatchAttribute Constraints

Sharing feedback from a DRA webhook consumer perspective. We've built a mutating admission webhook that builds ResourceClaimTemplates pairing NVIDIA GPUs with RDMA NICs for LLM inference workloads. Each GPU-NIC pair uses `matchAttribute` on `resource.kubernetes.io/pcieRoot` (string), and we optionally use `matchAttribute` on `dra.net/numaNode` (int) for NUMA co-location across NICs.

**What KEP-5491 improves for us (automatically):**

The intersection semantics for `matchAttribute` directly benefit our constraints with zero code changes. Today both drivers publish `pcieRoot` as a scalar. Under KEP-5491, if a future device (e.g., CPU DRA driver) publishes `pcieRoot` as a list because it spans multiple PCIe root complexes, our existing constraint still matches correctly — a GPU with scalar `"0000:00"` intersects with a CPU listing `["0000:00", "0000:80"]`. This is exactly right.

**Where we need guidance — our webhook also reads ResourceSlices directly:**

Beyond claim generation, we scan ResourceSlices for preflight availability checking and cluster-level allocation. This code assumes scalar attribute extraction:

- `getPCIeRoot()` reads `attr.StringValue`, ignores `StringValues` entirely
- NUMA zone grouping uses `int` as a map key from `attr.IntValue`
- GPU-NIC matching does a boolean map lookup `gpuRoots[nic.pcieRoot]` on a single string

If a driver migrates `pcieRoot` from `StringValue` to `StringValues`, our preflight silently returns empty strings and breaks pair matching.

**Questions:**

1. **Scalar-to-list migration pattern**: What's the recommended Go pattern for consumers reading attributes that may be scalar or list? Should we check `StringValues` first and fall back to `StringValue`?

2. **Driver migration guidance**: When a DRA driver adopts list-typed attributes for a previously-scalar field, should it publish both `StringValue` (backward compatibility) and `StringValues`? Or is the scalar-as-singleton-set treatment purely scheduler-side, and consumers must handle both forms? This could be great to have in the KEP to guide driver authors.

3. **CEL future-proofing**: Should new CEL selectors use `.includes()` even when the attribute is currently scalar? Our selectors use `==` for comparisons (e.g., `device.attributes["dra.net"].rdma == true`). The KEP notes `==` will fail at CEL compile time if a driver changes to list type. If `.includes()` is the recommended universal pattern, it would be worth calling out as a best practice in the docs.

Webhook source: [openshift-psap/dra-rail-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook)
Full gap analysis: [thameem-abbas/dra-llm-d-usability](https://github.com/thameem-abbas/dra-llm-d-usability)
