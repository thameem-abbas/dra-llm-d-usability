# Draft: Comment on kubernetes/enhancements#5732

**Target:** https://github.com/kubernetes/enhancements/issues/5732
**Action:** `gh` can post directly
**Status:** Ready for review

---

## Comment Body

### Use Case: GPU-NIC Topology-Aware Allocation via DRA Admission Webhook

We've built a mutating admission webhook for LLM inference workloads that reimplements several scheduler-level concerns in admission — specifically because DRA-aware topology scheduling isn't available yet. Sharing as a concrete use case for KEP-5732's beta work, particularly the DRA integration that was deferred from v1.36 alpha.

**What we do:** Convert a synthetic resource request (`dra.llm-d.io/gpu-nic-pair: "N"`) into full DRA objects — ResourceClaimTemplates with per-pair device requests, PCIe `matchAttribute` constraints, NUMA co-location constraints, CEL-based rail selectors, and opaque NIC driver parameters. The webhook coordinates two independent DRA drivers (`gpu.nvidia.com` and `dra.net`) that have no awareness of each other.

**What we had to reimplement in admission:**

| Webhook mechanism | Scheduler concern it duplicates |
|---|---|
| ResourceSlice scanning + per-node availability map | Scheduler's device accounting |
| NUMA zone bin-packing (pack small requests onto utilized zones, leave full zones for large) | Topology-aware placement policy |
| Rail exclusivity enforcement (each NIC on a distinct network fabric) | Multi-resource anti-collocation |
| Pod affinity/anti-affinity re-evaluation (~200 lines) | Core scheduler predicate logic |
| Priority queue with debounce batching (sort by pair count, serialize) | Scheduler's serial pod processing |
| 2-minute TTL pending reservations (`map[string]time.Time` keyed by `node:rail`) | Scheduler's atomic bind cycle |

The pending reservation TTL is particularly fragile — if the scheduler takes longer than 2 minutes, another pod can double-book the same rail. The webhook also can't observe actual DRA allocation outcomes; it guesses based on ResourceSlice state at admission time.

**Three topology dimensions we need the scheduler to handle:**

1. **PCIe root complex** — already expressible via `matchAttribute` within a single claim, but cross-claim PCIe affinity (e.g., "all GPUs on the same PCIe switch group") would need Placement-level constraints.
2. **NUMA zone** — requires bin-packing policy, not just co-location. Maps to a Placement topology key with a packing strategy.
3. **Network rail/fabric** — each NIC in a multi-pair request must land on a distinct parallel network. Maps to an attribute-uniqueness constraint.

**Questions for the community:**

1. Is DRA-aware topology scheduling planned for beta? We saw the DRATestPlugin PR was closed without merge in v1.36 — is there a timeline for this work?
2. Are there alternative approaches to multi-device, cross-driver topology constraints we should consider? (scheduler plugin, extender, or something else?)
3. Would our use case be useful as a test case for the DRA integration work? The webhook is open source: [openshift-psap/dra-rail-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook).

We've documented the full gap analysis at [thameem-abbas/dra-llm-d-usability](https://github.com/thameem-abbas/dra-llm-d-usability).
