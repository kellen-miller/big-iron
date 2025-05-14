# Development Roadmap for the Big Iron Hyperkernel

Big Iron is a **distributed hyperkernel** project aiming to combine multiple x86 PCs (connected via 1 GbE) into a single
unified system. In essence, it’s an “inverse hypervisor” where **multiple physical machines run one large virtual
machine (VM)**, as opposed to many VMs on one
machine ([TidalScale and inverted server virtualization – Blocks and Files](https://blocksandfiles.com/2022/08/17/tidalscale-and-inverted-server-virtualization/#:~:text=Nine,is%20behind%20the%20name%20TidalScale)).
The hyperkernel will virtualize CPUs, memory, and I/O across nodes so that a standard OS can run unmodified over the
aggregated hardware. Below is a phase-by-phase technical roadmap focusing on building this from scratch in Rust, with an
emphasis on quick iteration, modular design, and validation at each step.

## Phase 0: Prepare a Rapid Iteration Development Environment

To maximize developer efficiency early on, set up a local **simulation and debugging environment** before writing
complex code:

- **Emulate Hardware with QEMU:** Use QEMU/KVM to run the hypervisor as a guest on a development host. Enabling nested
  virtualization allows the Rust hypervisor (running in QEMU) to execute VT-x instructions and launch its own guest VM.
  This avoids constant reboots of physical machines – you can rebuild and run the hypervisor in a VM in seconds.
- **Multi-VM Cluster Simulation:** Leverage QEMU’s networking to simulate a cluster on one host. For example, start two
  or more QEMU instances (each will boot the hyperkernel on an emulated x86 PC) and connect them with a virtual
  network (using `-netdev socket` or a TAP bridge). This provides a controllable testbed for inter-node communication.
- **Debugging Tools:** Take advantage of QEMU’s GDB stub and serial console output. Embed a serial logger in the
  hypervisor to print debug info from bare-metal code. You can step through VM entry/exit in GDB or inspect memory on VM
  faults, greatly speeding up troubleshooting.
- **Reuse and Modularize:** Start a Rust project with `#![no_std]` for bare-metal. Use existing crates for low-level
  tasks – for instance, the `x86` crate offers abstractions for manipulating control registers, MSRs, and descriptor
  tables ([Hypervisor Development in Rust Part 1 - memN0ps](https://memn0ps.github.io/hypervisor-development-in-rust-part-1/#:~:text=This%20article%20covers%20the%20development,the%20fundamentals%20remain%20the%20same)).
  This avoids writing tons of assembly and reduces errors. Plan to isolate components (CPU virtualization, memory
  management, networking) into separate modules for independent testing where possible.

*Validation:* The environment itself can be validated by running a dummy “hello world” hypervisor on QEMU. For example,
write a minimal Rust kernel that boots and immediately exits, and confirm you can deploy it to multiple QEMU VMs
concurrently. Success is when you can launch, debug, and network two hypervisor instances on one machine easily. This
environment will be used in all subsequent phases to iterate quickly without needing the full hardware cluster.

## Phase 1: Build a Minimal Rust Type-1 Hypervisor (Proof of Concept)

The first development milestone is a minimal **Type-1 hypervisor** running directly on hardware (or VM) that can host a
single guest. The goal is to prove we can enter **VMX mode** and execute a guest OS under our Rust hypervisor:

- **Bare-Metal Bootstrapping:** Begin by booting the Rust hypervisor on a single x86 machine. You might use a simple
  bootloader (e.g., a custom assembly stub or Rust’s `bootloader` crate) to enter 64-bit mode and jump to the Rust
  hypervisor code. Set up identity-mapped page tables for the hypervisor itself and initialize basic hardware (disable
  interrupts or PIC, etc.) as needed for a controlled environment.
- **Enable CPU Virtualization:** Activate Intel VT-x or AMD-V in the processor. This involves setting the required
  control bits (e.g., CR4.VMXE) and executing VMXON (for VT-x) to enter hypervisor
  mode ([Hypervisor Development in Rust Part 1 - memN0ps](https://memn0ps.github.io/hypervisor-development-in-rust-part-1/#:~:text=1,instructions%20like%20VMLAUNCH%20and%20VMRESUME)).
  Use Rust to write to model-specific registers (MSRs) and configure the VMX capabilities (the `x86` crate can help
  abstract MSR
  bitfields ([Hypervisor Development in Rust Part 1 - memN0ps](https://memn0ps.github.io/hypervisor-development-in-rust-part-1/#:~:text=%7C%2063,)) ([Hypervisor Development in Rust Part 1 - memN0ps](https://memn0ps.github.io/hypervisor-development-in-rust-part-1/#:~:text=unsafe%20,return%20Err%28HypervisorError%3A%3AVMXBIOSLock))).
  Handle error cases (e.g., if the BIOS lock bit is set preventing VMXON).
- **Create a VMCS & Guest Context:** Allocate a region of memory for guest physical memory (for now, a small fixed
  chunk). Set up a VM Control Structure (VMCS) for a single virtual CPU. Populate the guest state in the VMCS with a
  minimal context – e.g., set the guest’s instruction pointer (RIP) to a known address in the guest memory and guest
  registers to initial values. Identity-map the guest memory in the Extended Page Tables (EPT) so that guest physical ==
  host physical for simplicity at this stage.
- **VM Entry/Exit Loop:** Invoke VMLAUNCH to start running the guest. The guest could be as simple as a few instructions
  that trigger a VM exit (for example, a CPUID or I/O instruction) so we can test the hypervisor’s trap handling.
  Implement a basic VM exit handler in the hypervisor: for now, just handle essential exits (CPUID can be passed through
  or faked, I/O can be ignored or logged, and HLT can be used to stop the guest). Keep this minimal – the aim is just to
  keep the guest running long enough to observe something.
- **Minimal Device I/O:** As a proof of concept, you might not virtualize any real devices yet. However, having a way
  for the guest to output something is useful. One quick hack: intercept writes to a specific port (e.g., 0xE9 port
  often used in bare-metal OS debugging) in the VM exit handler and print the character to the hypervisor console. This
  way a simple guest program can “print” via OUT instructions and you’ll see it on the hypervisor’s serial log.

**Validation:** Try running a trivial guest OS or code. For example, write a small 16-bit or 32-bit realmode program
that prints a message and halts, and load it as the guest. The hypervisor should start the guest, the guest issues a
print (trapped via port I/O or via a hypercall mechanism), and then halts. Seeing the expected message from the guest or
a correct VM exit reason in the hypervisor log indicates success. Another test is to boot an extremely minimal Linux
kernel or unikernel in the VM – at this stage it might panic due to unimplemented devices, but if it at least gets to
real mode or early boot under our hypervisor, it proves the concept. The key success criteria: **the hypervisor can
enter VMX mode and maintain control over a running guest** (e.g. you can intercept a known exit like CPUID and confirm
the hypervisor handled it). This minimal hypervisor, while not useful yet, establishes the foundation for virtualization
in Rust.

## Phase 2: Core Hypervisor Features – vCPU Management and Memory Virtualization

With a basic single-CPU VM working, the next step is to flesh out the core hypervisor so it can manage multiple virtual
CPUs and handle memory mapping more flexibly on one host. This phase lays the groundwork for scalability while still
focusing on a single-machine (single-node) hypervisor.

- **vCPU Management (Multi-core Support):** Initialize all physical cores (pCPUs) on the machine to participate in the
  hypervisor. When the hypervisor boots on a multicore x86, each hardware thread will start in the firmware – bring them
  into the hypervisor, similarly to how an OS brings up APs. Decide on a simple vCPU scheduling policy: for now, *
  *static one-to-one mapping** of guest vCPUs to physical cores is easiest (no time-slicing yet). For example, if the
  machine has 4 cores, you can support a guest with up to 4 vCPUs, each pinned to a different core. This static
  partitioning approach is used by some minimal hypervisors for
  simplicity ([GitHub - syswonder/hvisor: a Rust Hypervisor for mission-critical system](https://github.com/syswonder/hvisor#:~:text=,loongarch64)).
  Implement an inter-core communication mechanism (like a simple IPI) so the main core can start vCPU execution on other
  cores. Each core will load a VMCS for its assigned vCPU and perform a VMLAUNCH/VMRESUME loop. Ensure the hypervisor
  can handle concurrent VM exits on multiple cores (this may require per-core data structures or locks to avoid
  interference).
- **Memory Virtualization Management:** Replace the simple identity-mapped EPT from Phase 1 with a proper **second-level
  address translation** setup. The guest “physical” memory will be backed by host memory, but now the hypervisor should
  manage it through page tables so it can control access. Initially, continue to use a static allocation (e.g., a fixed
  1 GB region per VM) rather than dynamic allocation. The idea is to set up the EPT structures (PML4 down to page table)
  and fill them in with mappings for the guest memory region. This is still one big contiguous allocation on the host,
  but now we have the machinery to later change mappings or mark pages non-resident. Leverage the hardware support for
  multi-level translation: x86 hardware will translate guest-virtual -> guest-physical -> host-physical using two levels
  of page tables, with the hypervisor populating the second
  level ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=single%20point%20of%20failure,The)).
  Ensure the hypervisor can catch EPT violations (e.g., if a guest access is not mapped) – for now, it shouldn’t happen
  because we map everything, but this sets up the handler for later dynamic paging.
- **Basic Interrupt Virtualization:** As part of completing core functionality, handle interrupts in a minimal way. You
  may keep interrupts disabled in the guest initially, or allow a simple virtual timer interrupt. If using APIC
  virtualization or intercepting interrupts is too complex, this can be deferred. However, at least confirm that the
  hypervisor can survive external interrupts (timer or IPIs) while in VMX operation. You might configure the VMCS to
  trap interrupts to the hypervisor or use a paravirtual approach (guest uses hypercalls for yields) at first.
- **Rust Safety & Performance:** Continue to utilize Rust’s strengths – for example, create safe wrappers for VMCS
  fields read/write, to avoid common pointer bugs. Where performance matters (like the VM exit path), consider `unsafe`
  blocks with carefully audited code for speed. Measure the overhead of a simple hypercall or VM exit now: this gives a
  baseline to compare as features are added.

**Validation:** Test the multi-vCPU support by running a guest that can spawn threads or processes. For example, a
SMP-capable toy kernel or a modified Linux kernel that prints from multiple CPUs. If full OS boot is too heavy, you can
directly write a guest program that uses an atomic counter in memory: have two vCPUs (on two physical cores) increment a
shared memory location a million times, then check if the final result equals the expected sum. Initially, since all
vCPUs are on one machine with normal memory, this should work and the count should match (this is also a test for basic
memory consistency on one node). Also verify that each vCPU is truly executing in parallel on different cores by
measuring time or using performance counters. For memory management, you can simulate a page fault by deliberately
leaving a portion of the guest memory unmapped in EPT and accessing it – the hypervisor should catch the fault (VM exit
for EPT violation) and could, for example, map it on the fly or at least log that it trapped an access. Success criteria
for Phase 2: the hypervisor supports an SMP guest (e.g., 2–4 vCPUs) on a single host with correct execution and basic
memory virtualization in place (no manual interventions needed to keep the guest running). This sets the stage for
distributing these vCPUs and memory across multiple nodes.

## Phase 3: Inter-Node Networking and Messaging Layer

With a solid single-node hypervisor, we now introduce a second node and build the **communication layer** that will bind
nodes into one cluster. In this phase, each machine runs the hyperkernel, and we develop the ability for them to send
messages (over the 1 GbE NIC) to coordinate VM execution and memory management.

- **Network Driver in the Hypervisor:** Choose a network interface that is available on all target machines (and
  emulated in QEMU). A simple option is an Intel e1000 or RTL8139 Ethernet NIC. Implement a bare-bones NIC driver under
  the hypervisor’s control (since there’s no host OS, the hypervisor must program the NIC registers directly).
  Initially, polling the NIC’s TX/RX rings may be simpler than dealing with interrupts – the hypervisor main loop can
  periodically check for incoming packets. Bring up the link between two hyperkernel nodes: e.g., assign static MAC or
  IP addresses and verify basic connectivity.
- **Lightweight Messaging Protocol:** Design a minimal protocol for hypervisor-to-hypervisor communication. Treat the
  private Ethernet interconnect more like a system bus than a traditional
  network ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=The%20virtual%20machine%20aggregates%20the,the%20hyperkernel%20manage%20the%20second)) –
  we want low-latency, reliable delivery for small control messages. A simple approach is to use UDP over IPv4 for
  convenience (since UDP provides checksums and you can leverage existing IP stacks if desired), or even a custom
  Ethernet frame with a bespoke type for hyperkernel messages to avoid full IP overhead. In early development,
  simplicity is key: for example, define a fixed message format with fields for message type (e.g., “page request”,
  “page reply”, “CPU migrate”, etc.), sender/receiver IDs, and payload. Ensure that the messaging layer can pack and
  unpack these, and handle lost packets (maybe by simple timeouts and retransmission for critical ops, or rely on the
  reliability of a controlled switch environment).
- **Establish Node Coordination:** Implement an initial handshake when nodes start. For instance, when hyperkernel node
  2 comes online, it broadcasts or sends a “HELLO” message to node 1. They exchange basic info (like how much memory
  each has, etc.) and agree to form a single cluster. In this phase, you can designate one node as the **primary** just
  for setup purposes – e.g., node 1 might tell node 2 “I’m already running the VM” or “hold on, I’ll start the VM when
  ready”. The goal is to have a known state before we actually share work between them.
- **Integration with Hypervisor Loop:** Merge the networking with the hypervisor’s workflow. For example, you might
  dedicate one vCPU or core on each node to run a “network thread” (in practice, a loop in the hypervisor that checks
  for new messages and processes them). Alternatively, integrate message checks into the VM exit handler or a hypervisor
  timer tick, so that even when guest vCPUs are running, the hypervisor can occasionally process incoming messages. This
  is important so that, for instance, a request to fetch a memory page can be handled promptly even if the node’s vCPU
  is busy running a guest. Balance complexity by perhaps using one core for management tasks (similar to a management
  core).
- **Security Considerations:** At this stage, trust is assumed between nodes, so encryption isn’t necessary on the link,
  but you might implement basic checks (e.g., ignore messages from unknown node IDs) to avoid any malformed packet
  issues. Keep the focus on functionality first.

**Validation:** Using the multi-VM QEMU setup, test that the two hypervisors can communicate. A simple test is to
implement a “ping-pong” message: have node 1 send a test packet (“PING”) to node 2, where the hypervisor code recognizes
it and immediately replies with a “PONG”. On node 1, verify the response is received and correct. Measure the round-trip
time using timestamp counters if possible, to get a sense of latency (expected on the order of tens to hundreds of
microseconds with 1 GbE). Also test negative cases: e.g., unplug the virtual cable or shut down one side and ensure the
other side doesn’t hang indefinitely (time out or retry gracefully). By the end of Phase 3, we should have a **basic
messaging layer between hyperkernel nodes** working reliably. This is the backbone for all upcoming distributed
features.

## Phase 4: Distributed Memory – Remote Paging across Nodes

With communication in place, Big Iron can now treat **memory as a unified resource across the cluster**. In this phase,
we enable a VM running on one node to use memory that physically resides on another node, through a mechanism akin to
remote paging. The aim is to present a **single large guest physical memory** space that spans all nodes’ RAM.

- **Guest Physical Memory Partitioning:** Decide how to split the guest’s physical address space among the nodes. A
  simple scheme is to **statically partition by address ranges**. For example, in a two-node cluster, node 1 could be
  responsible for guest physical addresses 0–2GB and node 2 responsible for 2GB–4GB (assuming a 4GB guest). This
  “ownership” means node 2 initially holds the actual RAM for pages in its range. Record this mapping in a table so each
  hypervisor knows which node owns a given address (this can be as straightforward as a range check or as detailed as a
  map from page number -> owner node).
- **On-Demand Page Fetching:** Extend the EPT management to handle non-local pages. Initially, **do not preload all
  remote memory**; instead, let the guest demand pages. For instance, if the guest (running on node 1) tries to access a
  guest physical page that lies in node 2’s range, node 1’s EPT will not have a mapping, causing a VM exit (EPT
  violation). The node 1 hypervisor then pauses the guest vCPU and sends a **“page request” message** to node 2,
  specifying the page number needed. Node 2’s hypervisor receives this, looks up the page in its memory (since node 2 is
  the owner of that page), and replies with the page content (4KB payload) and an authorization for node 1 to map it.
- **Mapping Remote Pages:** Upon receiving the page data, node 1’s hypervisor allocates a local physical page (if not
  already allocated) and copies the data into it. It then updates its EPT to map the guest physical page to this local
  frame, and marks the mapping present. Now the guest can resume and continue as if the memory was always there.
  Effectively, we’ve **paged in** a remote memory page across the network. For initial implementation, treat this page
  as now *cached* on node 1. Possibly mark it read-only or owned to handle future writes (coherence will be handled in
  the next phase), but if we assume for now that only one node’s vCPU is actively using this page, it’s fine to keep it
  cached.
- **Eviction / Send-Back Mechanism:** If the guest keeps allocating new memory, node 1 might eventually pull a lot of
  pages from node 2. If node 1’s physical RAM is limited, implement a basic page eviction policy: for example, a
  least-recently-used cache of remote pages. When node 1 needs to free up space, it can choose a remote page it has
  cached, send the current content back to the owning node (node 2) in a “page unload” message (unless node 2 also
  discarded it, in which case no need), then drop its mapping. Node 2 would mark that it again holds the authoritative
  copy. This part can be rudimentary at first or even skipped if each node has enough RAM to hold all pages; the main
  point is to handle the scenario where a node doesn’t have infinite memory.
- **Optimize for Read-Only Pages:** For any pages that the guest only reads (code pages, or constant data), it’s
  inefficient to transfer ownership. As an optimization, you can allow **multiple nodes to cache read-only copies** of a
  page while the owner retains the master copy. In this phase, we might not fully implement this – but keep the idea in
  mind for coherence. Initially, you could simply always transfer the page and assume exclusive access. Coherence will
  refine this.
- **Networking Considerations:** Moving a 4KB page over 1 Gbps Ethernet has a non-trivial cost (~40 µs latency, and adds
  load). To mitigate performance issues, you can batch multiple pages if the guest faults on a sequence, or pipeline the
  requests. However, initially just get single-page requests working. Use the messaging layer for reliability; if a page
  response is lost, implement a retry or at least a timeout to avoid hanging the guest vCPU forever.

**Validation:** Now you can **run a workload that exceeds the memory of a single node**. For example, on a test cluster
of two QEMU nodes, deliberately give each node a small amount of RAM (say 256MB each) but configure the guest OS to
think it has 512MB total. Then run a memory-intensive program in the guest (such as allocating a large array or running
`memtester` in Linux). Monitor that the guest continues running when it uses more memory than one node can provide –
this indicates that remote paging is happening transparently. Instrument the hypervisors to log page fault events and
network transfers. You should see a count of page requests being handled between nodes. A success criterion is that the
guest can allocate and use memory up to the sum of the two nodes’ RAM with correct results (e.g., if the guest writes
distinct values to a large array spread across both nodes’ memory, reading them back yields the right values).
Performance will be slower due to network paging, but that’s expected. Also test scenario where the guest frees memory:
ensure that the hypervisor could potentially reclaim those pages (though a full garbage collection can be complex, you
can at least drop mappings if the guest explicitly frees and hints via balloon driver or just ignore for now). By the
end of Phase 4, **the distributed VM has a unified memory space** – an important milestone demonstrating that one node’s
CPU can seamlessly access memory on another node.

## Phase 5: vCPU Migration Between Nodes

With memory now fluid across the cluster, the next capability is to make **virtual CPUs (vCPUs) mobile** as well. In
this phase, we enable moving the execution of a guest vCPU from one physical host to another at runtime. This is
important for load balancing, resilience (moving work off a failing node), and treating all CPUs in the cluster as a
single
pool ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=a%20cluster%20of%20closely%20cooperating%2C,libraries%20or%20applications%20are%20required)).

- **Capturing vCPU State:** When a migration is triggered (either manually for testing, or automatically in a real
  scenario), the source node’s hypervisor must capture the complete state of the vCPU. This means stopping the vCPU (
  e.g., inject a hypervisor interrupt or use a control to cause a VM exit at a convenient point), then reading out the
  guest state from the VMCS: general purpose registers, PC (program counter), stack pointer, control registers (CR3 for
  page table, etc.), and any other guest MSRs or extended state (x87/SSE registers, etc., if needed for accuracy).
  Package this state into a message (it might be large, but still on the order of a few hundred bytes) to send to the
  target node.
- **Transferring and Reconstructing State:** Send the vCPU state over the network to the target node’s hypervisor in a *
  *“vCPU migrate” message**. The target node, upon receiving it, will create a new VMCS if one isn’t already allocated
  for this vCPU (or reuse an idle one). It then populates the VMCS guest fields with the received register state.
  Essentially, you are checkpointing the CPU and restoring it on another machine.
- **Memory Hand-off:** If the vCPU was the only one running on the source node, and you migrate it entirely to the
  target, then effectively the whole VM is now running on the target node. In that case, you might also transfer
  ownership of memory pages (e.g., source node could send all modified pages to target before finalizing migration).
  However, in the general case of multi-vCPU VMs, other vCPUs might still be active on other nodes, so memory is already
  shared. For initial migration implementation, we can assume a simpler scenario: **the VM has one vCPU** (or we migrate
  all vCPUs one by one in a coordinated way), so that after migration the source node is not actively running any part
  of the VM. This avoids coherence issues during the move. Use the remote paging mechanism to lazy-transfer memory: when
  the vCPU resumes on the target node, if it accesses a page that was last owned by the source, it will fault and fetch
  it over the network (just as in Phase 4). This means we don’t have to copy all memory up front. It trades migration
  time for on-demand paging, which is fine for a prototype.
- **Resume on Target:** Once state is loaded, enter the guest on the target node (VMRESUME). From the guest OS’s
  perspective, its CPU was paused for a moment and then continued – it should have no idea it switched physical hosts.
  The hyperkernel on the target now considers itself the host for that vCPU. The source node’s hypervisor can free the
  VMCS and any resources associated with the migrated vCPU.
- **Handling Active Multi-vCPU Case:** Eventually, you’ll want to migrate one out of several vCPUs while others are
  still running elsewhere. This is harder because it means after migration, two different nodes have active vCPUs that
  might access the same memory. We need full memory coherence for that (Phase 6). So, for now, it’s safest to either
  migrate the *entire* VM (all vCPUs move, essentially moving the whole workload to a new machine) or only use this when
  there’s a single vCPU. Later, after coherence, we can relax this.
- **Migration Triggers:** Implement an interface to trigger migration. For testing, this could be as simple as a
  hypervisor console command (e.g., typed over serial: “migrate vcpu0 to node2”) or even a built-in timer that after X
  seconds initiates a migration of the running VM. In a real system, triggers might be load imbalance (one node CPU
  overloaded) or fault tolerance (detected hardware issue, move VM away).

**Validation:** Start with a simple scenario: one VM, one vCPU, running on node 1. After the guest has been running for
a bit (say, it’s incrementing a counter or printing messages periodically), initiate a migration of that vCPU to node 2.
Expect the guest to perhaps pause momentarily (to snapshot state), then continue running on node 2. You can detect
success by observing that **the guest’s execution continues from the same point** after migration: for example, if it
was printing an incrementing number every second, the sequence should not reset or jump incorrectly. Measure the
downtime during migration – insert timestamps around the stop and resume to calculate how long the guest was paused.
This could be on the order of tens of milliseconds (mostly network transfer time for state and any key memory pages).
Also test migrating back to ensure the process is reversible. If the guest is more complex (say a small Linux VM), you
could try running a ping from inside the guest during migration to see if it drops a packet or two but then continues;
that would show the VM’s network stack survived the move (assuming the virtual NIC state moved or was reinitialized
seamlessly). Another test: migrate under memory load – e.g., the guest is using memory from both nodes, then move the
vCPU. The guest should still see its memory (via remote paging) after moving. By the end of Phase 5, we demonstrate that
**vCPU (and whole VM) can migrate between nodes** with minimal disruption, validating the idea of mobile “virtual
processors” in the
cluster ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=a%20cluster%20of%20closely%20cooperating%2C,libraries%20or%20applications%20are%20required)).
This sets the stage for having multiple vCPUs active on different nodes simultaneously.

## Phase 6: Distributed Shared Memory and Coherence Protocol

Up to now, we have only run one vCPU at a time across the cluster (or multiple vCPUs but confined to one node at a time)
to avoid consistency problems. In Phase 6, we tackle the most challenging part: **ensuring memory coherence across nodes
** so that **multiple vCPUs on different physical machines can concurrently access shared memory with correct results**.
Essentially, we implement a software-based **Distributed Coherent Shared Memory (DCSM)** for the
guest ([TidalScale and inverted server virtualization – Blocks and Files](https://blocksandfiles.com/2022/08/17/tidalscale-and-inverted-server-virtualization/#:~:text=Hat%20and%20Ubuntu%20have%20certified,memory%20computing)),
akin to a distributed cache protocol at the hypervisor level.

- **Memory Consistency Model:** We choose a memory consistency model to enforce. The simplest (for correctness) is to
  preserve **strong consistency (sequential consistency or at least x86-like consistency)** across the cluster.
  TidalScale’s hyperkernel ensures that the cluster’s memory is **“strongly cache-coherent”** and preserves the Intel
  memory
  ordering ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=address%3A%20the%20OS%20uses%20the,a%20single%20node%3F%20%20Does)),
  meaning a store by one vCPU will be seen by others as if they were on a single motherboard. We will implement
  coherence accordingly: no stale reads and writes are visible in the guest.
- **Ownership and States:** Enhance the page ownership concept. Each guest physical page can be in one of a few states:
  **Exclusive** to a node (that node has the only copy, possibly dirty), **Shared** (multiple nodes have read-only
  copies), or **Invalid** on a given node (not present locally). One node (typically the one that owns the physical
  frame or that last had it exclusively) will act as the “owner” or directory for that page to coordinate state.
  Maintain a metadata structure per page (or per chunks of pages) that tracks which nodes have a copy and what the
  current state is.
- **Read Sharing:** If a vCPU on node 1 tries to read a page that node 2 owns, we already fetch it (Phase 4). Now we
  will allow node 1 to keep that page cached while node 2 also potentially retains it. Both nodes mark it as **Read-Only
  **. The directory (say node 2 is the home for that page) knows that node 1 has a copy. This is the **Shared state**.
  Many nodes can have the page in Shared state for read-heavy workloads, enabling efficient access without repeated
  network fetches.
- **Write Invalidation:** If a vCPU on node 1 wants to write to a shared page, we must ensure it has exclusive rights.
  We will leverage the EPT permission to trap writes: mark any remotely cached pages as read-only in the EPT. So when
  node 1’s vCPU attempts a write, a VM exit occurs. The hypervisor on node 1 then sends a **“write request”** for that
  page to the owner (node 2). The owner node (node 2) sees that currently node 1 (and maybe others) have it in Shared
  state. To grant write access, the owner sends an **invalidation message** to all other nodes that have the page. Those
  nodes’ hypervisors receive the message and invalidate that page’s mapping in their EPT (and flush it from their local
  cache). Node 2 may either transfer ownership to node 1 at this point or simply tell node 1 “you have exclusive access
  now” (effectively making node 1 the owner until it’s done writing). Node 1’s hypervisor can then mark the page
  writable in EPT and resume the guest, which will now succeed in writing. This sequence ensures no other node has a
  stale copy once the write
  happens ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=address%3A%20the%20OS%20uses%20the,a%20single%20node%3F%20%20Does)).
- **TLB and Cache Flush:** After invalidation, remote nodes must flush any cached data for that page (including TLB
  entries on any vCPUs). The hyperkernel should take care of TLB shootdown similar to how an OS does on multicore: e.g.,
  send an IPI to vCPUs on that node to flush the specific guest physical page from their TLB if it might be cached.
  Because each hypervisor already traps guest operations, it can also flush the shadow EPT TLB (the CPU’s EPT cache) as
  needed. This guarantees that once an invalidation ack is sent back, no stale translation exists.
- **Write Propagation:** After node 1 writes to the page exclusively, what if another node (say node 3) later needs that
  page? In a typical MESI protocol, node 1 now has the only up-to-date copy (Modified state). So if node 3 requests the
  page, the current owner (which might be node 1 now) will have to send the latest data to node 3. We can implement this
  by having node 3’s request routed to the last writer. If we kept node 2 as directory, node 2 might ask node 1 to
  provide the data (“owner forward” model). Choose whichever is simpler: a centralized directory (home node always
  coordinates, more messages but simpler logic) or dynamic owner (faster but more complex tracking). The key is that the
  requester gets the latest copy.
- **No Centralized Global Lock:** Ensure the coherence protocol is distributed. There should be **no single master node
  for the whole system**; instead, each memory region/page can be managed by its home
  node ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=a%20system%20bus%20than%20a,fast%21%20Memory%20is%20always%20strongly)).
  Coordination messages happen per page or per request, so the system scales with number of pages, and there’s no single
  point of failure or bottleneck. We basically extend NUMA-like cache coherence over Ethernet – conceptually turning the
  network into an interconnect for our “L4 cache” of
  memory ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=Thanks%20For%20the%20Memories%20%E2%80%93,the%20first%20level%20page)).
