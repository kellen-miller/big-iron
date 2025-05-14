# Open-Source Hyperkernel Project Plan for a Distributed Virtual Machine

## Background and Objectives

Traditional approaches to scaling up computing involve expensive multiprocessor servers or specialized hardware, while
scaling out with clusters forces applications to be distributed-aware. The goal of this project is to **federate
multiple physical machines into one large virtual server** – a single system image that can run an unmodified Linux OS
and applications as if on a giant SMP machine. This concept, inspired by TidalScale’s “software-defined server” and
HyperKernel technology, uses a distributed hypervisor (or **hyperkernel**) to combine processors, RAM, storage, and I/O
of many nodes into one virtual machine. By leveraging modern, memory-safe languages (like Rust, Go, or Zig) for
implementation, we aim to maximize reliability and security. Key requirements include dynamic resource aggregation,
transparent memory coherence, live migration of execution, high fault tolerance, and minimal changes to existing
software.

**Objectives:**

- **Single System Image:** Present a unified Linux machine to the user and OS, hiding the cluster complexity. The OS
  will see one set of resources (CPUs, memory, etc.) and not require any modifications.
- **Aggregate Legacy Hardware:** Harness older, potentially heterogeneous PCs (x86_64 architecture) as building blocks
  for a larger virtual system. This extends the useful life of hardware by pooling their resources.
- **Dynamic Resource Allocation:** Allow processors (vCPUs) and memory pages to migrate or be reassigned across nodes at
  runtime to optimize locality and performance. I/O and storage resources should be accessible from any node as needed.
- **Memory Coherence:** Maintain a **distributed coherent shared memory (DCSM)** across nodes in software, so all RAM is
  accessible with proper synchronization (analogous to a huge NUMA shared memory). Use software-managed address
  translation layers to handle remote memory access transparently.
- **Fault Tolerance:** Achieve high availability via redundancy and fast recovery. The system should tolerate or quickly
  recover from node failures or OS crashes, with mechanisms for live node addition/removal and minimal downtime.
- **Modern Implementation:** Use memory-safe, high-performance languages and open-source tools to build the system. This
  reduces bugs (like buffer overruns) and leverages existing virtualization and networking libraries.
- **Open-Source Roadmap:** Provide a clear development roadmap from prototype to production, encouraging community
  collaboration on this “open hyperkernel” concept.

## System Architecture Overview

