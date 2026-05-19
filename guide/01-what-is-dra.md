# What Is Dynamic Resource Allocation?

## The Problem DRA Solves

Kubernetes was designed around resources that are fundamentally interchangeable and countable: CPU cores and memory bytes. Every CPU core behaves like every other CPU core. Every gigabyte of RAM is the same as any other gigabyte. The scheduler can treat these resources as simple numbers — add them up, subtract what's used, and check if there's enough left.

But modern workloads increasingly depend on specialized hardware that doesn't fit this model. GPUs, FPGAs, network interface cards, and TPUs are not fungible. They have distinct models, memory capacities, firmware versions, physical locations in the system, and interconnection topologies. Two GPUs might both be "GPUs," but one might have 80 GB of high-bandwidth memory while the other has 16 GB. They might live on different PCIe root complexes or in different NUMA zones. These differences matter deeply to performance.

Consider two concrete examples that expose the limits of treating devices as simple counts:

1. **Attribute filtering**: A machine learning training job needs a GPU with at least 80 GB of VRAM. The cluster has a mix of A100 GPUs (40 GB and 80 GB variants) and H100 GPUs (80 GB). Simply requesting "one GPU" could land the pod on a 40 GB card, and the job would fail. There's no standard way to express "a GPU with this minimum memory capacity."

2. **Cross-device topology constraints**: A high-performance network application needs a GPU for computation and a 100 Gbps NIC for data transfer. For maximum bandwidth, the GPU and NIC must share the same PCIe root complex — otherwise, traffic crosses slower inter-socket links. Requesting "one GPU and one NIC" gives no way to express "and they must be topologically adjacent."

Neither of these requirements can be expressed with the device plugin API or extended resources. Dynamic Resource Allocation exists to solve exactly these problems.

## How We Got Here: Device Plugins and Extended Resources

Device plugins have been the standard way to expose specialized hardware in Kubernetes since version 1.8. The model is straightforward: a driver runs on each node, registers a device type with the kubelet over gRPC, and reports how many devices it has. The kubelet handles allocation when pods land on the node. The API surface is minimal: Register (tell the kubelet you exist), ListAndWatch (report device counts and health), and Allocate (prepare devices for a container).

From the workload's perspective, these devices appear as extended resources — opaque integers in the pod spec. A pod might request `nvidia.com/gpu: 2` or `intel.com/qat: 1`. To the scheduler, these are just numbers. It sees "this node has 4 nvidia.com/gpu available, this pod wants 2, that fits" and makes a placement decision. The scheduler has no visibility into what those devices actually are — their models, capabilities, or physical relationships.

This works well for simple allocation scenarios. If your workload just needs "give me N GPUs" or "give me M virtual functions," device plugins and extended resources get the job done. The problems emerge when you need more:

- **No attribute filtering**: You can't distinguish an A100 from an H100, a 40 GB variant from an 80 GB variant, or a device running firmware version X from one running version Y. All devices with the same extended resource name look identical to the scheduler.

- **No cross-device constraints**: If your pod needs both a GPU and a NIC, you request two separate extended resources. The scheduler might place the pod on a node that has both, but it has no way to ensure those two devices share a NUMA node, PCIe root complex, or any other topological relationship.

- **No topology awareness**: Even for devices of the same type, you can't express preferences about their physical locations. "Give me two GPUs connected by NVLink" or "give me GPUs all in the same NUMA zone" cannot be represented.

- **All-or-nothing allocation**: You either get exactly what you requested or the pod doesn't schedule. There's no way to express "I prefer 4 GPUs but can run with 2" or "try to give me high-memory GPUs, but fall back to standard ones if unavailable."

The device plugin API will not be extended to address these limitations. It's a stable, feature-frozen interface. Dynamic Resource Allocation is the designated successor.

## DRA in One Paragraph

Dynamic Resource Allocation is a Kubernetes API — `resource.k8s.io/v1`, GA since Kubernetes 1.34 — that lets workloads describe what kind of device they need, not just how many. Device drivers publish their devices with typed attributes (model, memory, topology) into cluster-visible objects. Workloads write claims that match against those attributes using CEL expressions. The scheduler evaluates these claims against the inventory, finds matching devices, checks constraints between devices, and makes allocation decisions. This fundamentally shifts device allocation from the kubelet — where each node acts independently — into the scheduler, where a global view of the entire cluster's device inventory enables smarter placement and matching.

## Key Concepts at a Glance

**ResourceSlice**: A driver running on a node publishes its devices and their attributes into ResourceSlice objects. These are cluster-scoped resources that the scheduler reads to understand what hardware is available. Think of a ResourceSlice as the driver's inventory list: "Node X has these devices, each with these attributes."

**DeviceClass**: A cluster-scoped configuration object that names a category of devices (for example, `gpu.nvidia.com`). It identifies which driver manages these devices and can carry default selectors that automatically apply to all claims using this class. DeviceClasses let cluster administrators set policy: "Any claim asking for this class must match these baseline requirements."

**ResourceClaim**: A workload's request for devices. It lives in a namespace, is referenced by pods, and contains one or more device requests. Each request specifies a DeviceClass and can include CEL selectors that filter devices by attribute values. A ResourceClaim represents "I need devices that look like this."

**ResourceClaimTemplate**: A template that creates a new ResourceClaim for each pod that uses it. When multiple pods should share the same devices — for example, a multi-container sidecar pattern — they reference a single ResourceClaim directly. When each pod needs its own isolated set of devices — the more common case — the pod references a ResourceClaimTemplate, and the system creates a claim per pod.

**CEL Selectors**: Expressions written in the Common Expression Language that filter devices based on their attribute values. A selector might express "memory is at least 80 GB and model contains 'H100'." CEL is a simple, safe expression language designed for Kubernetes policy. Selectors are evaluated by the scheduler against each device's published attributes.

**Constraints**: Rules that express relationships between multiple devices within a single claim. The primary constraint type is `matchAttribute`, which requires two devices to share the same value for a specified attribute. For example, a constraint might require that a GPU and a NIC both have the same `pcieRoot` attribute value, ensuring they're topologically co-located.

## What's Next

You now understand the problem DRA solves, why device plugins couldn't address it, and the basic vocabulary of the DRA system. The next document walks through each of these objects in detail with real YAML and CEL expressions. It traces the full allocation lifecycle: how a driver registers devices, how a claim is written and matched, how the scheduler binds devices to a pod, and how those devices reach the container at runtime.

Link: [How DRA Works Under the Hood](02-how-dra-works.md)
