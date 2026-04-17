# QEMU Memory Management

This section describes how QEMU manages the memory of a specific VM.

## Guest VM Perspective (GPA)

### MemoryRegion: A "Memory" Block from the Guest's Perspective

In QEMU, the `MemoryRegion` structure type is used to represent a specific Guest physical memory region. This structure is defined in `include/exec/memory.h`:

```c
/** MemoryRegion:
 *
 * A structure representing a memory region.
 */
struct MemoryRegion {
    Object parent_obj;

    /* private: */

    /* The following fields should fit in a cache line */
    bool romd_mode;
    bool ram;
    bool subpage;
    bool readonly; /* For RAM regions */
    bool nonvolatile;
    bool rom_device;
    bool flush_coalesced_mmio;
    bool global_locking;
    uint8_t dirty_log_mask;
    bool is_iommu;
    RAMBlock *ram_block;
    Object *owner;

    const MemoryRegionOps *ops;
    void *opaque;
    MemoryRegion *container;	// Pointer to the parent MemoryRegion
    Int128 size;	// Memory region size
    hwaddr addr;	// Offset within the parent MR
    void (*destructor)(MemoryRegion *mr);
    uint64_t align;
    bool terminates;
    bool ram_device;
    bool enabled;
    bool warning_printed; /* For reservations */
    uint8_t vga_logging_count;
    MemoryRegion *alias;	// Only in alias MR, points to the actual MR
    hwaddr alias_offset;
    int32_t priority;
    QTAILQ_HEAD(, MemoryRegion) subregions;
    QTAILQ_ENTRY(MemoryRegion) subregions_link;
    QTAILQ_HEAD(, CoalescedMemoryRange) coalesced;
    const char *name;
    unsigned ioeventfd_nb;
    MemoryRegionIoeventfd *ioeventfds;
};
```

