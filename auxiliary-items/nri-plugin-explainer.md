# NRI Plugin: What It Does and Why It Exists

## What Is NRI

NRI (Node Resource Interface) is a hook framework for OCI-compatible container runtimes (CRI-O and containerd). NRI plugins register with the runtime and receive callbacks at specific points in a container's lifecycle — sandbox creation, container creation, container stop, sandbox removal. Plugins can inspect pod metadata, inject devices, modify container specs, and execute host-side operations like moving network interfaces between namespaces.

NRI is implemented in CRI-O 1.26+ and containerd 1.7+. Plugins communicate with the runtime via a Unix socket at `/var/run/nri/nri.sock` and register with a priority index that determines callback ordering when multiple plugins are active.

## Why DRANET Needs NRI (CDI Alone Is Not Enough)

DRA drivers return CDI (Container Device Interface) IDs from `PrepareResourceClaims()`. CDI is declarative — a JSON spec that describes device nodes, mount points, and environment variables. The container runtime reads the CDI spec and applies it when creating the container.

CDI works for GPUs: the NVIDIA driver returns CDI IDs like `nvidia.com/gpu=0`, and the runtime makes `/dev/nvidia0` visible in the container. No runtime-phase logic needed.

Network devices are different. Making a NIC available to a pod requires:

1. **Moving the host network interface into the pod's network namespace** — a privileged netlink operation that must happen after the pod sandbox is created but before containers start
2. **Configuring IP addresses, routes, routing rules, neighbors, VRF, MTU, and ethtool settings** inside the pod netns
3. **Moving RDMA link devices** into the pod netns (for RoCE and InfiniBand)
4. **Injecting RDMA character devices** (`/dev/infiniband/uverbsN`, `/dev/infiniband/rdma_cmN`) into containers
5. **Cleaning up** — returning devices to the host netns on pod stop

CDI can handle step 4 (device node injection) but cannot express steps 1–3 or 5. These require imperative, sequenced operations at specific lifecycle points. NRI provides exactly those hooks.

## The Dual-Hook Architecture

DRANET uses two separate hook systems because kubelet and the container runtime operate under different time budgets:

```
                    PHASE 1: DRA Hooks                    PHASE 2: NRI Hooks
                    (kubelet, time-relaxed)                (CRI-O/containerd, ~2s budget)

    Scheduler       PrepareResourceClaims()               RunPodSandbox()
    allocates  ───► ┌─────────────────────────┐      ───► ┌─────────────────────────┐
    device          │ Query cloud APIs         │           │ Read PodConfigStore      │
                    │ DHCP, ethtool discovery   │           │ Move NIC to pod netns    │
                    │ Build device config       │           │ Configure IPs/routes/MTU │
                    │ Store in PodConfigStore   │           │ Attach RDMA link device  │
                    │ (BoltDB)                  │           │ Update claim status      │
                    │ Return CDI IDs            │           └─────────────────────────┘
                    └─────────────────────────┘
                                                          CreateContainer()
                                                     ───► ┌─────────────────────────┐
                                                          │ Inject RDMA char devices │
                                                          │ (/dev/infiniband/*)      │
                                                          └─────────────────────────┘
```

**Why the split**: Phase 1 can take seconds (cloud API calls, DHCP, device discovery). Phase 2 must complete within the NRI plugin request timeout (default ~2s). By pre-computing everything in Phase 1 and storing it in BoltDB (the PodConfigStore), Phase 2 just reads the pre-computed config and executes fast host-side operations.

## NRI Hooks in DRANET

