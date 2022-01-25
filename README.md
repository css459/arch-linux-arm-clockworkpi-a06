# Arch Linux ARM for the ClockworkPi DevTerm A06

Author: Cole Smith

License: LGPL-2.1

# Introduction

This document will walk you through installing [Arch Linux ARM](https://archlinuxarm.org/) on the DevTerm A06. At the
time of writing, only Armbian is supported on the DevTerm.

We will create a root file system based on the **rock64** architecture (rk3328). This will include patching our
bootloader and kernel with the patches provided by ClockworkPi. Technically, the DevTerm A06's architecture is based on
the rk3399.

# Setup

This guide **assumes you are already using Arch Linux**. Some package names or procedures may differ depending on your
distribution.

In order to build the root filesystem, we will set up the following:

1. `aarch64` chroot environment + necessary configuration
2. `arm-none-eabi-gcc` build tools from ARM
3. Linux kernel
4. U-Boot bootloader
5. Additional packages for the A06

## Setting up Chroot Environment

We will start by creating an ARM chroot environment to build our root filesystem using
[this guide](https://nerdstuff.org/posts/2020/2020-003_simplest_way_to_create_an_arm_chroot/).

1. Install the required packages

```
$ yay -S binfmt-qemu-static and qemu-user-static base-devel arch-install-scripts

# systemctl restart systemd-binfmt.service
```

2. Verify that an `aarch64` executable exists in

```
$ ls /proc/sys/fs/binfmt_misc
```

3. Download the base root FS to use. We will use the `aarch64` tarball from Arch Linux ARM

```
$ wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
```

4. Create the mount point and extract the files as root (not via sudo)

```
$ mkdir root 
$ sudo su

# bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C root
# mount --bind root root
```

7. Chroot into the newly created environment

```
# arch-chroot root
```

8. Inside the chroot, populate and init the pacman keyring

```
# pacman-key --init
# pacman-key --populate archlinuxarm
```

9. Finally, update the packages

```
# pacman -Syu
```

## Configuring The Root Filesystem

We will start with lightly configuring our system before compiling the packages.

For this section, **all commands will be run inside the chroot**.

1. Install some useful tools

```
# pacman -S base-devel git vim wget ranger sudo iwd man
```

2. Set the Locale by editing `/etc/locale.gen` and uncommenting your required locales.
3. Run

```
# locale-gen
```

### Switch to the `alarm` user

1. We can avoid working with the root account by granting `alarm`, the default Arch Linux ARM user, `sudo` privileges.

```
# EDITOR=/usr/bin/vim visudo
```

2. And add the corresponding line for `alarm` after the one for `root`

```
alarm ALL=(ALL) ALL
```

3. Switch to the `alarm` user

```
# su alarm

$ cd
```

**NOTE**: The default password for the **alarm** user is **alarm**

## Acquiring GCC Build Tools

U-Boot depends on the `arm-none-eabi-gcc` executable to be built, and since this program is not available in the Arch
Linux ARM repositories, we will download it directly from
[ARM's website](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain/gnu-a/downloads)
.

1. Download the binaries

```
$ wget https://developer.arm.com/-/media/Files/downloads/gnu-a/10.3-2021.07/binrel/gcc-arm-10.3-2021.07-aarch64-arm-none-eabi.tar.xz
```

2. Extract the binaries to another directory

```
$ mkdir gcc 
$ tar -xJf gcc-arm-10.3-2021.07-aarch64-arm-none-eabi.tar.xz -C gcc
```

3. Add the toolchain to your `PATH`

```
$ cd gcc/gcc-arm-10.3-2021.07-aarch64-arm-none-eabi/bin
$ export PATH=$PATH:$(pwd)
$ cd
```

## Compiling The Packages

This repository contains pre-configured and patched Arch Linux packages for the DevTerm A06. The Linux kernel and U-Boot
are based off the **rock64** variants, already available in Arch Linux ARM, with patches provided by ClockWorkPi.

You can find these patches from
ClockworkPi [here](https://github.com/clockworkpi/DevTerm/tree/main/Code/patch/armbian_build_a06/patch).

### Download This Repository

1. Inside the `alarm` home folder of your `aarch64` chroot environment, clone this repository

```
$ git clone https://github.com/css459/arch-linux-arm-clockworkpi-a06.git
$ cd arch-linux-arm-clockworkpi-a06
```

### Compiling The Linux Kernel

1. Build the package. **This can take a long time!!** Especially since we are emulating an `aarch64`
   architecture. The package build tool `makepkg`, supports a flag called `MAKEFLAGS`. Below, we will append
   `MAKEFLAGS="-j$(nproc)"` to the `makepkg` command to instruct the compiler to use one worker for each core.

```
$ cd linux-clockworkpi-a06 
$ MAKEFLAGS="-j$(nproc)" makepkg -si 
$ cd ..
```

### Compiling U-Boot

1. Build the package, similar to above.

```
$ cd uboot-clockworkpi-a06 
$ MAKEFLAGS="-j$(nproc)" makepkg -si 
$ cd ..
```

**NOTE: DO NOT INSTALL THE BOOTLOADER TO THE DISK WHEN ASKED AFTER THIS STEP.** We will do this ourselves when we
prepare the SD card.

### Compiling Additional Packages

TODO

### Exit the chroot

1. Exit `alarm`

```
$ exit
```

2. Exit `root`

```
# exit
```

### Unmount The Root Filesystem

```
 # umonut root
```

# Tar The Root Filesystem

We are now ready to package up the root filesystem into a compressed tarball.

```
 # cd root
 # tar cpJf ../arch-linux-clockworkpi-a06-root-fs.tar.xz .
 # cd ..
```

Change ownership of the tarball and exit the `root` account

```
 # chown <user>:<user> arch-linux-clockworkpi-a06-root-fs.tar.xz
 # exit
```

**You now have a root filesystem tarball to bootstrap the SD card!**

## Prepare the SD Card

We will now put our prepared filesystem onto the SD card. We will follow
[Arch Linux ARM's guide for the rock64](https://archlinuxarm.org/platforms/armv8/rockchip/rock64), but use our tarball
in place of theirs.

1. Zero the beginning of the SD card

```
# dd if=/dev/zero of=/dev/sdX bs=1M count=32
```

2. Start fdisk to partition the SD card

```
# fdisk /dev/sdX
```

3. Inside fdisk,

    1. Type **o**. This will clear out any partitions on the drive
    2. Type **p** to list partitions. There should be no partitions left
    3. Type **n**, then **p** for primary, **1** for the first partition on the drive, **32768** for the first sector
    4. Press ENTER to accept the default last sector
    5. Write the partition table and exit by typing **w**

4. Create the **ext4** filesystem **without a Journal**

```
# mkfs.ext4 -L ROOT_ARCH -O ^has_journal /dev/sdX1
```

5. Mount the filesystem

```
# mount /dev/sdX1 /mnt
```

6. Install the root filesystem (as root not via sudo)

```
# sudo su
# bsdtar -xpf arch-linux-clockworkpi-a06-root-fs.tar.xz -C /mnt
# exit
```

7. Install the bootloader to the SD card

```
# cd /mnt/boot
# dd if=rksd_loader.img of=/dev/sdX seek=64 conv=notrunc
# dd if=u-boot.itb of=/dev/sdX seek=16384 conv=notrunc
```

8. Unmount and eject the SD card

```
# cd
# umount /mnt
# eject /dev/sdX
```

## Done!

The SD card is now ready to be booted by the DevTerm! Good luck!

# Acknowledgements

Very special thanks to **Max Fierke (@maxfierke)** from the Manjaro team for their help in debugging and kernel patching. 
The Linux kernel port in this repository uses their carefully designed patches, in place of the ones from ClockworkPi.

For more information about their kernel for the A06, [see here](https://gitlab.manjaro.org/manjaro-arm/packages/core/linux-clockworkpi-a06).

