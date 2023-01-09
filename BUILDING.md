# Building L4Re

Building an L4Re system from scratch involves several Git repositories and
multiple steps described in this document.

## Prerequisities

Depending on your host system, you might need to install some prerequisities.

On Debian 9.4, make sure you have the required packages installed together with
their dependencies by running the following command with sufficient privileges:

    [somedir] $ apt-get install git make liburi-perl libpar-packer-perl libgit-repository-perl libxml-mini-perl gcc g++ libc6-dev-i386 g++-multilib libncurses5-dev qemu xorriso mtools flex bison pkg-config gawk liburi-perl device-tree-compiler

On top of a fresh Fedora 27 install, you will need the following packages and
their dependencies:

    [somedir] $ dnf install perl-URI perl-PAR-Packer perl-Git-Repository-Plugin-AUTOLOAD perl-CPAN perl-Test perl-Text-Balanced gcc gcc-c++ glibc-devel.i686 ncurses-devel xorriso flex bison pkgconf-pkg-config gawk dtc
    [somedir] $ cpan install XML::Mini::Document

On Arch Linux, the following packages need to be installed:

    [somedir] $ pacman -S --needed base-devel dtc lib32-gcc-libs qemu qemu-arch-extra mtools

Additionally, these packages need to be installed from the AUR by a method of your choice:

    [somedir] $ yay -S perl-par-packer perl-git-repository perl-xml-mini perl-uri

## Getting and installing ham

