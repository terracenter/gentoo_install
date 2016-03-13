# Gentoo Installation

## Introduction
This file is a step by step how to install Gentoo guide. With systemd, uefi, luks and much more.... keep reading :D

## Installation steps

### Start live-cd environment
I like to use systemrescuecd to start a live-cd environment with uefi vars enabled, so, let's download it http://www.system-rescue-cd.org, then burn it or write into bootable usb and let's move on!

Check if it is really UEFI, the output should lists the UEFI variables
```
efivar -l
```

### Prepare Hard disk
Start disk partitioning, I like to use cgdisk, but others like gdisk, parted, ... any other are welcome as long as UEFI is supported :P
```
cgdisk /dev/sda
```

Create new GUID partition table and destroy everything on disk. Then create the following two partitions like below code.
I'm not making a swap because I have enough ram but feel free to make one if you really want it (inside crypt partition). 
```
gdisk -l /dev/sda
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         1050623   512.0 MiB   EF00  EFI
   2         1050624       500118158   238.0 GiB   8300  LVM
```

### Prepare crypt and filesystems
Before start creating crypt filesystem it's much important to check which is the best performance setup on your system because you can't chenge it once created, so:
```
cryptsetup benchmark
```

In my laptop the best performancing setup is aes-xts:
```
# Tests are approximate using memory only (no storage IO).
PBKDF2-sha1       306601 iterations per second for 256-bit key
PBKDF2-sha256     352818 iterations per second for 256-bit key
PBKDF2-sha512      94568 iterations per second for 256-bit key
PBKDF2-ripemd160  227951 iterations per second for 256-bit key
PBKDF2-whirlpool  121362 iterations per second for 256-bit key
#  Algorithm | Key |  Encryption |  Decryption
     aes-cbc   128b   604.2 MiB/s  2515.2 MiB/s
 serpent-cbc   128b    83.8 MiB/s   517.0 MiB/s
 twofish-cbc   128b   180.3 MiB/s   333.7 MiB/s
     aes-cbc   256b   449.5 MiB/s  1934.8 MiB/s
 serpent-cbc   256b    85.8 MiB/s   516.9 MiB/s
 twofish-cbc   256b   183.2 MiB/s   332.8 MiB/s
     aes-xts   256b  2110.6 MiB/s  2052.8 MiB/s
 serpent-xts   256b   518.3 MiB/s   502.5 MiB/s
 twofish-xts   256b   323.0 MiB/s   328.5 MiB/s
     aes-xts   512b  1677.8 MiB/s  1601.8 MiB/s
 serpent-xts   512b   518.9 MiB/s   502.4 MiB/s
 twofish-xts   512b   322.6 MiB/s   327.4 MiB/s
```

Create the encryption layer for the partition dedicated to lvm:
```
cryptsetup -v --cipher aes-xts-plain64 --key-size 256 -y luksFormat /dev/sda2
```

Then open our new crypted partition the create the lvm inside, remember that in this procedure the name *lvm* is only a trivial name, so you can use whenever you like:
```
cryptsetup open --type luks /dev/sda2 lvm
```

#### Create lvm partitions
In my setup I use one partition for *root* filesystem and other for *home* files, so modify if /usr, /tmp or other partitions should be on separate partitions in your setup:
```
pvcreate /dev/mapper/lvm
vgcreate vg0 /dev/mapper/lvm
lvcreate --size 50G vg0 --name root
lvcreate -l +100%FREE vg0 --name home
```

#### Create filesystems
Let's create the filesystems for our three partitions (two of them inside crypted lvm setup ;-) ):
```
mkfs.vfat -F32 /dev/sda1
mkfs.ext4 /dev/mapper/vg0-root
mkfs.ext4 /dev/mapper/vg0-home
```

#### Mount the new system
It's time to create a directory where we mount our new root filesystem to be able to chroot into our new system:
```
mount /dev/mapper/vg0-root /mnt/gentoo
mkdir -p /mnt/gentoo/boot
mount /dev/sda1 /mnt/gentoo/boot
mkdir /mnt/gentoo/home
mount /dev/mapper/vg0-home /mnt/gentoo/home
```

## Installing the Gentoo base system
Before installing Gentoo, make sure that the date and time are set correctly. A mis-configured clock may lead to strange results in the future!
`date`

If needed set correct date like (March 02 22:09 2016)
`date 030222092016`

### Downloading the stage tarball
To not start from a Linux from scratch installation, the Gentoo developed privides us with a Stage 3 which is a base binary semi working environment suitable to continue the Gentoo installation.
To do that we just only need to download it and uncompress in our filesystem tree:
```
cd /mnt/gentoo
wget http://mirror.eu.oneandone.net/linux/distributions/gentoo/gentoo/releases/amd64/autobuilds/current-stage3-amd64/stage3-amd64-20160225.tar.bz2
```

