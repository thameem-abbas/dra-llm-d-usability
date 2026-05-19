# How DRA Works Under the Hood

You've seen the core concepts — ResourceSlice, ResourceClaim, DeviceClass, CEL selectors, and constraints. Now let's look at how these pieces work together in practice. This document explains the DRA object model, shows real YAML examples, walks through the allocation lifecycle, and describes the kubelet plugin interface that drivers implement.

## The DRA Object Model

DRA uses four main resource types. Two are cluster-scoped (ResourceSlice and DeviceClass), and two are namespace-scoped (ResourceClaim and ResourceClaimTemplate). Each plays a distinct role in the device allocation process.

### ResourceSlice

ResourceSlices are published by DRA drivers running on each node. A driver discovers local devices (GPUs, NICs, FPGAs, or anything else) and publishes their capabilities as device entries in ResourceSlices. Each device entry has a name and a set of typed attributes. Attributes live under a domain namespace that matches the driver name.

ResourceSlices are read-only from the workload's perspective. Only drivers create and update them. The scheduler reads them to find candidate devices for claims.

Here's a minimal example from a GPU driver publishing two devices:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceSlice
metadata:
  name: node1-gpu-nvidia-0
  ownerReferences:
    - apiVersion: v1
      kind: Node
      name: node1
      uid: <node-uid>
driverName: gpu.nvidia.com
nodeName: node1
devices:
  - name: gpu-0
    basic:
      attributes:
        gpu.nvidia.com:
          uuid:
            version: "1.0.0"
            string: GPU-abc123
          model:
            version: "1.0.0"
            string: H100
          memory:
            version: "1.0.0"
            int: 80
        resource.kubernetes.io:
          pcieRoot:
            version: "1.0.0"
            string: "0000:00:00.0"
  - name: gpu-1
    basic:
      attributes:
        gpu.nvidia.com:
          uuid:
            version: "1.0.0"
            string: GPU-def456
          model:
            version: "1.0.0"
            string: H100
          memory:
            version: "1.0.0"
            int: 80
        resource.kubernetes.io:
          pcieRoot:
            version: "1.0.0"
            string: "0000:00:01.0"
```

Each device has attributes under two domains. The `gpu.nvidia.com` domain holds GPU-specific attributes like uuid, model, and memory. The `resource.kubernetes.io` domain holds topology attributes that are shared across driver types — in this case, the PCIe root complex. This shared attribute space enables cross-driver coordination without drivers needing to know about each other.

### DeviceClass

DeviceClasses are cluster-scoped resources that name categories of devices. A DeviceClass points to a driver and can optionally carry default selectors that apply to every claim using this class.

Here's a minimal example:

```yaml
apiVersion: resource.k8s.io/v1
kind: DeviceClass
metadata:
  name: gpu.nvidia.com
spec:
  selectors:
    - cel:
        expression: device.driver == "gpu.nvidia.com"
```

The DeviceClass name is typically the same as the driver name, but doesn't have to be. Workloads reference the DeviceClass name in their claims. The scheduler uses the class's selectors (if any) plus the claim's selectors to filter candidate devices.

### ResourceClaim

A ResourceClaim is the workload's request for devices. It lives in a namespace and is referenced by pods via `spec.resourceClaims`. A claim contains one or more device requests under `spec.devices.requests`. Each request uses the `exactly` sub-field (introduced in v1.34 and GA as of v1.36) to specify:

- `deviceClassName` — which class of device
- `count` — how many
- `selectors` — CEL expressions to filter candidates

Here's a minimal example requesting a single GPU with at least 80 GB of memory:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-claim
  namespace: default
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
                  device.attributes["gpu.nvidia.com"].memory >= 80
```

When a pod references this claim, the scheduler will find one GPU on a single node where the `gpu.nvidia.com` driver has published a device with `memory >= 80`.

### ResourceClaimTemplate

ResourceClaimTemplates create a new ResourceClaim for each pod that references them. This is the right choice when each pod needs its own devices (not shared across pods). The pod spec references the template using `source.resourceClaimTemplateName`:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaimTemplate
metadata:
  name: gpu-template
  namespace: default
spec:
  spec:
    devices:
      requests:
        - name: gpu
          exactly:
            deviceClassName: gpu.nvidia.com
            count: 1
---
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
  namespace: default
