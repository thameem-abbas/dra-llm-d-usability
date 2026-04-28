# Draft: WG-Device-Management Talking Points

**Target:** WG-Device-Management meeting
**Action:** User presents
**Status:** Ready for review

---

## Topic

"Workload API + DRA Integration: Who owns this gap?"

## Context to Establish (1 min)

We run LLM inference on LWS with DRA-based GPU-NIC pairing. Hit two layers of gaps:
- **Device layer:** Webhook reimplements scheduler topology logic (filed on KEP-5732)
- **Workload layer:** LWS has no ResourceClaim support, and the upstream path is circular

## The Circular Dependency — Partially Resolved (2 min)

Lay out the situation factually, then ask about remaining gaps:

1. **LWS #444** (ResourceClaimTemplate for LWS) → stalled, deferred to "Workload API integration"
2. **JobSet #762** (same gap for JobSet) → stalled, also "waiting on Workload API"
3. **KEP #4671** (Gang Scheduling / Workload API) → explicitly excludes ResourceClaims
4. **KEP #5488** (Multi-host DRA UX — the bridge) → closed by triage bot, April 2026
5. **KEP #5729** (ResourceClaim Support for Workloads) → **Alpha v1.36, active.** Adds ResourceClaimTemplates to PodGroup. Solves shared-claim-per-group.
6. **KEP #6012** (CompositePodGroup) → hierarchical grouping, delegates ResourceClaim work to KEP-5729

KEP-5729 breaks the circular dependency for the template model. But our use case needs per-replica unique claims (each pod gets its own GPU-NIC pair), not a shared claim across the group. The pool model also remains unaddressed.

## Questions (2 min)

1. Does KEP-5729 satisfy LWS #444 and JobSet #762, or do those issues need additional work beyond what KEP-5729 provides?
2. For workloads needing per-replica unique claims (each pod gets distinct GPU-NIC pair, not shared) — is there a planned extension to KEP-5729 or a separate mechanism?
3. **Pool-model gap** — distributing a finite set of pre-existing claims across replicas, reclaiming on scale-down. Not covered by KEP-5729 (template-only). Is anyone working on this?
4. Should LWS/JobSet adopt KEP-5729 now (alpha) or wait for beta? @johnbelamaric noted each controller still needs API changes regardless.
5. In the interim for per-replica claims, what's the recommended workaround? Custom controller? Mutating webhook? We'd rather contribute to existing work.

## Links to Share

- LWS gap analysis: https://github.com/thameem-abbas/dra-llm-d-usability/blob/main/lws-resourceclaim-gap.md
- Full usability repo: https://github.com/thameem-abbas/dra-llm-d-usability
