# Draft: Comment on kubernetes/enhancements#6081

**Target:** https://github.com/kubernetes/enhancements/pull/6081
**Action:** `gh` can post directly
**Status:** Ready for review

---

## Comment Body

We're hoping to adopt derivedAttributes and KEP-5491 concurrently for our cross-driver GPU-NIC pairing use case (`gpu.nvidia.com` + `dra.net`, [webhook source](https://github.com/openshift-psap/dra-rail-admission-webhook)).

From our [KEP-5491 feedback](https://github.com/kubernetes/enhancements/issues/5491#issuecomment-4337499390), @everpeace confirmed that scalar-as-singleton-set normalization is purely scheduler-side — the raw attribute type in ResourceSlice changes when a driver migrates from scalar to list.

How should derivedAttributes expressions handle this? If a driver migrates `pcieRoot` from `string` to `list(string)`, does `device.attributes['gpu.nvidia.com'].pcieRoot` break in the CEL expression? Should we be writing defensive expressions, or does the CEL environment normalize this?

Curious how others are thinking about this overlap.