spec:
  resourceClaims:
    - name: gpu
      resourceClaimTemplateName: gpu-template
  containers:
    - name: workload
      image: my-image
      resources:
        claims:
          - name: gpu
```

When the scheduler processes this pod, it creates a ResourceClaim from the template, allocates devices to it, and associates the claim with the pod. When the pod terminates, the claim is garbage collected.

## CEL Selectors: Filtering Devices by Attributes

CEL (Common Expression Language) expressions evaluate against each candidate device. The scheduler considers a device eligible for a request only if all of the request's CEL selectors evaluate to true.

The attribute access pattern is `device.attributes["driver-domain"].attributeName`. Here are some common examples:

**Numeric comparison** — select GPUs with at least 80 GB of memory:

```cel
device.attributes["gpu.nvidia.com"].memory >= 80
```

**Boolean check** — select NICs with RDMA support:

```cel
device.attributes["dra.net"].rdma == true
```

**String prefix matching** — select NICs in a specific subnet:

```cel
device.attributes["dra.net"].ipv4.startsWith("10.0.")
```

**The three-part guard idiom** — safe attribute access when devices from multiple drivers are present:

```cel
"gpu.nvidia.com" in device.attributes &&
"uuid" in device.attributes["gpu.nvidia.com"] &&
device.attributes["gpu.nvidia.com"]["uuid"] == "GPU-abc123"
```

This pattern is important. The scheduler evaluates selectors against every device in the cluster's ResourceSlices. If a selector tries to access an attribute that doesn't exist, the scheduler rejects the entire claim with an evaluation error. The three-part guard checks that the domain exists, then the attribute name exists, then compares the value. This allows selectors to safely run across devices from different drivers.

Without guards, a selector like `device.attributes["gpu.nvidia.com"].memory >= 80` will fail when evaluated against a network device that doesn't have the `gpu.nvidia.com` domain.

## Constraints: Relationships Between Devices

Constraints express relationships between devices in the same claim. They go in `spec.devices.constraints`. The `matchAttribute` constraint type requires that two or more devices share the same value for a named attribute.

This is how you request a GPU and NIC that are both on the same PCIe root complex:

```yaml
apiVersion: resource.k8s.io/v1
kind: ResourceClaim
metadata:
  name: gpu-nic-pair
  namespace: default
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
                  device.attributes["dra.net"].rdma == true
    constraints:
      - requests: ["gpu", "nic"]
        matchAttribute: resource.kubernetes.io/pcieRoot
```

The scheduler evaluates this claim by finding all eligible GPU and NIC pairs where both devices have the same `resource.kubernetes.io/pcieRoot` value. Neither driver needs to know about the other. The shared attribute (published by both drivers in the `resource.kubernetes.io` domain) and the scheduler constraint do the coordination.

This is the foundation of topology-aware scheduling in DRA. Drivers publish topology attributes in a shared domain. Workloads express topology requirements as constraints. The scheduler enforces them.

## The Allocation Lifecycle

Now let's walk through the full lifecycle of a pod requesting devices via DRA.

### Step by Step

1. **Driver starts** — the DRA driver runs as a DaemonSet on each node. It discovers local devices (GPUs, NICs, FPGAs, or anything else the driver manages). For each device, the driver builds a device entry with typed attributes and publishes it in a ResourceSlice object to the API server. This step happens continuously — if a device is added or removed, the driver updates the ResourceSlice.

2. **Admin creates DeviceClass** — a cluster administrator creates DeviceClass objects that name device categories and optionally set default selectors. This is cluster configuration, typically done once per driver type.

3. **User creates ResourceClaim** — a workload author creates a ResourceClaim (or ResourceClaimTemplate) describing what devices they need. The claim specifies the device class, count, selectors, and constraints.

4. **Pod references the claim** — the pod spec lists the claim under `spec.resourceClaims` and optionally maps it to specific containers using `resources.claims`.

5. **Scheduler evaluates** — when the pod is pending, the scheduler:
   - Reads ResourceSlices to find candidate devices matching the claim's selectors
   - Evaluates CEL expressions against each candidate
   - Checks constraints (like `matchAttribute`) across requests to ensure all devices in the claim can be satisfied simultaneously
   - Scores nodes based on available devices and other scheduling policies
   - Picks a node where all requests can be satisfied

6. **Scheduler writes allocation result** — the scheduler updates the ResourceClaim's `status.allocation` field with the specific devices allocated and the node they're on. At this point the claim is bound to a node and the devices are reserved.

7. **Kubelet calls the driver** — on the assigned node, the kubelet calls the DRA driver's `NodePrepareResources` gRPC method. The driver receives the list of allocated devices and sets them up (configures GPU settings, binds SR-IOV virtual functions, sets up RDMA queue pairs, or whatever the device needs).

8. **Containers start** — the pod's containers launch with access to the allocated devices. The kubelet passes device environment variables and CDI (Container Device Interface) device references to the container runtime.

### Allocation Flow Diagram

```
  Driver (DaemonSet)          API Server              Scheduler              Kubelet
       │                         │                       │                     │
       │── ResourceSlice ──────▶│                       │                     │
       │                         │                       │                     │
       │                         │◀── ResourceClaim ────│                     │
       │                         │                       │                     │
       │                         │    Pod scheduled ────▶│                     │
       │                         │                       │── evaluate claim    │
       │                         │                       │   find devices      │
       │                         │                       │   check constraints │
       │                         │                       │   pick node         │
       │                         │◀── allocation result ─│                     │
       │                         │                       │                     │
       │                         │── NodePrepareResources ──────────────────▶│
       │◀─── prepare devices ────│                       │                     │
       │                         │                       │              start containers
