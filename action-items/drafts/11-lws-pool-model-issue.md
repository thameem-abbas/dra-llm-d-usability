# Draft: New Issue on kubernetes-sigs/lws

**Target:** https://github.com/kubernetes-sigs/lws/issues
**Action:** `gh` can post directly — review before posting (new issue on external repo)
**Status:** Ready for review
**Blocked on:** Item 10 (clarify template vs pool model need)

---

## Issue Title

Discussion: Finite-pool ResourceClaim distribution across LWS replicas (distinct from #444)

## Issue Body

### Context

Issue #444 proposes ResourceClaimTemplate support for LWS — creating a new ResourceClaim per replica from a template. We have a related but distinct need and wanted to check if others share it before proposing a design.

We're running LLM inference workloads (prefill/decode disaggregation) using LWS with DRA-based GPU-NIC pairing. Our current setup uses a mutating admission webhook to create ResourceClaimTemplates per pod at admission time ([openshift-psap/dra-rail-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook)).

### The gap: pool-model claim distribution

#444 covers the **template model** — create new claim per replica, 1:1 lifecycle. There's a second model that doesn't appear to be covered by any LWS issue or upstream KEP:

**Pool model:** A fixed number of pre-existing ResourceClaims (e.g., IMEX channels, TPU slices, licensed accelerators, pre-configured GPU partitions) need to be distributed across LWS replicas. When a replica scales down, its claims are returned to the pool for reuse by future replicas.

| | Template Model (#444) | Pool Model (this discussion) |
|---|---|---|
| Claim lifecycle | Created per replica, deleted with it | Pre-existing, assigned/reclaimed |
| Scaling up | New claim from template | Claim assigned from pool |
| Scaling down | Claim deleted | Claim returned to pool |
| Resource exhaustion | Unbounded (DRA scheduler decides) | Bounded by pool size |

### Upstream landscape

We looked at the broader ecosystem and found this gap isn't addressed anywhere:

- **KEP #5488** (DRA: UX for multiple multi-host resources) — was the closest unified solution but was [closed by triage bot](https://github.com/kubernetes/enhancements/issues/5488) in April 2026 with no replacement
- **KEP #4671** (Gang Scheduling / Workload API) — explicitly excludes ResourceClaims
- **KEP #6012** (CompositePodGroup, targeting Alpha v1.37) — too early to tell if it covers this
- **JobSet #762** — same template-model gap for JobSet, also stalled

### Questions for the community

1. Is anyone else hitting this need? Are there use cases beyond the ones we listed (IMEX, TPU slices, licensed resources)?
2. Should this be a separate issue from #444, or should #444's scope expand to cover both models?
3. Given that #444 is deferred to Workload API integration — should pool-model support follow the same path, or could it be implemented independently (e.g., as a controller that watches LWS replicas and manages claim bindings)?
4. @Edwinhr716 — you mentioned in #444 that the plan is to fold ResourceClaim support into Workload API work. Does the pool model change that calculus at all?

Full gap analysis: [thameem-abbas/dra-llm-d-usability](https://github.com/thameem-abbas/dra-llm-d-usability/blob/main/lws-resourceclaim-gap.md)
