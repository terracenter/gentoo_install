# Gentoo install guide

## Introduction
Gentoo is a Linux distribution where unlike binary distros like Arch, Debian and many others, software are compiled locally according the user preferences and optimizations.

The name of Gentoo comes from the penguin specie who are the fastest swimming penguin in the world.

I've been using Gentoo since 2002 with some stages in Debian and Arch, but I always return to the source. What I like most from Gentoo are the possibility to get everything under control, deep customization and how do I learn from it.

This is not a generic guide that everybody can simply follow to get Gentoo installed in their system. This guide is so focused to everybody who wants to:
1. Install Gentoo Linux
2. Learn from a very funny Linux distro and its installation process
3. Want to have a systemd flavored Gentoo.

If you are that kind of person, speak *emerge* and enter :wink:

## Installation steps
> Before continue it's important to know that this guide if very focused to uefi, crypt/luks disk, lvm partitioning and systemd init system. If don't want to stick to any of these configurations it's also possible to follow this guide but remember to keep your eye on these steps to choose wherever you like. :thumbsup:

### Start live-cd environment
The first we need to install our Gentoo is a live-cd environment with uefi vars enabled. I like [systemrescuecd](http://www.system-rescue-cd.org) which is Gentoo based and has uefi vars enabled, so download it and write into [bootable usb](https://www.system-rescue-cd.org/Sysresccd-manual-en_How_to_install_SystemRescueCd_on_an_USB-stick) and let's move on!

Once you have you live-cd environment started simply check if it's really UEFI:
```
efivar -l
```
If the output listed the UEFI variables we can go on!

### Prepare Hard disk
There some available options to do that job, I prefer to use gdisk, but others like cgdisk or parted should do the job.
```
gdisk /dev/sda
```
Create new GUID partition table and destroy everything on disk:
```
o (Create a new empty GUID partition table (GPT))
Proceed? Y
```

Then create the following two partitions like below code. I'm not making a swap because I have enough ram but feel free to make one if you really want it (inside crypt partition).
```
n (Add a new partition)
Partition number 1
First sector 2048 (default)
Last sector +512M
Hex code EF00

n (Add a new partition)
Partition number 2
First sector 1050624 (default)
Last sector (press Enter to use remaining disk)
Hex code 8300
```
If everything looks good, save and quit:
```
w
Y
```
Partitions should look something like this:
```
gdisk -l /dev/sda
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         1050623   512.0 MiB   EF00  EFI
   2         1050624       500118158   238.0 GiB   8300  LVM
```

#### Prepare crypt container
Let's set up the encryption container where the lvm volume will be. Just before start creating it it's much important to check which is the best performance setup on your system because you can't change it once created, run:
```
cryptsetup benchmark
```
In my laptop the best performance setup is aes-xts:
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
Once we know that proceed with the creation itself:
```
cryptsetup -v --cipher aes-xts-plain64 --key-size 256 -y luksFormat /dev/sda2
```
Then open up our new crypt container to be able to create the lvm inside. In this procedure the label *cryptcontainer* is only a trivial name, so you can use whenever you like:
```
cryptsetup open --type luks /dev/sda2 cryptcontainer
```
#### Create lvm volumes
In that point we've our luks container and using lvm minimize the partitioning design time, because you can modify whenever you need. In my setup I use one partition for *root* filesystem and other for *home* files, feel free to add as much as you need/like:
```
pvcreate /dev/mapper/cryptcontainer
vgcreate vg0 /dev/mapper/cryptcontainer
lvcreate --size 50G vg0 --name root
lvcreate -l +100%FREE vg0 --name home
```
#### Create filesystems
Let's create the filesystems for our three partitions (two of them inside crypt lvm setup :v:):
```
mkfs.vfat -F32 /dev/sda1
mkfs.ext4 /dev/mapper/vg0-root
mkfs.ext4 /dev/mapper/vg0-home
```
#### Mount the new system
It's time to mount our partitions:
```
mount /dev/mapper/vg0-root /mnt/gentoo
mkdir -p /mnt/gentoo/boot
mount /dev/sda1 /mnt/gentoo/boot
mkdir /mnt/gentoo/home
mount /dev/mapper/vg0-home /mnt/gentoo/home
```

### Installing the Gentoo base system
Before installing Gentoo, make sure that the date and time are set correctly. A mis-configured clock may lead to strange results in the future! To check our current system date just run:
```
date
```
If needed set correct date like (March 02 22:09 2016):
```
date 030222092016
```
#### Install the stage3 tarball
In order to avoid starting from a Linux from scratch, the Gentoo developers provide us with a Stage 3 which is a base binary semi-working non-bootable environment suitable to save us such an amount of time. To do that we just only need to download it and uncompress in our filesystem tree:
```
cd /mnt/gentoo
wget http://mirror.eu.oneandone.net/linux/distributions/gentoo/gentoo/releases/amd64/autobuilds/current-stage3-amd64/stage3-amd64-20160225.tar.bz2
```
Unpacking the stage tarball into our local disk:
```
tar xvjpf stage3-*.tar.bz2 --xattrs
```
#### Configuring compile options:
As I said, Gentoo is a great Linux distro because its deep customization so portage is its cornerstone.

Portage is the Gentoo autobuild system, similar to packing systems like apt, yum or pacman in binary distros. It's inspired by FreeBSD port system. Pre-compiled binaries are also available, but these are out of reach of this guide.

Wikipedia says:
> Portage is Gentoo's software distribution and package management system. The original design was based on the ports system used by the Berkeley Software Distribution (BSD) based operating systems. The portage tree contains over 10,000 packages ready for installation in a Gentoo system.

Portage describe how to build the packages based on CPU architecture Flags and which features are available for every package with what are known as "use flags".

Let's see how to setting up portage with our system CPU flags:
```
nano -w /mnt/gentoo/etc/portage/make.conf
```
These are my Intel Core i7 available flags:
```
CFLAGS="-march=core-avx2 -O2 -pipe -mabm -madx -mavx256-split-unaligned-load -mavx256-split-unaligned-store -mprfchw -mrdseed"
CXXFLAGS="${CFLAGS}"
```
And MAKEOPTS describe how many allowed parallel jobs are available to use by building system. Usually I use the same that my CPU has, but somebody says that *number_of_cpu + 1* is the best option. I don't care
We can see how many cores we have running:
```
grep processor /proc/cpuinfo
processor       : 0
processor       : 1
processor       : 2
processor       : 3
```
Then I set it inside make.conf like:
```
MAKEOPTS="-j4"
```
If you want to know exactly which is the best option here you can read this post to choose your best option [MAKEOPTS=”-j${core} +1″ is NOT the best optimization](https://blogs.gentoo.org/ago/2013/01/14/makeopts-jcore-1-is-not-the-best-optimization/).
#### Selecting mirrors
Set the best mirrors for your location. Be sure that you have Internet access from your live-cd:
```
mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
```
#### Configuring the main Gentoo repository
Copy the official repository file and select which you want to use:
```
mkdir /mnt/gentoo/etc/portage/repos.conf
cp /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
nano -w /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```
#### Copy DNS info
Copy the dns information from your working live-cd environment:
```
cp -L /etc/resolv.conf /mnt/gentoo/etc/
```
#### Mounting the necessary filesystems
In addition to lvm filesystem we created in our local disk, there's other pseudo-filesystems which are created during system boot that are needed to chroot into our new environment:
```
mount -t proc proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
```
#### Entering the new environment
Chroot into the new environment:
```
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```
Great! We are now inside our final Gentoo system but we need to bake it a little more time.
### Configuring Portage
Portage consists of two main parts, the ebuild system and emerge. The ebuild system takes care of the actual work of building and installing packages, while emerge provides an interface to ebuild: managing an ebuild repository, resolving dependencies and similar issues.

First we need to synchronize the remote repositories with the local Portage tree to knows what packages are available to be installed.

#### Installing a Portage snapshot
Download an snapshot from the remote repositories we defined inside make.conf:
```
emerge-webrsync
```
#### Updating the Portage tree
Then update the snapshot with the latest version of the repos:
```
emerge --sync
```
#### Choosing the right profile
A Portage profile specifies default values for global and per-package USE flags, specifies default values for most variables found in /etc/portage/make.conf, and defines a set of system packages. The profiles are maintained by the Gentoo developers as part of the Portage tree (/usr/portage/profiles).

List the available profiles:
```
eselect profile list
```
From the output list we must select our best fitting option. Once our system is based on systemd, the best profile we can choose is the *systemd*. Of course we can choose others like Gnome which needs systemd to run, but we want a minimal environment to be configured.
```
[..]
[12]  default/linux/amd64/13.0/systemd
[..]
```
Choose the profile with eselect utility:
```
eselect profile set 12
```
#### Configuring the USE variable
As we said [USE flags](https://wiki.gentoo.org/wiki/USE_flag) are a core feature of Gentoo, and a good understanding of how to deal with them is needed for administering a Gentoo system.

The USE variable allows the system wide setting or deactivation of USE flags in a space separated list while are set inside [/etc/portage/make.conf](https://wiki.gentoo.org/wiki//etc/portage/make.conf#USE) or for better fine grained per package control we should set them inside [/etc/portage/package.use](https://wiki.gentoo.org/wiki//etc/portage/package.use).

If we use tools like *ufed* we can easelly set system wide Flags since I prefer to choose it manually to set per package defined Flags and use as minimum as possible as system-wide. So, if you choose the easy/system-wide way simply install ufed with:
```
emerge ufed
```
And then run it to select the USE Flag with a beautiful user interface :smile:
```
ufed
```
If you, like me, want to get everything under control, then you would love an utility named *quse* from *portage-utils* package. This utility tell us which package use any specified flag.
For example, if you want to know which packages use the *dbus* flag we simply need to run:
```
quse dbus
```
And all packages will be shown to us.

Ok, nothing better than test it by ourself!
```
emerge app-portage/portage-utils
```

These are the system wide USE Flags described in my make.conf file:
```
nano -w /etc/portage/make.conf
```
```
USE="X aac alsa aspell bash-completion boost branding caps contrib \
     cpudetection cryptsetup dbus exif filecaps flac gd gpm gstreamer gzip \
     hardened hddtemp highlight hostname imagemagick int64 introspection \
     jemalloc jpeg jpeg2k json kdbus kmod kms libav libnotify lm_sensors lz4 \
     lzma lzo mime mp3 mp4 mpeg nano-syntax networkmanager nss numa ogg \
     opencl opengl pango pcap pdf pie png pulseaudio python recode samba sdl \
     seccomp simplexml slang smp sockets sound spell sse sse2 ssh \
     startup-notification svg systemd tcmalloc theora threads \
     timezone truetype udisks upnp upnp-av upower usb video vim vim-syntax \
     vorbis wavpack wayland wayland-compositor wifi xcomposite xfs xft \
     xinerama xkb xml xmlrpc xorg xpm xrandr xwayland xz zeroconf \
     zsh-completion -cups"
```
The rest of the flags are per package defined inside */etc/portage/package.use/* and here you can choose to set everything inside a file or not, simply arrange them as you like. I created these files (you can get it from this same repository):
```
ls /etc/portage/package.use/
```
```
admin devel emulation fonts fs games media network office perl system terms themes utils xorg
```
USE flags are not the only optimizations we want from our portage system. Other flags corresponding to the instruction sets and other features specific to the x86 (amd64) architecture are configured into the variable called CPU_FLAGS_X86.

We can simply use a useful python script to auto-detect which of them we should set. Install *app-portage/cpuinfo2cpuflags*:
```
emerge --ask app-portage/cpuinfo2cpuflags
```
And then run it to get those values:
```
cpuinfo2cpuflags-x86
```
```
CPU_FLAGS_X86="aes avx avx2 fma3 mmx mmxext pni popcnt sse sse2 sse3 sse4_1 sse4_2 ssse3
```
Put the output line into */etc/portage/make.conf*
```
cpuinfo2cpuflags-x86 >> /etc/portage/make.conf
```
If you would like (and believe me you should) get deeper knowledge of portage system, don't hesitate to read the official documentation [wiki.gentoo.org](https://wiki.gentoo.org/wiki/Portage).
### Configuring base system
In addition to Portage there's some other options should be configured before the end of the bake :grin:
#### Timezone
Configuring the time zone of our system's clock:
```
echo "Europe/Madrid" > /etc/timezone
```
Next, reconfigure the *sys-libs/timezone-data* package, which will update the /etc/localtime file based on the /etc/timezone entry. The /etc/localtime file is used by the system C library to know the timezone the system is in:
```
emerge --config sys-libs/timezone-data
```
#### Configure locales
Locales are a set of information that most programs use for determining country and language specific settings. To set the system locales we need to set it inside:
```
nano -w /etc/locale.gen
```
And then execute locale-gen to generate all the locales specified in the /etc/locale.gen file and write them to the locale-archive (/usr/lib/locale/locale-archive).
```
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
```
eselect locale set 2
```
Now reload the environment:
```
env-update && source /etc/profile && export PS1="(chroot) $PS1"
```
#### Installing Systemd
Before continue we must remove udev and openrc otherwise we've cyclic dependencies in the future:
```
emerge --deselect sys-fs/udev
emerge --unmerge sys-fs/udev
```
Also be sure that we don't have nothing related with openrc or systemd as masked package:
```
emerge --deselect sys-apps/openrc
emerge --unmerge sys-apps/openrc
rm /etc/portage/package.mask/systemd
```
Once we have our system prepared it's time to ensure that we have systemd installed with all required USE flags:
```
emerge -av app-portage/gentoolkit
euse -E cryptsetup systemd gudev dbus
emerge -av sys-apps/systemd
emerge -av sys-apps/dbus
```
### Configuring Linux kernel
While Portage is the core of Gentoo Linux system the Linux kernel is the core of the operating system and offers an interface for programs to access the hardware. The kernel contains most of the device drivers.

To create a kernel, it is necessary to install the kernel source code first. The Gentoo recommended kernel sources for a desktop system are, of course, sys-kernel/gentoo-sources. These are maintained by the Gentoo developers, and patched to fix security vulnerabilities, functional problems, as well as to improve compatibility with rare system architectures. But as always in Gentoo there's other options also available.


To get a full list of kernel sources with short descriptions can be found by searching with emerge:
```
emerge --search sources
```

#### Installing the sources
Let's install the gentoo-sources:
```
emerge --ask sys-kernel/gentoo-sources
```
Now it is time to configure and compile the kernel sources. There are two approaches to do that job:
1. The kernel is manually built and install.
2. A tool called genkernel is used to automatically build and install the Linux kernel.

In both cases we must configure it manually that normally is the most difficult procedure a Linux user ever has to perform. Nothing is less true - after configuring a couple of kernels no-one even remembers that it was difficult :wink:

So I'll explain how to use genkernel which help us to maintain config files and some more automations :smiley:

However, one thing is true: it is vital to know the system when a kernel is configured manually. Start by install required packages:
```
emerge --ask sys-apps/pciutils
```

Edit */etc/genkernel.conf* to set our preferences:
```
nano -w /etc/genkernel.conf
```
```
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
Notice that LVM, LUKS and UDEV must be set on to have this system works otherwise will not boot.

Once configured then simply run:
```
genkernel all
```
#### Configuring the modules
If we need to auto-load a kernel module each time to system boots we should specify it in */etc/conf.d/modules* file.

You can list your available modules with:
```
find /lib/modules/<kernel version>/ -type f -iname '*.o' -or -iname '*.ko' | less
```
#### Installing firmware
Some drivers require additional firmware to be installed on the system before they work. This is often the case for network interfaces, especially wireless network interfaces. Most of the firmware is packaged in *sys-kernel/linux-firmware*, so installing them are almost needed in a laptop system:
```
emerge --ask sys-kernel/linux-firmware
```
#### LVM Configuration
Install lvm tools if it's not yet installed:
```
emerge -av sys-fs/lvm2
```
Then edit the package configurations:
```
nano -w /etc/lvm/lvm.conf
```
```
use_lvmetad = 1
issue_discards = 1
volume_list = ["vg0"] # Our VG volume name, check with vgdisplay
```
#### Fstab
Before editing fstab we need to know which UUID are using our devices inside and outside lvm and luks volumes:
```
blkid /dev/mapper/vg0-root | awk '{print $2}' | sed 's/"//g'
UUID="576e229c-cf68-4010-8d85-ff8149158416"
blkid /dev/mapper/vg0-home | awk '{print $2}' | sed 's/"//g'
UUID="95fa5807-ea57-4cf5-b717-74f4aba190e2"
```
Then edit fstab:
```
nano -w /etc/fstab
```
```
/dev/sda1                                       /boot   vfat    noatime                                         1 2
UUID="576e229c-cf68-4010-8d85-ff8149158416"     /       ext4    discard,noatime,commit=600,errors=remount-ro    0 1
UUID="95fa5807-ea57-4cf5-b717-74f4aba190e2"     /home   ext4    discard,noatime,commit=600                      0 0
tmpfs                                           /var/tmp tmpfs  nodev,nosuid    								0 0
tmpfs                                           /tmp    tmpfs   nodev,nosuid    								0 0
```
#### Configuring crypttab
Warning!!! As we don't have encrypted partitions other than root which must be mounted by systemd before the whole system start we don't need to set it up there, so, our crypttab must be empty.
#### Configuring mtab
In the past some utilities wrote information (like mount options) into /etc/mtab and thus it was supposed to be a regular file. Nowadays all software is supposed to avoid this problem. Still, before switching the file to become a symbolic link to /proc/self/mounts.

To create the symlink, run:
```
ln -sf /proc/self/mounts /etc/mtab
```
### Systemd boot (bootloader)
Verify your EFI variables are accessible:
```
emerge -av sys-libs/efivar
efivar -l
```
Verify that you have mounted /boot
mount | grep boot

Install systemd-boot:
```
bootctl --path=/boot install
```
It will copy the systemd-boot binary to your EFI System Partition ($esp/EFI/systemd/systemd-bootx64.efi and $esp/EFI/Boot/BOOTX64.EFI - both of which are identical - on x64 systems) and add systemd-boot itself as the default EFI application (default boot entry) loaded by the EFI Boot Manager.
#### Add bootloader entries
Add one entry into bootloader with this options:
```
nano -w /boot/loader/entries/gentoo.conf
```
```
title    Gentoo Linux
efi      /kernel-genkernel-x86_64-4.4.4-gentoo
options  initrd=/initramfs-genkernel-x86_64-4.4.4-gentoo crypt_root=/dev/sda2 root=/dev/mapper/vg0-root ro dolvm
```
Edit default loader:
```
nano -w /boot/loader/loader.conf
```
```
default gentoo
timeout 3
```
### Rebooting into our new Gentoo systemd :D
#### Enable lvm2
```
systemctl enable lvm2-lvmetad.service
```
#### Change the root password
While in chroot we need to change the root password of our new system just before rebooting it.
```
passwd
```
#### And reboot (you don't need to cross your fingers, everything should goes fine :shrink:)
```
exit # To exit from chroot
sync # To sync filesystems
reboot
```
## Post-installation
### Setting the Hostname
When booted using systemd, a tool called hostnamectl exists for editing /etc/hostname and /etc/machine-info. So we don't need to edit the file manually simlpy run:
```
hostnamectl set-hostname <HOSTNAME>
```
### Configuring Network
Let's plug our new Gentoo system to the world!! :earth_africa:

We can choose between two options, to use systemd as network manager or standalone one, I prefer networkmanager but here are the two options:
#### Using systemd-networkd
systemd-networkd is useful for simple configuration of wired network interfaces. It is disabled by default.

To configure systemd-networkd we must create a \*.network file under /etc/systemd/network. Here is an example for a simple DHCP configuration:
```
nano -w /etc/systemd/network/50-dhcp.network
```
```
[Match]
Name=en*
[Network]
DHCP=yes
```
And then tell systemd to manage and start that service:
```
systemctl enable systemd-networkd.service
systemctl start systemd-networkd.service
```
#### Using NetworkManager
Often NetworkManager is used to configure network settings. I personally use nmtui because it's easy and has a ncurses client. Just install it:
```
emerge networkmanager
```
And now simply run the following command and follow a guided configuration process through nmtui:
```
nmtui
```
### Setting locales
Yes, you're right, we set the locales before but once booted with systemd, the tool localectl is used to set locale and console or X11 keymaps. So, let's set it again to be sure that everything goes fine.

To change the system locale, run the following command:
```
localectl set-locale LANG=en_US.utf8
```
Change the virtual console keymap:
```
localectl set-keymap es
```
And finally, to set the X11 layout:
```
localectl set-x11-keymap es
```
### Setting time and date
Time and date can be set using the timedatectl utility. That will also allow users to set up synchronization without needing to rely on net-misc/ntp or other providers than systemd's own implementation.

To set the local time of the system clock directly:
```
timedatectl set-time "yyyy-MM-dd hh:mm:ss"
```
To set time zone:
```
timedatectl list-timezones
timedatectl set-timezone Europe/Madrid
```
Set systemd-timesyncd as a simple SNTP daemon. Systemd-timesyncd that only implements a client side, focusing only on querying time from one remote server. It should be more than appropriate for most installations.
```
timedatectl set-ntp true
```
To check the status of the daemon:
```
timedatectl status
```
