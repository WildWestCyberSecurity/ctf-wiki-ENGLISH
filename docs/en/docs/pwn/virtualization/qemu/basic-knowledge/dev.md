# QEMU Device Emulation

QEMU independently performs device emulation in user space, and virtual devices are invoked by other VMs through interfaces provided by the hypervisor. Since device emulation is independent of the hypervisor, we can emulate any device, and the emulated device can be shared among other hypervisors.

This chapter describes how QEMU performs device emulation.

## QEMU Device I/O Handling

When a VM accesses the physical memory/port corresponding to a certain virtual device, control is transferred from the VM to the Hypervisor. At this point, QEMU handles the event differently depending on the type of event that triggered the VM-exit.

> accel/kvm/kvm-all.c

```c
int kvm_cpu_exec(CPUState *cpu)
{
    //...

    do {
        //...

        run_ret = kvm_vcpu_ioctl(cpu, KVM_RUN, 0);

        // VCPU exits execution, handle the corresponding event

        trace_kvm_run_exit(cpu->cpu_index, run->exit_reason);
        switch (run->exit_reason) {
        case KVM_EXIT_IO:
            DPRINTF("handle_io\n");
            /* Called outside BQL */
            kvm_handle_io(run->io.port, attrs,
                          (uint8_t *)run + run->io.data_offset,
                          run->io.direction,
                          run->io.size,
                          run->io.count);
            ret = 0;
            break;
        case KVM_EXIT_MMIO:
            DPRINTF("handle_mmio\n");
            /* Called outside BQL */
            address_space_rw(&address_space_memory,
                             run->mmio.phys_addr, attrs,
                             run->mmio.data,
                             run->mmio.len,
                             run->mmio.is_write);
            ret = 0;
            break;
```

### MMIO

For MMIO, `address_space_rw()` is called. This function first flattens the global address space `address_space_memory` into a `FlatView` and then calls the corresponding function to perform the read/write operation.

> softmmu/physmem.c

```c
MemTxResult address_space_read_full(AddressSpace *as, hwaddr addr,
                                    MemTxAttrs attrs, void *buf, hwaddr len)
{
    MemTxResult result = MEMTX_OK;
    FlatView *fv;

    if (len > 0) {
        RCU_READ_LOCK_GUARD();
        fv = address_space_to_flatview(as);
        result = flatview_read(fv, addr, attrs, buf, len);
    }

    return result;
}

MemTxResult address_space_write(AddressSpace *as, hwaddr addr,
                                MemTxAttrs attrs,
                                const void *buf, hwaddr len)
{
    MemTxResult result = MEMTX_OK;
    FlatView *fv;

    if (len > 0) {
        RCU_READ_LOCK_GUARD();
        fv = address_space_to_flatview(as);
        result = flatview_write(fv, addr, attrs, buf, len);
    }

    return result;
}

MemTxResult address_space_rw(AddressSpace *as, hwaddr addr, MemTxAttrs attrs,
                             void *buf, hwaddr len, bool is_write)
{
    if (is_write) {
        return address_space_write(as, addr, attrs, buf, len);
    } else {
        return address_space_read_full(as, addr, attrs, buf, len);
    }
}
```

The operation functions ultimately find the target `MemoryRegion` corresponding to the target memory based on the `FlatView`. For MRs that have read/write pointers defined in their function table, the corresponding function pointer is called to complete the memory access. Since there is too much code, we won't expand further here:

> softmmu/physmem.c

```c
/* Called from RCU critical section.  */
static MemTxResult flatview_write(FlatView *fv, hwaddr addr, MemTxAttrs attrs,
                                  const void *buf, hwaddr len)
{
    hwaddr l;
    hwaddr addr1;
    MemoryRegion *mr;

    l = len;
    mr = flatview_translate(fv, addr, &addr1, &l, true, attrs);
    if (!flatview_access_allowed(mr, attrs, addr, len)) {
        return MEMTX_ACCESS_ERROR;
    }
    return flatview_write_continue(fv, addr, attrs, buf, len,
                                   addr1, l, mr);
}

/* Called from RCU critical section.  */
static MemTxResult flatview_read(FlatView *fv, hwaddr addr,
                                 MemTxAttrs attrs, void *buf, hwaddr len)
{
    hwaddr l;
    hwaddr addr1;
    MemoryRegion *mr;

    l = len;
    mr = flatview_translate(fv, addr, &addr1, &l, false, attrs);
    if (!flatview_access_allowed(mr, attrs, addr, len)) {
        return MEMTX_ACCESS_ERROR;
    }
    return flatview_read_continue(fv, addr, attrs, buf, len,
                                  addr1, l, mr);
}
```

### PMIO

For `PMIO`, the `kvm_handle_io()` function is called. This function is actually a wrapper around `address_space_rw()`, except it uses the **port address space** `address_space_io`. It ultimately also calls the read/write functions in the corresponding `MemoryRegion`'s function table.

```c
static void kvm_handle_io(uint16_t port, MemTxAttrs attrs, void *data, int direction,
                          int size, uint32_t count)
{
    int i;
    uint8_t *ptr = data;

    for (i = 0; i < count; i++) {
        address_space_rw(&address_space_io, port, attrs,
                         ptr, size,
                         direction == KVM_EXIT_IO_OUT);
        ptr += size;
    }
}
```

## QEMU PCI Devices

> Under construction.

## REFERENCE

"QEMU/KVM Source Code Analysis and Applications" — Li Qiang

[【HARDWARE.0x00】PCI Device Practical Guide](https://arttnba3.cn/2022/08/30/HARDWARE-0X00-PCI_DEVICE/)

[【VIRT.0x02】Introduction to System Virtualization](https://arttnba3.cn/2022/08/29/VURTUALIZATION-0X02-BASIC_KNOWLEDGE)
