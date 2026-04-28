# Draft: KubeCon / DevConf Talk Proposal

**Target:** KubeCon NA/EU or DevConf CFP
**Action:** User submits via conference portal
**Status:** Ready for review

---

## Title

Rail-Aware GPU-NIC Pairing with DRA: What the Kubernetes Scheduler Can't Do Yet

## Abstract (250 words)

Dynamic Resource Allocation (DRA) reached GA in Kubernetes 1.34, enabling structured device requests with CEL selectors and cross-device constraints. But for AI/HPC workloads requiring GPU-NIC RDMA pairing with PCIe affinity, NUMA locality, and multi-rail network isolation, the scheduler still can't express the topology constraints that matter.

We'll walk through a DRA admission webhook that fills this gap for LLM inference workloads targeting prefill/decode disaggregation on multi-node B200 clusters. The webhook converts a single synthetic resource request into full DRA objects — pairing GPUs and NICs across two independent DRA drivers that have no awareness of each other, using matchAttribute constraints on PCIe root, CEL-based network rail selection, and NUMA-aware bin-packing.

We'll cover three things the audience can take away:
1. **CEL selector patterns** for cross-driver device pairing that go beyond upstream examples — rail-specific RDMA selection, explicit device pinning, and the three-part guard idiom for defensive attribute checking.
2. **Operational patterns** for DRA webhooks at scale — orphan cleanup for ResourceClaimTemplates without ownerReferences, and priority queue batching to prevent rail collisions during concurrent rollouts.
3. **Where upstream is headed** — how KEP-5732 (Topology-Aware Scheduling) and KEP-5491 (List-Typed Attributes) will eventually move this logic into the scheduler, and what's still missing.

This talk is for platform engineers building DRA-based device allocation and anyone interested in the gap between DRA's API and real-world multi-device topology requirements.

## Topic

Cloud Infrastructure / Kubernetes / AI Infrastructure

## Session Type

Presentation (35 min) or Lightning Talk (10 min)

## Benefits to the Ecosystem

- First public case study of DRA usage for complex multi-device topology
- Reusable CEL patterns and operational patterns for DRA webhook authors
- Direct feedback loop to SIG-Scheduling on KEP-5732 DRA integration priorities
