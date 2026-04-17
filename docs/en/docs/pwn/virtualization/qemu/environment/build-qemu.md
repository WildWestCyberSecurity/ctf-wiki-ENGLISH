# Downloading and Building QEMU

This article introduces how to build QEMU from source code.

## Obtaining QEMU Source Code

We can go to [QEMU's official website](https://download.qemu.org) to download the source code for the corresponding version:

```shell
$ wget https://download.qemu.org/qemu-7.0.0.tar.xz
$ tar -xf qemu-7.0.0.tar.xz
```

Alternatively, we can obtain it directly from [GitHub](https://github.com/qemu/qemu):

```shell
$ git clone git@github.com:qemu/qemu.git
```

## Building QEMU

First, install some necessary dependencies:

```shell
$ sudo apt -y install ninja-build build-essential zlib1g-dev pkg-config libglib2.0-dev binutils-dev libpixman-1-dev libfdt-dev
```

Next, create a build directory and configure the corresponding build options:

```shell
$ mkdir build && cd build
build$ ../qemu-7.0.0/configure --enable-kvm --target-list=x86_64-softmmu --enable-debug
```

Here we manually specified the following build options:

- `--enable-kvm`: Enable KVM support.
- `--target-list=<architecture name>`: Specify the CPU architecture to build. Here we specify `x86_64-softmmu`, which means we want to build for the x86 64-bit CPU architecture.
- `--enable-debug`: Enable debugging of QEMU.

Then simply run `make`:

```shell
build$ make -j$(nproc)
```

After the build is complete, you will see a new executable `qemu-system_x86-64` in the current directory, which is the QEMU binary itself.

If you want to launch the self-built QEMU from the command line, you can run the `make install` command, which will automatically install it to the `/bin` directory:

```shell
build$ sudo make install
```

## Debugging QEMU

QEMU allows us to debug the virtual machine (e.g., debugging the Linux kernel) through additional parameters like `-s` or `-gdb tcp::1234`. However, sometimes we want to **debug the QEMU binary itself** (e.g., debugging custom emulated devices). In this case, we need to treat the QEMU process on the Host as the debugging target.

Since QEMU is essentially just a process running on the host machine, we can simply find its corresponding PID and directly use `gdb attach` for debugging.

## REFERENCE

[【VIRT.0x00】Qemu - I：Qemu 简易食用指南](https://arttnba3.cn/2022/07/15/VIRTUALIZATION-0X00-QEMU-PART-I/)

[QEMU 源码编译](https://hlyani.github.io/notes/openstack/qemu_make.html)
