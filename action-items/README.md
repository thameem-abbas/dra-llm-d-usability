# Action Items: Upstream DRA Engagement

Tracked tasks for getting conversations started with Kubernetes SIG-Scheduling, SIG-Node, and WG-Device-Management.

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

## Recommended Sequence

```
Week 1:  Items 1, 2        (file issues — starts conversations)
Week 2:  Item 4            (broadcast to SIG channels)
Week 3:  Item 5            (present at SIG meeting)
Week 4+: Items 3, 6, 7     (PRs and proposals after relationships established)
Ongoing: Items 8, 9        (background work, no external dependency)
```
