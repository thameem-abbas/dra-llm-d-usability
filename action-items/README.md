# Action Items: Upstream DRA Engagement

Tracked tasks for getting conversations started with Kubernetes SIG-Scheduling, SIG-Node, WG-Device-Management, and LWS/JobSet maintainers.

## Status Key

- [ ] Not started
- [x] Done
- [~] In progress

---

## Tier 1: Open Upstream Issues/Discussions

### 1. File KEP-5732 use case on kubernetes/enhancements
- **Status:** [ ] Not started
- **Target:** [kubernetes/enhancements#5732](https://github.com/kubernetes/enhancements/issues/5732)
- **Content:** [kep-5732-use-case.md](../kep-5732-use-case.md)
- **Tags:** SIG-Scheduling, WG-Device-Management
- **Key ask:** DRA-aware topology scheduling in beta — our webhook proves the need
- **Why now:** DRA integration explicitly deferred to beta in v1.36. They need real-world use cases to prioritize.
- **Upstream link:** _(fill when posted)_

### 2. Comment on KEP-5491 with consumer feedback
- **Status:** [ ] Not started
- **Target:** [kubernetes/enhancements#5491](https://github.com/kubernetes/enhancements/issues/5491)
- **Content:** [kep-5491-feedback.md](../kep-5491-feedback.md)
- **Key ask:** Scalar-to-list migration pattern, `.includes()` best practice guidance
- **Why now:** Alpha in v1.36, needs real-world feedback for beta graduation criteria.
- **Upstream link:** _(fill when posted)_

### 3. Submit CEL selector patterns PR to kubernetes/website
- **Status:** [ ] Not started
- **Target:** `kubernetes/website` — DRA docs section
- **Content:** [cel-selector-patterns.md](../cel-selector-patterns.md)
- **Target page:** `content/en/docs/concepts/scheduling-eviction/dynamic-resource-allocation.md` or new subpage
- **Why now:** After items 1-2 get traction — establishes credibility first.
- **Upstream link:** _(fill when PR opened)_

---

## Tier 2: Community Engagement

### 4. Post best-practices to SIG-Node
- **Status:** [ ] Not started
- **Target:** `#sig-node` on Kubernetes Slack + `kubernetes-sig-node@googlegroups.com`
- **Content:** [dra-webhook-best-practices.md](../dra-webhook-best-practices.md)
- **Frame as:** "Patterns we learned building a DRA webhook — sharing for others"
- **Links back to:** Items 1-2 for context

### 5. Present at SIG-Scheduling meeting
- **Status:** [ ] Not started
- **Target:** [SIG-Scheduling meeting schedule](https://github.com/kubernetes/community/tree/master/sig-scheduling)
- **Format:** 5-minute slot
- **Topic:** "Production DRA webhook doing scheduler work — KEP-5732 beta priorities"
- **Evidence:** allocator.go NUMA packing, rail selection, pod affinity filtering

### 6. Draft KubeCon / DevConf talk proposal
- **Status:** [ ] Not started
- **Title:** "Rail-Aware GPU-NIC Pairing with DRA: What the Scheduler Can't Do Yet"
- **Abstract source:** kep-5732-use-case.md + dra-webhook-best-practices.md
- **Action:** Check next KubeCon NA/EU CFP deadlines

---

## Tier 3: Code Contributions

### 7. Evaluate KEP-4816 FirstAvailable adoption
- **Status:** [ ] Not started
- **Scope:** Replace custom rail fallback logic in allocator with native prioritized requests
- **Dependency:** KEP-4816 GA in v1.36 — safe to depend on
- **Benefit:** Reduces allocator complexity, proves we adopt upstream features

### 8. Prototype KEP-5517 ResourcePoolStatusRequest in preflight
- **Status:** [ ] Not started
- **Scope:** Replace direct ResourceSlice scanning with new API in preflight.go
- **Constraint:** Alpha — experimental branch only
- **Benefit:** Shows alignment with upstream direction

### 9. Open source housekeeping
- **Status:** [ ] Not started
- **Tasks:**
  - Verify repo URL in docs matches actual public repo
  - Ensure webhook repo README links to this repo's upstream discussions
  - Add upstream engagement section to webhook repo README

---

## Tier 4: LWS / Workload-Level DRA Gaps

### 10. Clarify internal requirements: template vs pool model
- **Status:** [ ] Not started
- **Context:** [lws-resourceclaim-gap.md](../lws-resourceclaim-gap.md)
- **Scope:** Document whether our need is template-model (like LWS #444 — new claim per replica) or pool-model (distribute finite pre-existing claims across replicas, reclaim on scale-down)
- **Why it matters:** Pool model is not covered by ANY upstream issue or KEP. If that's our need, we must file a new issue.

### 11. Monitor KEP #6012 (CompositePodGroup)
- **Status:** [ ] Not started
- **Target:** [KEP PR #6017](https://github.com/kubernetes/enhancements/pull/6017)
- **Why:** Most promising upstream vehicle for per-replica ResourceClaim scoping. Targeting Alpha in k8s 1.37.
- **Action:** Watch PR, attend WG-Device-Management meetings where this is discussed.

### 12. File new LWS issue for pool-model claim distribution (if needed)
- **Status:** [ ] Not started — blocked on item 10
- **Target:** [kubernetes-sigs/lws](https://github.com/kubernetes-sigs/lws/issues)
- **Scope:** Distinct from #444 (template model). Describe: finite hardware resources distributed across LWS replicas, reclaimed on scale-down. Reference #444, KEP #5488 (closed), JobSet #762.
- **Why now:** KEP #5488 was the closest unified solution and is dead. LWS #444 is stalled. Someone needs to push.

### 13. Engage LWS / JobSet maintainers on ResourceClaim integration
- **Status:** [ ] Not started
- **Contacts:** @Edwinhr716 (LWS), @kannon92 (JobSet), @johnbelamaric (WG-Device-Management)
- **Scope:** Attend WG-Device-Management meetings. Raise that KEP #5488 is dead, everyone is blocked on Workload API, and Workload API has no ResourceClaim story. Propose concrete path forward.
- **Consider:** Whether reviving KEP #5488 or filing a replacement is worth pursuing.

### 14. Evaluate workaround for pool-model claim distribution
- **Status:** [ ] Not started — blocked on item 10
- **Options:**
  - **A: Custom controller** — watches LWS replicas, binds pre-existing ResourceClaims via label/annotation matching. Most Kubernetes-native.
  - **B: Mutating webhook** — injects ResourceClaim refs into pod specs at creation, pulling from pool. Simpler but less lifecycle-aware.
  - **C: External lease** — init container acquires claim lease from coordination service. Most flexible, adds operational complexity.

---

## Recommended Sequence

```
Week 1:  Items 1, 2, 10    (file DRA issues + clarify LWS requirements)
Week 2:  Items 4, 11       (broadcast to SIG channels + start monitoring KEP #6012)
Week 3:  Items 5, 13       (present at SIG-Scheduling + engage LWS/JobSet maintainers)
Week 4+: Items 3, 6, 7, 12 (PRs, proposals, and LWS issue after relationships established)
Ongoing: Items 8, 9, 14    (background work, no external dependency)
```