```

The flow separates concerns cleanly. The driver publishes device inventory and prepares devices when told to. The scheduler makes allocation decisions. The kubelet orchestrates the prepare step and container startup.

### What Happens on Pod Deletion

When a pod using DRA is deleted:

1. The kubelet calls the driver's `NodeUnprepareResources` method. The driver tears down device state (releases GPU contexts, removes VF bindings, cleans up RDMA resources, etc.).

2. The ResourceClaim is deallocated — the scheduler clears the `status.allocation` field. The devices become available for other pods.

3. If the claim came from a ResourceClaimTemplate, the claim itself is garbage collected. If it was a manually created ResourceClaim that multiple pods might share, it stays around (deallocated) and can be reused.

## The Kubelet DRA Plugin Interface

DRA drivers implement a gRPC interface registered with the kubelet on each node. This interface replaced the older device plugin API (which used Register → ListAndWatch → Allocate) with a cleaner separation of concerns.

The kubelet DRA plugin interface has two main methods:

- **`NodePrepareResources`** — called after the scheduler allocates devices but before the pod starts. The driver receives a list of device names that have been allocated to the claim. The driver sets up whatever the device needs: configure GPU settings, bind an SR-IOV virtual function to the pod's network namespace, set up RDMA queue pairs, mount device nodes, generate CDI specifications, or anything else required to make the device usable by the pod's containers.

- **`NodeUnprepareResources`** — called after the pod terminates. The driver cleans up device state, releases resources, and makes the devices available for future allocations.

The kubelet manages the lifecycle. Drivers don't need to watch pods, track allocation state, or coordinate with the scheduler. They respond to prepare and unprepare calls. This makes drivers simpler and more reliable.

Here's the sequence in detail:

1. The scheduler allocates devices and writes `status.allocation` on the ResourceClaim.
2. The kubelet sees the pod is assigned to its node and has a claim with an allocation.
3. The kubelet looks up the driver named in the ResourceSlice that contains the allocated devices.
4. The kubelet calls the driver's `NodePrepareResources` with the claim UID and the list of allocated device names.
5. The driver does whatever setup is needed and returns CDI device references to the kubelet.
6. The kubelet passes those CDI references to the container runtime.
7. The container runtime uses CDI to inject device nodes, environment variables, and mounts into the container.

When the pod terminates, the kubelet calls `NodeUnprepareResources` with the same claim UID and device list. The driver tears everything down.

This flow works for any device type. The scheduler doesn't need device-specific logic. The kubelet doesn't need device-specific logic. All device-specific knowledge lives in the driver.

## What's Next

You now understand how DRA works under the hood — the object model, CEL selectors, constraints, the allocation lifecycle, and the kubelet plugin interface. The next two documents cover the actual DRA drivers available today in the Kubernetes ecosystem.

The next document explores GPU DRA drivers: NVIDIA's official driver, Intel's driver for data center GPUs, and AMD's experimental driver. You'll see how each driver publishes GPU attributes, how to write claims for GPU workloads, and what challenges remain.

Link: [GPU DRA Drivers](03-gpu-dra-drivers.md)
