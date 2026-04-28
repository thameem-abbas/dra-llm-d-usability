# CEL Device Selector Patterns for Dynamic Resource Allocation

CEL selectors in DRA ResourceClaims enable precise device matching that goes beyond what DeviceClass names alone can express. The patterns below come from a production admission webhook that pairs GPUs with RDMA-capable NICs for distributed AI workloads. Each pattern is self-contained and reusable in any DRA-based device allocation pipeline.

> **API version:** All examples use the `resource.k8s.io/v1` API, which became GA in Kubernetes 1.34. In the v1 schema, `deviceClassName`, `count`, and `selectors` are nested under an `exactly` sub-field within each device request. If you are migrating from the earlier v1beta1 API, move these three fields from the request top level into `exactly`.

## Pattern 1: Rail-Specific RDMA Selection

**Problem:** In multi-rail RDMA fabrics, each rail is a separate IP subnet. A NIC must be pinned to a specific rail so that source-based policy routing directs traffic through the correct switch plane.

**Selector:**

```cel
device.attributes["dra.net"].rdma == true && device.attributes["dra.net"].ipv4.startsWith("10.0.")
```

This combines two conditions in a single CEL expression: the NIC must be RDMA-capable, and its IPv4 address must fall within the target rail's subnet. The `startsWith` string method makes prefix matching concise without requiring CIDR parsing.

**ResourceClaim excerpt:**

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: nic-rail-0
spec:
  devices:
    requests:
      - name: nic
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true &&
                  device.attributes["dra.net"].ipv4.startsWith("10.0.")
```

**Configuration-driven generation:** The webhook builds this selector dynamically from a config that lists each rail's subnet prefix:

```yaml
nicConfig:
  rdmaRequired: true
  rails:
    - subnet: "10.0.0.0/16"
      gateway: "10.0.0.1"
      ipv4Prefix: "10.0."
    - subnet: "10.1.0.0/16"
      gateway: "10.1.0.1"
      ipv4Prefix: "10.1."
```

**Fallback:** When no rails are configured, the selector simplifies to RDMA-only:

```cel
device.attributes["dra.net"].rdma == true
```

This keeps the claim valid on clusters without multi-rail networking.

## Pattern 2: Explicit Device Pinning

**Problem:** An administrator knows the exact physical topology -- which GPU is wired to which NIC on which PCIe root complex -- and wants to hard-pin allocations to specific devices by unique identifier.

**Selector:**

```cel
"gpu.nvidia.com" in device.attributes &&
"uuid" in device.attributes["gpu.nvidia.com"] &&
device.attributes["gpu.nvidia.com"]["uuid"] == "GPU-abc123"
```

**The three-part guard idiom:** The expression first checks that the attribute domain exists (`"gpu.nvidia.com" in device.attributes`), then that the specific attribute name exists within that domain (`"uuid" in device.attributes["gpu.nvidia.com"]`), and only then compares the value. This defensive pattern prevents CEL evaluation errors when a device from a different driver does not publish the attribute domain at all. Omitting either guard can cause the scheduler to reject the entire claim with an evaluation error rather than simply skipping the non-matching device.

**ResourceClaim excerpt:**

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-pinned
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
          selectors:
            - cel:
                expression: >-
                  "gpu.nvidia.com" in device.attributes &&
                  "uuid" in device.attributes["gpu.nvidia.com"] &&
                  device.attributes["gpu.nvidia.com"]["uuid"] == "GPU-abc123"
      - name: nic
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  "dra.net" in device.attributes &&
                  "name" in device.attributes["dra.net"] &&
                  device.attributes["dra.net"]["name"] == "ens2f0np0"
```

**When to use vs MatchAttribute:** Use explicit pinning when the topology is known ahead of time and maintained in configuration (e.g., a ConfigMap). Use MatchAttribute when devices publish a shared attribute (like `pcieRoot`) and you want the scheduler to discover matching pairs dynamically. Explicit pinning is deterministic but requires per-node configuration; MatchAttribute is automatic but depends on drivers publishing compatible attributes.

## Pattern 3: Cross-Driver Topology Alignment

**Problem:** A GPU and a NIC come from different DRA drivers (`gpu.nvidia.com` and `dra.net`). Neither driver knows about the other, but they must be allocated on the same PCIe root complex for optimal data transfer. Additionally, the NIC must be on a specific RDMA rail.

**Solution:** Combine a `matchAttribute` constraint with per-request CEL selectors. The constraint bridges the two drivers through a standardized attribute that both publish; the CEL selector further narrows the NIC to the desired rail.

**Full ResourceClaimSpec:**

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-nic-pair
spec:
  devices:
    requests:
      - name: gpu
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
      - name: nic
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true &&
                  device.attributes["dra.net"].ipv4.startsWith("10.0.")
    constraints:
      - requests: ["gpu", "nic"]
        matchAttribute: resource.kubernetes.io/pcieRoot
```

The `matchAttribute` field references `resource.kubernetes.io/pcieRoot`, an attribute that both the GPU driver and the NIC driver publish independently. The DRA scheduler ensures that the allocated GPU and NIC share the same value for this attribute, meaning they sit under the same PCIe root complex. The CEL selector on the NIC request runs independently, filtering candidates to RDMA-capable interfaces on the correct rail. Both conditions must be satisfied for an allocation to succeed.

This pattern is the foundation for multi-device, cross-driver topology alignment. Neither driver needs any awareness of the other; the shared attribute and the scheduler constraint do all the coordination.

## Combining Patterns: Multi-Pair Claims with NUMA Affinity

In production, these patterns compose. A request for N GPU+NIC pairs generates N request pairs, N PCIe constraints, and (optionally) a NUMA constraint across all NICs. Here is a 2-pair claim with rail pinning and NUMA co-location:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-nic-2-numa
spec:
  devices:
    requests:
      - name: gpu-0
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
      - name: nic-0
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true &&
                  device.attributes["dra.net"].ipv4.startsWith("10.0.")
      - name: gpu-1
        exactly:
          deviceClassName: gpu.nvidia.com
          count: 1
      - name: nic-1
        exactly:
          deviceClassName: dranet
          count: 1
          selectors:
            - cel:
                expression: >-
                  device.attributes["dra.net"].rdma == true &&
                  device.attributes["dra.net"].ipv4.startsWith("10.1.")
    constraints:
      # PCIe root pairing: each GPU must share a PCIe root with its paired NIC
      - requests: ["gpu-0", "nic-0"]
        matchAttribute: resource.kubernetes.io/pcieRoot
      - requests: ["gpu-1", "nic-1"]
        matchAttribute: resource.kubernetes.io/pcieRoot
      # NUMA co-location: all NICs must be on the same NUMA node
      - requests: ["nic-0", "nic-1"]
        matchAttribute: dra.net/numaNode
```

Each GPU+NIC pair is bound by a `pcieRoot` constraint (Pattern 3). Each NIC is filtered to a specific rail via CEL (Pattern 1). The final constraint ensures all NICs (and by transitive topology, all GPUs) land on the same NUMA node. Together, these give the scheduler enough information to produce a topology-optimal allocation without any driver needing cross-driver awareness.

For workloads that need all 8 GPU+NIC pairs on a node (spanning both NUMA zones), the NUMA constraint is omitted, and the webhook sets `allow-cross-numa` to signal that cross-NUMA allocation is acceptable.
