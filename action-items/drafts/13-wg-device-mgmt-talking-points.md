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

## The Circular Dependency (2 min)

Lay out the situation factually, then ask who's working on it:

1. **LWS #444** (ResourceClaimTemplate for LWS) → stalled, deferred to "Workload API integration"
2. **JobSet #762** (same gap for JobSet) → stalled, also "waiting on Workload API"
3. **KEP #4671** (Gang Scheduling / Workload API) → explicitly excludes ResourceClaims
4. **KEP #5488** (Multi-host DRA UX — the bridge) → closed by triage bot, April 2026, no replacement
5. **KEP #6012** (CompositePodGroup) → too early, targeting Alpha v1.37

Everyone is waiting on a solution that doesn't exist. @johnbelamaric noted in KEP #5488 discussions that each controller still needs API changes regardless of Workload integration.

## Questions (2 min)

1. Now that KEP #5488 is closed — what is the intended path for workload controllers (LWS, JobSet) to integrate with DRA? Is someone planning a replacement KEP?
2. Should LWS/JobSet proceed independently with ResourceClaim support rather than waiting for a unified solution? @johnbelamaric's point suggests they could.
3. Does KEP #6012 (CompositePodGroup) have ResourceClaim scoping in its design? If not, should we propose it?
4. We also identified a **pool-model gap** — distributing a finite set of pre-existing claims across replicas, reclaiming on scale-down. This isn't covered by #444 or #762 (both template-model). Is anyone working on this, or is there an approach we're not seeing?
5. In the interim, what's the recommended workaround? Custom controller watching LWS replicas? Mutating webhook? We'd rather contribute to existing work than build something new.

## Links to Share

- LWS gap analysis: https://github.com/thameem-abbas/dra-llm-d-usability/blob/main/lws-resourceclaim-gap.md
- Full usability repo: https://github.com/thameem-abbas/dra-llm-d-usability
