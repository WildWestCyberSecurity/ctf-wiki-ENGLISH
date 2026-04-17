# Introduction to Virtualization Technology

What is virtualization? In a narrow sense, when people talk about virtualization in daily life, they mainly refer to _virtual machines_ (Virtual Machine), i.e., **using virtualization technology to virtualize a single computer into multiple logical computers** — this is actually a branch of virtualization technology with an abstraction granularity of a single computer: `system virtualization`.

In computer science, **virtualization** refers to a resource management technique that "**logically abstracts various physical resources of a computer to present different virtual resources**". With virtualization technology, we can break the indivisible nature of physical structures — a single physical resource can be presented to users as multiple virtual resources, and multiple physical resources can also be presented as a single physical resource.

Through virtualization technology, we can achieve dynamic resource allocation, flexible scheduling, cross-domain sharing, and more, thereby improving resource utilization.

![overview.png](./figure/overview.png)

> The physical resources mentioned here include **CPU, memory, disk space, network adapters**, etc.

Here the author quotes a passage from a classic virtualization technology textbook:

> In abstract terms, virtualization is the logical representation of resources, unconstrained by physical limitations. Specifically, virtualization technology is implemented by adding a virtual layer to the system. The virtualization layer abstracts the underlying resources into a different form of resources and provides them to the upper layer for use. Through spatial partitioning, time-sharing, and emulation, virtualization can abstract a single resource into multiple ones. Conversely, virtualization can also abstract multiple resources into a single one.
>
> — "System Virtualization: Principles and Implementation"

The implementation of virtualization technology actually stems from the multi-layered bottom-up abstraction structure of modern computer systems: "**Each layer presents an abstraction to the layer above it, and each layer only needs to know the abstract interface of the layer below, without needing to understand its internal workings**" — it is not hard to imagine that **as long as we can provide the same abstract interface to the upper layer through some means, from the upper layer's perspective we appear to be the normal resources provided by that layer, thus achieving virtualization of that layer.**

![layer.png](./figure/layer.png)

From this, looking at both sides of the physical layer and the virtual layer, we have two important qualifiers in virtualization:

- "**Host**": The physical resource side.
- "**Guest**": The virtual resource side.

Depending on the resource type, we can append different nouns after these two qualifiers: for example, we call a physical machine a `Host Machine`, and the virtual machine running on it a `Guest Machine`; correspondingly, if an operating system runs on the host machine, it is called the `Host OS`, while the operating system running inside the virtual machine is called the `Guest OS`.

From this, we classify virtualization at different abstraction layers as follows:

- **Virtualization at the hardware abstraction layer**: Implementing virtual machines through a virtual hardware abstraction layer, presenting the Guest OS with a hardware abstraction layer identical or similar to the physical hardware. This is also known as "**system-level virtualization**" (e.g., VMWare, Xen).
- Virtualization at the operating system layer: This typically refers to the OS kernel providing multiple isolated user-space instances (commonly called containers). These user-space instances appear to their users as real computers with their own independent networks, file systems, etc. (e.g., VServer).
- Virtualization at the library level: By virtualizing the application-level library function service interfaces of the operating system, applications can run seamlessly across different operating systems without modification (e.g., Wine, WSL).
- Virtualization at the programming language level: This type of virtual machine runs process-level virtual architectures that do not exist in hardware. Program code is **translated** into machine language by the virtual machine's runtime support system before execution, belonging to process-level virtualization (e.g., JVM).

> For example, the VFS in the Linux kernel is a subsystem that fits the concept of virtualization very well: from the perspective of upper-layer calls, what we see are unified API interfaces, while the specific implementations of different file systems are hidden below the VFS layer. We only need to know how to use abstract APIs like open, read, write, etc. at this abstraction layer, without needing to care about the internal implementation of ext4 or ntfs underneath.
>
> Virtualization works the same way. From the Guest side, we can only see the unified virtual resource interfaces, or rather, the Host presents virtualized resource interfaces to us, whose behavior is consistent with physical devices.

The virtualization technology we commonly refer to is mainly **virtualization at the hardware abstraction layer**, i.e., "**system-level virtualization**": using virtualization technology to virtualize a single computer into multiple logical computers.

Depending on the different types of physical resources, we can further subdivide into:

- **Compute virtualization**: Virtualization of CPU and memory resources.
- **Network virtualization**: Virtualization of network link resources.
- **I/O virtualization**: Virtualization of I/O resources.
- **Storage virtualization**: Virtualization of disk storage resources.