- **Data Structures and Algorithms:** Implement a directory table for pages (maybe hash table keyed by page number ->
  record of owner and sharers). Use locks or atomic operations to update this safely if multiple events happen (though
  likely serialize operations per page). Keep the protocol simple at first: e.g., allow only one outstanding write
  request on a page at a time (queue others if needed). This simplifies reasoning and is acceptable given the relatively
  slow network operations.
- **Performance Optimizations:** Coherence will add overhead. To mitigate this: *batch invalidations* (if writing a
  whole page, that’s the granularity anyway; but if doing sequential writes, you might keep exclusive access until
  someone else asks for it back, to avoid ping-ponging). *Reduce protocol chatter*: for example, use a single “ownership
  handoff” message that implies invalidation if only one other node had it. Also consider larger page sizes (2MB huge
  pages) for rarely shared regions to cut down on number of coherence operations, though that can wait until profiling
  shows need.

**Validation:** Coherence is correct if the guest OS and programs cannot tell that memory is distributed. To test this,
design concurrent workloads across nodes. One test is a classic producer-consumer: run two threads on two different
physical nodes (pin vCPU0 on node1 and vCPU1 on node2). Have them use a shared memory buffer to pass data (for example,
thread A writes a value to a memory location and thread B busy-waits until it sees that value, then maybe writes an
acknowledgment). This should work reliably – thread B must see the update from A **promptly** once the write is done,
and B’s write must be seen by A, etc., with no data corruption. Without coherence, this would fail (B might never see
A’s write or see an out-of-date value). Another test is to run standard multi-threaded software on the VM. For instance,
boot a full OS on the cluster (Linux SMP) with two vCPUs on different nodes. Then run something like `make -j2` to
compile a program, or run a database benchmark with threads – these will stress shared memory (locks, semaphores, etc.).
If the system is coherent, the OS won’t crash and the computations will be correct. You can also use memory consistency
test suites (there are litmus tests for memory ordering) to ensure that the behavior is at least as strict as expected (
x86 expects writes to be seen in order, etc., which our invalidation scheme preserves by making writes atomic to all
nodes ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=cache%20coherent,The))).
Instrument performance: measure how many microseconds a remote write invalidation takes, or how much extra latency a
lock acquisition between two nodes has versus on one node. This helps identify bottlenecks for future optimization. The
ultimate success criterion for Phase 6 is that **the VM can truly run with vCPUs on multiple nodes concurrently, with a
coherent shared memory view**. At this point, Big Iron achieves the core promise: it behaves like a giant SMP machine
spread over the
network ([TidalScale and inverted server virtualization – Blocks and Files](https://blocksandfiles.com/2022/08/17/tidalscale-and-inverted-server-virtualization/#:~:text=Hat%20and%20Ubuntu%20have%20certified,memory%20computing)) ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=address%3A%20the%20OS%20uses%20the,a%20single%20node%3F%20%20Does)).

## Phase 7: Orchestration and Cluster Management Tools

Finally, build out the **orchestration and management layer** to make Big Iron usable and maintainable. This includes
how to start the distributed VM, how to monitor it, and how to add or remove nodes – all with a focus on reliability and
developer/operator convenience rather than manual hackery.

- **Cluster Boot and VM Launch:** Develop a procedure for booting the unified system. One approach: designate one
  hyperkernel node as the **bootstrap leader** that loads the guest OS kernel (for example, it could have the disk
  attached or a BIOS that boots the OS). During Phase 3 we allowed a primary for handshake; expand on that. The leader
  can initialize the guest memory (perhaps load the OS image into the distributed memory space) and start the first
  vCPU. Other nodes, upon receiving the “start” message, will know to participate by launching their vCPUs in halted
  state until the OS brings them online (the guest OS, when it boots, will use APIC signals or ACPI tables to start
  secondary CPUs – the hyperkernel should intercept those and start vCPUs on other nodes accordingly). Prepare the *
  *unified hardware description** for the guest: e.g., present an ACPI table that lists the total number of CPUs (across
  all nodes) and total memory. The hyperkernel can cooperate to provide a consistent view – possibly the leader node can
  generate these tables including CPUs that will actually run on other nodes. This way, a standard OS sees all resources
  from the start.
- **Dynamic Node Management:** Implement the ability to **add or remove nodes** from the cluster. For adding a node: a
  new hyperkernel instance joins the network and announces itself. The existing cluster could either require a reboot of
  the guest to include the new resources (simpler), or if advanced, perform hot-add of memory and CPUs. Linux, for
  example, supports CPU hotplug and memory hotplug – the hyperkernel could simulate this (trigger ACPI hotplug events
  for new CPUs/memory when a node joins). Removal (especially unplanned) is trickier: if a node fails, the remaining
  nodes should detect the loss (missed heartbeats or no response). Since our design avoids any single shared state, a
  lost node means some pages and possibly vCPUs have vanished. In a basic system, that might crash the VM (like losing a
  board in a physical server). A more robust design (future goal) is to live-migrate the at-risk vCPUs/pages off a node
  preemptively when it shows signs of
  failure ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=it%20bring%20down%20the%20entire,respond%20when%20thresholds%20are%20exceeded)).
  At least plan for graceful shutdown: provide a command to migrate all workload off a node and then remove it from the
  cluster without stopping the VM.
