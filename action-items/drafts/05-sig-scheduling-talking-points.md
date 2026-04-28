# Draft: SIG-Scheduling Meeting Talking Points

**Target:** [SIG-Scheduling meeting](https://github.com/kubernetes/community/tree/master/sig-scheduling)
**Action:** User presents (5-minute slot)
**Status:** Ready for review

---

## Title

"GPU-NIC Topology Allocation: A DRA Webhook Doing Scheduler Work"

## Talking Points (5 minutes)

### Setup (1 min)

We run a DRA admission webhook for LLM inference that pairs GPUs with RDMA NICs. It works, but ~1500 lines of it are scheduler logic that shouldn't be in an admission webhook. Sharing because it's directly relevant to KEP-5732's DRA integration work.

### What we reimplement (2 min)

Three things the webhook does that should be scheduler concerns:

1. **NUMA bin-packing** — group NIC slots by NUMA zone, pack small requests onto utilized zones, leave full zones for large requests. 50 lines of placement policy in `selectSlots()`.

2. **Pod affinity/anti-affinity** — 200 lines re-evaluating nodeSelector, nodeAffinity, podAntiAffinity, podAffinity — including tracking webhook-pinned-but-not-yet-scheduled pods. Exact duplication of scheduler predicates.

3. **Rail exclusivity** — each NIC must land on a distinct network rail (subnet). No DRA primitive for "each device in this set must have a distinct value for attribute X."

### The fragile parts (1 min)

- **Pending reservations with 2-minute TTL** — between admission and scheduling, another pod can claim the same rail. If scheduler is slow, double-booking.
- **Stale ResourceSlice reads** — webhook sees a point-in-time snapshot. Can't observe actual allocation outcomes.
- **Priority queue** — we debounce-batch concurrent admissions and sort by demand. This is scheduler-level serialization done in a webhook.

### Questions (1 min)

1. Is KEP-5732 DRA integration the intended answer, or should we look at scheduler plugins/extenders?
2. For NUMA-aware bin-packing — is there an existing plugin, or is this genuinely a new scheduler capability?
3. Would our webhook be useful as a test case for the DRA integration beta work?

### Links

- Webhook: https://github.com/openshift-psap/dra-rail-admission-webhook
- Gap analysis: https://github.com/thameem-abbas/dra-llm-d-usability
- KEP-5732 comment: _(link once item 1 is posted)_
