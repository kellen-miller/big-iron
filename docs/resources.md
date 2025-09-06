# Resources

## General

- [Single System Image (SSI)](https://en.wikipedia.org/wiki/Single_system_image) - Wikipedia article explaining the
  concept of a single system image in distributed computing, where multiple computers appear as a single unified system.
- [Dr. Jonathan Nassi's Speaking Engagements](https://www.nassi.com/speaking-engagements.html) - Information about
  speaking engagements by Dr. Jonathan Nassi, who has expertise in neuroscience and technology.
- [OSDev Wiki Expanded Main Page](https://wiki.osdev.org/Expanded_Main_Page) - Comprehensive wiki resource for operating
  system development with articles on various OS concepts and implementation details.
- [Software-Defined NUMA Servers](https://www.nextplatform.com/2022/09/12/why-arent-there-software-defined-numa-servers-everywhere/) -
  Article discussing the challenges and potential of software-defined NUMA (Non-Uniform Memory Access) server
  architectures.
- [Apache Mesos](https://mesos.apache.org/) - A cluster resource manager that abstracts CPU, memory, storage, and other
  resources across machines, enabling fault-tolerant distributed systems and “datacenter as a computer” scheduling.

## Operating Systems

### Implementations

- [Diosix](https://github.com/diodesign/diosix) - A microkernel-based operating system written in Rust that supports
  hardware-assisted virtualization.
- [XV6 RISC-V User Programs](https://github.com/mit-pdos/xv6-riscv/tree/riscv/user) - User-space programs from the XV6
  operating system, a teaching OS developed at MIT, ported to RISC-V architecture.
- [HubrisOS](https://github.com/oxidecomputer/hubris) - Hubris is a microcontroller operating environment designed for
  deeply-embedded systems with reliability requirements. Its design was initially proposed in RFD41, but has evolved
  considerably since then.
- [RTIC](https://rtic.rs/2/book/en/) - Real-Time Interrupt-driven Concurrency for Rust, a framework for writing
  real-time applications in Rust.
- [TockOS](https://github.com/tock/tock) - Tock is a flexible, real-time operating system for embedded systems.
- [DroneOS](https://github.com/drone-os/drone-core) - DroneOS is a free and open-source operating system for embedded
  devices.
- [OS1](https://github.com/SauravMaheshkar/os1) - Bare Bones rust os implementation
- [ARM64 OS](https://github.com/rust-embedded/rust-raspberrypi-OS-tutorials) - Bare bones ARM64 tutorial
- [RISCV64 OS](https://osblog.stephenmarz.com/index.html) - Making a RISC-V Operating System using Rust

### Articles

- [Writing a Freestanding Rust Binary](https://os.phil-opp.com/freestanding-rust-binary/) - Guide on creating a
  freestanding Rust binary that can run without an operating system, part of the "Writing an OS in Rust" blog series.
- [Rust OS Development](https://github.com/rust-osdev) - GitHub organization focused on operating system development
  using Rust, offering various tools and libraries for OS developers.
- [Rust Kernel Adventures](https://not-matthias.github.io/posts/rust-kernel-adventures/) - Blog post exploring custom
  memory allocators in Rust for kernel development, discussing implementation techniques and challenges.
- [Kernel Driver Development with Rust](https://not-matthias.github.io/posts/kernel-driver-with-rust/) - Tutorial on
  developing kernel drivers using Rust, covering the basics of kernel module development and Rust's safety features in
  kernel context.
- [Kernel Printing with Rust](https://not-matthias.github.io/posts/kernel-printing-with-rust/) - Guide on implementing
  printing functionality in a kernel using Rust, explaining how to set up logging and output in a kernel environment.
- [Kernel Driver Development with Rust (2022 Update)](https://not-matthias.github.io/posts/kernel-driver-with-rust-2022/) -
  Updated tutorial on kernel driver development with Rust, incorporating newer Rust features and kernel development
  techniques.

### Videos

- [Operating System Development Tutorials](https://www.youtube.com/watch?v=r0t8K0SRzR4&list=PL5GPYFCBKv4a_oDzW8d6BXMlTOZb34CUB) -
  YouTube playlist with tutorials on operating system development fundamentals.
- [OS Development Video Series](https://www.youtube.com/watch?v=QUH1moScriw&list=PLFOS-Gn3aXROWUHhB-QTrruOmy26qgr2W) -
  YouTube playlist covering operating system development topics and practical implementation.
- [Kernel Development Tutorial Series](https://www.youtube.com/watch?v=WabeOICAOq4&list=PLSkhUfcCXvqFJAuFbABktmLaQvJwKxJ3i) -
  YouTube playlist with tutorials on kernel development concepts and implementation.

## Hypervisors

### Implementations

- [Rust-VMM](https://github.com/rust-vmm) - GitHub organization dedicated to developing virtualization components in
  Rust, providing building blocks for VMMs (Virtual Machine Monitors).
- [HVisor](https://github.com/syswonder/hvisor) - A hypervisor implementation written in Rust, designed for educational
  purposes and research in virtualization technology.
- [Hypervisor GitHub Collection](https://github.com/stars/not-matthias/lists/hypervisor) - Curated list of
  hypervisor-related projects on GitHub, compiled by not-matthias.

### Articles

- [Hypervisor Development in Rust (Part 1)](https://memn0ps.github.io/hypervisor-development-in-rust-part-1/) - Tutorial
  series on developing a hypervisor using the Rust programming language, covering fundamental concepts and
  implementation details.
- [Hypervisor From Scratch (Part 1)](https://rayanfam.com/topics/hypervisor-from-scratch-part-1/) - Comprehensive
  tutorial series on building a hypervisor from scratch, covering virtualization concepts and implementation.
- [Virtualization Technology Overview](https://www.youtube.com/watch?v=7igpsgCZJY4) - Video explaining virtualization
  technology concepts, principles, and applications.

### Videos

- [Hypervisor Development YouTube Series](https://www.youtube.com/watch?v=mpPbKEeWIHU&list=PLxN4E629pPnJxCQCLy7E0SQY_zuumOVyZ) -
  Video tutorial series covering hypervisor development concepts and implementation techniques.
- [Virtual Machine Monitor Concepts](https://www.youtube.com/watch?v=ce00rLMPoF8) - Video explaining the fundamental
  concepts behind Virtual Machine Monitors (VMMs) and their implementation.
- [Hypervisor Architecture Overview](https://www.youtube.com/watch?v=-Zc4kv1x22M) - Video providing an overview of
  hypervisor architectures, types, and design considerations.
- [Virtualization Security Considerations](https://www.youtube.com/watch?v=83euuVIvYcM) - Video discussing security
  aspects and considerations in virtualization environments and hypervisor implementations.

## Kernel

### Exokernel

- [BareMetal OS](https://github.com/ReturnInfinity/BareMetal?tab=readme-ov-file) - A 64-bit operating system written in
  Assembly language for x86-64 processors, designed with a minimal footprint and maximum performance.
- [x86 Bare Metal Examples](https://github.com/cirosantilli/x86-bare-metal-examples) - Collection of minimalist x86
  operating system examples to demonstrate bare metal programming concepts.
- [CS630 Operating Systems Course](https://www.cs.usfca.edu/~cruse/cs630f08/) - Educational materials from a University
  of San Francisco course on operating systems focusing on bare metal development.
- [MIT PDOS Exokernel](https://pdos.csail.mit.edu/archive/exo/) - Research project on exokernel architecture from MIT's
  Parallel and Distributed Operating Systems group, focusing on application-level resource management.
- [Barrelfish OS](https://barrelfish.org/documentation.html) - Research operating system implementing a multikernel
  architecture, developed as a collaboration between ETH Zurich and Microsoft Research.

### Multikernel

- [Azalea](https://github.com/oslab-swrc/Azalea) - A multikernel operating system designed for manycore systems,
  focusing on scalability and performance across many CPU cores.
- [Barrelfish Architecture Paper](https://www.trustworthy.systems/publications/nicta_full_text/5618.pdf) - Research
  paper detailing the architecture and design principles of the Barrelfish multikernel operating system.
- [Multiprogramming a 64-bit Multicore Processor (Bhardwaj)](https://www.usenix.org/system/files/osdi21-bhardwaj.pdf) -
  Research paper introducing innovations in multikernel design for modern hardware architectures.
- [OSDI'21 Bhardwaj Presentation](https://www.usenix.org/system/files/osdi21_slides_bhardwaj.pdf) - Presentation slides
  describing multikernel innovations presented at the USENIX OSDI conference.
- [IHK/McKernel](https://ihkmckernel.readthedocs.io/en/latest/) - A lightweight multikernel operating system designed
  for high-performance computing environments.
- [Arrakis OS](https://arrakis.cs.washington.edu/) - Research operating system that reduces kernel involvement by giving
  applications direct access to virtualized hardware.
- [Quest OS](https://www.questos.org/) - A real-time multikernel operating system designed for mixed criticality systems
  with timing constraints.
- [Distributed Shared Persistent Memory (DSPM)](https://dl.acm.org/doi/10.1145/3546591.3547525) - Research paper on
  distributed memory architecture relevant to multikernel operating system design.
- [IX Project](https://ix-project.github.io/ix/) - A research operating system designed for high throughput and low
  latency on multicore servers.
- [Rack OS](https://sourceforge.net/p/rack-os/wiki/Home/) - A distributed operating system project designed for
  rack-scale computing environments.