- **Distributed Management (No Single Point):** Ensure that, after initial bootstrap, the system has **no single point
  of failure** in
  management ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=a%20system%20bus%20than%20a,fast%21%20Memory%20is%20always%20strongly)).
  Each node’s hypervisor should be capable of making decisions about its pages and vCPUs without always deferring to a
  master. Achieve consensus or agreement via simple protocols: e.g., when adding a node, all current nodes update their
  tables to include the new one; when removing, all agree to drop it. This could be done via a small cluster management
  service built into the hyperkernel (like an election of a coordinator for a particular action, or a config sync). Keep
  it simple and static if possible (for a fixed small cluster, you might not need complex algorithms).
- **Monitoring and Instrumentation:** Develop tools to observe the system’s behavior, which is crucial for iterative
  development and for users. For example, each hyperkernel could expose a debugging console (over serial or telnet)
  where one can query state: “list pages cached locally”, “list vCPUs running here”, “show network latency metrics”,
  etc. Also consider integrating performance counters: measure how many remote page fetches per second, how much network
  bandwidth used, CPU utilization on each node, etc. This will help developers optimize and also serve as a basic
  monitoring solution for an operator of Big Iron.
- **Testing and CI:** By this phase, the system is complex. Set up a continuous integration test that automatically
  boots a small cluster (2-node in QEMU) and runs a suite of regression tests (like the ones described in validations
  for each phase). This ensures that new changes don’t break existing features – critical as you refine the hyperkernel.
  Also test edge cases in a controlled way: e.g., deliberately inject a 100ms network delay in QEMU to see how the
  hyperkernel copes, or limit one node’s memory to see if paging still works under pressure.
