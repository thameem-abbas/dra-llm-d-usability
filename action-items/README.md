# Action Items: Upstream DRA Engagement

Reframed around three tracks: tack-on contributions to existing KEPs, one new KEP proposal, and community engagement. Based on analysis of which gaps map to existing upstream work vs what needs new proposals.

## Status Key

- [ ] Not started
- [x] Done
- [~] In progress

---

## Track 1: Tack-On Contributions (Highest ROI)

### 1. KEP-5732: File production use case for intra-node DRA topology scheduling
- **Status:** [ ] Not started
- **Target:** [kubernetes/enhancements#5732](https://github.com/kubernetes/enhancements/issues/5732)
- **Content:** [kep-5732-use-case.md](../kep-5732-use-case.md)
- **Tags:** SIG-Scheduling, WG-Device-Management
- **Why tack on:** KEP-5732's deferred DRA integration is exactly our gap. Our allocator.go (22 functions of topology logic) is production proof this is needed. Gaps in NUMA packing, PCIe affinity, batch mutation, and pending reservation tracking all collapse if the scheduler handles topology-aware DRA allocation natively.
- **Questions for upstream:**
  - Is DRA-aware topology scheduling planned for beta (v1.37)? What's the timeline?
  - Would our GPU-NIC pairing use case (cross-driver: `gpu.nvidia.com` + `dra.net`, PCIe root matching, NUMA co-location) be useful as a co-design test case?
  - Our batch mutation and pending reservation patterns exist because admission webhooks lack the scheduler's global view — does KEP-5732 aim to eliminate this class of workaround?
- **Upstream link:** _(fill when posted)_

### 2. KEP-5729: Does per-PodGroup ResourceClaim extend to per-replica unique claims?
- **Status:** [ ] Not started
- **Target:** [kubernetes/enhancements#5729](https://github.com/kubernetes/enhancements/issues/5729)
- **Contacts:** @helayoty, @mortent, @nojnhuh (KEP-5729 authors)
- **Why tack on:** KEP-5729 (Alpha v1.36) adds ResourceClaimTemplates to PodGroup — shared claim per group. Our use case needs per-replica unique claims (each pod gets its own GPU-NIC pair). This is a natural extension of the same API surface and controller.
- **Questions for upstream:**
  - Can PodGroup ResourceClaimTemplates generate per-replica claims (one claim per PodGroup instance), or only one shared claim per group?
  - If per-replica: does the naming scheme support deterministic per-replica references?
  - Is pool model (finite pre-existing claims distributed across replicas, reclaimed on scale-down) in scope for KEP-5729, or does it need a separate proposal?
  - Does KEP-5729 satisfy LWS #444 and JobSet #762, or do those issues need additional work?
- **Upstream link:** _(fill when posted)_

### 3. KEP-5491: Consumer feedback on scalar-to-list attribute transition
- **Status:** [ ] Not started
- **Target:** [kubernetes/enhancements#5491](https://github.com/kubernetes/enhancements/issues/5491)
- **Content:** [kep-5491-feedback.md](../kep-5491-feedback.md)
- **Why tack on:** Our webhook uses `matchAttribute` on scalar `pcieRoot` and `numaNode`. KEP-5491 changes MatchAttribute semantics to set intersection. Forward-compatible for us, but our preflight code reads ResourceSlices directly and assumes scalar extraction.
- **Questions for upstream:**
  - What's the recommended Go pattern for consumers reading attributes that may be scalar or list?
  - Should new CEL selectors use `.includes()` even when the attribute is currently scalar, as future-proofing?
  - When a driver migrates an attribute from scalar to list, should it publish both fields for backward compatibility?
- **Upstream link:** _(fill when posted)_

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
- **Draft:** [04-sig-node-slack-message.md](drafts/04-sig-node-slack-message.md)
- **Frame as:** Two patterns from production — orphan cleanup and batch mutation. Are others hitting these? Are there better approaches?

### 7. SIG-Scheduling: Present topology allocation use case
- **Status:** [ ] Not started
- **Target:** [SIG-Scheduling meeting](https://github.com/kubernetes/community/tree/master/sig-scheduling)
- **Format:** 5-minute slot
- **Draft:** [05-sig-scheduling-talking-points.md](drafts/05-sig-scheduling-talking-points.md)
- **Frame as:** Production evidence for KEP-5732 DRA integration + gauge interest in device availability query API (item 4)

### 8. WG-Device-Management: KEP-5729 + per-replica claims + pool model
- **Status:** [ ] Not started
- **Target:** WG-Device-Management meeting
- **Draft:** [13-wg-device-mgmt-talking-points.md](drafts/13-wg-device-mgmt-talking-points.md)
- **Contacts:** @helayoty, @mortent (KEP-5729), @Edwinhr716 (LWS), @kannon92 (JobSet), @johnbelamaric
- **Frame as:** KEP-5729 partially resolves the circular dependency. Two remaining gaps: per-replica unique claims and pool model. Who owns these?

### 9. KubeCon / DevConf talk proposal
- **Status:** [ ] Not started
- **Working title:** "Rail-Aware GPU-NIC Pairing with DRA: What the Scheduler Can't Do Yet"
- **Draft:** [06-kubecon-abstract.md](drafts/06-kubecon-abstract.md)
- **Action:** Check next KubeCon NA/EU CFP deadlines

---

## Track 4: Internal Decisions

### 10. Resolve: What ResourceClaim model does LLM inference need?
- **Status:** [ ] Not started
- **Context:** [lws-resourceclaim-gap.md](../lws-resourceclaim-gap.md)
- **Background:** KEP-5729 covers shared-claim-per-PodGroup (template model). Still missing: per-replica unique claims and pool model.
- **Questions to resolve internally:**
  - Do our workloads always need per-replica unique claims, or are there shared-claim scenarios KEP-5729 already covers?
  - Are there fixed-pool scenarios (IMEX channels, licensed accelerators) that need pool model?
  - Should we file a pool-model issue on LWS, or wait for KEP-5729 authors to clarify scope?
- **Blocks:** Items 11, 12

### 11. File LWS pool-model issue (if needed)
- **Status:** [ ] Not started — blocked on item 10
- **Target:** [kubernetes-sigs/lws](https://github.com/kubernetes-sigs/lws/issues)
- **Draft:** [12-lws-pool-model-issue.md](drafts/12-lws-pool-model-issue.md)

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

---

## Recommended Sequence

```
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