([TidalScale Software Defined Server | IIS](https://www.iistech.com/blog/tidalscale-software-defined-server)) *Figure:
Conceptual architecture of a software-defined server cluster. Multiple physical nodes run HyperKernel instances that
combine their CPU (cores) and RAM into one large virtual machine. The unified system runs a single OS and application
stack, with machine-learning–driven self-optimization aligning workloads to resources dynamically (figure based on
TidalScale’s architecture).*

At a high level, the system consists of a cluster of PCs each running a **Hyperkernel Node Module**, all cooperating to
present a single virtual machine. The architecture can be visualized in layers: **hardware nodes at the base, a
distributed hyperkernel layer across all nodes, and a unified guest OS on top**. Key architectural components include:

- **Hyperkernel Node (Type-1 Hypervisor):** A lightweight hypervisor running on each physical node directly on the
  hardware. It virtualizes the local CPU, memory, and I/O into **vCPUs, vMemory pages, and vI/O devices** that can be
  reallocated cluster-wide. Each node’s hyperkernel instance works in concert with others to form one virtual
  motherboard for the guest OS.
- **Distributed Memory Manager:** A software coherence mechanism that treats the sum of all nodes’ RAM as a single
  address space. It maintains a uniform memory abstraction (all of memory appears accessible with equal semantics) to
  the OS. In practice, each node’s memory acts like an additional cache level (L4) for the cluster, caching portions of
  the global memory. This manager handles page fault traps, remote page fetches, replication of read-only pages, and
  invalidation of writes to keep memory consistent across nodes.
- **Virtual CPU Scheduler:** A component that binds the guest’s virtual CPUs to physical CPUs across the cluster. It can
  **migrate vCPU execution contexts** between nodes in milliseconds to move computation closer to data (or vice versa).
  This scheduler monitors where each vCPU (which the guest OS sees as a processor core) is running and can pause,
  transfer, and resume it on another machine by sending its state over the network.
- **Network Fabric:** The interconnect that links all hyperkernel nodes, carrying page data, vCPU states, and
  coordination messages. The design assumes a high-speed Ethernet or similar network (10 Gbps or better) with low
  latency. All node-to-node communication for memory and CPU migration uses this fabric, ideally on a dedicated VLAN or
  physically separate network for performance. Advanced configurations may use RDMA (InfiniBand/RoCE) to accelerate
  memory operations, though the system can function with standard TCP/IP if latency is within tolerable ranges.
- **I/O Aggregation Layer:** A mechanism to virtualize and share I/O devices (disk storage, NICs, etc.) across the
  cluster. It ensures the guest OS can use collective storage and network interfaces as if they were local. For example,
  disks on multiple nodes might be exposed as a single virtual block device (using network RAID or a distributed file
  system), and physical NICs might be bridged to appear as one virtual NIC to the OS. The hyperkernel can forward I/O
  requests over the cluster network to whichever node owns a particular device, or even migrate virtual I/O controllers
  between nodes similar to vCPUs if needed.

These components together allow the cluster to behave as one system. **Unmodified operating systems perceive a large SMP
machine** with many cores and a huge memory pool, while the hyperkernel layer transparently handles the distribution and
coherence. The following sections detail each aspect of the design.

## Hyperkernel Node Design and Operation

Each physical node runs an instance of the hyperkernel, which is a specialized virtualization layer providing both local
control and cluster-level coordination. Key design points for the hyperkernel node module include:

- **Type-1 Hypervisor:** The hyperkernel runs directly on hardware (replacing the host OS) to have full control over CPU
  and memory. It boots on each node and then links with other nodes’ hyperkernels to form the distributed virtual
  machine. This avoids the overhead of a host OS and allows low-level management of traps, page tables, and device DMA.
  Minimal services (for bootstrapping or management) could run alongside, but the hyperkernel is primarily in charge.
- **Guest OS Boot Strapping:** One node (e.g., the cluster leader) will start the process of booting the unified guest
  OS. It will create the initial virtual machine context (vCPUs, memory map) and load the OS kernel. During boot, the
  hyperkernel presents the illusion of contiguous memory spanning all nodes and a CPU count equal to the sum of cores.
  The other hyperkernel instances contribute their resources to this VM at boot time. All nodes’ hyperkernels
  synchronize to agree on the hardware configuration visible to the guest (e.g., ACPI tables describing CPUs and
  memory). From the guest’s perspective, it’s booting on a single machine with many CPUs and large RAM.
- **CPU Virtualization & VM Exits:** The hyperkernel leverages hardware virtualization extensions (Intel VT-x/VT-d or
  AMD-V) so that the guest OS runs in a virtualized context on each physical CPU core. Privileged operations or accesses
  that need handling (like interrupts, or memory accesses outside the mapped range) cause a VM exit trap to the
  hyperkernel. For instance, if a vCPU running on Node A accesses a memory page currently on Node B, the hardware will
  trap (exit) because that guest physical address is not presently mapped on Node A. The hyperkernel intercepts such
  traps to orchestrate the necessary data movement or context switch.
- **Memory Address Translation:** Each node’s hyperkernel maintains extended page tables (EPT) or second-level address
  translation for the guest’s memory. **Locally present pages** are mapped normally to physical DRAM, so memory access
  hits are handled at full speed. If the guest tries to access a page that is not in local memory, the missing
  translation triggers a page fault VM exit. The hyperkernel then consults the distributed memory manager to locate that
  page. If it finds that the page resides on another node, it has options: (a) **Remote Fetch** – treat it like a cache
  miss and fetch the page data into local memory (and update the page table), or (b) **vCPU Migration** – suspend the
  vCPU and resume it on the node where the page resides, so the page can be accessed locally there. This decision is
  based on a cost model (considering network cost, access patterns, etc.).
- **Lightweight vCPU State Transfer:** Migrating a running vCPU between nodes involves transferring its CPU state (
  registers, program counter, etc.) and rebuilding its context on the target node’s hardware. The hyperkernel uses
  optimized mechanisms provided by virtualization extensions to save and load CPU state. As noted in TidalScale’s
  design, the state size can be on the order of only a few kilobytes (≈6.4 KB), making it feasible to send in a single
  network message. The origin node’s hyperkernel captures the vCPU state, sends it over the cluster fabric, and the
  destination hyperkernel injects that state into a halted local CPU to resume the thread. The guest OS is unaware this
  migration happened; from its perspective, the thread simply took slightly longer for that memory access. If subsequent
  accesses hit other remote pages, the process may repeat, but the system tries to minimize migrations by intelligently
  clustering a vCPU with the pages it needs next.
- **Memory Coherence Handling:** For memory reads, the system can replicate pages across nodes safely (multiple nodes
  can cache a copy of a read-only page). On a write, however, the hyperkernel must ensure only one up-to-date copy
  exists. If a vCPU attempts to write to a page that has been copied elsewhere, the hyperkernel will invalidate those
  other copies (remote entries) before allowing the write (similar to a distributed MESI protocol). This might involve
  sending invalidation messages to other hyperkernels that hold the page, ensuring coherence. By handling this at the
  hypervisor layer, we **guarantee coherent shared memory** across the cluster with no changes to the guest OS’s memory
  model.
- **I/O and Interrupt Virtualization:** Each hyperkernel instance may virtualize I/O devices present on that physical
  node and present them to the guest as part of a unified I/O subsystem. For example, if one node has a GPU or a disk,
  the hyperkernel can expose it to the guest (possibly via a virtual PCI device). Accesses to that device from the guest
  will be trapped and forwarded to the physical device’s node. This might use a **virtio**-style framework over the
  network (e.g., a block device request is sent to the node that has the disk, which then performs the IO and returns
  the result). Additionally, device interrupts must be routed to the guest no matter where they originate. The
  hyperkernels coordinate to forward interrupts from physical devices to the guest OS (which sees virtual interrupts).
  In some cases, **I/O context migration** could occur: for instance, a virtual NIC could potentially float between
  nodes, handing off responsibilities, though initially a simpler static assignment (each device owned by one node’s
  hyperkernel) will be implemented. All of this is transparent to the OS, which thinks it’s using normal devices.

In summary, the hyperkernel on each node acts like a part of a distributed VMM (Virtual Machine Monitor). **Everything
physical is virtualized and made mobile** – CPU, memory, and I/O states can all “flit” between machines under
hyperkernel control. The nodes collaborate peer-to-peer; there is no single point of control after initialization, which
avoids bottlenecks. The design effectively **inverts traditional virtualization**: instead of splitting one machine into
many VMs, we’re merging many machines into one VM.

## Distributed Memory Model and Coherent Shared Memory

A core challenge is presenting a **single, coherent memory space** across multiple computers. Our architecture uses a
software-managed distributed shared memory approach, deeply integrated with the virtualization layer:

- **Global Physical Address Space:** The guest OS is given a contiguous physical memory space (guest “physical”
  addresses) that actually spans all nodes. For example, if Node1 has 16GB and Node2 has 16GB, the guest might see 32GB
  total. The hyperkernel on each node knows which portions of this address space it physically hosts. One strategy is to
  partition the guest physical memory range among nodes (e.g., Node1 hosts addresses 0–16GB, Node2 hosts 16–32GB).
  Alternatively, memory could be interleaved or dynamically assigned. A fixed partitioning simplifies lookup of a page’s
  home node, but dynamic assignment allows moving pages between nodes for load balancing. The plan is to start with a
  static partition (each page has a home node) and later allow migration of pages for performance tuning.
- **Cache and L4 Memory Concept:** The system treats each node’s memory as an **L4 cache** for the cluster’s memory. In
  hardware, L1–L3 caches are on the CPU; here L4 is effectively “memory elsewhere in cluster”. When a vCPU on a node
  accesses memory, if that memory is local (in L1–L3 or local DRAM), it’s fast. If not local, it’s like a cache miss –
  the hyperkernel must retrieve it from remote memory. Because even a 10 GbE network is orders of magnitude slower than
  local RAM, we mitigate the cost by moving the *compute* to the data whenever feasible (vCPU migration). This way, a
  workload that touches a large chunk of data on Node2 will ideally execute on Node2’s CPUs, avoiding constant network
  transfers. In effect, data and computation are dynamically reunited to minimize expensive remote accesses.
- **Page Fault Handling:** On a miss (remote access), the default action might be to fetch the page over the network
  into the requesting node’s memory, as in traditional distributed shared memory systems. The hyperkernel sends a
  request to the remote node that has the page, obtains the data, maps it into the local EPT, and resumes the vCPU. This
  is appropriate for read-heavy pages that may be accessed repeatedly (caching them locally). The challenge is if a page
  is heavily written by multiple nodes – in such cases, bouncing it around is expensive. That’s where migrating the vCPU
  to the page’s node can be a win. The hyperkernel employs a **cost heuristic**: if the page access is isolated or
  read-mostly, copy the page; if the page is part of a locality set of many nearby addresses or likely to be repeatedly
  modified, move the computation to the page’s location. These decisions can even be refined by runtime statistics or
  machine learning (discussed in Performance section).
- **Memory Consistency Protocol:** To keep memory coherent, we implement a software protocol analogous to CPU cache
  coherence (MESI/MOESI). Initially, each memory page has a single canonical copy at its home node. Other nodes can
  cache a copy for read-only purposes; we mark those copies as read-only in their page tables. If a vCPU on another node
  writes to the page, the hyperkernel invalidates or updates other copies:
    - For strict consistency, we use an **invalidate-on-write** protocol: before a write is allowed, any other cached
      copies are invalidated (their entries removed or marked invalid) and future accesses by other nodes will trap. The
      writing node becomes the sole owner until it eventually evicts or until another node demands it.
    - Optionally, a more advanced protocol could allow **migrate-on-write**: move the page’s home to the writer’s node (
      especially if that node will likely become the primary user). This essentially transfers ownership.  
      In either case, the system ensures that at most one node can modify a page at a time, preserving coherence. These
      operations happen transparently during the guest’s execution.
- **Software TLB and Address Translation:** Each hyperkernel maintains knowledge of where each guest page resides. This
  can be done via a distributed hash table keyed by page number or by a master directory. A straightforward approach is
  for each page’s home node to act as the directory for that page, tracking which nodes have a cached copy. When a node
  needs a page, it queries the home node (or a global service) to get the current owner and any sharers, then proceeds
  to fetch or migrate appropriately. This directory could itself be partitioned by address ranges to avoid a single
  bottleneck. The hyperkernels exchange messages to update these records on page migrations or copy events.
- **NUMA and Heterogeneity Handling:** The unified memory is presented to the OS as uniform, but under the hood, access
  times vary. The hyperkernel can optionally expose the cluster as a NUMA system to the OS (e.g., each physical node as
  a NUMA node) if the OS can make use of that information for scheduling. However, TidalScale found it effective to hide
  NUMA and maintain the illusion of uniform memory. We may start by hiding the NUMA and relying on our own placement
  optimization. Regarding heterogeneous nodes (different memory sizes or speeds), the hyperkernel can weight decisions
  based on node capabilities. If one machine has slower RAM or interconnect, the system might prefer to keep hot pages
  on a faster node. The memory allocation algorithm on boot can distribute pages proportionally to capacities, and
  during runtime, pages might be migrated away from an overloaded or slower node to maintain performance.
- **Memory Over-commitment and Swap:** In later phases, we might allow the cluster to **over-commit memory** (i.e., the
  sum of guest memory exceeds physical total by using disk swap across nodes). Initially, we will avoid this for
  simplicity and performance. Eventually integrating a distributed swap mechanism or using a node’s SSD as a
  cluster-wide swap space (accessible by others) could provide an extra safety net if memory is exhausted.

By managing memory at this granular page level, the system can preserve coherence and give the **single system image**
illusion even across a network. The design leverages the fact that modern workloads exhibit locality (per Denning’s
working set principle), so not all pages are heavily shared across nodes at the same time. Our aim is to have most
memory accesses served at local (or effectively local) speed, and only incur network penalties infrequently. In
practice, as long as working sets can be localized and writes aren’t contended across nodes, the performance can
approach that of a real SMP machine.

## Network Fabric and Communication Layer

The performance and reliability of the cluster network are critical, as this architecture transforms network operations
into memory and CPU operations. Key considerations for the network fabric include:

- **Topology and Bandwidth:** A high-bandwidth, low-latency network is recommended (e.g., 10 GbE, 40 GbE, or better). In
  lab environments, even **commodity 10 Gb/sec Ethernet** has been shown to be sufficient when combined with intelligent
  placement algorithms. All nodes should be interconnected through a switch or direct links such that the latency
  between any two is as low as possible (a few tens of microseconds ideally). For our open-source project targeting
  older PCs, Gigabit Ethernet (1 GbE) is a more common baseline – it will work functionally but will limit performance.
  Therefore, for serious use-cases, upgrading old machines with inexpensive 10 GbE NICs or using technologies like
  bonding multiple 1Gb links is advisable.
- **Private Network / VLAN:** The cluster will use a dedicated network (or VLAN) for hyperkernel communication. This
  isolates the traffic from general LAN noise and improves security. The hyperkernels set up a private overlay network
  for all coordination messages, memory transfers, and state migration. We can allocate a private IP range or use MAC
  addresses known to cluster nodes only. Optionally, a secondary network could be used for redundancy (if the primary
  fails or is saturated).
- **Protocol:** Communication can use UDP or TCP sockets, or even raw Ethernet frames for lower overhead. TidalScale’s
  approach used raw Ethernet frames with a simple protocol on top – we could adopt a similar strategy. For reliability,
  a lightweight acknowledgement or re-transmit scheme will be needed (or just use TCP initially for simplicity, then
  optimize later). If using standard TCP/UDP, we might leverage existing libraries (e.g., ZeroMQ or nanomsg for
  messaging, RDMA libraries for direct memory access). The latency is key: we want the overhead of moving a vCPU or
  fetching a page to be minimal compared to the cost of using a local resource. At 10 GbE, a 6 KB vCPU state or a 4 KB
  memory page can be transferred in much less than a millisecond, so the network latency is the dominant factor rather
  than throughput for single operations.
- **Message Types:** The system will have a defined set of network messages, such as:
    - *Page Request/Reply:* Node A asks Node B for a page (with an identifier for the page number). B replies with the 4
      KB data (or a refusal if it has been invalidated and B doesn’t actually have it anymore, in which case a directory
      lookup is needed).
    - *Invalidate:* Sent by a node that is about to write to a page, informing others to drop or mark their copy
      invalid. Could be a multicast to all sharers of that page.
    - *vCPU Migration Request:* A node tells another to take over a vCPU execution, including the state. This might be
      paired with a state transfer message or the state could be pulled by the target.
    - *Join/Leave:* When a node is added or removed, a broadcast is made to update cluster membership.
    - *Heartbeat/Health:* Periodic pings or health info to ensure all nodes are responsive (vital for fault detection).
    - *Synchronization:* Occasional messages to sync global state (like time sync, though we can let the guest handle
      clock sync via NTP if needed). Also, a barrier or consensus when performing cluster-wide actions (like quiescing
      the VM to checkpoint).

  We will design a custom binary protocol for these to minimize overhead. Using a memory-safe language means we can
  safely decode/encode these messages without buffer errors. If performance tests show overhead, we can move to raw
  Ethernet frames and a custom handler (using something like DPDK for user-space packet processing).
- **Orchestration Channel:** Separate from the data plane, there may be a control/orchestration service (possibly
  running in user-space on one node or an external node) to manage higher-level commands (like instructing to add a
  node, initiating shutdown, etc.). This could use a REST API or gRPC for convenience, but those commands are not in the
  hot path of memory/CPU operations.
- **Security on the Network:** The cluster network should be secured to prevent unauthorized access. At minimum, nodes
  can authenticate each other on join (using a shared key or certificate). If running on an untrusted network, we could
  employ encryption (IPsec or TLS for the control messages). The memory and vCPU transfers ideally happen in a closed
  environment, but adding encryption is possible at some cost to latency. This is discussed more in the Security
  section.

By optimizing the communication patterns (e.g., bundling multiple page transfers in one packet if sequential pages are
needed, or using jumbo frames), we aim to make remote memory access as efficient as possible. **Empirical data suggests
that with smart vCPU placement, the network is hardly taxed at all, even at 10Gbps, because most memory accesses become
local after optimization.** In effect, the network serves as a "memory backplane" for cache misses. Ensuring the network
fabric is robust and low-latency will directly improve the system’s performance and responsiveness.

## Orchestration and Cluster Management

Managing multiple nodes as one requires orchestration beyond the low-level hyperkernel operations. We design a
management layer that handles configuration, node membership, and lifecycle events:

- **Cluster Initialization:** A primary node (e.g., the one with the boot disk or chosen by the user) will initiate the
  cluster. On startup, each hyperkernel will announce itself or respond to a discovery mechanism. This could be done via
  a simple broadcast where one node is configured as the master of a particular VM instance. That master coordinates
  collecting resource information from all nodes (CPU counts, memory sizes, device list) and composes the unified
  virtual hardware that will be exposed to the OS. Once the configuration is set, the master triggers the boot of the
  guest OS on the cluster. After boot, the “master” role is less important, and any coordination needed (like directory
  of pages or deciding migrations) is done collectively or through smaller leader election for specific tasks.
- **Node Addition (Scale-Out):** To add a node to a running system (for example, to increase available RAM or replace a
  failing unit), the orchestration service will put the system into a state that can integrate the new resource.
  Ideally, Linux supports physical hot-plug of CPUs and memory. We can leverage **CPU and memory hotplug** features: the
  hyperkernel can present the new node’s CPUs as new CPU ids to the OS (on Linux, offline CPUs can be brought online)
  and memory as a hot-added NUMA region. The orchestration would coordinate this: notify the guest OS (via ACPI or a
  custom driver) that new resources are available. Another approach is to treat the new node as initially a spare and
  gradually start scheduling work there (without necessarily telling the OS it’s a new NUMA node, which might complicate
  things). This area will require careful handling to ensure the OS recognizes new capacity. In early stages, it may be
  acceptable to only add nodes when the VM is not running or at boot time; dynamic addition can come later.
- **Node Removal (Scale-In or Failure):** Removing a node (either deliberately for maintenance or due to an outage) is
  challenging because it may be hosting critical data or threads. A controlled removal (maintenance mode) would involve
  first migrating all vCPUs off that node and copying any pages that node exclusively has to others (essentially
  evacuating it). Once the node’s contributions are migrated, the hyperkernels can update the cluster view so that node
  is no longer used. The guest OS can be informed to offline those CPUs and memory (as if a NUMA node was removed, which
  some OSes support in a limited fashion). In case of an unexpected failure (node crash or network loss), the
  orchestration layer needs to detect it (via missed heartbeats) and initiate recovery: possibly pause the entire VM (
  stop all vCPUs cluster-wide) to prevent further damage, then attempt to reconstruct lost pages from a replica or from
  disk (if we had checkpointed). If a page was only on the failed node and not replicated, the guest OS process that
  needed it may be unrecoverable – this scenario might lead to a crash in the guest. Our plan will incorporate **fault
  tolerance strategies** (next section) to minimize this risk. After isolating the failed node, the remaining nodes can
  resume the guest, minus the resources of that node. This is akin to a degraded mode operation.
- **Resource Scheduling & Load Balancing:** At a higher level, the system can have policies for how to allocate
  workloads across nodes. For example, on a multi-tenant cluster (if we ever allow multiple VMs), the orchestrator could
  place different VMs on different sets of nodes. But in our single-VM scenario, the concern is more about balancing
  usage: if one node is much slower or has less capacity, the system may want to preferentially use other nodes to avoid
  bottlenecks. The orchestrator can monitor metrics (CPU utilization, memory pressure, network usage) from each
  hyperkernel (they can report stats) and adjust the strategy (for example, trigger more aggressive migration away from
  an overloaded node).
- **Configuration Management:** An open-source project should allow flexibility in configuration. A config file or
  interface will let the user specify which nodes participate, their roles, and any special parameters (like network
  addresses, using RDMA or not, etc.). The orchestrator reads this and sets up the environment accordingly. We can use
  existing config management or even a simple etcd cluster to store the cluster state (node list, etc.).
- **Monitoring and Control Interface:** Provide a dashboard or CLI tools to observe the system. For example, an admin
  might query the system for the distribution of pages, the current migration decisions, or the health of each node.
  This can be facilitated by a small web service running as part of the management layer, or integration with tools like
  Prometheus for metrics. Having visibility is important for tuning and trust in the system.
- **Cluster Metadata & Consensus:** Some information (like who is in the cluster, who is primary) requires consensus. We
  can use a lightweight consensus algorithm (such as Raft via an etcd instance or a custom simple election) to avoid
  split-brain scenarios. This ensures that at any time, nodes agree on the composition of the cluster and what the
  global system state is. For example, only one node should be initializing the OS boot; that “leader” is chosen at
  start or via config.

In summary, the orchestration layer ties together the hyperkernels into a coherent whole and provides the tools to
manage the cluster lifecycle. This is the part of the system that will deliver on the promise of elasticity (
adding/removing nodes on the fly) and user-friendly operation (one command to start a cluster VM, etc.). Initially, a
rudimentary orchestrator (maybe a script or manual commands) will suffice to launch the prototype, but as we progress,
this will evolve into a robust management component.

## Performance Optimization Strategies (Locality & Learning)

While the basic design will functionally create a unified VM, performance optimization is crucial to make it efficient.
The system should actively optimize where code runs and where data resides:

- **Dynamic Locality Optimization:** As described earlier, the hyperkernel decides whether to move data or execution
  based on access patterns. We will implement a **cost function** that evaluates likely future accesses. For example, if
  a vCPU is sequentially scanning a large array that spans many pages, it might be better to migrate that vCPU to the
  node owning each next chunk as it goes (if the chunks are huge). Conversely, if multiple vCPUs are all reading the
  same large dataset, replicating that dataset across nodes (caching it) is beneficial. These decisions can be
  hard-coded heuristics initially (e.g., a threshold on how many times a page is accessed remotely before migrating the
  thread vs copying the page).
- **Machine Learning Approach:** Taking inspiration from TidalScale, we plan to incorporate a machine learning component
  to improve these decisions over time. The hyperkernel can collect statistics such as: how often each vCPU had to stall
  for remote memory, how often migrations happened, and the access pattern of each vCPU (like its working set size,
  temporal locality). Using this data, a learning algorithm (which could be as simple as adaptive feedback or as complex
  as a reinforcement learning agent) adjusts the strategy. For example, the system can learn that a certain
  application (like a database) has a pattern of accessing a set of pages repeatedly, so it proactively groups those
  pages on one node or pre-copies them when possible. TidalScale’s team mentions using machine learning to continuously
  enhance CPU and memory placement, watching what each vCPU saw recently and predicting what it will need next. We can
  start with straightforward adaptive algorithms (like LRU for page caching, or reacting to frequent faults by
  migration) and evolve towards more sophisticated predictive models (perhaps using online training to recognize
  patterns).
- **Real-Time Monitoring and Feedback:** Each hyperkernel instance can locally monitor the “stall” time a vCPU
  experiences waiting for data, and how often it causes network traffic. These metrics are shared among peers. If one
  node notices it’s becoming a hotspot (many others requesting pages from it), it could offload some pages or
  temporarily push execution to itself to serve those requests in a batch. This collective introspection means each node
  not only optimizes locally but also contributes to a **global optimization**. The design will include a background
  thread on each node that exchanges summary stats and suggests adjustments (like “Node2 to Node3: I’m seeing you
  request a lot from me, perhaps I should migrate process X to you” or vice versa).
- **Resource Alignment:** Another angle of optimization is aligning CPU and memory resources to the workload’s needs. If
  the guest OS spawns many threads but only actively uses a subset at a time, the hyperkernel might consolidate active
  vCPUs on fewer nodes (to reduce cross-traffic) and put other nodes in a standby or low-power state until needed.
  Conversely, if the workload is truly parallel and touches disjoint data that nicely maps to different nodes, the
  system should avoid unnecessary moves and let each node handle its portion. This essentially becomes an automatic NUMA
  placement problem: we want to schedule threads on nodes where their memory is, similar to how an OS would schedule on
  the correct NUMA node in a big machine. Here, because memory location can change, it’s a two-way street – we can move
  the thread to memory or memory to thread. Finding the optimal balance is complex, which is why adaptive algorithms and
  potentially ML are valuable.
- **Benchmarking and Tuning:** As part of the project plan, we will include extensive benchmarking with various
  workloads (e.g., in-memory databases, analytics, HPC simulations, etc.) to identify performance bottlenecks. These
  tests will guide tuning of our cost models. For instance, if we find the network latency is the biggest culprit, we
  may bias more towards migrating vCPUs (so that after one-time cost, the next many accesses are local). If we find
  certain access patterns cause thrashing (like ping-ponging a page between two nodes), we might implement a rule to
  temporarily replicate and diverge that page (if possible) or to serialize that access through one node.
- **Use of Modern Hardware Features:** We will also consider leveraging any modern features in the hardware to improve
  performance. For example, if available, **RDMA** could allow one node to read/write another’s memory without involving
  the remote CPU, cutting down software overhead for page copies. Also, technologies like Intel ADQ (Application Device
  Queues) might help prioritize our traffic on a NIC. If some nodes have Non-Volatile Memory (NVDIMM or Optane) that can
  act as an extension of RAM, perhaps that could be a global swap space that all nodes use commonly, thereby reducing
  the worst-case delay for missing pages.
- **Scalability Limits:** Part of optimizing is recognizing limits. Initially, we might support a cluster of, say, up to
  8 nodes. As we optimize, we want to ensure the design scales further (16, 32 nodes). The concern is increased network
  chatter or directory management overhead. We plan to test scaling and possibly adjust the architecture (for example,
  if broadcasting invalidations to 31 other nodes is too slow, we might introduce a hierarchical cluster arrangement or
  limit the fan-out by grouping nodes). Our design can incorporate hierarchy: e.g., treat subsets of nodes as tighter
  coherence domains and only have occasional exchanges between groups. This is analogous to multi-level NUMA in very
  large machines.

Ultimately, **the aim is to get as close as possible to bare-metal performance of a single machine** by smartly
amortizing the cost of distributed operations. By continuously learning and adapting, the system can handle a wide range
of workloads effectively – many applications could run unmodified and see near-native performance on this distributed
virtual
machine ([](https://www.ssrc.us/media/pubs/df492b91a126802e0199dfe6ef3484a4c3f08171.pdf#:~:text=Each%20hyperkernel%20instance%20main%02tains%20a,to%20quickly%20adapt%20to%20the)).
Performance tuning will be an ongoing focus through the roadmap milestones, with clear metrics (like how much slower vs
real hardware for a given app) guiding improvements.

## Security Considerations

Security is crucial in a system that spans multiple machines, especially using memory sharing and network communication.
We address security at multiple levels:

- **Memory Safety in Implementation:** By using languages like **Rust or Zig for the hyperkernel development**, we
  inherently reduce vulnerabilities such as buffer overflows, use-after-free, and null pointer dereferences in the
  hypervisor code. A memory-safe implementation means the core that handles guest memory and cross-node messaging is
  less likely to have exploitable
  bugs ([Rust device backends for every hypervisor | Blog - Linaro](https://www.linaro.org/blog/rust-device-backends-for-every-hypervisor/#:~:text=Rust%20device%20backends%20for%20every,crosvm)) ([Exploring the Rust-VMM Ecosystem – Building Blocks for Custom ...](https://www.linkedin.com/pulse/beyond-containers-exploring-microvm-revolution-part-5-moon-hee-lee-mrrfc#:~:text=Exploring%20the%20Rust,Performance%20close)).
  Rust, for instance, can ensure that pointer arithmetic and memory accesses are checked at compile time, and its borrow
  checker prevents many race conditions. This is especially important because a bug in the hyperkernel could compromise
  the entire system. (Low-level unsafe blocks might still be needed for hardware access, but they will be minimal and
  heavily reviewed).
- **Isolation of Guest OS:** Although the cluster appears as one machine to the guest, each node’s hyperkernel will
  enforce strict isolation of the guest context. The guest OS will not be allowed to directly execute privileged
  instructions except through the controlled virtualized interface. This is similar to any hypervisor security – the
  guest is “jailed” in a VM environment. Even though our VM spans nodes, we must ensure that a malicious or compromised
  guest can’t break out to the hyperkernel or host hardware on any node. Techniques like extended page table (EPT)
  protection, CPU rings, and not exposing unnecessary hypercalls will be applied. We will conduct security audits and
  possibly formal verification on critical sections (e.g., the page fault handler) to ensure they handle all cases
  safely.
- **Inter-Node Communication Security:** Within the cluster, hyperkernels communicate presumably in a trusted
  environment. However, if the cluster network could be sniffed or altered (say the nodes are connected over a broader
  network), we need to secure the channel. We will authenticate all cluster messages – e.g., include an HMAC or digital
  signature with a cluster-wide shared secret for each message to prevent tampering or replay. We may also encrypt the
  traffic using lightweight symmetric encryption (AES-GCM, for example) if confidentiality is a concern (for instance,
  if sensitive data in memory pages might be exposed on the wire). The encryption keys can be established at cluster
  startup via a key exchange. This ensures that even if someone taps the network, they cannot interpret memory contents
  or inject fake page data.
- **Node Authentication and Trust:** When adding a node to the cluster, there should be a verification step. This could
  involve a pre-shared cluster key or certificate that each hyperkernel instance has. A node should prove its identity (
  and that it’s running an untampered hyperkernel version) before being trusted with memory from others. We can explore
  using TPMs (Trusted Platform Modules) or secure boot measurements: each hyperkernel could attest its integrity to a
  master node. This prevents a scenario where an attacker introduces a rogue machine into the cluster to siphon data.
- **Failure Containment:** If one node is compromised or malfunctions in a way that violates protocol (e.g., it’s not
  responding or sending corrupt data), the others should detect this (via timeouts or bad checksum/HMAC detections) and
  isolate that node. The system might then either attempt recovery without it or shut down gracefully if it can’t
  continue. Containment means not allowing a single faulty node to bring down the whole system or corrupt global state
  arbitrarily. For example, if a node starts sending inconsistent page data, we could compare with a secondary copy (if
  available) or at least flag an error and stop using that node.
- **Guest Security Perspective:** From the guest OS’s point of view, running on a distributed hyperkernel should be
  nearly identical to running on a normal machine, so normal OS security practices (firewalls, user isolation, etc.)
  remain in effect. One difference is that the timing and ordering of memory accesses might be slightly different (due
  to network latency). We need to ensure this doesn’t break assumptions or introduce new side channels. For instance,
  could an attacker on one VM thread infer something by measuring delays that occur due to remote memory fetches? Such
  side-channel considerations are quite advanced, but we mention them for completeness. If multiple user-level processes
  are running on the guest, the usual OS-managed isolation is in place; our hyperkernel will not let one process read
  memory of another except via normal OS mechanisms.
- **Secure Updates and Management:** The project will also include secure methods to update the hyperkernel on each
  node (since it’s essentially the OS of each physical machine). We might use digitally signed updates and a controlled
  rollout process (update one node at a time while VM is down or in maintenance) to patch any vulnerabilities. The
  management interface itself (APIs or CLI) should be access-controlled (only cluster admins can issue commands).
- **Minimal Attack Surface:** We will keep the hyperkernel lean – no extraneous services running on the nodes outside of
  what’s necessary for virtualization. Each extra service (like an SSH server or a debug interface) could be an entry
  point for attackers, so ideally the hyperkernel OS is single-purpose. Management actions could be done from a separate
  management daemon possibly on one node or an external host that communicates with hyperkernels through a secure
  channel. By minimizing what's running, we reduce the attack surface on each physical node.

In an open-source context, transparency of code will help with security auditing. Community contributions can be vetted,
and over time we can even pursue a security certification of the hyperkernel. **Using memory-safe languages and strong
cryptographic practices for node communication gives a solid foundation for a secure system
** ([Rust device backends for every hypervisor | Blog - Linaro](https://www.linaro.org/blog/rust-device-backends-for-every-hypervisor/#:~:text=Rust%20device%20backends%20for%20every,crosvm)) ([Exploring the Rust-VMM Ecosystem – Building Blocks for Custom ...](https://www.linkedin.com/pulse/beyond-containers-exploring-microvm-revolution-part-5-moon-hee-lee-mrrfc#:~:text=Exploring%20the%20Rust,Performance%20close)).
We acknowledge that a distributed system has more complexity (hence more potential vulnerabilities) than a
single-machine VM, but careful design and proactive security measures can mitigate these risks.

## Fault Tolerance and High Availability

High fault tolerance means the system continues to operate (or recovers quickly) even if some components fail. This is
particularly challenging here, because losing a node could mean losing part of the "memory" or "CPU" of the one big VM.
Our strategy for fault tolerance includes:

- **Proactive Failure Mitigation:** Similar to TidalScale’s *TidalGuard* feature, we will integrate monitoring to
  predict failures and act **before** they happen. This involves monitoring hardware signals: e.g., CPU temperature, ECC
  memory error counts, SMART data on disks, network link flapping, etc. If a node shows signs of instability (e.g.,
  corrected memory errors climbing or it stops responding to health pings), the orchestrator can initiate a controlled
  removal of that node (migrate its responsibilities away) *prior* to it failing completely. Many hardware failures can
  be anticipated (90% according to TidalScale) and avoided by such proactive moves. This effectively adds extra “nines”
  of uptime by reducing unexpected crashes.
- **Memory Redundancy (Mirroring):** For critical memory pages (for example, kernel code or crucial data structures of
  the OS), we can maintain redundant copies on multiple nodes. This is analogous to RAID for memory. Perhaps the system
  could replicate every page on two distinct nodes (either always, or selectively for important ones). If one node dies,
  the memory pages it held might also exist on a backup node. Then the remaining nodes could recover the VM state.
  However, mirroring everything has a high cost (essentially halves memory capacity and doubles network traffic on
  writes). A more targeted approach is to let the user or system tag certain portions of memory or certain processes as
  high criticality, and only mirror those. Alternatively, a future optimization is to use an erasure coding scheme
  across nodes’ memory to tolerate failure of one node with less than 100% overhead. In initial phases, full mirroring
  of all memory is an option for smaller clusters where losing capacity is acceptable for the benefit of resilience.
- **Checkpointing and Recovery:** Another safety net is periodic checkpointing of the entire VM state to durable
  storage. For example, every N minutes, the system could quiesce (pause the VM very briefly) and have each node write
  out the pages it currently hosts, plus CPU states, to a shared disk or to each other (so that each node has a copy of
  others’ data). This would allow a *restart* of the VM from a recent state if a catastrophic failure happens. The
  downtime in such a recovery might be on the order of the checkpoint interval plus reboot time – not ideal for truly
  seamless HA, but it prevents total loss. Over time, we could refine this into more continuous logging (like record
  each dirty page or each migration in a log) to have a journal that can replay the VM state on other nodes almost
  up-to-last-moment (similar to how some fault tolerant systems log non-deterministic events).
- **Fast Failover of vCPUs:** If a node fails, any vCPUs running there obviously stop. The orchestrator can detect the
  failure (missed heartbeats) and should immediately signal a stop to all other vCPUs (to put the whole VM on hold).
  Then, for each vCPU that was on the failed node, we check if its last state was known elsewhere (perhaps we were in
  the middle of migrating it or it had recently moved). It’s tricky because if the node died, we lose whatever state was
  in its CPU registers at that moment. If we have no copy, that thread is lost – which could bring down the OS if it was
  e.g. a kernel thread. To handle this better, we might run certain critical vCPUs in lockstep on two nodes. An extreme
  approach is akin to VMware FT: run two copies of the VM (or at least two copies of each vCPU) on different nodes,
  keeping them in sync by executing the same instructions. This doubles CPU usage but one can take over instantly if the
  other fails. However, doing that for many cores is probably impractical. Instead, perhaps do it for just one core that
  runs the OS kernel’s most critical tasks (like the bootstrap processor). Alternatively, rely on the OS’s ability to
  survive losing a CPU (some OSes can if it wasn’t the only one). We should test scenarios of losing a CPU and see if
  Linux can continue; if not, ensure that CPU has a backup.
- **Distributed I/O and Storage Redundancy:** If the cluster’s storage is distributed, we must ensure that a disk
  failure or node with a disk failure doesn’t corrupt the guest’s filesystem. Using a network RAID (say RAID-1 or 5
  across disks in different nodes) or a distributed file system (like Ceph, GlusterFS) for the guest OS volumes would
  add fault tolerance. That way, any disk I/O has redundancy. For networking, if one node had the only physical NIC and
  it failed, the cluster would lose external connectivity. To avoid that, we should allow multiple physical NICs (across
  nodes) to be bonded or one acting as backup so the guest network traffic can failover to another path. The hyperkernel
  could present a virtual NIC that is actually backed by multiple physical NICs on different nodes for redundancy.
- **Hot Swap and Repair:** The design should allow an operator to remove a bad node and later reintroduce a repaired
  node (or a new node) to take its place, all without shutting down the guest OS completely. The orchestration aspect
  already covers adding/removing nodes; here we emphasize doing it in response to failures. Ideally the guest OS
  continues running throughout, perhaps with a momentary pause. In cases where the failure impact was too severe (e.g.,
  multiple nodes failing at once or a failure in the middle of a critical un-mirrored memory update), a short downtime
  might be unavoidable – but the goal is to recover automatically when possible.
- **Testing Fault Scenarios:** As part of development, we will simulate failures (power off a node abruptly, disconnect
  network, corrupt memory artificially) to test the system’s resilience. This will help refine the strategies above. We
  also plan to incorporate the ability to snapshot the running state so that if something goes wrong, we can debug by
  examining the distributed state post-mortem (a sort of core dump of the cluster VM).

While achieving **zero-downtime continuity** in the face of a node failure is extremely challenging, our multi-pronged
approach (proactive mitigation, redundancy, and fast failover) aims to get as close as possible. The expectation is that
common hardware failures can be handled by replacing the component with no OS interruption (just as TidalScale claims to
add two extra nines of uptime by avoiding 90% of failures). For catastrophic events, reducing recovery time and avoiding
data loss is the priority.

## Implementation Tools and Open-Source Components

Building this system from scratch is a huge effort, but we can leverage existing open-source technologies and libraries
to accelerate development and ensure robustness. Below is a breakdown of suggested tools and libraries for various parts
of the implementation:

- **Virtualization Layer:** We will use the Linux KVM API for low-level virtualization support, or alternatively the
  FreeBSD bhyve codebase, as a starting point. KVM provides a stable interface to create VMs, assign vCPUs, and manage
  memory, which we can use on each node (with our custom logic on top to coordinate between nodes). There is also the *
  *rust-vmm** project (a set of Rust crates for virtualization) which can be extremely useful if we choose Rust. It
  provides components for vCPU management, virtio device emulation, and memory management that we can adapt to a
  distributed
  context (["VMWare rewritten in Rust : r/rust - Reddit](https://www.reddit.com/r/rust/comments/18ekjaf/vmware_rewritten_in_rust/#:~:text=,It%20provides%20a)) ([Rust device backends for every hypervisor | Blog - Linaro](https://www.linaro.org/blog/rust-device-backends-for-every-hypervisor/#:~:text=Rust%20device%20backends%20for%20every,crosvm)).
  Another open-source VMM is **Cloud Hypervisor** (by Intel/Cloud Native Computing Foundation) which is written in Rust;
  its code for launching and managing VMs could be
  instructive ([cloud-hypervisor/cloud-hypervisor - GitHub](https://github.com/cloud-hypervisor/cloud-hypervisor#:~:text=cloud,MSHV)).
  We may end up writing a lot of custom code for the distributed aspect, but these projects provide reference
  implementations of core virtualization tasks.
- **Networking and RPC:** For the cluster interconnect, we can utilize libraries like **DPDK** or **netmap** to achieve
  high throughput user-space packet processing if needed (for example, to send/receive page data with minimal kernel
  overhead). Initially, standard sockets in Rust (with `tokio` for async IO or Go’s net library) are fine for
  simplicity. If using Rust, crates like `smoltcp` (a small TCP/IP stack) could allow a fully in-hypervisor network
  implementation without needing a host network stack. For control plane communication, **gRPC** or **Cap’n Proto**
  could be used for structured messages (like join requests, etc.), but for the data plane (memory and vCPU transfers)
  we will likely use custom binary protocols for efficiency. Also, if reliability is a concern, employing a messaging
  library like ZeroMQ could simplify the pub-sub patterns for state updates or heartbeats.
- **Synchronization and Coordination:** **etcd** (with its Raft consensus) is a candidate to manage cluster state
  consistency (like membership, leader election). We might embed an etcd client or run a small etcd cluster on the
  nodes (possibly on top of the hyperkernel or a companion process in each hyperkernel’s userland). Alternatively, we
  implement a simple consensus with timeouts for leader election since our scenario is relatively straightforward (one
  VM, known nodes). For threading and locks within the hyperkernel, Rust’s `std::sync` or `crossbeam` or Go’s
  channels/goroutines can be used. Care must be taken to avoid deadlocks especially when multiple nodes interact (we may
  need a well-defined lock hierarchy or mostly lock-free design for inter-node ops).
- **Memory Management:** We will likely write custom code for managing the distributed page tables and cache, but can
  draw on algorithms from prior art. Libraries for data structures (like a concurrent hash map to track page locations)
  will help – e.g., Rust’s `dashmap` (thread-safe map) could be useful. If using userfaultfd (Linux user-space page
  fault handling), we can utilize the Linux system call to trap page faults in user space – this is how post-copy live
  migration in QEMU works, and we could adapt it for distributed paging. Essentially, we could run the guest OS inside a
  user-space process and let userfaultfd notify us of missing pages, then fetch from network. This might be a stepping
  stone prototype (with less efficiency) before fully kernel-integrated hyperkernel.
- **Device Emulation:** For virtual devices (especially disk and network for the guest), we can use the **Virtio**
  specification. Rust-VMM has virtio device implementations that we can adapt, or we can integrate with QEMU’s device
  models by modifying QEMU to understand a distributed backend. Another interesting tool is **VFIO** (for direct device
  assignment); if we wanted to assign a physical device in one node to the guest, we could use VFIO on that node and the
  hyperkernel would simply let the guest driver run, but that device wouldn’t be accessible if the vCPU using it isn’t
  on that node. So more likely, we stick to virtio-net, virtio-blk, etc., and implement those such that requests are
  sent over the cluster network to the node that has the physical device. This can be done with a service listening on
  those nodes, or an RPC call via our hyperkernel messaging.
- **Programming Languages:** We have a strong preference for **Rust** for the hyperkernel core due to its performance
  and safety. Rust also allows low-level access (inline assembly for special CPU instructions if needed, and crates for
  accessing MSRs, etc.). We might write some bootstrap or prototype parts in C (especially if modifying an existing
  hypervisor like KVM or bhyve), but then gradually port to Rust. **Go** could be used for higher-level orchestration
  tools or CLI, since Go is excellent for quick development of networked services (like a management API server). **Zig
  ** is another option similar to C but safer; it could be used for the hyperkernel if we find it easier to interface
  with hardware, but currently Rust has more momentum in systems programming. Ultimately, we may end up with a mix:
  hyperkernel in Rust, management daemons in Go, and possibly small assembly stubs for very specific tasks.
- **Open-Source Projects to Reference or Reuse:** Aside from rust-vmm and Cloud Hypervisor, we will look at older SSI
  projects like **Kerrighed** and **OpenMOSIX** (both were Linux-based SSI clusters). Their source code (though dated)
  can give insight into process migration and distributed scheduling. Kerrighed, for instance, was an extension to the
  Linux kernel for cluster-wide paging and process
  migration ([Kerrighed - Wikipedia](https://en.wikipedia.org/wiki/Kerrighed#:~:text=Kerrighed%20is%20an%20open%20source,at%20the%20Paris%20research%20group)) ([jeanparpaillon/kerrighed-tools: Kerrighed tools (upstream ... - GitHub](https://github.com/jeanparpaillon/kerrighed-tools#:~:text=Kerrighed%20offers%20the%20view%20of,a%20set)).
  While our approach is different (we don’t modify the guest OS), the concepts overlap. If any of their algorithms or
  even code (licensed under GPL likely) is adaptable, we should leverage that rather than reinventing. Additionally,
  research prototypes like **Popcorn Linux** (a Linux variant for multi-ISA multiprocessing) might have relevant code
  for migrating threads between heterogeneous processors.
- **Testing and Continuous Integration:** Use QEMU to simulate multiple nodes on a single machine for testing (by
  running multiple QEMU instances with our hyperkernel code, we can simulate the cluster without needing physical
  hardware). This will be essential for automated testing. We can use CI pipelines to run a matrix of tests (e.g.,
  memory intensive workloads vs CPU intensive) on each commit. Tools like **k8s** or containers might even help simulate
  network partitions or delays in a controlled manner to test fault handling. For debugging distributed issues, we might
  integrate logging frameworks (perhaps conditional compile to enable verbose logging of decisions and message
  exchange).

The table below summarizes some components and tools:

| **Subsystem**              | **Technology / Libraries**                            | **Purpose**                                                                                    |
|----------------------------|-------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Hypervisor Core            | Rust (rust-vmm, KVM bindings, bhyve code)             | Low-level VM creation, CPU and memory virtualization, trap handling in a safe language.        |
| Memory Coherence Directory | Custom (Rust dashmap or etcd)                         | Track page ownership and caching; possibly etcd for consensus on changes.                      |
| Networking (Data Plane)    | Sockets/DPDK + Rust `tokio` or Go net                 | Efficient transfer of pages and vCPU states; async handling of requests.                       |
| Networking (Control Plane) | gRPC or custom REST (Go) + TLS                        | Node coordination, cluster management commands (join/leave, health, etc.) securely.            |
| Device Emulation           | Virtio (rust-vmm crates or QEMU backends)             | Present unified disk and NIC to guest; forward I/O requests to proper node.                    |
| Monitoring & Logging       | Prometheus + Grafana, Log aggregation                 | Export metrics (latency, migrations, cache hits) and collect logs from all nodes for analysis. |
| Build & CI                 | Cargo (for Rust) / Go modules, Jenkins/GitHub Actions | Build artifacts for each node, test on multiple VMs, ensure reproducibility and reliability.   |

By harnessing these open-source building blocks, we save time and stand on the shoulders of prior work. The emphasis is
on integration and innovation in the distributed aspects, rather than writing everything from scratch.

## Development Roadmap and Milestones

Developing a full distributed hyperkernel is an ambitious project. We outline a phased roadmap with milestones to
incrementally build and validate the system:

1. **Phase 0: Research and Design (Current)** – Complete the design specification (this document) and gather a team.
   Investigate existing projects (KVM, rust-vmm, etc.) to choose starting points. Outcome: clear architecture (
   components defined), choice of implementation language(s), and setup of development environment and repository.

2. **Phase 1: Single-Node Hypervisor Prototype** – Build a minimal hypervisor that can run Linux on a single node using
   our chosen stack (e.g., a simple VM monitor in Rust on KVM or bare-metal). This establishes the baseline VM launch,
   vCPU management, and basic I/O (perhaps just a console). We ensure we can boot an unmodified Linux on this hypervisor
   on one machine. Milestone: *Boot a single-node VM with our hypervisor and run a test program.*

3. **Phase 2: Two-Node Memory Federation (Prototype)** – Extend the prototype to use memory from a second node. For this
   milestone, we might cheat a bit: run the guest OS fully on Node1’s CPU, but whenever it accesses a certain high
   memory range, trap and fetch from Node2 via network. We can implement this by marking that range as non-resident on
   Node1. This will prove out the distributed memory mechanism (userfaultfd or hyperkernel trap and network fetch). vCPU
   might still remain on Node1 (no migration yet). Milestone: *Guest OS can access memory on Node2 (e.g., allocate and
   use an array that spans nodes) transparently.* Performance will be poor if it thrashes, but functionally it works.

4. **Phase 3: vCPU Migration** – Implement the ability to migrate a vCPU between Node1 and Node2. This involves freezing
   execution, sending register state, and resuming on the other side, as well as transferring the identity of that vCPU
   in the hypervisor. Start with manual or triggered migration (e.g., a debug command to push a vCPU to the other node).
   Then integrate it with the page fault handler: on a remote page fault, instead of fetching the page, migrate the vCPU
   to the remote node (simple policy to test). Milestone: *A running process can move between nodes without crashing.*
   For example, a program that intentionally touches memory on both nodes will cause the hyperkernel to chase it back
   and forth.

5. **Phase 4: Coherence and Coordination** – Expand to more nodes (3 or 4) and implement a basic coherence directory. At
   this stage, formalize how we track page ownership and sharing. Implement replication of read-only pages (so one page
   can be cached on two nodes). Also implement invalidation on write. This will likely involve a central coordinator or
   a distributed agreement on who owns what (we can pick a simple approach like master node as directory for now). Also,
   build rudimentary scheduling: e.g., a timer to check if a vCPU should be moved closer to its frequent memory.
   Milestone: *Demonstrate a 3-node cluster running a memory-intensive app with correct results (no data corruption) and
   measure the overhead.* We should see that reads can happen from local caches and that writes are visible globally.

6. **Phase 5: I/O Integration** – Up to now, our VM may have been using minimal I/O (maybe a single disk image on
   Node1). Here, enable distributed I/O: allow the guest to use a disk that is actually on Node2 or a NIC on Node3.
   Implement a virtio-blk backend where Node1’s hyperkernel forwards block requests to Node2 over the network, for
   example. Also, handle networking for the guest: possibly create a virtual NIC that bridges to the physical NIC of all
   nodes (so if one goes down, others can still reach out). This might involve writing a small virtual switch or using
   an existing SDN solution. Milestone: *Guest OS can perform I/O (file read/write, network ping) utilizing devices on
   remote nodes.* This is critical for full functionality.

7. **Phase 6: Management & Usability** – Develop the management tools to simplify using the system. For example, a CLI
   to launch a cluster VM: “`openhyperkernel start -n node1,node2,node3 -m 64G -c 32`” which would start the service on
   those nodes and combine them. Implement status commands to query resource usage. Also build a simple web UI or use
   existing monitoring to display cluster status (CPU load per node, etc.). This phase also includes refining the
   configuration (maybe making nodes aware of heterogeneous speeds, etc., so include that info in config). Milestone:
   *User can easily start and stop the distributed VM, and observe its operation via a management interface.* At this
   point, basic functionality is achieved.

8. **Phase 7: Optimization (Performance Tuning)** – Iterate on the algorithms for page placement and vCPU scheduling.
   Introduce the machine learning or adaptive heuristics module to improve locality. This will involve collecting
   traces, perhaps using some standard benchmarks (SPECint, SPECjbb, etc.) to train or adjust parameters. Optimize
   network usage (e.g., pipeline page transfers, compress data if beneficial, etc.). Possibly add support for RDMA if
   available to speed up memory copying. Milestone: *Performance within X% of native for certain workloads.* (We define
   a target, e.g., for a given memory-heavy benchmark, the slowdown is no more than 20% compared to if it ran on a real
   SMP of equivalent specs).

9. **Phase 8: Fault Tolerance Features** – Implement redundancy: memory mirroring option and health monitoring. For
   health, integrate with IPMI or utilize each node’s sensors to catch issues. Create a procedure for node failover:
   test by killing one node’s hyperkernel and see if the guest can survive (maybe it pauses and resumes). Implement at
   least one strategy for surviving a failure, e.g., if using mirrored memory, then have a failover node ready to take
   over. Also implement live node addition/removal (via CPU/memory hotplug in the guest if possible). Milestone:
   *Cluster can handle a planned removal of a node without crashing the guest.* Perhaps even an unplanned removal with
   only a brief pause. Document how far we got in seamless recovery.

10. **Phase 9: Heterogeneity Support** – Test the system on nodes with different CPU models, memory sizes, etc. Add
    features to handle discrepancies: e.g., CPU feature flags masking (so the guest only uses instruction sets common to
    all nodes). If some nodes are much slower, implement weight adjustments in the scheduler so they aren’t overloaded.
    Milestone: *Demonstrate the VM spanning an older PC and a newer PC, still functioning correctly.* Performance might
    differ but it works and is optimized.

11. **Phase 10: Polishing and Production Readiness** – Hardening the code, writing documentation and tutorials,
    packaging (perhaps create bootable ISO for the hyperkernel so users can easily set up nodes). Conduct extensive
    testing for memory leaks, race conditions, etc. Possibly engage third-party audit for security. By this phase, the
    project should be open-source on a platform like GitHub, inviting contributions. We will address any remaining
    issues, improve installation (maybe provide an Ansible script or similar to set up cluster nodes), and prepare for a
    1.0 release. Milestone: *Release version 1.0 of the open-source distributed hyperkernel, with documentation and
    real-world test cases.*

Each phase builds on the previous, ensuring that at any point we have a working (if not fully featured) system. After
Phase 5 or 6, early adopters could experiment with it on non-critical workloads. By Phase 10, it should be robust and
feature-complete. Throughout, we will maintain open communication with a community (mailing list or forum) to get
feedback, especially from anyone who might test it on their own hardware.

## Conclusion

This project envisions a powerful capability: **turning a cluster of ordinary PCs into a single massive Linux system**
through software alone. By drawing inspiration from TidalScale’s software-defined server and past single-system-image
research, and by utilizing modern programming practices, we aim to create an open-source implementation that is
accessible and adaptable. The architectural blueprint we’ve outlined covers the key challenges of CPU/memory
aggregation, coherence, performance optimization, and fault tolerance.

If successful, this distributed virtual machine (DVM) hyperkernel will enable new possibilities – from upcycling retired
hardware for big-memory applications, to providing a flexible alternative to buying large proprietary servers.
Unmodified applications from databases to scientific simulations could run on a “virtual big iron” composed of many
smaller machines. The use of memory-safe languages and open collaboration will ensure the system is reliable and secure
for the community.

In summary, this technical plan sets the stage for a multi-year development effort culminating in a production-ready,
open-source software-defined server platform. By following the roadmap and steadily overcoming each technical hurdle, we
can realize a system where **“a rack of PCs appears as one giant computer”**, available to anyone with the hardware and
the desire to push the boundaries of virtualization technology.

**Sources:** This design is informed by prior work on distributed hypervisors and single system image clusters, notably
TidalScale’s HyperKernel architecture, which demonstrates the feasibility of software-managed coherent clusters, as well
as academic research in distributed shared memory. The plan combines these concepts with modern systems engineering
practices to outline an open-source path forward.