- **Future Enhancements Planning:** While not part of the immediate roadmap, note down ideas like incorporating machine
  learning for dynamic optimization (as TidalScale did to optimize page
  placement ([TidalScale and inverted server virtualization – Blocks and Files](https://blocksandfiles.com/2022/08/17/tidalscale-and-inverted-server-virtualization/#:~:text=Nine,is%20behind%20the%20name%20TidalScale)) ([TidalScale and inverted server virtualization – Blocks and Files](https://blocksandfiles.com/2022/08/17/tidalscale-and-inverted-server-virtualization/#:~:text=Hat%20and%20Ubuntu%20have%20certified,memory%20computing))),
  support for more NIC bandwidth (10GbE, etc.), or adding virtualization of I/O devices (perhaps one node’s disk and
  network can be used by the guest, requiring an I/O virtualization strategy or device assignment). These can be tackled
  once the basic platform is stable.

**Validation:** The final system can be demonstrated with an end-to-end integration test: boot a full OS (e.g., a Linux
kernel) on a cluster of physical PCs running the Big Iron hyperkernel. The OS should see a single machine with, say, N
CPUs and the sum of all nodes’ memory. Run realistic workloads: for example, a memory-heavy database that uses more RAM
than any one node has (to stress remote paging) and a CPU-intensive task spread across threads (to use all nodes’ CPUs).
Monitor that performance scales reasonably and, most importantly, the correctness is maintained (no data corruption, no
crashes of the guest OS). Try orchestrating operations: add a node on the fly and observe the OS possibly recognizing
new resources (or at least using them if they were pre-configured); remove a node by migrating its work off and shutting
it down, ensuring the OS keeps running on the remaining resources. A specific success scenario could be: start with 2
nodes, run a workload, then introduce a third node and see improved performance as the hyperkernel begins scheduling
vCPUs there or using its memory. Another scenario: simulate a node failure – perhaps kill one hyperkernel instance – and
verify the VM either survives (if you’ve implemented redundancy) or at least that it shuts down cleanly without
corrupting data (in a controlled fail-stop manner). By passing these tests, Big Iron will have proven to be a **modular,
safe, and scalable distributed hypervisor** platform.

---

Throughout all phases, the guiding principles are **modularity, incremental development, and rapid feedback**. We
started with a tiny Rust hypervisor and gradually added vCPU scheduling, memory management, networking, and then
distribution features one by one. Each phase had clear validation criteria (from simple VM execution to multi-node
consistency tests) to ensure progress is concrete. By focusing on technical fundamentals first (and deferring complex
policy or optimization decisions to later), this roadmap allows the team to get something working at each step and build
confidence. The use of Rust assures memory safety in the hyperkernel core, reducing bugs, while the emphasis on using
QEMU and controlled tests accelerates the development cycle. The end result, if each milestone is met, will be a
functioning Big Iron hyperkernel: a distributed Type-1 hypervisor that **binds a cluster of x86 machines into one “big”
virtual machine** running a standard OS with no
modifications ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=A%20DVM%20aggregates%20an%20entire,libraries%20or%20applications%20are%20required)) ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=The%20virtual%20machine%20aggregates%20the,fast%21%20Memory%20is%20always%20strongly)).
Each technical component – from vCPU migration to coherent paging – will have been built and vetted in stages, resulting
in a robust, scalable system.