DRANET implements four NRI hooks ([source](https://github.com/kubernetes-sigs/dranet/blob/main/pkg/driver/nri_hooks.go)):

### `RunPodSandbox` — NIC injection and RDMA link attachment

Fires when the container runtime creates the pod sandbox (network namespace). This is the primary hook:

1. **Lookup**: Retrieves device config from PodConfigStore using pod UID
2. **NIC move** (`attachNetdevToNS`): Moves the host interface into the pod netns via netlink. Applies:
   - IP addresses and MAC
   - Ethtool configuration (ring buffers, offloads)
   - eBPF program management (attach or detach)
   - VRF table configuration
   - Routing table entries
   - Policy rules (source-based routing for multi-rail RoCE)
   - Neighbor entries (ARP/NDP)
3. **RDMA link move** (`attachRdmaToNS`): Moves the RDMA link device into the pod netns (exclusive mode only; shared mode uses character devices via `CreateContainer`)
4. **Status update**: Sets `NetworkDeviceReady` and `RDMALinkReady` conditions on the ResourceClaim status. Status updates are non-blocking (goroutine with 3s timeout) to avoid delaying the hook response.

For InfiniBand-only devices (no netdev), only the RDMA link move and status update execute.

### `CreateContainer` — RDMA character device injection

Fires when a container is about to be created within the pod. Injects RDMA character devices (`/dev/infiniband/uverbsN`, `/dev/infiniband/rdma_cmN`) as `LinuxDevice` entries in the container adjustment. These are the userspace access points for RDMA operations (ibverbs, rdma-core).

### `StopPodSandbox` — cleanup

Fires on pod shutdown. Moves network interfaces and RDMA link devices back to the host namespace. Best-effort — errors are logged but don't fail the shutdown. The kernel cleans up when the namespace is deleted regardless.

### `Synchronize` — state reconciliation

Fires on NRI plugin startup. Receives the list of live pods from the runtime. Prunes stale PodConfigStore entries for pods that were deleted while the driver was down.

## The Timeout Problem

NRI plugins have a request timeout enforced by the container runtime. Exceeding it causes the runtime to kill the plugin's connection.

**Default timeout**: CRI-O and containerd both default to ~2 seconds. This is sufficient for simple device injection but not for multi-VF RDMA operations on 8-GPU nodes, where `RunPodSandbox` must process 8 NICs sequentially (netns move + routing + RDMA attach per NIC).

**Symptom**: DRANET crashes with `/var/run/nri/nri.sock: connection refused` every ~25 minutes under load. CRI-O terminates the NRI plugin connection when a hook exceeds the timeout, and subsequent pods fail until the plugin re-registers.

**Fix**: Increase `nri_plugin_request_timeout` in CRI-O configuration. Required on every GPU worker node.

### CRI-O Configuration

```ini
[crio.nri]
enable_nri = true
nri_plugin_request_timeout = "30s"
nri_plugin_registration_timeout = "10s"
```

### OpenShift MachineConfig

For OpenShift clusters, deploy via MachineConfig targeting the GPU worker MachineConfigPool:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-nri-timeout-gpu-workers
  labels:
    machineconfiguration.openshift.io/role: <gpu-worker-mcp>
spec:
  config:
    storage:
      files:
      - path: /etc/crio/crio.conf.d/10-nri-timeout.conf
        mode: 0644
        contents:
          source: "data:text/plain;charset=utf-8,[crio.nri]%0Aenable_nri%20=%20true%0Anri_plugin_request_timeout%20=%20%2230s%22%0Anri_plugin_registration_timeout%20=%20%2210s%22%0A"
```

**Note**: 30s is sufficient for current 8-GPU node configurations. For larger nodes or slower storage, increase further. The dual-hook architecture means only the fast path runs here — all slow operations (cloud API, DHCP) already completed in the DRA Prepare phase.

## NRI + Composite Driver

The composite driver's shadow claims flow through DRANET's NRI hooks identically to normal DRA claims:

```
Composite driver Prepare
  → Creates shadow NIC claim (pre-filled allocation + opaque params)
  → gRPC to DRANET → DRANET stores config in PodConfigStore
  → CDI IDs returned to kubelet
     → CRI-O applies CDI specs (GPU visible)
     → NRI RunPodSandbox fires → DRANET reads PodConfigStore → NIC moves to pod netns
     → NRI CreateContainer fires → RDMA char devices injected
```

DRANET's NRI hooks look up config by pod UID in PodConfigStore. Whether the config was stored from a normal DRA claim or a composite driver's shadow claim is transparent — PodConfigStore just maps pod UID → device configs. This was validated end-to-end: PodConfigStore lookup succeeds, `attachNetdevToNS` and `attachRdmaToNS` execute normally, and claim status updates apply to the shadow claim via `ownerReference`.

## Operational Notes

- **Plugin registration**: DRANET registers as NRI plugin index 42. It only acts on pods that have entries in its PodConfigStore — pods without DRA network claims are passed through without modification. Safe to run alongside other NRI plugins (e.g., SR-IOV).

- **Idempotency**: NRI hooks must be idempotent. A pod that fails to start is retried locally by the container runtime multiple times — each retry fires the same hooks. DRANET handles this: if a device is already in the target netns, the move is a no-op.

- **RDMA modes**: DRANET supports two RDMA modes:
  - **Exclusive** (`rdmaSharedMode: false`): RDMA link device is moved to pod netns via `RunPodSandbox`. Pod has full RDMA device isolation.
  - **Shared** (`rdmaSharedMode: true`): RDMA link stays in host netns. Only character devices are injected via `CreateContainer`. Multiple pods can share the same RDMA device.

- **Status updates are async**: Claim status updates (`NetworkDeviceReady`, `RDMALinkReady` conditions) are applied in a background goroutine with a 3-second timeout. This prevents slow API server responses from blocking the NRI hook and triggering the timeout kill.
