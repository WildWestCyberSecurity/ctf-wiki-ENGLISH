# Writing QEMU Emulated Devices

This section introduces how to write a new emulated device in QEMU.

> Note 1: Before getting started, you may need to review some [PCI device basics](https://arttnba3.cn/2022/08/30/HARDWARE-0X00-PCI_DEVICE/)
>
> Note 2: QEMU officially provides an educational device example in `hw/misc/edu.c`, and Red Hat provides a test device in `hw/misc/pci-testdev.c`. We can refer to these two devices to build our own device.

## QEMU Object Model

In QEMU, there is a system called the **QEMU Object Model** for implementing object-oriented programming. It mainly consists of these four components:

- `Type`: Used to define the basic attributes of a "class", such as the class name, size, constructor, etc.
- `Class`: Used to define the static content of a "class", such as static data stored in the class, method function pointers, etc.
- `Object`: A dynamically allocated concrete instance of a "class", storing the dynamic data of the class.
- `Property`: An accessor for dynamic object data, which can be inspected through the monitor interface.

Similar to Golang, QOM uses member embedding to achieve class inheritance, where the parent class exists as the first member `parent` of the class structure. Therefore, multiple inheritance is not supported.

> See [this presentation](https://www.linux-kvm.org/images/f/f6/2012-forum-QOM_CPU.pdf)

### I. TypeInfo - Basic Attributes of a Class

The `TypeInfo` structure is used to define the basic attributes of a `class`. This structure is defined in `include/qom/object.h`:

```c
/**
 * TypeInfo:
 * @name: The name of the type.
 * @parent: The name of the parent type.
 * @instance_size: The size of the object (derivative of #Object).
 *   If @instance_size is 0, then the size of the object will be the size of the parent object.
 * @instance_init: This function is called to initialize an object (i.e., constructor).
 *   The parent class will already be initialized, so the subclass only needs to initialize its own members.
 * @instance_post_init: This function is called to finish initialization of an object,
 *   after all @instance_init functions have been called.
 * @instance_finalize: This function is called during object destruction. It is
 *   called before the parent class's @instance_finalize.
 *   An object should only free its own members in this function.
 * @abstract: If this field is true, the class is abstract and cannot be directly instantiated.
 * @class_size: The size of the class object of this object (derivative of #Object).
 *   If @class_size is 0, then the size of the class will be the size of the parent class.
 *   This allows a type to avoid implementing an explicit class when no additional virtual functions are added.
 * @class_init: This function is called after all parent classes have been initialized,
 *   to allow a class to set its default virtual method pointers.
 *   This also allows this function to override virtual methods of the parent class.
 * @class_base_init: This function is called for all base classes after all parent classes
 *   have been initialized but before the class itself is initialized.
 *   This function is used to undo the effects of memcpy from parent class to subclass.
 * @class_data: Data passed to @class_init and @class_base_init,
 *   which is useful when building dynamic types.
 * @interfaces: The interfaces associated with this type.
 *   It should point to a static array terminated by a zero-filled element.
 */
struct TypeInfo
{
    const char *name;
    const char *parent;

    size_t instance_size;
    void (*instance_init)(Object *obj);
    void (*instance_post_init)(Object *obj);
    void (*instance_finalize)(Object *obj);

    bool abstract;
    size_t class_size;

    void (*class_init)(ObjectClass *klass, void *data);
    void (*class_base_init)(ObjectClass *klass, void *data);
    void *class_data;

    InterfaceInfo *interfaces;
};
```

When we want to define a **class** in QEMU, we actually need to define a variable of type `TypeInfo`. The following is an example of defining a custom class in QEMU:

```c
static const TypeInfo a3_type_info = {
    .name = "a3_type",
    .parent = TYPE_OBJECT,
    .interfaces = (InterfaceInfo[]) {
        { },
    },
}

static void a3_register_types(void) {
    type_register_static(&a3_type_info);
}

type_init(a3_register_types);
```

`type_init()` is essentially a wrapper around the `constructor` gcc attribute. Its purpose is to add a function to an `init_array`. When the QEMU program starts, before entering the main function, it will first call the functions in the `init_array`. Therefore, our custom function will be called here, whose purpose is to call `type_register_static()` to register our custom type `a3_type_info` into the global type table.

### II. Class - Static Content of a Class

After defining a class through a `TypeInfo` structure, we also need to define a Class structure to define the static content of this class, including function tables, static members, etc. It should inherit from the corresponding Class structure type. For example, if we want to define a new machine class, its Class should inherit from `MachineClass`.

The ultimate parent of all Class structure types is the `ObjectClass` structure:

```c
/**
 * ObjectClass:
 *
 * The base class for all classes. #ObjectClass contains only an integer type handler.
 */
struct ObjectClass
{
    /*< private >*/
    Type type;
    GSList *interfaces;

    const char *object_cast_cache[OBJECT_CLASS_CAST_CACHE];
    const char *class_cast_cache[OBJECT_CLASS_CAST_CACHE];

    ObjectUnparent *unparent;

    GHashTable *properties;
};
```

Below is a simplest example:

```c
struct A3Class
{
    /*< private >*/
    ObjectClass parent;
}
```

After completing the Class definition, we should also add the Class size and Class constructor to the previously defined `a3_type_info`:

```c
static void a3_class_init(ObjectClass *oc, void *data)
{
    // The oc parameter here is the newly created Class; there is only one global instance
    // We should cast it to our own Class type, then perform the corresponding operations
    // do something
}

static const TypeInfo a3_type_info = {
    .name = "a3_type",
    .parent = TYPE_OBJECT,
    .class_size = sizeof(A3Class),
    .class_init = a3_class_init,
    .interfaces = (InterfaceInfo[]) {
        { },
    },
}
```

### III. Object - Class Instance Object

We also need to define a corresponding Object type to represent an instance object. It contains the actual specific data of the class and should inherit from the corresponding Object structure type. For example, if we want to define a new machine type, its instance type should inherit from `MachineState`.

The ultimate parent of all Object structure types is the `Object` structure:

```c
/**
 * Object:
 *
 * The base class for all objects. The first member of this object is a pointer to #ObjectClass.
 * Since C guarantees that the first member of a structure is organized at the 0-byte offset,
 * as long as any subclass places its parent class as the first member, we can directly cast it to an #Object.
 *
 * Therefore, #Object contains a reference to the object's class as its first member.
 * This allows identifying the real type of an object at runtime.
 */
struct Object
{
    /*< private >*/
    ObjectClass *class;
    ObjectFree *free;
    GHashTable *properties;
    uint32_t ref;
    Object *parent;
};
```

Below is an example:

```c
struct A3Object
{
    /*< private >*/
    Object parent;
}
```

After completing the Object definition, we should also add the Object size and Object constructor to the previously defined `a3_type_info`:

```c
static void a3_object_init(Object *obj)
{
    // The obj parameter here is the dynamically created type instance
    // do something
}

static const TypeInfo a3_type_info = {
    .name = "a3_type",
    .parent = TYPE_OBJECT,
    .instance_init = a3_object_init,
    .instance_size = sizeof(A3Object),
    .class_size = sizeof(A3Class),
    .class_init = a3_class_init,
    .interfaces = (InterfaceInfo[]) {
        { },
    },
}
```

### IV. Class Creation and Destruction

Similar to using `new` and `delete` in C++ to create and destroy a class instance, in QOM we should use `object_new()` and `object_delete()` to create and destroy a QOM class instance. This is essentially `allocating/freeing class space + explicitly calling the constructor/destructor`.

QOM determines the type of class instance to create by the class name, i.e., `TypeInfo->name`. When creating a class instance, QEMU traverses all TypeInfo entries and finds the one with the matching name, then calls the corresponding constructor and points its base class `Object->class` to the corresponding class.

Below is an example:

```c
// create a QOM object
A3Object *a3obj = object_new("a3_type");
// delete a QOM object
object_delete(a3obj);
```

## Writing PCI Devices in QEMU

With all this QEMU-related knowledge covered, we can now start writing PCI devices in QEMU. Here the author will write a simplest QEMU device, with the source code placed in `hw/misc/a3dev.c`.

The base class for PCI device instances in QEMU is `PCIDevice`, so we should create a class that inherits from `PCIDevice` to represent our device instance. Here the author only declares two `MemoryRegion`s for MMIO and PMIO, and a buffer for data storage:

```c
#define A3DEV_BUF_SIZE 0x100

typedef struct A3PCIDevState {
    /*< private >*/
    PCIDevice parent_obj;

    /*< public >*/
    MemoryRegion mmio;
    MemoryRegion pmio;
    uint8_t buf[A3DEV_BUF_SIZE];
} A3PCIDevState;
```

And define an empty Class template, inheriting from the PCI device's static type `PCIDeviceClass`. However, this step is not required — in fact, we can directly use `PCIDeviceClass` as the Class for our device class:

```c
typedef struct A3PCIDevClass {
    /*< private >*/
    PCIDeviceClass parent;
} A3PCIDevClass;
```

And two macros for casting parent class to subclass, because QOM basic functions mostly pass parent class pointers, so we need macros for type checking + casting. This is also a common practice in QEMU:

```c
#define TYPE_A3DEV_PCI "a3dev-pci"
#define A3DEV_PCI(obj) \
    OBJECT_CHECK(A3PCIDevState, (obj), TYPE_A3DEV_PCI)
#define A3DEV_PCI_GET_CLASS(obj) \
    OBJECT_GET_CLASS(A3PCIDevClass, obj, TYPE_A3DEV_PCI)
#define A3DEV_PCI_CLASS(klass) \
    OBJECT_CLASS_CHECK(A3PCIDevClass, klass, TYPE_A3DEV_PCI)
```

Now let's define the MMIO and PMIO operation functions. Here the author simply sets them to read/write the device's internal buffer, and declares the function tables for the two MemoryRegions. Note that the `hwaddr` type parameter passed here is actually a relative address, not an absolute address:

```c
static uint64_t
a3dev_read(void *opaque, hwaddr addr, unsigned size)
{
    A3PCIDevState *ds = A3DEV_PCI(opaque);
    uint64_t val = ~0LL;

    if (size > 8)
        return val;

    if (addr + size > A3DEV_BUF_SIZE)
        return val;
    
    memcpy(&val, &ds->buf[addr], size);
    return val;
}

static void
a3dev_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    A3PCIDevState *ds = A3DEV_PCI(opaque);

    if (size > 8)
        return ;

    if (addr + size > A3DEV_BUF_SIZE)
        return ;
    
    memcpy(&ds->buf[addr], &val, size);
}

static uint64_t
a3dev_mmio_read(void *opaque, hwaddr addr, unsigned size)
{
    return a3dev_read(opaque, addr, size);
}

static uint64_t
a3dev_pmio_read(void *opaque, hwaddr addr, unsigned size)
{
    return a3dev_read(opaque, addr, size);
}

static void
a3dev_mmio_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    a3dev_write(opaque, addr, val, size);
}

static void
a3dev_pmio_write(void *opaque, hwaddr addr, uint64_t val, unsigned size)
{
    a3dev_write(opaque, addr, val, size);
}

static const MemoryRegionOps a3dev_mmio_ops = {
    .read = a3dev_mmio_read,
    .write = a3dev_mmio_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
};

static const MemoryRegionOps a3dev_pmio_ops = {
    .read = a3dev_pmio_read,
    .write = a3dev_pmio_write,
    .endianness = DEVICE_LITTLE_ENDIAN,
};
```

Next is the device instance initialization function. `PCIDeviceClass` defines a function pointer named `realize` that is called to initialize the PCI device when it is loaded. So here we also define our own initialization function. The work we need to do is basically just initializing two `MemoryRegion`s. `memory_region_init_io()` initializes these two `MemoryRegion`s and sets the function table to our specified function table, while `pci_register_bar()` is used to register BARs:

```c
static void a3dev_realize(PCIDevice *pci_dev, Error **errp)
{
    A3PCIDevState *ds = A3DEV_PCI(pci_dev);

    memory_region_init_io(&ds->mmio, OBJECT(ds), &a3dev_mmio_ops,
                        pci_dev, "a3dev-mmio", A3DEV_BUF_SIZE);
    pci_register_bar(pci_dev, 0, PCI_BASE_ADDRESS_SPACE_MEMORY, &ds->mmio);
    memory_region_init_io(&ds->pmio, OBJECT(ds), &a3dev_pmio_ops,
                        pci_dev, "a3dev-pmio", A3DEV_BUF_SIZE);
    pci_register_bar(pci_dev, 1, PCI_BASE_ADDRESS_SPACE_IO, &ds->pmio);
}
```

Finally, the Class and Object (i.e., instance) initialization functions. Note that in the Class initialization function, we should set a series of basic attributes of the parent class `PCIDeviceClass` (which are the basic attributes of the PCI device):

```c
static void a3dev_instance_init(Object *obj)
{
    // do something
}

static void a3dev_class_init(ObjectClass *oc, void *data)
{
    DeviceClass *dc = DEVICE_CLASS(oc);
    PCIDeviceClass *pci = PCI_DEVICE_CLASS(oc);

    pci->realize = a3dev_realize;
    pci->vendor_id = PCI_VENDOR_ID_QEMU;
    pci->device_id = 0x1919;
    pci->revision = 0x81;
    pci->class_id = PCI_CLASS_OTHERS;

    dc->desc = "arttnba3 test PCI device";
    set_bit(DEVICE_CATEGORY_MISC, dc->categories);
}
```

Last is registering the TypeInfo for our PCI device type. Don't forget that **we should add the PCI interface to our interfaces**:

```c
static const TypeInfo a3dev_type_info = {
    .name = TYPE_A3DEV_PCI,
    .parent = TYPE_PCI_DEVICE,
    .instance_init = a3dev_instance_init,
    .instance_size = sizeof(A3PCIDevState),
    .class_size = sizeof(A3PCIDevClass),
    .class_init = a3dev_class_init,
    .interfaces = (InterfaceInfo[]) {
        { INTERFACE_CONVENTIONAL_PCI_DEVICE },
        { },
    },
};

static void a3dev_register_types(void) {
    type_register_static(&a3dev_type_info);
}

type_init(a3dev_register_types);
```

Finally, we add our new device to the meson build system. Add the following statement in `hw/misc/meson.build`:

```meson
softmmu_ss.add(when: 'CONFIG_PCI_A3DEV', if_true: files('a3dev.c'))
```

And add the following content in `hw/misc/Kconfig`, which means our device will be compiled when `CONFIG_PCI_DEVICES=y`:

```kconfig
config PCI_A3DEV
    bool
    default y if PCI_DEVICES
    depends on PCI
```

After compiling QEMU and appending `-device a3dev-pci`, then booting any Linux system, we can see our newly added PCI device using the `lspci` command:

![Viewing the newly added PCI device using lspci](./figure/new-qemu-dev-lspci.png)

We can use the following programs to test our device's input/output. Note that this requires root privileges:

> PMIO: After using iopl to change port permissions, you can read/write ports through in/out instructions

```c
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/io.h>

int main(int argc, char **argv, char **envp)
{
        unsigned short port_addr;

        if (argc < 2) {
                puts("[x] no port provided!");
                exit(EXIT_FAILURE);
        }

        if (iopl(3) < 0) {
                puts("[x] no privilege!");
                exit(EXIT_FAILURE);
        }

        port_addr = atoi(argv[1]);

        printf("[+] a3dev port addr start at: %d\n", port_addr);

        puts("[*] now writing into a3dev-pci...");

        for (int i = 0; i < 0x100 / 4; i++) {
                outl(i, port_addr + i * 4);
        }

        puts("[+] writing done!");

        printf("[*] now reading from a3dev-pci...");
        for (int i = 0; i < 0x100 / 4; i++) {
                if (i % 8 == 0) {
                        printf("\n[--%d--]", port_addr + i * 4);
                }
                printf(" %d ", inl(port_addr + i * 4));
        }

        puts("\n[+] reading done!");
}
```

> MMIO: Use mmap to map the device's `resource0` file under the `sys` directory for direct read/write access

```c
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdint.h>
#include <sys/mman.h>

void mmio_write(uint32_t *addr, uint32_t val)
{
        *addr = val;
}

uint32_t mmio_read(uint32_t *addr)
{
        return *addr;
}

int main(int argc, char **argv, char **envp)
{
        uint32_t *mmio_addr;
        int dev_fd;

        dev_fd = open("/sys/devices/pci0000:00/0000:00:04.0/resource0",
                        O_RDWR | O_SYNC);
        if (dev_fd < 0) {
                puts("[x] failed to open mmio file! wrong path or no root!");
                exit(EXIT_FAILURE);
        }

        mmio_addr = (uint32_t*)
                mmap(0, 0x1000, PROT_READ | PROT_WRITE, MAP_SHARED, dev_fd, 0);
        if (mmio_addr == MAP_FAILED) {
                puts("failed to mmap!");
                exit(EXIT_FAILURE);
        }

        puts("[*] start writing to a3dev-pci...");
        for (int i = 0; i < 0x100 / 4; i++) {
                mmio_write(mmio_addr + i, i);
        }
        puts("[+] write done!");

        printf("[*] start reading from a3dev-pci...");
        for (int i = 0; i < 0x100 / 4; i++) {
                if (i % 8 == 0) {
                        printf("\n[--%p--]", mmio_addr);
                }
                printf(" %u ", mmio_read(mmio_addr + i));
        }
        puts("\n[+] read done!");
}
```

## REFERENCE

[【VIRT.0x00】Qemu - I: Qemu Practical Guide](https://arttnba3.cn/2022/07/15/VIRTUALIZATION-0X00-QEMU-PART-I/)

[QOM Vadis?Taking Objects To The CPU And Beyond](https://www.linux-kvm.org/images/f/f6/2012-forum-QOM_CPU.pdf)

[Emulating Devices in QEMU - Zhihu](https://zhuanlan.zhihu.com/p/57526565)