**Sources:**

1. Mellor, C. *“TidalScale and inverted server virtualization.”* *Blocks and Files*, 17 Aug 2022. (Overview of
   TidalScale’s distributed hyperkernel
   concept) ([TidalScale and inverted server virtualization – Blocks and Files](https://blocksandfiles.com/2022/08/17/tidalscale-and-inverted-server-virtualization/#:~:text=Nine,is%20behind%20the%20name%20TidalScale)) ([TidalScale and inverted server virtualization – Blocks and Files](https://blocksandfiles.com/2022/08/17/tidalscale-and-inverted-server-virtualization/#:~:text=Hat%20and%20Ubuntu%20have%20certified,memory%20computing))

2. Ike Nassi, *Resilience in a Distributed Virtual Machine*, UCSC Tech Talk, May 2024. (Slides on the DVM/Hyperkernel
   architecture and its
   properties) ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=a%20system%20bus%20than%20a,read%20only%20pages%20across%20the)) ([2024-05 Ike Nassi Resiliance talk.pptx](file://file-HCemeQBbUoobfeKV8r4omz#:~:text=address%3A%20the%20OS%20uses%20the,a%20single%20node%3F%20%20Does))

3. **hvisor** Project – *Rust Type-1 Hypervisor*. *GitHub*, 2023. (Demonstrates a minimalist Rust hypervisor with static
   CPU and memory
   partitioning) ([GitHub - syswonder/hvisor: a Rust Hypervisor for mission-critical system](https://github.com/syswonder/hvisor#:~:text=,loongarch64))

4. memN0ps. *“Hypervisor Development in Rust – Part 1.”* *memn0ps.github.io*, 2023. (Describes using Rust `x86` crate
   and VT-x to build a
   hypervisor) ([Hypervisor Development in Rust Part 1 - memN0ps](https://memn0ps.github.io/hypervisor-development-in-rust-part-1/#:~:text=This%20article%20covers%20the%20development,the%20fundamentals%20remain%20the%20same))