## System Virtualization Overview

### Basic Model

Let us first define a virtual machine:

- A **Virtual Machine** (VM) is a virtualized instance of a computer that has its own virtual hardware (such as CPU, memory, devices, etc.) and can perform functions nearly identical to those of a computer, including running applications and operating systems.

For a computer, we can simply abstract it into the three-layer model shown in the figure below. From bottom to top, the layers are the **physical hardware layer, operating system layer, and application layer**. Therefore, we can view a virtual machine instance as a **logical computer** with the layered structure shown in the figure:

![](./figure/computer.png)

However, the operation of a virtual machine requires physical environment support, and virtual machine instances cannot appear or disappear out of thin air. Therefore, we introduce a new concept — **VMM**, i.e., `Virtual Machine Monitor`, also known as `Hypervisor`. This is a software layer between the VM and the hardware, **responsible for creating, destroying VMs, and providing the runtime environment for VMs**: the `virtual hardware abstraction layer`.

![](./figure/virt-hardware-layer.png)

In 1974, Gerald J. Popek and Robert P. Goldberg published their joint paper [*"Formal Requirements for Virtualizable Third Generation Architectures"*](https://dl.acm.org/doi/pdf/10.1145/361011.361073), in which they proposed three sufficient conditions for a VMM that satisfies virtualization system architecture requirements, known as `Popek and Goldberg virtualization requirements`:

- **Equivalence** (essentially identical): A program running under a VMM **should behave identically to the same program running directly on an equivalent physical machine**.
- **Resource control**: The VMM has **complete control** over virtual resources, including resource allocation, monitoring, and reclamation.
- **Efficiency**: The frequently used portion of machine instructions should **execute directly on the hardware without VMM intervention**.

From this, the paper proposed two Hypervisor schemes, which have become the two most mainstream approaches today:

- `Type I`: **The Hypervisor runs directly on the hardware, i.e., the Hypervisor acts as the Host OS to directly manage hardware resources**. For example, `VMware ESXI` is a Hypervisor that uses this architecture.

  ![Type I VMM](./figure/type-i-vmm.png)

- `Type II`: **The Hypervisor runs on a traditional operating system, running in parallel with other applications**. For example, `Qemu` and `VMware Player` are Hypervisors that use this architecture.

  ![Type II VMM](./figure/type-ii-vmm.png)

Now we introduce the concept of `sensitive instructions`, which are **instructions that operate on privileged resources**, such as I/O operations, modifying page table registers, etc. For our VMM to have complete control over system resources, **sensitive instructions must be completed under the supervision of the VMM, or executed by the VMM**. Therefore, if all privileged instructions in an architecture are sensitive instructions, we can use **Ring Compression** to implement the virtual environment:

- The VMM runs at the highest privilege level, and the Guest VM runs at a lower privilege level. When the Guest VM executes a sensitive instruction, it traps into the VMM at the highest privilege level, at which point the VMM can emulate the behavior of the sensitive instruction.

From this, we obtain the classic model of system virtualization — `Trap & Emulate`:

- We divide the operating system into two execution modes: "user mode" and "privileged mode". In user mode, only non-privileged instructions can be executed directly. When a privileged instruction is encountered, it triggers an exception, trapping into the corresponding handler code in privileged mode.
- The Guest VM runs in user mode, so that ordinary instructions can be executed directly on the CPU. When the Guest VM executes a **sensitive instruction**, it **triggers an exception, at which point the VMM intervenes and emulates the expected behavior**.

![](./figure/trap-emulate.png)

Since hardware physical resources also come in different types, we classify virtualization technologies for different types of physical resources as follows:

- CPU virtualization
- Memory virtualization
- I/O virtualization

## REFERENCE

"System Virtualization: Principles and Implementation" — Intel Open Source Technology Center

[【VIRT.0x02】Introduction to System Virtualization](https://arttnba3.cn/2022/08/29/VURTUALIZATION-0X02-BASIC_KNOWLEDGE/)

[Intel® 64 and IA-32 Architectures Software Developer's Manual Volume 3C: System Programming Guide, Part 3](https://cdrdv2.intel.com/v1/dl/getContent/671506)

[Relearning Docker - part1: A Brief Introduction to Virtualization Technology](https://zhuanlan.zhihu.com/p/363922044)

[Formal Requirements for Virtualizable Third Generation Architectures](https://dl.acm.org/doi/pdf/10.1145/361011.361073)
