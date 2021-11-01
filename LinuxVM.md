# Running a Linux guest VM

This tutorial will teach you to run a VM with an unmodified version of Linux as
a guest on L4Re. L4Re can act in the hypervisor capacity on several platforms,
but in this tutorial we will use the QEMU virtual machine of the
*qemu-system-aarch64* target.

## Terminology

In some parts of this text we are using the term *aarch64* to refer to the
64-bit ARM architecture, while in some other parts we use *arm64* or even
*ARMv8* to refer to the same. This may seem somewhat arbitrary and confusing at
the same time, but is to a large degree dictated by the conventions of the
components at hand. QEMU and toolchain-related uses typically require *aarch64*
while Fiasco and L4Re typically refer to *arm64*. The bottom line is that these
terms are largely interchangeable and should not confuse you.

## Prerequisities

You will need several ingredients:

  * aarch64 toolchain
  * L4Re sources
  * Linux kernel sources
  * Linux aarch64 ramdisk
  * QEMU 3.1 with aarch64 support

### aarch64 toolchain

As you will be building for aarch64, you are going to need the aarch64 build
tools for both gcc and g++.  The easiest option is to get a ready-to-use
toolchain from
[Linaro](https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/)
or from your distribution. Examples in this tutorial were tested with Linaro GCC
6.3-2017.05 and 7.3-2018.05 and assume that a functionally equivalent toolchain
is installed in some directory where the shell can find it. You can verify that
your toolchain is ready by running the following command:

    [somedir] $ aarch64-linux-gnu-gcc --version
    aarch64-linux-gnu-gcc (Linaro GCC 7.3-2018.05) 7.3.1 20180425 [linaro-7.3-2018.05 revision d29120a424ecfbc167ef90065c0eeb7f91977701]
    Copyright (C) 2017 Free Software Foundation, Inc.
    This is free software; see the source for copying conditions.  There is NO
    warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.

If your output looks very different (i.e. you get an error instead), then there
is likely something wrong with your setup.

### Preparing L4Re sources

The following assumes you are familiar with the procedures described in the
[Building L4Re](BUILDING) tutorial and that you have successfully used *ham* to
check out L4Re sources into a top-level directory consistently called *somedir*
throughout this text (but you can use a different name).

Start by making sure you have the latest sources:

    [somedir] $ ham sync

You should end up with a source tree that contains at least the following
components:

    fiasco
    l4/pkg
    ├── bootstrap
    ├── drivers-frst
    ├── l4re-core
    ├── l4virtio
    ├── libfdt
    ├── libvcpu
    └── uvmm

Continue by configuring the Fiasco microkernel:

    [somedir] $ cd fiasco
    [somedir/fiasco] $ make B=../build-fiasco-aarch64
    [somedir/fiasco] $ cd ../build-fiasco-aarch64
    [somedir/build-fiasco-aarch64] $ make config

In *Target configuration*, make sure to select:

  * Architecture: ARM processor family
  * Platform: QEMU ARM Virtual Platform
  * CPU: ARM Cortex-A57 CPU
  * Virtualization: enable

In *Debugging*, make sure the following is NOT selected:

  * JDB Kernel Debugger

Save the configuration and build Fiasco:

    [somedir/build-fiasco-aarch64] $ make -j 6

Once Fiasco is built, move on to configuring L4Re:

    [somedir/build-fiasco-aarch64] $ cd ../l4
    [somedir/l4] $ make B=../build-aarch64
    [somedir/l4] $ cd ../build-aarch64
    [somedir/build-aarch64] $ make config

Make sure to select:

  * Target Architecture: ARM64 Architecture (AArch64)
  * CPU Variant: ARMv8-A Type CPU
  * Platform Selection: QEMU ARM Virtual Platform

Once configured, build L4Re like this:

    [somedir/build-aarch64] $ make -j 6
    [somedir/build-aarch64] $ cd ..

Finishing this step will give you the Fiasco binary, the L4Re binaries and the
device tree binary. You will need all of those later.

### Preparing Linux sources

In the following we use Linux 4.19, but any reasonably new version of Linux will
do.

Create a build directory for an out-of-tree Linux build:

    [somedir] $ mkdir build-linux-aarch64

Afterwards proceed to clone Linux sources using git:

    [somedir] $ git clone git://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
    [somedir] $ cd linux
    [somedir/linux] $ git checkout v4.19     # or any other reasonable version

Alternatively, download Linux 4.19 tarball and work with that instead:

    [somedir] $ curl https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.19.8.tar.xz | tar xvfJ -
    [somedir] $ ln -s linux-4.19.8/ linux
    [somedir] $ cd linux

Now it's time to configure and build Linux. Default configuration should be fine:

    [somedir/linux] $ make O=../build-linux-aarch64 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig
    [somedir/linux] $ make O=../build-linux-aarch64 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j 6
    [somedir/linux] $ cd ..

This should produce a compressed kernel image in
*somedir/build-linux-aarch64/arch/arm64/boot/Image.gz*. You will need that later.

### Getting a Linux ramdisk

