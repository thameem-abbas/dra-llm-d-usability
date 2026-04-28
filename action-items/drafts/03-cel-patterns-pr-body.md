# Draft: kubernetes/website PR for CEL Patterns

**Target:** https://github.com/kubernetes/website
**Action:** User opens PR (I can prep the fork/branch, user reviews before pushing)
**Status:** Ready for review
**Note:** Check with SIG-Docs and #sig-node first (item 4) to confirm there's no existing effort and the right location.

---

## PR Title

docs: Add CEL device selector patterns for DRA cross-driver topology

## PR Body

## Summary

- Add three production-tested CEL selector patterns for DRA device allocation
- Patterns cover: rail-specific RDMA selection, explicit device pinning with defensive guards, and cross-driver topology alignment via `matchAttribute` + CEL
- All examples use `resource.k8s.io/v1` API with `exactly` sub-field

## Motivation

The current DRA documentation has basic CEL examples but doesn't cover:
- Cross-driver topology alignment (GPU from `gpu.nvidia.com` + NIC from `dra.net` on same PCIe root)
- The three-part guard idiom for defensive attribute domain/name checking
- Rail-specific RDMA selection using `startsWith` for subnet prefix matching
- Composing multiple patterns (PCIe pairing + rail selection + NUMA co-location)

These patterns come from a production webhook and are reusable by anyone building DRA-based device allocation.

## Content

See [cel-selector-patterns.md](https://github.com/thameem-abbas/dra-llm-d-usability/blob/main/cel-selector-patterns.md) for the full content.

## Questions for reviewers

- Should this live in the main DRA concepts page or as a separate "patterns" / "examples" subpage?
- Are there other teams with CEL patterns that should be included alongside these?
- Is there an existing effort to expand DRA CEL documentation that we should contribute to instead?

## Test plan

- [ ] Content renders correctly in Hugo
- [ ] YAML examples are valid against `resource.k8s.io/v1` schema
- [ ] Links to upstream KEPs and repos are correct
