# Draft: Slack/Mailing List Message for SIG-Node

**Target:** `#sig-node` on Kubernetes Slack + `kubernetes-sig-node@googlegroups.com`
**Action:** User posts manually
**Status:** Ready for review

---

## Message

**Subject:** Patterns from building a DRA mutating webhook for GPU-NIC pairing — anyone else hitting these?

We've been running a DRA admission webhook in production that pairs GPUs with RDMA NICs for LLM inference (prefill/decode disaggregation). Along the way we hit two operational problems that required non-obvious solutions, and we're curious if others building DRA webhooks have found better approaches.

**Problem 1: Orphaned ResourceClaimTemplates.** Our webhook creates ResourceClaimTemplates during pod admission, then patches the pod to reference them. But the pod doesn't exist yet when the template is created, so there's no ownerReference — Kubernetes GC won't clean them up. We built a reconciler that detects unreferenced templates, annotates them with a timestamp, waits a grace period (to avoid racing the scheduler), and optionally auto-deletes. Is anyone else dealing with this? Is there a simpler approach we're missing?

**Problem 2: Rail collisions under concurrent admission.** When a Deployment rolls out N replicas simultaneously, N admission requests hit the webhook at once. Without coordination, multiple pods grab the same network rail. We added a debounce-batch-serialize pipeline (collect requests for 3s, sort by resource demand, process largest-first). This feels like the wrong layer — is there a scheduler plugin or DRA mechanism that handles this?

We also noticed KEP-5729 (ResourceClaim Support for Workloads, alpha in v1.36) adds ResourceClaimTemplates to PodGroups — this looks like it addresses the LWS/JobSet claim lifecycle gap. Has anyone tried it with multi-device DRA claims?

Wrote up both patterns in detail: https://github.com/thameem-abbas/dra-llm-d-usability/blob/main/dra-webhook-best-practices.md

Webhook source: https://github.com/openshift-psap/dra-rail-admission-webhook