Besides the Linux kernel itself, you are also going to need an initial ramdisk
with some programs on it. You have basically two options: either build your own
ramdisk from scratch or use a ramdisk that someone else made for you. Both
options are explained below. In any case, create a directory for the resulting
ramdisk:

    [somedir] $ mkdir ramdisk

#### Downloading an existing ramdisk

If you prefer to use a pre-built Busybox-based ramdisk, you can use one from TU
Dresden:

    [somedir] $ curl http://os.inf.tu-dresden.de/~adam/dl/ramdisks/ramdisk-armv8-64.cpio.gz >ramdisk/ramdisk-armv8-64.cpio.gz

#### Building your own ramdisk

If you prefer to build the ramdisk yourself, the following instructions will
teach you to build a ramdisk based on [Busybox](https://www.busybox.net/).

First, create a busybox build directory:

    [somedir] $ mkdir build-busybox-aarch64

Continue by cloning Busybox repository using git:

    [somedir] $ git clone https://git.busybox.net/busybox
    [somedir] $ cd busybox
    [somedir/busybox] $ git checkout 1_29_stable     # or another reasonable version

Alternatively, get a Busybox tarball:

    [somedir] $ curl https://busybox.net/downloads/busybox-1.29.3.tar.bz2 | tar xvfj -
    [somedir] $ ln -s busybox-1.29.3/ busybox
    [somedir] $ cd busybox

Then start configuring it:

    [somedir/busybox] $ make O=../build-busybox-aarch64 defconfig
    [somedir/busybox] $ make O=../build-busybox-aarch64 menuconfig

In menuconfig, go to *Settings* and in the *Build Options* section enable:

  * Build static binary (no shared libs)

Save the configuration and proceed to build and install:

    [somedir/busybox] $ cd ../build-busybox-aarch64
    [somedir/build-busybox-aarch64] $ make CROSS_COMPILE=aarch64-linux-gnu- install -j 6

What remains to be done is to create a compressed CPIO archive from the contents
of the *_install* directory:

    [somedir/build-busybox-aarch64] $ cd _install
    [somedir/build-busybox-aarch64/_install] $ find . | cpio -H newc -o | gzip > ../../ramdisk/ramdisk-armv8-64.cpio.gz
    [somedir/build-busybox-aarch64/_install] $ cd ../..

### Preparing QEMU

QEMU got support for aarch64 virtualization in version 3.1 (December 2018), so
make sure your QEMU is not older than that. Your distribution might provide
ready-to-use QEMU packages with aarch64 support, but building your own aarch64
QEMU is also an option.

Start by creating a build directory:

    [somedir] $ mkdir build-qemu-aarch64

Then clone the QEMU git repository:

    [somedir] $ git clone git://git.qemu.org/qemu.git
    [somedir] $ cd qemu
    [somedir/qemu] $ git checkout v3.1.0          # or any release >=v3.1.0
    [somedir/qemu] $ cd ../build-qemu-aarch64

Or download a QEMU tarball:

    [somedir] $ curl https://download.qemu.org/qemu-3.1.0.tar.xz | tar xvfJ -
    [somedir] $ ln -s qemu-3.1.0/ qemu
    [somedir] $ cd build-qemu-aarch64

In either case, you can now configure QEMU:

    [somedir/build-qemu-aarch64] $ ../qemu/configure --target-list=aarch64-softmmu
    [somedir/build-qemu-aarch64] $ make -j 6
    [somedir/build-qemu-aarch64] $ make install   # make sure to run with sufficient privileges
    [somedir/build-qemu-aarch64] $ cd ..

This will install qemu-system-aarch64 into */usr/local/bin*. Make sure it is in
your PATH.

## Putting it all together

Unlike the simple *hello* example in the BUILDING tutorial, running a VM with a
Linux guest is more involved and requires a little bit of configuration.  Now
that all the prerequisities were installed, built or downloaded, you are ready to
put everything together. First of all, you need to create a directory for
configuration files:

    [somedir] $ mkdir conf
    [somedir] $ cd conf

### Writing a basic ned script

In L4Re, components of the resulting system are described and connected together
in a so-called ned script. The name is dervied from *ned*, which is an init
process component that spawns other components and interconnects them via IPC
channels (i.e. IPC gate capabilities) according to a Lua configuration file
passed to it as an argument.

For our scenario you need to write your own ned script. In the configuration
directory, create a file called *uvmm-basic.ned* with the following content:

````lua
local L4 = require "L4";

L4.default_loader:startv({
  log = L4.Env.log,
  caps = {
    ram  = L4.Env.user_factory:create(
      L4.Proto.Dataspace,
      128 * 1024 * 1024,                   -- size in MB
      L4.Mem_alloc_flags.Continuous |
        L4.Mem_alloc_flags.Pinned |
        L4.Mem_alloc_flags.Super_pages,
      21                                   -- alignment
    ):m("rws");
  }
}, "rom/uvmm",
  "-i",  -- place guest RAM using the host-physical addresses of the backing memory
  "--dtb", "rom/virt-arm_virt-64.dtb",
  "--ramdisk", "rom/ramdisk-armv8-64.cpio.gz", "--kernel", "rom/Image.gz",
  "--cmdline", "console=hvc0 earlyprintk=1 rdinit=/bin/sh");
````

This ned script is as basic as it gets. It only supports a single VM and does
not pass any host I/O devices to the VM, but already contains everything
necessary for doing just that without additional bells and whistles. More
complex scenarios (e.g. involving multiple VMs) will require a more complex ned
script and possibly also additional L4Re components.  For these cases, L4Re
provides a convenience wrapper. You shall see in a later example how to use it.

### Specifying boot modules

For the above ned script, you also need to add an entry to
*somedir/l4/conf/modules.list* so that *make E=uvmm-basic qemu* below knows
what binaries and config files to load. Open the file and append the following
lines:

````
entry uvmm-basic
kernel fiasco -serial_esc
roottask moe rom/uvmm-basic.ned
module uvmm
module l4re
module ned
module virt-arm_virt-64.dtb
module ramdisk-armv8-64.cpio.gz
module uvmm-basic.ned
module[uncompress] Image.gz
````

### Creating Makeconf.boot

In order to save yourself from typing (or copy-and-pasting) ridiculously long
command lines full of variable definitions, you need to do one last thing before
starting the VM. Go to *somedir/l4/conf* and copy *Makeconf.boot.example* to
*Makeconf.boot* (assuming it does not exist yet). Then edit *Makeconf.boot* and
make sure that:

  * somedir/build-fiasco-aarch64
  * somedir/ramdisk
  * somedir/conf
  * somedir/build-linux-aarch64/arch/arm64/boot/

are all included in the definition of *MODULE_SEARCH_PATH*. The individual paths
need to be absolute and spearated by colons. An example definition of
*MODULE_SEARCH_PATH* might look as follows. Just make sure to change *SOMEDIR*
according to your environment:

    SOMEDIR = /absolute/path/to/your/somedir
    MODULE_SEARCH_PATH = $(SOMEDIR)/build-fiasco-aarch64:$(SOMEDIR)/ramdisk:$(SOMEDIR)/conf:$(SOMEDIR)/build-linux-aarch64/arch/arm64/boot/

At the same time, make sure that *QEMU_OPTIONS* for aarch64 are defined in the
following fashion:

    QEMU_OPTIONS-arm64 = -M virt,virtualization=true -cpu cortex-a57 -m 1024 -display none

Do not modify the *-m 1024* part as it seems to be essential at the moment and
trying a different size might result in a failure.

### Spawning the Linux VM

At this point you are ready to start the basic VM scenario:

    [somedir/l4/conf] $ cd ../../build-aarch64
    [somedir/build-aarch64] $ make E=uvmm-basic qemu

This will spawn QEMU and run L4Re inside of it, including the *uvmm* virtual
machine monitor which eventually starts the Linux guest. Linux should boot in a
couple of seconds and you should be able to interact with it over its serial
console afterwards.

## Using the vmm convenience wrapper

Invoking *io* and *uvmm* manually like we do in the uvmm-basic scenario above
can quickly become tedious and impractical when more flexibility and
functionality is desired. L4Re therefore provides a convenience wrapper to
abstract away the repetetive parts. This wrapper is located in
*somedir/l4/pkg/uvmm/configs/vmm.lua*.

At the expense of hiding some interesting details under the cover, our
uvmm-basic scenario can be rewritten as follows. In *somedir/conf* create a new
ned script called *uvmm.ned*:

````lua
package.path = "rom/?.lua";

local L4 = require "L4";
local vmm = require "vmm";

vmm.start_vm({
  id = 1,
  mem = 128,
  mon = false,
  rd = "rom/ramdisk-armv8-64.cpio.gz",
  fdt = "rom/virt-arm_virt-64.dtb",
  bootargs = "console=hvc0 earlyprintk=1 rdinit=/bin/sh",
  kernel = "rom/Image.gz",
  log = L4.Env.log,
  ext_args = { "-i" }
});
````

You will also need to accompany it with a corresponding new entry in
*somedir/l4/conf/modules.list*:

````
entry uvmm
kernel fiasco -serial_esc
roottask moe rom/uvmm.ned
module uvmm
module l4re
module ned
module virt-arm_virt-64.dtb
module ramdisk-armv8-64.cpio.gz
module[shell] echo $SRC_BASE_ABS/pkg/uvmm/configs/vmm.lua
module uvmm.ned
module[uncompress] Image.gz
````

You can now run this scenario:

    [somedir/l4/conf] $ cd ../../build-aarch64
    [somedir/build-aarch64] $ make E=uvmm qemu

As with the basic scenario, you should be able to interact with the guest after
it boots.

## Conclusion

In this tutorial you have learned to build, configure and run a basic Linux VM
scenario in the L4Re virtual machine monitor *uvmm*. Parts not covered here
include running multiple VMs, using a console multiplexer and a virtual network
switch. Likewise, we didn't show how to pass a host I/O device to the guest.
