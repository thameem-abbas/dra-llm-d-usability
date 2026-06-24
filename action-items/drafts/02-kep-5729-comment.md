# Draft: Comment on kubernetes/enhancements#5729

**Target:** https://github.com/kubernetes/enhancements/issues/5729
**Action:** `gh` can post directly
**Status:** Posted
**Upstream link:** https://github.com/kubernetes/enhancements/issues/5729#issuecomment-4337498700

### Follow-up (May 2026)

**nojnhuh responded:** Per-replica unique claims is "a different problem than what this KEP is addressing." KEP-5729 covers shared claims per PodGroup only.

**johnbelamaric** acknowledged it as a related problem and committed to think about it (May 1, 2026).

**Implication:** Per-replica unique claims confirmed as a gap. Potential paths:
- [#6048](https://github.com/kubernetes/enhancements/issues/6048) (Consume From Resource Claim) may address the pool model side
- Composite Device / Proxy Driver concepts discussed with johnbelamaric may address the cross-driver pairing side
- A follow-up comment connecting #6048 to our per-replica need may be warranted

---

## Comment Body

### Consumer Question: Does KEP-5729 support per-replica unique claims, or only shared claims per PodGroup?

We've built a DRA admission webhook for LLM inference that pairs GPUs with RDMA NICs across two independent DRA drivers (`gpu.nvidia.com` and `dra.net`). Each pod needs its own unique GPU-NIC pair — not a shared claim across the group.

KEP-5729 adds ResourceClaimTemplates to PodGroups, which solves the lifecycle management problem beautifully for shared claims. But we need to understand whether it extends to per-replica unique claims.

**Our use case:**

A LeaderWorkerSet with N replicas, where each replica's pod needs:
- 1 GPU (from `gpu.nvidia.com`) paired with 1 NIC (from `dra.net`) on the same PCIe root complex
- Each pod's GPU-NIC pair is distinct — no sharing between replicas
- Claims include CEL selectors (rail-specific RDMA matching) and `matchAttribute` constraints (PCIe root affinity)

Today our webhook creates a ResourceClaimTemplate per pod at admission time. LWS #444 proposed per-replica template creation, but is stalled.

**Questions:**

1. Can a PodGroup's `resourceClaims[].resourceClaimTemplateName` generate one ResourceClaim per pod in the group (per-replica), or does it generate one shared ResourceClaim for the entire PodGroup?

2. If per-replica is in scope: does the naming scheme (`{podgroup}-{claim-name}-{generated}`) support deterministic per-pod references? Our pods need to know which specific claim they own at scheduling time.

3. If per-replica is NOT in scope: is this expected to be handled by the individual pod's `spec.resourceClaims` (existing mechanism), with KEP-5729 only covering the shared-claim case? If so, the lifecycle management gap for per-replica claims remains open.

4. Is pool-model claim distribution in scope for this KEP? (Distributing a finite set of pre-existing ResourceClaims across replicas, reclaiming on scale-down.) This is a distinct lifecycle from template-based creation.

5. Does KEP-5729 satisfy LWS #444 and JobSet #762? If so, those stalled issues could be closed or redirected.

**Context:**

We've documented the full LWS ResourceClaim gap analysis at [thameem-abbas/dra-llm-d-usability](https://github.com/thameem-abbas/dra-llm-d-usability/blob/main/lws-resourceclaim-gap.md), including the template vs pool model distinction and the upstream circular dependency that KEP-5729 partially resolves.

Webhook source: [openshift-psap/dra-rail-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook)
