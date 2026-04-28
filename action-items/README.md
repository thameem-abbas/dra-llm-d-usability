# Action Items: Upstream DRA Engagement

Discussion items and open questions for Kubernetes SIG-Scheduling, SIG-Node, WG-Device-Management, and LWS/JobSet maintainers.

## Status Key

- [ ] Not started
- [x] Done
- [~] In progress

---

## Tier 1: Upstream Discussions

### 1. KEP-5732: Is topology-aware DRA scheduling on the beta roadmap?
- **Status:** [ ] Not started
- **Target:** [kubernetes/enhancements#5732](https://github.com/kubernetes/enhancements/issues/5732)
- **Content:** [kep-5732-use-case.md](../kep-5732-use-case.md)
- **Tags:** SIG-Scheduling, WG-Device-Management
- **Context:** v1.36 alpha shipped topology-aware scheduling but DRA integration was explicitly deferred — the DRATestPlugin PR was closed without merge. We built a production webhook that reimplements NUMA packing, PCIe affinity, rail selection, and pod affinity filtering in admission because the scheduler can't do it yet.
- **Questions for upstream:**
  - Is DRA-aware topology scheduling planned for beta? What's the timeline?
  - Are there other approaches to multi-device, cross-driver topology constraints that we should consider instead of waiting for Placement primitives?
  - Would our use case (GPU-NIC pairing across `gpu.nvidia.com` and `dra.net` drivers) be a useful test case for the DRA integration work?
- **Upstream link:** _(fill when posted)_

### 2. KEP-5491: How should consumers handle the scalar-to-list attribute transition?
- **Status:** [ ] Not started
- **Target:** [kubernetes/enhancements#5491](https://github.com/kubernetes/enhancements/issues/5491)
- **Content:** [kep-5491-feedback.md](../kep-5491-feedback.md)
- **Context:** Our webhook uses `matchAttribute` on `resource.kubernetes.io/pcieRoot` (string) and `dra.net/numaNode` (int). The intersection semantics benefit our constraints automatically — but our preflight code reads ResourceSlices directly and assumes scalar extraction (`attr.StringValue`).
- **Questions for upstream:**
  - What's the recommended Go pattern for consumers reading attributes that may be scalar or list? Check `StringValues` first, fall back to `StringValue`? Or will the API guarantee `StringValue` is populated when the list has exactly one element?
  - Should all new CEL selectors use `.includes()` even when the attribute is currently scalar, as future-proofing? If so, is this documented anywhere?
  - When a DRA driver migrates an attribute from scalar to list, should it publish both fields for backward compatibility, or is there a versioning mechanism?
- **Upstream link:** _(fill when posted)_

### 3. CEL selector patterns: Would DRA docs benefit from cross-driver examples?
- **Status:** [ ] Not started
- **Target:** `kubernetes/website` — DRA docs section
- **Content:** [cel-selector-patterns.md](../cel-selector-patterns.md)
- **Context:** Upstream DRA documentation has basic CEL examples but nothing covering cross-driver topology alignment, the three-part guard idiom for defensive attribute checking, or rail-specific RDMA selection. We have three production-tested patterns.
- **Questions before filing PR:**
  - Is there an existing effort to expand DRA CEL examples? We don't want to duplicate work.
  - Should these live in the main DRA concepts page or as a separate "patterns" subpage?
  - Are there other teams with CEL patterns they'd want to contribute alongside ours?
- **Upstream link:** _(fill when PR opened)_

---

## Tier 2: Community Conversations

### 4. DRA webhook operational patterns: Are others hitting the same problems?
- **Status:** [ ] Not started
- **Target:** `#sig-node` on Kubernetes Slack + `kubernetes-sig-node@googlegroups.com`
- **Content:** [dra-webhook-best-practices.md](../dra-webhook-best-practices.md)
- **Frame as:** We learned two patterns building a DRA admission webhook for GPU-NIC pairing — orphan cleanup and batch mutation. Sharing in case others are hitting similar issues. Are there better approaches we're missing?
- **Open questions to pose:**
  - Is anyone else building DRA mutating webhooks that create ResourceClaimTemplates during admission? How are you handling orphaned resources?
  - For concurrent admission (e.g., Deployment rollouts), are there alternatives to our debounce-sort-serialize pattern? Is a scheduler plugin a better fit?
  - KEP-5517 (ResourcePoolStatusRequest) looks like it could replace our direct ResourceSlice scanning — has anyone prototyped it?

### 5. SIG-Scheduling: What's the right layer for multi-device topology allocation?
- **Status:** [ ] Not started
- **Target:** [SIG-Scheduling meeting schedule](https://github.com/kubernetes/community/tree/master/sig-scheduling)
- **Format:** 5-minute slot
- **Context:** Our webhook duplicates ~200 lines of scheduler predicate logic (nodeSelector, nodeAffinity, podAffinity/Anti-Affinity) plus NUMA bin-packing and rail exclusivity. This is clearly the wrong layer.
- **Questions to raise:**
  - Is KEP-5732 + DRA integration the intended long-term answer, or should we be looking at scheduler plugins / extenders?
  - For NUMA-aware bin-packing (pack small requests onto utilized zones, leave full zones for large requests) — is there an existing scheduler plugin that handles this, or is it genuinely new?
  - The priority queue pattern (batch concurrent admissions, process largest-first) — is this something the scheduler should handle natively for DRA workloads?

### 6. KubeCon / DevConf: Is there appetite for a DRA production case study?
- **Status:** [ ] Not started
- **Working title:** "Rail-Aware GPU-NIC Pairing with DRA: What the Scheduler Can't Do Yet"
- **Abstract source:** kep-5732-use-case.md + dra-webhook-best-practices.md
- **Open questions:**
  - Which venue is best — KubeCon NA/EU, DevConf, or a SIG-hosted deep-dive session?
  - Should this be framed as a DRA success story (it works for complex use cases) or a gap analysis (here's what's missing)?
  - Are there co-speakers from SIG-Scheduling or WG-Device-Management who would strengthen the submission?
- **Action:** Check next KubeCon NA/EU CFP deadlines

---

## Tier 3: Code Contributions

### 7. KEP-4816: Can `FirstAvailable` replace our custom rail fallback?
- **Status:** [ ] Not started
- **Context:** KEP-4816 is GA in v1.36. Our allocator has custom fallback logic when preferred rails aren't available. `FirstAvailable` lets us express "prefer rail 0, accept rail 1, accept any RDMA NIC" as a single prioritized device request.
- **Open questions:**
  - Does `FirstAvailable` work with `matchAttribute` constraints? (i.e., can each subrequest have its own constraint scope?)
  - The KEP notes no cross-claim consistency — different claims in the same pod may select different alternatives. Does this break our rail isolation guarantee?
  - Is there a performance cost to evaluating 8 subrequests per NIC when we have 8 rails?

### 8. KEP-5517: Is `ResourcePoolStatusRequest` ready for preflight prototyping?
- **Status:** [ ] Not started
- **Context:** Our preflight checker scans all ResourceSlices from `dra.net`, extracts device attributes, and builds per-node availability maps. KEP-5517 proposes a purpose-built API for this.
- **Open questions:**
  - Does the API expose per-device attributes (NUMA zone, PCIe root, IPv4) or just aggregate pool counts?
  - Can we query availability filtered by CEL selectors, or is that client-side post-processing?
  - Alpha stability — is the API surface likely to change significantly before beta?
- **Constraint:** Experimental branch only until API stabilizes.

### 9. Open source housekeeping
- **Status:** [ ] Not started
- **Tasks:**
  - Verify repo URL in docs matches actual public repo
  - Ensure webhook repo README links to this repo's upstream discussions
  - Add upstream engagement section to webhook repo README

---

## Tier 4: LWS / Workload-Level DRA Gaps

### 10. What ResourceClaim model does LLM inference actually need?
- **Status:** [ ] Not started
- **Context:** [lws-resourceclaim-gap.md](../lws-resourceclaim-gap.md)
- **Background:** Two models exist conceptually but neither is implemented:
  - **Template model** (LWS #444): Create new claim per replica from template, shared across leader + workers. Stalled, deferred to Workload API.
  - **Pool model** (not covered anywhere): Distribute finite pre-existing claims (IMEX channels, licensed accelerators) across replicas, reclaim on scale-down.
- **Questions to resolve internally:**
  - Which model do our LLM inference workloads actually need? Is it always template (create on demand) or are there fixed-pool scenarios?
  - If both — which is more urgent? Template model has an existing (stalled) issue; pool model needs a new issue filed.
  - Could the webhook's current per-pod claim creation be extended to replica-level sharing, or does that require LWS controller changes?

### 11. KEP #6012 (CompositePodGroup): Will it cover ResourceClaim scoping?
- **Status:** [ ] Not started
- **Target:** [KEP PR #6017](https://github.com/kubernetes/enhancements/pull/6017)
- **Context:** CompositePodGroup targets Alpha in k8s 1.37. It's the most promising upstream vehicle for per-replica resource scoping — but it's too early to tell if ResourceClaims are in scope.
- **Questions to raise on the PR:**
  - Does the CompositePodGroup design accommodate ResourceClaim distribution across sub-groups?
  - Is there a mechanism for "this sub-group of pods shares a ResourceClaim" vs "each pod gets its own"?
  - How does this interact with KEP-5732's SchedulingConstraints.DRAConstraints?

### 12. LWS: Should we file a pool-model issue separate from #444?
- **Status:** [ ] Not started — blocked on item 10
- **Target:** [kubernetes-sigs/lws](https://github.com/kubernetes-sigs/lws/issues)
- **Context:** LWS #444 covers template model only. Pool model (finite claims, assign/reclaim on scale) is a fundamentally different lifecycle. KEP #5488 (the closest unified solution) is dead.
- **Questions to consider before filing:**
  - Is the LWS community receptive to new ResourceClaim work given #444 is stalled? Or would this be seen as scope creep?
  - Should this be filed on LWS, or is it better as a WG-Device-Management cross-project issue?
  - Are there other workload controllers (JobSet, Kueue) that need the same pool model? Filing jointly may carry more weight.

### 13. Upstream coordination: Who owns the Workload API + DRA integration gap?
- **Status:** [ ] Not started
- **Contacts:** @Edwinhr716 (LWS), @kannon92 (JobSet), @johnbelamaric (WG-Device-Management)
- **Context:** Circular dependency exists — LWS #444 deferred to Workload API, Workload API (KEP #4671) excludes ResourceClaims, KEP #5488 (the bridge) is dead. Everyone is waiting on something that doesn't exist.
- **Questions to raise at WG-Device-Management:**
  - Now that KEP #5488 is closed, what is the intended path for workload controllers to integrate with DRA?
  - Is someone planning to file a replacement KEP, or is this expected to emerge from KEP #6012?
  - In the interim, are workarounds (custom controller, webhook, external lease) the recommended approach, or is there a lighter-weight upstream path we're missing?
  - @johnbelamaric noted each controller still needs API changes regardless of Workload integration — does this mean LWS/JobSet should proceed independently rather than waiting?

### 14. What's the best workaround while upstream sorts this out?
- **Status:** [ ] Not started — blocked on item 10
- **Context:** If we need pool-model claim distribution before upstream delivers (likely late 2027+), we need a workaround.
- **Options under consideration:**
  - **A: Custom controller** — watches LWS replicas, binds pre-existing ResourceClaims via label/annotation matching. Most Kubernetes-native, best lifecycle awareness.
  - **B: Mutating webhook** — injects ResourceClaim refs into pod specs at creation, pulling from pool. Simpler but less lifecycle-aware (no reclaim on scale-down without reconciler).
  - **C: External lease** — init container acquires claim lease from coordination service. Most flexible, adds operational complexity.
- **Open questions:**
  - Are there other approaches we haven't considered? (e.g., Kueue integration, DRA driver-side pooling)
  - Has anyone in the community built a claim pool controller? We'd rather contribute to existing work than start from scratch.
  - Does the webhook approach (option B) compose with our existing dra-admission-webhook, or would they conflict?

---

## Recommended Sequence

```
Week 1:  Items 1, 2, 10    (file DRA discussions + clarify LWS requirements)
Week 2:  Items 4, 11       (broadcast to SIG channels + start monitoring KEP #6012)
Week 3:  Items 5, 13       (present at SIG-Scheduling + engage LWS/JobSet maintainers)
Week 4+: Items 3, 6, 7, 12 (PRs, proposals, and LWS issue after relationships established)
Ongoing: Items 8, 9, 14    (background work, no external dependency)
```