Unpacking the stage tarball:
```
tar xvjpf stage3-*.tar.bz2 --xattrs
```

#### Configuring compile options:
```
nano -w /mnt/gentoo/etc/portage/make.conf
CFLAGS="-march=core-avx2 -O2 -pipe -mabm -madx -mavx256-split-unaligned-load -mavx256-split-unaligned-store -mprfchw -mrdseed"
CXXFLAGS="${CFLAGS}"
MAKEOPTS="-j4"
```

#### Selecting mirrors
`mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf`

#### Configuring the main Gentoo repository
```
mkdir /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
nano -w /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```

#### Copy DNS info
`cp -L /etc/resolv.conf /mnt/gentoo/etc/`

#### Mounting the necessary filesystems
```
mount -t proc proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
```

#### Entering the new environment
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"

### Configuring Portage
Installing a Portage snapshot
`emerge-webrsync`

Updating the Portage tree
`emerge --sync`

Choosing the right profile
`eselect profile list`

[..]
[12]  default/linux/amd64/13.0/systemd
[..]

`eselect profile set 12`

### Configuring the USE variable
`emerge ufed`

Or simply edit *make.conf* file:
```
nano -w /etc/portage/make.conf
USE="X aac alsa aspell bash-completion boost branding caps contrib \
     cpudetection cryptsetup dbus exif filecaps flac gd gpm gstreamer gzip \
     hardened hddtemp highlight hostname imagemagick int64 introspection \
     jemalloc jpeg jpeg2k json kdbus kmod kms libav libnotify lm_sensors lz4 \
     lzma lzo mime mp3 mp4 mpeg nano-syntax networkmanager nss numa ogg \
     opencl opengl pango pcap pdf pie png pulseaudio python recode samba sdl \
     seccomp simplexml slang smp sockets sound spell sse sse2 ssh \
     startup-notification svg symlink systemd tcmalloc theora threads \
     timezone truetype udisks upnp upnp-av upower usb video vim vim-syntax \
     vorbis wavpack wayland wayland-compositor wifi xcomposite xfs xft \
     xinerama xkb xml xmlrpc xorg xpm xrandr xwayland xz zeroconf \
     zsh-completion -cups"
```
The USE flags corresponding to the instruction sets and other features specific to the x86 (amd64) architecture are configured into the variable called CPU_FLAGS_X86.
We can simply use a useful python script to autodetect which of them we should select:
`emerge --ask app-portage/cpuinfo2cpuflags`

And then:
`cpuinfo2cpuflags-x86`

Copy the output line into */etc/portage/make.conf*
`echo "CPU_FLAGS_X86="aes avx avx2 fma3 mmx mmxext pni popcnt sse sse2 sse3 sse4_1 sse4_2 ssse3" >> /etc/portage/make.conf`

### Timezone
`echo "Europe/Madrid" > /etc/timezone`

Next, reconfigure the sys-libs/timezone-data package, which will update the /etc/localtime file based on the /etc/timezone entry.
The /etc/localtime file is used by the system C library to know the timezone the system is in.
`emerge --config sys-libs/timezone-data`

### Configure locales
```
nano -w /etc/locale.gen
locale-gen
```
Once done, it is now time to set the system-wide locale settings. Again we use eselect for this, now with the locale module.
```
eselect locale list
Available targets for the LANG variable:
  [1]   C
  [2]   en_US.utf8
  [3]   POSIX
  [ ]   (free form)
```
Choose which locales would you like to active and then do it manually editing */etc/env.d/02locale* or automatically by:
`eselect locale set 2`

Now reload the environment:
env-update && source /etc/profile && export PS1="(chroot) $PS1"

## Systemd and udev
Before continue we must remove udev otherwise we've ciclic dependencies
emerge --deselect sys-fs/udev
emerge --unmerge sys-fs/udev

## Configuring the Linux kernel
Installing the sources
`emerge --ask sys-kernel/gentoo-sources`

Now it is time to configure and compile the kernel sources. There are two approaches for this:

1. The kernel is manually built and install.
2. A tool called genkernel is used to automatically build and install the Linux kernel.

In both cases we must configure it manually that normally is the most difficul procedure a Linux user ever has to perform. Nothing is less true - after configuring a couple of kernels no-one even remembers that it was difficult ;)
So I'll explain how to use genkernel which help us to mantain config files and some more automations :D

However, one thing is true: it is vital to know the system when a kernel is configured manually.
`emerge --ask sys-apps/pciutils`

