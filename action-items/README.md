# Action Items: Upstream DRA Engagement

Reframed around three tracks: tack-on contributions to existing KEPs, one new KEP proposal, and community engagement. Based on analysis of which gaps map to existing upstream work vs what needs new proposals.

## Status Key

- [ ] Not started
- [x] Done
- [~] In progress

---

## Track 1: Tack-On Contributions (Highest ROI)

### 1. KEP-5732: File use case for intra-node DRA topology scheduling
- **Status:** [x] Posted
- **Target:** [kubernetes/enhancements#5732](https://github.com/kubernetes/enhancements/issues/5732)
- **Content:** [kep-5732-use-case.md](../kep-5732-use-case.md)
- **Tags:** SIG-Scheduling, WG-Device-Management
- **Why tack on:** KEP-5732's deferred DRA integration is exactly our gap. Our allocator.go (22 functions of topology logic) demonstrates this is needed. Gaps in NUMA packing, PCIe affinity, batch mutation, and pending reservation tracking all collapse if the scheduler handles topology-aware DRA allocation natively.
- **Questions for upstream:**
  - Is DRA-aware topology scheduling planned for beta (v1.37)? What's the timeline?
  - Would our GPU-NIC pairing use case (cross-driver: `gpu.nvidia.com` + `dra.net`, PCIe root matching, NUMA co-location) be useful as a co-design test case?
  - Our batch mutation and pending reservation patterns exist because admission webhooks lack the scheduler's global view — does KEP-5732 aim to eliminate this class of workaround?
- **Upstream link:** https://github.com/kubernetes/enhancements/issues/5732#issuecomment-4337498195

### 2. KEP-5729: Does per-PodGroup ResourceClaim extend to per-replica unique claims?
- **Status:** [x] Posted
- **Target:** [kubernetes/enhancements#5729](https://github.com/kubernetes/enhancements/issues/5729)
- **Contacts:** @helayoty, @mortent, @nojnhuh (KEP-5729 authors)
- **Why tack on:** KEP-5729 (Alpha v1.36) adds ResourceClaimTemplates to PodGroup — shared claim per group. Our use case needs per-replica unique claims (each pod gets its own GPU-NIC pair). This is a natural extension of the same API surface and controller.
- **Questions for upstream:**
  - Can PodGroup ResourceClaimTemplates generate per-replica claims (one claim per PodGroup instance), or only one shared claim per group?
  - If per-replica: does the naming scheme support deterministic per-replica references?
  - Is pool model (finite pre-existing claims distributed across replicas, reclaimed on scale-down) in scope for KEP-5729, or does it need a separate proposal?
  - Does KEP-5729 satisfy LWS #444 and JobSet #762, or do those issues need additional work?
- **Upstream link:** https://github.com/kubernetes/enhancements/issues/5729#issuecomment-4337498700
- **Findings (May 2026):**
  - nojnhuh (KEP author) responded: per-replica unique claims is "a different problem than what this KEP is addressing"
  - johnbelamaric acknowledged it as a related problem and committed to think about it (May 1, 2026)
  - **Confirmed gap**: KEP-5729 solves shared-claim-per-PodGroup only. Per-replica unique claims need a separate mechanism.
  - See [#6048](https://github.com/kubernetes/enhancements/issues/6048) for a related "Consume From" proposal that may address the pool model side.

### 3. KEP-5491: Consumer feedback on scalar-to-list attribute transition
- **Status:** [x] Posted
- **Target:** [kubernetes/enhancements#5491](https://github.com/kubernetes/enhancements/issues/5491)
- **Content:** [kep-5491-feedback.md](../kep-5491-feedback.md)
- **Why tack on:** Our webhook uses `matchAttribute` on scalar `pcieRoot` and `numaNode`. KEP-5491 changes MatchAttribute semantics to set intersection. Forward-compatible for us, but our preflight code reads ResourceSlices directly and assumes scalar extraction.
- **Questions for upstream:**
  - What's the recommended Go pattern for consumers reading attributes that may be scalar or list?
  - Should new CEL selectors use `.includes()` even when the attribute is currently scalar, as future-proofing?
  - When a driver migrates an attribute from scalar to list, should it publish both fields for backward compatibility?
- **Upstream link:** https://github.com/kubernetes/enhancements/issues/5491#issuecomment-4337499390
- **Findings (May 2026):**
  - everpeace responded directly to all three questions (May 28, 2026):
  - **Scalar-to-list migration**: Consumer code needs explicit care for the type change. No API-level convenience — consumers must check both `StringValue` and `StringValues`.
  - **Driver migration**: scalar-as-singleton-set treatment is purely scheduler-side. When a driver ships a type change, all consumers of that attribute need explicit handling. everpeace will document this migration path in the KEP.
  - **CEL `.includes()`**: Yes — recommended universal pattern. Works on scalar values too. Migration path: (1) migrate `==` to `.includes()` in driver-side and user-side manifests with sufficient migration period, (2) then driver ships the type change.
  - **Related**: everpeace noted [KEP-5978 (ClusterResourceClaimTemplate)](https://github.com/kubernetes/enhancements/issues/5978) relates to our "define complex claims simply" use case.
  - **Timeline**: KEP-5491 staying alpha, targeting second alpha for v1.37.
  - **Impact on us**: Update our CEL selectors to use `.includes()` proactively. Update preflight code to handle both scalar and list attribute forms.

### 17. KEP-6080: Engage on Derived Attributes for cross-driver topology
- **Status:** [x] Posted
- **Target:** [kubernetes/enhancements#6081](https://github.com/kubernetes/enhancements/pull/6081) (PR) / [#6080](https://github.com/kubernetes/enhancements/issues/6080) (issue)
- **Contacts:** @gauravkghildiyal (author), @johnbelamaric (tagged us directly)
- **Why tack on:** derivedAttributes eliminates our dependency on standardized attribute names for cross-driver matchAttribute constraints. Our Pattern 3 requires both `gpu.nvidia.com` and `dra.net` to publish `resource.kubernetes.io/pcieRoot` — derivedAttributes lets each request extract its own key via CEL, bridging disparate schemas without forced standardization.
- **Draft:** [17-kep-6080-comment.md](drafts/17-kep-6080-comment.md)
- **Questions for upstream:**
  - Can multiple derivedAttributes per request be matched in separate constraints simultaneously?
  - Are there CEL cost budget limits for derivedAttributes expressions?
  - Do computed derived values appear in ResourceClaim allocation status for downstream consumers?
  - How does this interact with KEP-5491 list-typed attributes?
- **Priority:** High — johnbelamaric directly tagged @thameem-abbas on PR #6081, v1.37 target.
- **Upstream link:** https://github.com/kubernetes/enhancements/pull/6081#issuecomment-4606653735
- **Update (June 2026):**
  - kad LGTM (May 29): "derived attributes will be good help to express complex topological concepts"
  - pohly self-assigned to PR (June 2) — now reviewing
  - **Urgency: Post our comment before PR merges — gaining traction fast**

---

## Track 2: New KEP Proposal

### 4. Propose: Device Availability Query API
- **Status:** [ ] Not started
- **Target:** New KEP on kubernetes/enhancements (SIG-Scheduling or SIG-Node)
- **Why new KEP:** No upstream mechanism exists for external consumers to query per-device availability with attribute-level filtering. Our preflight scans ResourceSlices directly to answer "how many unallocated RDMA NICs on NUMA zone 0 of node X?" — this is undocumented, unsupported, and breaks when slice format changes. Broad applicability: any DRA admission webhook, capacity-aware controller, or autoscaler hits this wall.
- **Scope:**
  - Request/response API for querying device pool availability
  - Filter by driver, device class, attribute selectors (CEL)
  - Return per-node, per-topology-zone availability counts
  - Read-only — no allocation side effects
- **Prior art:** KEP-5517 was misidentified as this but is actually scheduler-internal accounting. Nothing else exists.
- **Next steps:**
  - Validate demand — are other DRA webhook/controller authors scanning ResourceSlices directly?
  - Draft problem statement for SIG-Scheduling discussion (item 7)
  - Identify sponsors

---

## Track 3: Community Engagement

### 5. CEL selector patterns: PR to kubernetes/website
- **Status:** [ ] Not started
- **Target:** `kubernetes/website` — DRA docs section
- **Content:** [cel-selector-patterns.md](../cel-selector-patterns.md)
- **Fork/Branch:** `thameem-abbas/website:docs/dra-cel-selector-patterns`
- **Questions before filing PR:**
  - Is there an existing effort to expand DRA CEL examples?
  - Should these live in the main DRA concepts page or as a separate "patterns" subpage?
  - Are there other teams with CEL patterns to contribute alongside ours?
- **Upstream link:** _(fill when PR opened)_

### 6. DRA webhook operational patterns: Slack + mailing list
- **Status:** [ ] Not started
- **Target:** `#sig-node` on Kubernetes Slack + `kubernetes-sig-node@googlegroups.com`
- **Content:** [dra-webhook-best-practices.md](../dra-webhook-best-practices.md)
- **Draft:** [06-sig-node-slack-message.md](drafts/06-sig-node-slack-message.md)
- **Frame as:** Two patterns from building a DRA webhook — orphan cleanup and batch mutation. Are others hitting these? Are there better approaches?

### 7. SIG-Scheduling: Present topology allocation use case
- **Status:** [~] In progress — invited to present
- **Target:** [SIG-Scheduling meeting](https://github.com/kubernetes/community/tree/master/sig-scheduling)
- **Format:** 5-minute slot
- **Draft:** [07-sig-scheduling-talking-points.md](drafts/07-sig-scheduling-talking-points.md)
- **Frame as:** Concrete evidence for KEP-5732 DRA integration + gauge interest in device availability query API (item 4)
- **Update (May 2026):** kannon92 invited us to present at Monday 12pm EST calls (May 7, on KEP-5732 issue). **Need to respond and schedule a date.** KEP-5732 now milestoned to v1.37 — good timing for this presentation.

### 8. WG-Device-Management: KEP-5729 + per-replica claims + pool model
- **Status:** [ ] Not started
- **Target:** WG-Device-Management meeting
- **Draft:** [08-wg-device-mgmt-talking-points.md](drafts/08-wg-device-mgmt-talking-points.md)
- **Contacts:** @helayoty, @mortent (KEP-5729), @Edwinhr716 (LWS), @kannon92 (JobSet), @johnbelamaric
- **Frame as:** KEP-5729 partially resolves the circular dependency. Two remaining gaps: per-replica unique claims and pool model. Who owns these?

### 9. KubeCon / DevConf talk proposal
- **Status:** [ ] Not started
- **Working title:** "Rail-Aware GPU-NIC Pairing with DRA: What the Scheduler Can't Do Yet"
- **Draft:** [09-kubecon-abstract.md](drafts/09-kubecon-abstract.md)
- **Action:** Check next KubeCon NA/EU CFP deadlines

---

## Track 4: Internal Decisions

### 10. Resolve: What ResourceClaim model does LLM inference need?
- **Status:** [ ] Not started
- **Context:** [lws-resourceclaim-gap.md](../lws-resourceclaim-gap.md)
- **Background:** KEP-5729 confirmed to cover shared-claim-per-PodGroup only (May 2026). [#6048](https://github.com/kubernetes/enhancements/issues/6048) proposes a "Consume From" reservation mechanism that may address the pool model. johnbelamaric wants to get something into the v1.37 KEP pipeline.
- **Questions to resolve internally:**
  - Do our workloads always need per-replica unique claims, or are there shared-claim scenarios KEP-5729 already covers?
  - Are there fixed-pool scenarios (IMEX channels, licensed accelerators) that need pool model?
  - Does #6048's "Consume From" model fit our LWS claim distribution needs?
  - Should we engage on #6048 directly or file a separate pool-model issue on LWS?
- **Blocks:** Items 11, 12

### 11. File LWS pool-model issue (if needed)
- **Status:** [ ] Not started — blocked on item 10
- **Target:** [kubernetes-sigs/lws](https://github.com/kubernetes-sigs/lws/issues)
- **Draft:** [11-lws-pool-model-issue.md](drafts/11-lws-pool-model-issue.md)
- **Note:** May be partially superseded by [#6048](https://github.com/kubernetes/enhancements/issues/6048). If #6048's Consume-From model covers the pool use case, this LWS issue should reference #6048 rather than proposing a separate mechanism.

### 12. Evaluate interim workaround for per-replica claims
- **Status:** [ ] Not started — blocked on item 10
- **Options:** Custom controller, mutating webhook extension, or external lease
- **Context:** If KEP-5729 doesn't extend to per-replica, we need a bridge. Prefer contributing to existing work over building new.

### 13. Open source housekeeping
- **Status:** [ ] Not started
- **Tasks:**
  - Verify repo URL in docs matches actual public repo
  - Ensure webhook repo README links to this repo's upstream discussions
  - Add upstream engagement section to webhook repo README

### 14. Engage on #6048 (DRA: Consume From Resource Claim)
- **Status:** [x] Posted — our comment received engagement
- **Target:** [kubernetes/enhancements#6048](https://github.com/kubernetes/enhancements/issues/6048)
- **Contacts:** @johnbelamaric (author), @44past4, @pravk03, @wojtek-t
- **Why:** Directly addresses pool model gap. johnbelamaric authored this and explicitly asked for our input.
- **Draft:** [14-issue-6048-comment.md](drafts/14-issue-6048-comment.md)
- **Findings (May 2026):**
  - **Pushed to v1.38** — pravk03 and johnbelamaric confirmed not enough time for v1.37.
  - **Our first-pod-as-leader concern validated**: wojtek-t went further — reservations should attach to "intent of workload" (Workload object), not runtime representation. Even PodGroup/CompositePodGroup may not be the right level.
  - **Use case doc needed**: johnbelamaric wants to start with written use cases before design. @44past4 has internal doc — extract use cases to public doc as starting point.
  - **Next step**: Help with use case doc. Our pool model use cases (IMEX channels, licensed accelerators, pre-configured GPU partitions) are directly relevant. Consider contributing to the public use case document when it materializes.

### 15. Provide diagrams to johnbelamaric
- **Status:** [ ] Not started
- **Context:** johnbelamaric explicitly asked for pictures showing:
  - GPU-NIC topology per node (added to [kep-5732-use-case.md](../kep-5732-use-case.md))
  - How rails are selected (indexing model)
  - Cross-pod constraints
  - CPU alignment requirements
  - ResourceClaimTemplates being generated
- **Action:** Share repo link with updated diagrams. Consider additional diagrams for AKS topology if different from IBM Cloud.

### 16. Follow up on johnbelamaric meeting outcomes
- **Status:** [ ] Not started
- **Context:** Meeting held May 1, 2026. Follow up on:
  - Per-replica unique claims — johnbelamaric said he would think about it
  - Composite Device / Proxy Driver concepts — which approach for v1.37?
  - v1.37 KEP pipeline timeline and what's feasible
  - How AKS topology differs from IBM Cloud topology

---

## Recommended Sequence

```
Week 1:  Items 14, 15, 16, 17 (#6048 engagement + diagrams + meeting follow-up + KEP-6080 comment — v1.37 urgency)
Week 1:  Items 1, 2, 10       (file KEP-5732 use case + ask KEP-5729 authors + internal decision)
Week 2:  Items 6, 7            (broadcast to SIG channels)
Week 3:  Items 3, 8            (KEP-5491 feedback + WG-Device-Management)
Week 4+: Items 4, 5, 9, 11    (new KEP draft, CEL PR, KubeCon, LWS issue)
Ongoing: Items 12, 13          (workaround eval + housekeeping)
```

---

## Evaluated — Not Applicable

KEPs reviewed and determined not to overlap with our use case:

- **KEP-4816 (Prioritized Alternatives / `FirstAvailable`)** — Solves per-pod device fallback (prefer H100, accept A100). Our problem is coordinated rail assignment across pods — each pod needs a *distinct* rail, not a *preferred* rail.
- **KEP-5055 (Device Taints)** — Marks devices as unhealthy/offline for maintenance. Our preflight checks allocation state (unallocated NICs with rdma+ifName attributes), not device health.
- **KEP-5007 (Binding Conditions)** — Post-allocation device readiness (wait for fabric-attached GPU to be physically connected). Our preflight is pre-allocation capacity checking.
- **KEP-5517 (Node Allocatable Resources)** — Misidentified as "ResourcePoolStatusRequest." Actually solves scheduler-internal double-counting of CPU/memory between DRA and pod.spec.resources. Not a device availability query API.
