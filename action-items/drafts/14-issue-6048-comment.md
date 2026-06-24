# Draft: Comment on kubernetes/enhancements#6048

**Target:** https://github.com/kubernetes/enhancements/issues/6048
**Action:** `gh` can post directly
**Status:** Ready for review

---

## Comment Body

### Consumer Use Case: Pool Model for LLM Inference GPU-NIC Pairs

We have a concrete use case for the "Consume From" pattern from our DRA admission webhook work. Context: we pair NVIDIA GPUs with RDMA NICs for disaggregated LLM inference (prefill/decode) on multi-node clusters using LeaderWorkerSet (LWS). Full gap analysis: [thameem-abbas/dra-llm-d-usability](https://github.com/thameem-abbas/dra-llm-d-usability).

**How this maps to our needs:**

Today our webhook creates ResourceClaimTemplates per-pod at admission time — each claim pairs a GPU and NIC on the same PCIe root with CEL selectors for rail-specific RDMA and `matchAttribute` on `resource.kubernetes.io/pcieRoot`. This is the "template model." The missing piece is what we call the "pool model":

A fixed set of pre-existing GPU-NIC pair claims (e.g., 4 IMEX channels, 8 TPU slices, N licensed accelerators) distributed across LWS replicas, reclaimed on scale-down. Today there is no upstream mechanism for this — we identified this gap in [lws-resourceclaim-gap.md](https://github.com/thameem-abbas/dra-llm-d-usability/blob/main/lws-resourceclaim-gap.md).

The "Consume From" reservation model could solve this: a reservation defines the pool, and each LWS replica pod consumes a subset (e.g., 2 GPU-NIC pairs) from the reservation.

**Node topology for reference:**

```
                      Node (8 GPU-NIC pairs)
    ┌─────────────────────────────────────────────────────┐
    │                                                     │
    │   NUMA Zone 0                  NUMA Zone 1          │
    │  ┌───────────────────┐       ┌───────────────────┐  │
    │  │  PCIe Root 0x00   │       │  PCIe Root 0x80   │  │
    │  │  GPU-0 ── NIC-0   │       │  GPU-4 ── NIC-4   │  │
    │  │  (Rail0)          │       │  (Rail0)          │  │
    │  │  GPU-1 ── NIC-1   │       │  GPU-5 ── NIC-5   │  │
    │  │  (Rail1)          │       │  (Rail1)          │  │
    │  │  GPU-2 ── NIC-2   │       │  GPU-6 ── NIC-6   │  │
    │  │  (Rail2)          │       │  (Rail2)          │  │
    │  │  GPU-3 ── NIC-3   │       │  GPU-7 ── NIC-7   │  │
    │  │  (Rail3)          │       │  (Rail3)          │  │
    │  └───────────────────┘       └───────────────────┘  │
    │                                                     │
    │  Rails: Rail0=10.0.x.x  Rail1=10.1.x.x             │
    │         Rail2=10.2.x.x  Rail3=10.3.x.x             │
    │  Drivers: GPU→gpu.nvidia.com  NIC→dra.net           │
    └─────────────────────────────────────────────────────┘
```

**Rail selection / indexing model:**
- Zero-based indexing. Rail index maps to subnet prefix (Rail 2 → `10.2.0.x`).
- NUMA-aware packing: small requests (1-2 pairs) pack into the most-utilized NUMA zone; large requests (4+ pairs) target the most-available zone.
- Rail isolation: each NIC in a request must land on a distinct rail subnet. No two NICs in the same claim should share a rail.

**Cross-pod constraints:**
- Currently minimal. Our environment allows cross-rail communication via per-subnet gateway.
- Main constraint: avoid crossing NUMA zones within a single pod's allocation. A 4-pair request crossing zones takes a performance hit (not yet quantified).
- In environments without cross-rail communication, a "rail" attribute on the reservation might be needed to ensure all pods in a group use the same set of rails.

**Questions:**

1. **Reservation vs claim-based approach**: For our pool model (fixed number of GPU-NIC pairs known at deployment time), does the Reservation object approach or the claim-based approach fit better? The pool size is bounded and known upfront.

2. **LWS replica scaling**: When a LWS replica scales down, does the reservation automatically reclaim the subset that pod was consuming? How does the reservation track which subset is assigned to which pod?

3. **Cross-driver reservations**: Our pairs span two drivers (`gpu.nvidia.com` + `dra.net`). Can a single reservation include devices from multiple drivers with topology constraints (e.g., `matchAttribute` on `pcieRoot`)?

4. **Relationship to KEP-5729**: KEP-5729 covers shared claims per PodGroup but confirmed per-replica unique claims out of scope. Is "Consume From" intended to complement KEP-5729 (shared claim defines the pool, consume-from distributes subsets)? Or is it a separate mechanism?

5. **NUMA-aware subset selection**: When a pod consumes a subset from the reservation, can it express topology preferences (e.g., "give me 2 pairs from the same NUMA zone")? Or is subset selection purely positional?

Webhook source: [openshift-psap/dra-rail-admission-webhook](https://github.com/openshift-psap/dra-rail-admission-webhook)