Edit */etc/genkernel.conf* to set our preferences:
```
nano -w /etc/genkernel.conf

INSTALL="yes"
OLDCONFIG="yes"
MENUCONFIG="yes"
NCONFIG="no"
CLEAN="yes"
MRPROPER="yes"
MOUNTBOOT="yes"
SYMLINK="no"
SAVE_CONFIG="yes"
USECOLOR="yes"
CLEAR_CACHE_DIR="yes"
POSTCLEAR="1"
MAKEOPTS="-j5"
LVM="yes"
LUKS="yes"
DMRAID="no"
UDEV="yes"
BOOTDIR="/boot"
GK_SHARE="${GK_SHARE:-/usr/share/genkernel}"
CACHE_DIR="/var/cache/genkernel"
DISTDIR="/var/lib/genkernel/src"
LOGFILE="/var/log/genkernel.log"
LOGLEVEL=1
DEFAULT_KERNEL_SOURCE="/usr/src/linux"
COMPRESS_INITRD="yes"
COMPRESS_INITRD_TYPE="best"
```

Notice that LVM, LUKS and UDEV must be set on to have this system works. Otherwise will not boot.

And then simply run:
`genkernel all`

## Configuring the modules
If we need to autoload a kernel module each time to system boots we should specify it in */etc/conf.d/modules* file.

You can list your available modules with:
`find /lib/modules/<kernel version>/ -type f -iname '*.o' -or -iname '*.ko' | less`

## Installing firmware
Some drivers require additional firmware to be installed on the system before they work. This is often the case for network interfaces, especially wireless network interfaces. 
Most of the firmware is packaged in sys-kernel/linux-firmware:
`emerge --ask sys-kernel/linux-firmware`

## LVM Configuration
Install lvm tools if its not yet installed:
`emerge -av sys-fs/lvm2`

nano -w /etc/lvm/lvm.conf

use_lvmetad = 1
issue_discards = 1
volume_list = ["vg0"] # Our VG volume name, check with vgdisplay

## Fstab

Before editing fstab we need to know which UUID are using our devices inside and outside lvm and luks volumes:

blkid /dev/mapper/vg0-root | awk '{print $2}' | sed 's/"//g'
UUID="576e229c-cf68-4010-8d85-ff8149158416"

blkid /dev/mapper/vg0-home | awk '{print $2}' | sed 's/"//g'
UUID="95fa5807-ea57-4cf5-b717-74f4aba190e2"

Then edit fstab:

nano -w /etc/fstab
```
/dev/sda1                                       /boot   vfat    noatime                                         1 2
UUID="576e229c-cf68-4010-8d85-ff8149158416"     /       ext4    discard,noatime,commit=600,errors=remount-ro    0 1
UUID="95fa5807-ea57-4cf5-b717-74f4aba190e2"     /home   ext4    discard,noatime,commit=600                      0 0
tmpfs                                           /var/tmp tmpfs  nodev,nosuid    								0 0
tmpfs                                           /tmp    tmpfs   nodev,nosuid    								0 0
```

## Crypttab
Warning!!! As we don't have encrypted partitions other than root which must be mounted by systemd before the whole system start we don't need to set it up there, so, our crypttab must be empty.

## Emerging systemd

ln -sf /proc/self/mounts /etc/mtab ?????????

Ensure that you have systemd installed with all required USE flags
emerge -av app-portage/gentoolkit
euse -E cryptsetup systemd gudev dbus
emerge -av sys-apps/systemd
emerge -av sys-apps/dbus

## Change the root password
While in chroot we need to change the root password of our new system just before rebooting it.
`passwd`

## Systemd boot (bootloader)
Verify your EFI variables are accessible:
```
emerge -av sys-libs/efivar
efivar -l
```

Verify that you have mounted /boot
mount | grep boot

Install systemd-boot:
`bootctl --path=/boot install`
It will copy the systemd-boot binary to your EFI System Partition ($esp/EFI/systemd/systemd-bootx64.efi and $esp/EFI/Boot/BOOTX64.EFI - both of which are identical - on x64 systems) and add systemd-boot itself as the default EFI application (default boot entry) loaded by the EFI Boot Manager.

### Add bootloader entries
Find the UUID of /dev/sda2 and insert it into the file 
blkid /dev/sda2 | awk '{print $2}' | sed 's/"//g' > /boot/loader/entries/gentoo.conf`
`nano -w /boot/loader/entries/gentoo.conf`
```
title    Gentoo Linux
linux    /kernel-genkernel-x86_64-4.4.4-gentoo
initrd   /initramfs-genkernel-x86_64-4.4.4-gentoo
options  cryptdevice=UUID=eace91f3-270e-4bc3-b0a3-b4a869fdbbfd:lvm:allow-discards root=/dev/mapper/vg0-root ro dolvm
```
Edit default loader
`nano -w /boot/loader/loader.conf`
```
default gentoo
timeout 3
```

## Enable lvm2
`systemctl enable lvm2-lvmetad.service`