L4Re is composed of several loosely coupled Git repositories. While it is
theoretically possible to manage them individually using Git alone, it is
recommended to use [ham](https://github.com/kernkonzept/ham) to manage the whole
set at once.

    [somedir] $ git clone https://github.com/kernkonzept/ham.git
    [somedir] $ cd ham
    [somedir/ham] $ make

Make sure to include ham in your PATH.

## Getting the Fiasco and L4Re sources using ham

Use ham to get the L4Re project manifest and all its constituent repositories:

    [somedir/ham] $ cd ..
    [somedir] $ ham init -u https://github.com/kernkonzept/manifest.git
    [somedir] $ ham sync

If ham sync is terminated early or fails to sync refer to the
[troubleshooting](#troubleshooting) information.

## Building L4Re

When building L4Re for the first time, you first need to create and configure
a build directory:

    [somedir] cd l4
    [somedir/l4] $ make B=../build-i386
    [somedir/l4] $ cd ../build-i386
    [somedir/build-i386] $ make config    # Make sure to select the X86-32 target architecture and exit

Note that you need to choose the X86-32 target architecture before saving the
configuration in order to build L4Re for x86. Also note that the build directory
can be arbitrary. *build-i386* is used as an example here.

The build directory is now ready and you can build the L4Re binaries:

    [somedir/build-i386] $ make           # Optionally use -j X to make the build parallel

The release L4Re binaries reside in the *bin* subdirectory of the build
directory. For the x86 configuration, this is *bin/x86_gen/l4f*:

    [somedir/build-i386] $ file bin/x86_gen/l4f/hello
    bin/x86_gen/l4f/hello: ELF 32-bit LSB executable, Intel 80386, version 1 (GNU/Linux), statically linked, with debug_info, not stripped

## Building Fiasco.OC

For running the L4Re binaries, you are going to need the
[Fiasco.OC](https://github.com/kernkonzept/fiasco) microkernel, which is cloned
by ham as part of the L4Re manifest.

Create and configure the Fiasco build directory:

    [somedir/build-i386] cd ../fiasco
    [somedir/fiasco] $ make B=../build-fiasco-i386
    [somedir/fiasco] $ cd ../build-fiasco-i386
    [somedir/build-fiasco-i386] $ make config    # Make sure to select Intel IA-32 processor family as Architecture in Target configuration and exit

And finally, build Fiasco itself:

    [somedir/build-fiasco-i386] $ make           # Optionally use -j X to make the build parallel

Like in the L4Re case above, the build directory is created and configured only
once. The resulting Fiasco binary is called *fiasco*.

## Running the Hello world! program

Now that you have sucessfully built Fiasco and L4Re, it is time to verify that
they were built correctly by running a simple demo scenario with a sample
program called *hello*:

    [somedir/build-fiasco-i386] cd ../build-i386
    [somedir/build-i386] $ make E=hello qemu MODULE_SEARCH_PATH="${PWD}/../build-fiasco-i386"

This will run the scenario in QEMU without creating any bootable images. After
a short while, you should see the message "Hello World!" printed in 1-second
intervals on the virtual QEMU screen.

If you prefer having a bootable ISO instead, you can generate one like this:

    [somedir/build-i386] $ make E=hello grub2iso MODULE_SEARCH_PATH="${PWD}/../build-fiasco-i386"

The generated ISO image can be found in the *images* subdirectory.

## Making life easier with Makeconf.boot and Makeconf.local

If you copy *somedir/l4/conf/Makeconf.boot.example* to
*somedir/l4/conf/Makeconf.boot* and point the *FIASCO_PATH-x86* variable in
there to your Fiasco build directory, you wouldn't have to specify it in the
*MODULE_SEARCH_PATH* environment variable on the command line everytime.

*Makeconf.boot* is also the place to tune various QEMU and platform-specific
options.

In a similar way, you can create *Makeconf.local* in your build directory and
define the variables used during the configuration phase (e.g. the
cross-compiler configuration, see below) there.

## Cross-compiling for ARM and MIPS

If you are going to cross-compile L4Re and Fiasco for ARM or MIPS, you'll need
to install the respective QEMU packages for the target platforms and
cross-compilers.

Your distribution most likely provides some cross-compiler packages for selected
platforms. Debian cross-compilers (packaged in g++-arm-linux-gnueabihf) have been
known to work with L4Re and Fiasco, but your mileage may vary. If unsure or out
of luck with your distribution, try installing the following and point your PATH
environment variable to it:

  * Linaro Toolchains for ARM
    * 32-bit ARMv7 Cortex-A, hard-float, little-endian: [arm-linux-gnueabihf](https://releases.linaro.org/components/toolchain/binaries/latest-7/arm-linux-gnueabihf/)
    * 64-bit ARMv8 Cortex-A, little-endian: [aarch64-linux-gnu](https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu/)
  * Codescape GNU Tools for MIPS
    * MIPS32R5 and MIPS64R5: [mips-mti-linux-gnu](https://codescape.mips.com/components/toolchain/2017.10-08/Codescape.GNU.Tools.Package.2017.10-08.for.MIPS.MTI.Linux.CentOS-5.x86_64.tar.gz)
    * MIPS32R6 and MIPS64R6: [mips-img-linux-gnu](https://codescape.mips.com/components/toolchain/2017.10-08/Codescape.GNU.Tools.Package.2017.10-08.for.MIPS.IMG.Linux.CentOS-5.x86_64.tar.gz)

The cross-compilation of L4Re and Fiasco is similar to the normal build:

    [somedir/l4] $ make B=../build-arm
    [somedir/l4] $ cd ../build-arm

When cross-compiling, you have to specify the prefix of the cross-tools you
wish to use in the *CROSS_COMPILE* variable at some point before the config
phase.

For example, when cross-compiling for the 32-bit ARM Versatile Express
Cortex-A15, you need to specify the *CROSS_COMPILE* variable in a file called
*Makeconf.local*.  Adjustments might be necessary for different cross-compilers
and platforms:

    [somedir/build-arm] $ echo "CROSS_COMPILE:=arm-linux-gnueabihf-" > Makeconf.local
    [somedir/build-arm] $ make config    # Select ARM architecture, ARMv7a CPU variant and ARM Versatile Express A15 platform
    [somedir/build-arm] $ make

Alternatively, you can define the *CROSS_COMPILE* variable on the command line:

    [somedir/build-arm] $ make config CROSS_COMPILE=arm-linux-gnueabihf-    # Select ARM architecture, ARMv7a CPU variant and ARM Versatile Express A15 platform
    [somedir/build-arm] $ make CROSS_COMPILE=arm-linux-gnueabihf-

In both cases, don't forget to configure the system for the ARM architecture, the ARMv7A CPU variant and the ARM Versatile Express A15 platform.

Cross-compiling Fiasco is analogous to cross-compiling L4Re:

    [somedir/build-arm] $ cd ../fiasco
    [somedir/fiasco] $ make B=../build-fiasco-arm
    [somedir/fiasco] $ cd ../build-fiasco-arm
    [somedir/build-fiasco-arm] $ echo "CROSS_COMPILE:=arm-linux-gnueabihf-" > Makeconf.local
    [somedir/build-fiasco-arm] $ make config    # Select ARM processor family as Architecture, ARM RealView Platform as Platform, Versatile Express as Realview Platform and ARM Cortex-A15 CPU as CPU
    [somedir/build-fiasco-arm] $ make

When all is built, run the *hello* scenario in QEMU (assuming *qemu-system-arm*
is installed on the system):

    [somedir/build-fiasco-arm] $ cd ../build-arm
    [somedir/build-arm] $ make E=hello qemu MODULE_SEARCH_PATH="${PWD}/../build-fiasco-arm" QEMU_OPTIONS="-M vexpress-a15 -m 2047 -cpu cortex-a15 -serial stdio -display none" PLATFORM_TYPE=rv_vexpress_a15

Note that besides *MODULE_SEARCH_PATH*, also *QEMU_OPTIONS* and *PLATFORM_TYPE* can be conveniently specified in *somedir/l4/conf/Makeconf.boot*.

## Troubleshooting

Building Fiasco and L4Re is a procedure complex enough to offer many
opportunities for things to go wrong. Here are the most common issues and their
causes:

### Recovering from ham sync errors

If ham sync is terminated early using Ctrl-C or encounters network errors such
as the following an incomplete manifest may have been synced.

    mk: fatal: The remote end hung up unexpectedly
    mk: fatal: early EOF
    mk: fatal: index-pack failed

It is usually best to start again with an empty directory. Alternatively, it
may be possible to selectively force sync some packages listed in
.ham/manifest.xml:

    [somedir] $ ham sync --force-local-update <manifest package name>

If network issues are suspected try:

    [somedir] $ ham sync --max-connections=1

### Missing top-level make file

If the ham sync operation was incomplete the *mk* package may be missing. In
this case it's best to restart ham sync from an empty directory. Running *make*
in the top-level directory results in this error indicating the top-level
Makefile is missing:

    make: *** No targets specified and no makefile found.  Stop.

### Missing multilib

If you get the following error when creating a build directory on a 64-bit
host, make sure that multilib (or libraries supplementing it, e.g.
glibc-devel.i686 on Fedora) are installed:

    /usr/include/linux/errno.h:1:23: fatal error: asm/errno.h: No such file or directory
    #include <asm/errno.h>
                          ^
    compilation terminated.

### Forgetting to set the architecture during *make config* before cross-compilation

If you forget to set the respective architecture during the configuration step
before cross-compilation, you may get failures that look like:

    arm-linux-gnueabihf-gcc: error: unrecognized command line option ‘-m32’
    Makefile:372: recipe for target 'Makeconf.bid.local-internal-names' failed
    make[5]: *** [Makeconf.bid.local-internal-names] Error 1

### L4Re stops during boot after Sigma0 output

If you are on the aarch64 architecture and observe the boot process to stop
some lines after

    SIGMA0: Hello!

but before

    MOE: Hello world

Then it is likely that the Fiasco kernel is configured to support
virtualization but the L4Re user-land is not. Please ensure the L4Re
configuration features "CONFIG_KERNEL_CPU_VIRT=y" (Kernel supports
virtualization) and rebuild the L4Re.