According to the [QEMU official documentation](https://www.qemu.org/docs/master/devel/memory.html#types-of-regions), although numerous memory region objects are all declared as the C structure type MemoryRegion, they can be classified into eight types of MemoryRegion based on their functional characteristics and application scenarios. These eight types of memory regions essentially cover the common memory access needs of the Guest. The eight types include:

- RAM MemoryRegion: Initialized via `memory_region_init_ram()`. Represents a concrete block of Host memory that is allocated for Guest use.
- ROM MemoryRegion: Initialized via `memory_region_init_rom()`. Similar in principle to RAM MR, but used only for reading (equivalent to directly accessing the host memory region), with write operations disabled.
- MMIO MemoryRegion: Initialized via `memory_region_init_io()`. A guest virtual machine memory segment where each read or write ultimately calls callback functions registered in QEMU code, used to emulate the MMIO read/write flow.
- ROM device MemoryRegion: Initialized via `memory_region_init_rom_device()`. Read operations work similarly to RAM MR's direct memory access, while write operations work similarly to MMIO MR's callback function invocation.
- IOMMU MemoryRegion: Initialized via `memory_region_init_iommu()`. As the name suggests, this MR type is used only for IOMMU modeling, not for simple devices. Accessing an address in this type of MR triggers address translation and forwards the access to other target memory regions.
- container MemoryRegion: Initialized via `memory_region_init()`. It does not actually point to a memory region but only contains other MRs, each at a different offset. Container MRs are used to group multiple MRs into a single unit for representation and management. For example, a PCI BAR may consist of a RAM MR and an MMIO MR.
- alias MemoryRegion: Initialized via `memory_region_init_alias()`. It exists as an alias for another MemoryRegion entity and does not point to actual memory.
- reservation MemoryRegion: Initialized by passing a `NULL` callback parameter to `memory_region_init_io()`. It is primarily used for debugging, and the I/O space it occupies should not be handled by QEMU itself. A typical use case is tracking portions of the address space that are handled by the host kernel when KVM is enabled.

Container MRs and other MRs form a tree structure, where the container is the root node and entities are child nodes, as shown below:

```
                            struct MemoryRegion
                            +------------------------+                                         
                            |name                    |                                         
                            |  (const char *)        |                                         
                            +------------------------+                                         
                            |addr                    |                                         
                            |  (hwaddr)              |                                         
                            |size                    |                                         
                            |  (Int128)              |                                         
                            +------------------------+                                         
                            |subregions              |                                         
                            |    QTAILQ_HEAD()       |                                         
                            +------------------------+                                         
                                       |
                                       |
               ----+-------------------+---------------------+----
                   |                                         |
                   |                                         |
                   |                                         |

     struct MemoryRegion                            struct MemoryRegion
     +------------------------+                     +------------------------+
     |name                    |                     |name                    |
     |  (const char *)        |                     |  (const char *)        |
     +------------------------+                     +------------------------+
     |addr                    |                     |addr                    |
     |  (hwaddr)              |                     |  (hwaddr)              |
     |size                    |                     |size                    |
     |  (Int128)              |                     |  (Int128)              |
     +------------------------+                     +------------------------+
     |subregions              |                     |subregions              |
     |    QTAILQ_HEAD()       |                     |    QTAILQ_HEAD()       |
     +------------------------+                     +------------------------+
```

Correspondingly, based on the OOP philosophy, the member functions of MemoryRegion are encapsulated in the function table `MemoryRegionOps`:

```c
/*
 * Memory region callbacks
 */
struct MemoryRegionOps {
    /* Read from the memory region. @addr is relative to @mr; @size is in bytes. */
    uint64_t (*read)(void *opaque,
                     hwaddr addr,
                     unsigned size);
    /* Write to the memory region. @addr is relative to @mr; @size is in bytes. */
    void (*write)(void *opaque,
                  hwaddr addr,
                  uint64_t data,
                  unsigned size);

    MemTxResult (*read_with_attrs)(void *opaque,
                                   hwaddr addr,
                                   uint64_t *data,
                                   unsigned size,
                                   MemTxAttrs attrs);
    MemTxResult (*write_with_attrs)(void *opaque,
                                    hwaddr addr,
                                    uint64_t data,
                                    unsigned size,
                                    MemTxAttrs attrs);

    enum device_endian endianness;
    /* Guest-visible constraints: */
    struct {
        /* If nonzero, specifies the access size bounds beyond which
         * a machine check is raised.
         */
        unsigned min_access_size;
        unsigned max_access_size;
        /* If true, unaligned accesses are supported.  Otherwise unaligned
         * accesses throw machine checks.
         */
         bool unaligned;
        /*
         * If present and returns #false, the transaction is not accepted
         * by the device (and results in machine-dependent behavior such
         * as a machine check exception).
         */
        bool (*accepts)(void *opaque, hwaddr addr,
                        unsigned size, bool is_write,
                        MemTxAttrs attrs);
    } valid;
    /* Internal implementation constraints: */
    struct {
        /* If nonzero, specifies the minimum implemented size.
         * Smaller sizes will be rounded up, and a partial result will be returned.
         */
        unsigned min_access_size;
        /* If nonzero, specifies the maximum implemented size.
         * Larger sizes will be done as a series of accesses with smaller sizes.
         */
        unsigned max_access_size;
        /* If true, unaligned accesses are supported.
         * Otherwise all accesses are converted to (possibly multiple) aligned accesses.
         */
        bool unaligned;
    } impl;
};
```

When our Guest reads/writes memory in the virtual machine, inside QEMU it actually calls `address_space_rw()`. For regular RAM memory, it directly operates on the memory corresponding to the MR. For MMIO, it ultimately calls the corresponding `MR->ops->read()` or `MR->ops->write()`.

Similarly, for interface unification, **PMIO implementation in QEMU is also accomplished through MemoryRegion**. We can think of a group of ports as a block of Guest memory from QEMU's perspective.

> Almost all CTF QEMU Pwn challenges involve defining a custom device with corresponding MMIO/PMIO operations.

### FlatView: The Guest Physical Address Space Corresponding to the MR Tree

QEMU manages the Guest's physical address space through a tree structure of MemoryRegions, supporting dynamic adjustments (such as hot-plugging devices). However, this complex nested and overlapping structure is not suitable for direct interaction with kernel modules like KVM. Therefore, QEMU uses `FlatView` to represent **the linear Guest address space represented by a MemoryRegion tree**. It "flattens" the tree structure into a list where each entry records the start address (GPA), size, and attributes (such as RAM/MMIO) of a contiguous memory region, eliminating nesting relationships and simplifying kernel processing. For example, KVM needs an explicit physical memory layout to configure EPT (Extended Page Table), and `FlatView` provides a flat memory description that can be directly mapped, avoiding the need for the kernel to parse complex tree structures. `FlatView` uses an array of `FlatRange` structure pointers to store the address information corresponding to different `MemoryRegion`s. Each `FlatRange` represents **a single MemoryRegion's linear physical address space from the Guest's perspective** along with properties such as whether it is read-only. The address ranges represented by different `FlatRange`s do not overlap.

```c
/* Range of memory in the global map.  Addresses are absolute. */
struct FlatRange {
    MemoryRegion *mr;
    hwaddr offset_in_region;
    AddrRange addr;
    uint8_t dirty_log_mask;
    bool romd_mode;
    bool readonly;
    bool nonvolatile;
};

//...

/* Flattened global view of current active memory hierarchy.  Kept in sorted
 * order.
 */
struct FlatView {
    struct rcu_head rcu;
    unsigned ref;
    FlatRange *ranges;
    unsigned nr;
    unsigned nr_allocated;
    struct AddressSpaceDispatch *dispatch;
    MemoryRegion *root;
};
```

### AddressSpace: Different Types of Guest Address Spaces

The `AddressSpace` structure represents **different types of address spaces from the Guest's perspective**. On x86, there are essentially only two: `address_space_memory` and `address_space_io`.

A single `AddressSpace` structure is associated with the root node of a MemoryRegion tree and uses a `FlatView` structure to establish the flattened memory space of that tree.

```c
/**
 * struct AddressSpace: describes a mapping of addresses to #MemoryRegion objects
 */
struct AddressSpace {
    /* private: */
    struct rcu_head rcu;
    char *name;
    MemoryRegion *root;

    /* Accessed via RCU.  */
    struct FlatView *current_map;

    int ioeventfd_nb;
    struct MemoryRegionIoeventfd *ioeventfds;
    QTAILQ_HEAD(, MemoryListener) listeners;
    QTAILQ_ENTRY(AddressSpace) address_spaces_link;
};
```

Ultimately, we can obtain the following overview diagram:

![](./figure/qemu_mm.png)

## Host VMM Perspective (HVA)

### RAMBlock: Host Virtual Memory Corresponding to an MR

The `RAMBlock` structure represents **the Host virtual memory information occupied by a single entity MemoryRegion**. Multiple `RAMBlock` structures form a singly linked list.

The more important members are:

- `mr`: The MemoryRegion corresponding to this RAMBlock (i.e., HVA → GPA)
- `host`: The HVA corresponding to the GVA, typically obtained by QEMU through `mmap()` (if KVM is not used)

```c
struct RAMBlock {
    struct rcu_head rcu;
    struct MemoryRegion *mr;
    uint8_t *host;
    uint8_t *colo_cache; /* For colo, VM's ram cache */
    ram_addr_t offset;
    ram_addr_t used_length;
    ram_addr_t max_length;
    void (*resized)(const char*, uint64_t length, void *host);
    uint32_t flags;
    /* Protected by iothread lock.  */
    char idstr[256];
    /* RCU-enabled, writes protected by the ramlist lock */
    QLIST_ENTRY(RAMBlock) next;
    QLIST_HEAD(, RAMBlockNotifier) ramblock_notifiers;
    int fd;
    size_t page_size;
    /* dirty bitmap used during migration */
    unsigned long *bmap;
    /* bitmap of already received pages in postcopy */
    unsigned long *receivedmap;

    /*
     * bitmap to track already cleared dirty bitmap.  When the bit is
     * set, it means the corresponding memory chunk needs a log-clear.
     * Set this up to non-NULL to enable the capability to postpone
     * and split clearing of dirty bitmap on the remote node (e.g.,
     * KVM).  The bitmap will be set only when doing global sync.
     *
     * It is only used during src side of ram migration, and it is
     * protected by the global ram_state.bitmap_mutex.
     *
     * NOTE: this bitmap is different comparing to the other bitmaps
     * in that one bit can represent multiple guest pages (which is
     * decided by the `clear_bmap_shift' variable below).  On
     * destination side, this should always be NULL, and the variable
     * `clear_bmap_shift' is meaningless.
     */
    unsigned long *clear_bmap;
    uint8_t clear_bmap_shift;

    /*
     * RAM block length that corresponds to the used_length on the migration
     * source (after RAM block sizes were synchronized). Especially, after
     * starting to run the guest, used_length and postcopy_length can differ.
     * Used to register/unregister uffd handlers and as the size of the received
     * bitmap. Receiving any page beyond this length will bail out, as it
     * could not have been valid on the source.
     */
    ram_addr_t postcopy_length;
};
```

The corresponding relationship is shown in the figure below:

![](./figure/mr_ramblock_subregion.png)

## REFERENCE

[QEMU Official Documentation](https://www.qemu.org/docs/master/devel/memory.html#types-of-regions)

[understanding qemu - MemoryRegion](https://richardweiyang-2.gitbook.io/understanding_qemu/00-as/02-memoryregion)

[QEMU Memory Analysis (Part 1): Key Structures for Memory Virtualization](https://www.cnblogs.com/edver/p/14470706.html)

[QEMU Memory Emulation](https://66ring.github.io/2021/04/13/universe/qemu/qemu_softmmu/)

[QEMU Memory Model](https://richardweiyang-2.gitbook.io/kernel-exploring/00-kvm/01-memory_virtualization/01_1-qemu_memory_model)

[【VIRT.0x00】Qemu - I: Qemu Practical Guide](https://arttnba3.cn/2022/07/15/VIRTUALIZATION-0X00-QEMU-PART-I/)
