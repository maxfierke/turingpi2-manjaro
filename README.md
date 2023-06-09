# UNDERCONSTRUCTION, PROBABLY NOT WORKING YET

# Manjaro ARM for Turing Pi 2 SoCs

Maintainer: Max Fierke

README based on from: Cole Smith & Yatao Li's ["Arch Linux ARM for ClockworkPi A06"](https://github.com/yatli/arch-linux-arm-clockworkpi-a06/tree/main)

License: LGPL-2.1

# Introduction

This document will walk you through installing [Manjaro ARM](https://archlinuxarm.org/) on a few of the supported SoCs for the Turing Pi 2 cluster board.

We will create a root file system for supporting ~~three~~ two different SoCs:
  * ~~**soquartz** (rk3566)~~
    * Use the existing images: https://github.com/manjaro-arm/soquartz-cm4-images/releases
  * **NVIDIA Jetson TX2 NX** (tegra186, p3636-0001)
  * **NVIDIA Jetson Nano** (tegra210, p3448-0002)

I may expand the supported SoCs if/when I acquire them. (e.g. I suspect I'll get a Turing RK1 at some point)

## Caution
If you are running advanced filesystems on your host (for example `zfs`), don’t try to prepare the image on that host. 
`mkinitcpio` in chroot doesn’t like that, and may cause damage to your host filesystem. Please use a virtual machine, or use a prebuilt image instead.

## Quickstart

Pre-built root filesystem(s) are provided in the **Releases** tab. Skip to the
**Prepare to Flash** if using a pre-built image for eMMC on **NVIDIA Jetson**.

**NOTE:** Please note the following defaults for the release filesystem:

* Root password: `root`
* Timezone: `US/Central`
* Locale: `en_US.UTF-8`
* Hostname: `turingpi2-node0x`

# Setup

This guide **assumes you are already using Arch Linux**. Some package names or procedures may differ depending on your
distribution.

In order to build the root filesystem, we will set up the following:

1. `aarch64` chroot environment + necessary configuration
2. (optional) `arm-none-eabi-gcc` build tools from ARM
3. Linux kernel
4. U-Boot bootloader
5. Additional packages for running on Turing Pi 2

## Setting up Chroot Environment

We will start by creating an ARM chroot environment to build our root filesystem using
[this guide](https://nerdstuff.org/posts/2020/2020-003_simplest_way_to_create_an_arm_chroot/).

1. Install the required packages

```
$ sudo pacman -S base-devel qemu-user-static qemu-user-binfmt arch-install-scripts
$ sudo systemctl restart systemd-binfmt.service
```

2. Verify that an `aarch64` executable exists in

```
$ ls /proc/sys/fs/binfmt_misc
```

3. Download the base root FS to use. We will use the latest `aarch64` tarball from Manjaro ARM

Find the latest here: https://github.com/manjaro-arm/rootfs/releases

e.g.

```
$ wget https://github.com/manjaro-arm/rootfs/releases/latest/download/Manjaro-ARM-aarch64-latest.tar.gz
```

4. Create the mount point and extract the files as root (not via sudo)

```
$ sudo su
# mkdir rootfs

# bsdtar -xpf Manjaro-ARM-aarch64-latest.tar.gz -C rootfs
# mount --bind rootfs rootfs
# touch rootfs/MANJARO-ARM-IMAGE-BUILD
# mkdir -p rootfs/etc
# ln -sf ../usr/lib/os-release rootfs/etc/os-release

```

**NOTE**: It's very important that this `rootfs` folder is owned by **root**. Otherwise, you will
get `unsafe path transition`
errors in the final build.

5. Chroot into the newly created environment

```
# arch-chroot rootfs
```

6. Seed the keys

```
# pacman-key --init
# pacman-key --populate archlinuxarm manjaro manjaro-arm
# pacman-mirrors -f10
```

6. Finally, update the packages

```
# pacman -Syu
```

## Configuring The Root Filesystem

We will start with lightly configuring our system before compiling the packages.

For this section, **all commands will be run inside the chroot**.

This is largely cribbed from [manjaro-arm-installer](https://gitlab.manjaro.org/manjaro-arm/applications/manjaro-arm-installer)

0. Grab the latest `arm-profiles` from Manjaro ARM

```
# pacman -S manjaro-arm-tools
# getarmprofiles
```

1. Install the packages needed for a minimal base

```
# PKG_SHARED=$(grep "^[^#;]" /usr/share/manjaro-arm-tools/profiles/arm-profiles/editions/shared | awk '{print $1}')
# PKG_EDITION=$(grep "^[^#;]" /usr/share/manjaro-arm-tools/profiles/arm-profiles/editions/minimal | awk '{print $1}')
# pacman -S manjaro-system manjaro-release systemd systemd-libs base-devel git vim wget ranger sudo man iputils linux-firmware $PKG_SHARED $PKG_EDITION
```

3. Enable (and disable) services

```
# EDITION_SERVICES=$(grep "^[^#;]" /usr/share/manjaro-arm-tools/profiles/arm-profiles/services/minimal | grep -v "bootsplash" | awk '{print $1}')
# systemctl enable getty.target haveged.service $EDITION_SERVICES
# sed -i s/"enable systemd-resolved.service"/"#enable systemd-resolved.service"/ /usr/lib/systemd/system-preset/90-systemd.preset
```

4. Setup a bunch of system config

```
# chmod u+s /usr/bin/ping
```

4. Set the Locale by editing `/etc/locale.gen` and uncommenting your required locales.

5. Set your default language by editing `/etc/locale.conf`

```
# echo "LANG=en_US.UTF-8" | tee --append /etc/locale.conf
```

5. Set your keymap and font by editing `/etc/vconsole.conf`

```
KEYMAP=us
FONT=Lat2-Terminus16
```

4. Run

```
# locale-gen
```

5. Reference partitions in `/etc/fstab`

```
# echo 'LABEL=ROOT_MNJRO    /    f2fs    defaults,noatime    0    0' >> /etc/fstab
# echo 'LABEL=BOOT_MNJRO    /boot    ext4    defaults    0    0' >> /etc/fstab
```

6. Set the time using `timedatectl`. To list supported timezones: `timedatectl list-timezones`

```
# timedatectl set-timezone "US/Central"
# timedatectl set-ntp true
```

**NOTE**: This may affect the timezone of your system outside the chroot, if so, you may reset
your host system timezone after finishing the root tarball. 


7. Set the system clock

```
# hwclock --systohc
```

8. Set the hostname to whatever you like

```
# echo 'turingpi2-node0x' > /etc/hostname
```

9. Assign the root password

```
# passwd
```

### Create and switch to a non-root user

**NOTE**: The default user and password are setup here as **manjaro**, but you can use whatever you want below,
just substitute anything you want in for `manjaro`.

1. We can avoid working with the root account by creating and granting our own user (e.g. `manjaro`) `sudo` privileges.

```
# useradd -m -G wheel,sys,audio,input,video,storage,lp,network,users,power -s /bin/bash manjaro
# passwd manjaro
```

2. Switch to the `manjaro` user

```
# su manjaro
$ cd
```

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

This repository contains pre-configured and patched Arch Linux packages for
the relevant SoCs. The Linux kernel and U-Boots are based off the multi-platform
AArch64 kernel from Manjaro ARM, which is based on the kernel already available
in Arch Linux ARM, with additional configuration for Tegra devices.

Eventually, I hope to get these config changes into the default Manjaro ARM
kernel, and then we just need to worry about packaging and flashing.

### Download This Repository

1. Inside the `manjaro` home folder of your `aarch64` chroot environment, clone this repository

```
$ git clone https://github.com/maxfierke/turingpi2-manjaro.git
$ cd turingpi2-manjaro
```

### Compiling The Linux Kernel

1. Build the package. **This can take a long time!!** Especially if we are emulating an `aarch64`
   architecture. The package build tool `makepkg`, supports a flag called `MAKEFLAGS`. Below, we will append
   `MAKEFLAGS="-j$(nproc)"` to the `makepkg` command to instruct the compiler to use one worker for each core.

```
$ cd linux-turingpi2
$ MAKEFLAGS="-j$(nproc)" makepkg -si
$ cp -r /boot/dtbs ../turingpi2-manjaro/dtbs
$ cd ..
```

### Compiling U-Boot

Build each u-boot from upstream. We're not creating these as `pacman` packages,
because we're going to flash them via NVIDIA's flashing utilities.

```
$ cd ..
$ git clone https://github.com/OE4T/u-boot-tegra.git
$ cd u-boot-tegra
$ git checkout patches-v2022.07 # Or whatever the latest is / version you want to target
```

#### Jetson Nano (P3448-0002 / P3450-0002)

```
$ make O=build/p3450-0002 p3450-0002_defconfig
$ make O=build/p3450-0002
$ cp build/p3450-0002/u-boot.bin ../turingpi2-manjaro/uboot-jetson-nano/u-boot.bin
$ cp build/p3450-0002/u-boot.dtb ../turingpi2-manjaro/uboot-jetson-nano/u-boot.dtb
```

#### Jetson TX2 NX (P3636-0001)

```
$ make O=build/p3636-0001 p3636-0001_defconfig
$ make O=build/p3636-0001
$ cp build/p3636-0001/u-boot.bin ../turingpi2-manjaro/uboot-jetson-tx2-nx/u-boot.bin
$ cp build/p3636-0001/u-boot.dtb ../turingpi2-manjaro/uboot-jetson-tx2-nx/u-boot.dtb
```

### Compiling post-install package

```
$ cd linux-turingpi2
$ cd jetson-post-install
$ makepkg -si
$ cd ..
```

### Clean up the build dependencies


```
$ exit # Exit manjaro user and return to root
# pacman -Rs base-devel git vim wget ranger xmlto docbook-xsl inetutils bc dtc manjaro-arm-tools
# rm -f /var/cache/pacman/pkg/*.pkg.tar.xz*
# rm -f MANJARO-ARM-IMAGE-BUILD
# rm -f /var/log/* /var/log/journal/* /etc/*.pacnew /usr/lib/systemd/system/systemd-firstboot.service /etc/machine-id
# rm -rf /home/manjaro/gcc* /home/manjaro/.cache/ /home/manjaro/.bash_history
```

### Exit the chroot

1. Exit `manjaro`

```
$ exit
```

2. Exit `root`

```
# exit
```

### Unmount The Root Filesystem

```
 # umount root
```

# Tar The Root Filesystem

We are now ready to package up the root filesystem into a compressed tarball.
Optionally, we can save the built packages.

```
 # cd rootfs
 # mv home/manjaro/turingpi2-manjaro ../
 # tar cpJf ../turingpi2-manjaro-rootfs.tar.xz .
 # cd ..
```

Change ownership of the tarball and exit the `root` account

```
 # chown <user>:<user> turingpi2-manjaro-rootfs.tar.xz
 # exit
```

**You now have a root filesystem tarball to bootstrap the SD card!**

## Prepare to Flash

**NOTE**: Much of this is extrapolated from the [Porting to Arch](https://elinux.org/Jetson/Porting_Arch)
guide for the NVIDIA Jetson TK1 and [Upstream](https://elinux.org/Jetson/Nano/Upstream) guide for the NVIDIA Jetson Nano

1. Grab the latest Linux for Tegra (L4T) release for the given board:
  * **Jetson Nano**: [L4T 32.7.3](https://developer.nvidia.com/downloads/remetpack-463r32releasev73t210jetson-210linur3273aarch64tbz2)
  * **Jetson TX2 NX**: [L4T 32.7.3](https://developer.nvidia.com/downloads/remksjetpack-463r32releasev73t186jetsonlinur3273aarch64tbz2)

  and place it within this repo (it will be git ignored)

2. Extract it. You should end up with a `Linux_for_Tegra` directory within the extracted archive, e.g.

```
$ tar -xf Jetson-210_Linux_R32.7.3_aarch64.tbz2
$ ls -la Jetson-210_Linux_R32.7.3_aarch64
```

3. Copy the rootfs tarball into the `L4T_RELEASE/Linux_for_Tegra/rootfs` directory, e.g.

```
$ cp turingpi2-manjaro-rootfs.tar.xz Jetson-210_Linux_R32.7.3_aarch64/Linux_for_Tegra/rootfs
```

## Flash the u-boot and DTB for the chip

### Jetson Nano

```
$ cd Jetson-210_Linux_R32.7.3_aarch64/Linux_for_Tegra
$ sudo ./flash.sh -k LNX -K ../../uboot-jetson-nano/u-boot.bin -d ../../uboot-jetson-nano/u-boot.dtb jetson-nano-emmc mmcblk0p1
$ sudo ./flash.sh -k DTB -d ../../rootfs/boot/dtbs/nvidia/tegra210-p3450-0000.dtb jetson-nano-emmc mmcblk0p1
```

### Jetson TX2 NX

```
$ cd Jetson-186_Linux_R32.7.3_aarch64/Linux_for_Tegra
$ sudo ./flash.sh -k kernel -K ../../uboot-jetson-tx2-nx/u-boot.bin -d ../../uboot-jetson-tx2-nx/u-boot.dtb jetson-xavier-nx-devkit-tx2-nx mmcblk0p1
$ sudo ./flash.sh -k kernel-dtb -d ../../rootfs/boot/dtbs/nvidia/tegra186-p3509-0000+p3636-0001.dtb jetson-xavier-nx-devkit-tx2-nx mmcblk0p1
```

## Flash the filesystem to the eMMC for the chip

TBD

## Next Steps

Check out the [post-install suggestions](https://wiki.archlinux.org/title/General_recommendations) from Arch Linux for
further configuration.

## Troubleshooting

If you run into issues where you cannot SSH or it does not appear to be booting, please check the debugging output
via UART:

1. Connect a USB-to-serial cable on the UART header for the specific node on Turing Pi 2 board
  - Node 1 requires UART through the GPIO pins, see the Turing Pi 2 docs for more details
3. Connect the other end to your Linux system, you should now see a new device: `/dev/ttyUSB0`
4. Monitor the connection with `sudo stty -F /dev/ttyUSB0 115200 && sudo cat /dev/ttyUSB0`
  * Or use `picocom`, `sudo picocom -b 115200 /dev/ttyUSB0`
6. Power on your Turing Pi 2 and/or the specific node and monitor for errors

# Acknowledgements

I stole this README format from Cole Smith and Yatao Li's guide on running
Arch Linux ARM on the ClockworkPi A06.

The Linux and PKGBUILD is based on the Manjaro ARM kernel PKGBUILD from the
Manjaro ARM team (Dan Johansen, Furkan Salman Kardame, and others)
