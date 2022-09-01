This is a project based on [lfs book](https://www.linuxfromscratch.org/) which is about building its own Linux distribution, from scratch. Disk partitioning, build of cross-compilation tools, packages from sources, kernel configuration & build, system configuration, internet connection and finally personnal pimp. This project gives us a great overall of what a distribution is made of, how it was born.

### Presentation
For **security** reasons this project must be done in a VM. We have to build this distribution from a linux, as we are sure to have the required build tools and a compatible filesystem. At the end we must dual boot this machine with the help of **grub**.
Consequently there are two ways to be wrong: mix sources/libraries between host and guest, and miss the grub configuration so that the host could never start again.
So do this in a VM make this project **sand-boxed**, but also gives a great liberty of action as we can **snapshot** the state of the machine so we can **go back** if necessary.

As I lack space on my computer, I choosed to host the LFS on an **Alpine system**, which is a distribution made to be the smallest as possible (this is why it's the favorite one for docker). I took the **virtual alpine release** which is free of desktop environment. I just had to add few missing packages to complete my build tools.
So I gave it 20GB of disk space to run, in the idea to split at the end 10GB per distribution.

### Big picture
A 20GB disk is partitionned for the host. At the end, a bootloader will have to choose to boot one disk or another (each one leading to one or another **system**). So the LFS has to be on another disk. We can make it by **partionning** the host disk.
Here we are on a **BIOS** starter and we are limited to 4 primary partitions. 3 are already used by the host so the way to do this is to create an **extended** partition, which is kind of primary type, then to create inside it 3 partitions we want: **boot** (60MB is enough for this project), **swap** (1 or 2 GB) and **root** (I gave 11.3GB). The filesystem we must set on these partitions are respectively **ext4, swap and ext4**. Filesystem is a core technology of a system as it defines the way the system sees the disk, roams it and so on...

`rc-status`: status of running deamons
`apk info bash`: apk is the packet manager on alpine

### Disk partition
```h
ft-linux:/# fdisk -l
Disk /dev/sda: 20 GB, 21923627008 bytes, 42819584 sectors
2665 cylinders, 255 heads, 63 sectors/track
Units: sectors of 1 * 512 = 512 bytes

Device  Boot StartCHS    EndCHS        StartLBA     EndLBA    Sectors  Size Id Type
/dev/sda1 *  0,32,33     12,223,19         2048     206847     204800  100M 83 Linux
/dev/sda2    12,223,20   535,10,51       206848    8595455    8388608 4096M 82 Linux swap
/dev/sda3    535,10,52   899,130,63     8595456   14450687    5855232 2859M 83 Linux
/dev/sda4    899,131,1   1023,254,63   14450688   42819583   28368896 13.5G  5 Extended
/dev/sda5    1023,254,63 1023,254,63   18923520   42819583   23896064 11.3G 83 Linux
/dev/sda6    899,163,33  924,127,35    14452736   14852095     399360  195M 83 Linux
/dev/sda7    924,160,5   1023,254,63   14854144   18921471    4067328 1986M 82 Linux swap
```
*`fdisk -l` show us all disks, even those which aren't mounted*

We can't see the extended partition on the host as it is a new one reserved for LFS so it is not mounted on the host. The configuration file to choose which disk is mounted at startup is `/etc/fstab` and we will set it on lfs system later.
In order to start to build the lfs system, we create a directory at `/mnt/lfs` and mount the root and boot disks on it as follow ($LFS=/mnt/lfs):
`mount -v -t ext4 /dev/sda5 $LFS`
`mount -v -t ext4 /dev/sda6 $LFS/boot`
`swapon /dev/sda7`

`mount` (information on mounted disks)
`findmnt` (tree of mounted disks & filesystem)

We will mount them again if we reboot the host. (check with `mount`)
Now the extended partition is accessible from the host.

### Environnment
Create a lfs user with rights on lfs filesystem by saying lfs user is the owner of it.
`adduser lfs`
`chown -v lfs $LFS/{usr{,/*},lib,var,etc,bin,sbin,tools}` 
`chown -v lfs $LFS/sources`

### Compilation

A SBU is a time unit depending on your system. Binutils package is the reference so after timing this build you can know how long a SBU takes on your system.
1 SBU:
```
real	3m48.897s
user	3m17.906s
sys	0m26.999s
```
*This is my SBU time result*

There is two ways to indicate to `make` to use many processors at compilation time to accelerate the building:
`export MAKEFLAGS='-j4'` or `make -j4`


`wget -c https://ftp.gnu.org/gnu/mpc/mpc-1.2.1.tar.xz -O - | tar x -J` download a packet, output it on stdout and extract it

`du -ch -d 0 gcc-11.2.0` size of a directory

`find /usr/{lib,libexec} -name \*.la -delete` 

> Device creation is usually done by Udev during startup. Since this new system does not yet contain Udev and has not yet been started, it is necessary to mount and populate /dev manually. This is done by double mounting the host system's /dev directory. Duplicate mount is a special type of mount that allows you to mirror a directory or mount point to another location
```
mount -v --bind /dev $LFS/dev
mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
```

Enter the chroot environment:
```
chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin     \
    /bin/bash --login
```

> On Linux-based systems, the /dev directory is used to hold file-based devices, "nodes", which refer to system devices. Each "node" refers to a device, which may or may not exist. User applications can use these "nodes" to interact with the device. For example, the X graphics server will "listen" to /dev/input/mice which refers to the mouse and move the pointer on the screen.

**udev**
Udev is the new system for managing the /dev directory, designed to overcome the limitations put forward by previous versions of /dev, and provide a robust link. In order to create and name the devices in /dev, the "nodes" which correspond to system devices, udev makes the link between the information given by sysfs and the rules given by the user.

**sysfs**
Sysfs has been made official with the 2.6 series kernels. It is managed by the kernel to export basic information about devices currently connected to the system. Udev uses this information to create the 'nodes' corresponding to the peripherals of your computer. Sysfs is mounted on /sys and you can browse it: you can look at these files before diving into udev.

**systemd**
It is the first program launched by the kernel (therefore it has the PID N°1) and it is responsible for launching all the following programs in order until obtaining an operational system for the user, according to the determined mode (single user, multi-user, graphic). It is also responsible for restarting or shutting down your computer properly.
It was developed in order to better understand the management of a multitasking system, particularly in terms of dependency between the different services launched at startup with the main objective of optimizing system performance.
Its role is more extensive than Upstart, it also manages the mounting of different file systems and introduces a new log system called "The Journal".
It introduces the notion of unity. A unit represents a configuration file. Among others, a unit can be a service (*.service), a target (*.target), a mount (*.mount), a socket (*.socket)…

**BIOS**
> The BIOS in modern PCs initializes and tests the system hardware components (Power-on self-test), and loads a boot loader (like grub) from a mass storage device which then initializes an operating system. In the era of DOS, the BIOS provided BIOS interrupt calls for the keyboard, display, storage, and other input/output (I/O) devices that standardized an interface to application programs and the operating system. More recent operating systems do not use the BIOS interrupt calls after startup.
Most BIOS implementations are specifically designed to work with a particular computer or motherboard model, by interfacing with various devices especially system chipset.
Unified Extensible Firmware Interface (UEFI) is a successor to the legacy PC BIOS, aiming to address its technical limitations.

https://en.wikipedia.org/wiki/BIOS
![42aa6be3455d9fe99e691b1cb34d9730.jpg](../_resources/42aa6be3455d9fe99e691b1cb34d9730.jpg)

**UEFI**
> The Unified Extensible Firmware Interface (UEFI) is a publicly available specification that defines a software interface between an operating system and platform firmware. UEFI replaces the legacy Basic Input/Output System (BIOS) boot firmware originally present in all IBM PC-compatible personal computers, with most UEFI firmware implementations providing support for legacy BIOS services. UEFI can support remote diagnostics and repair of computers, even with no operating system installed.

> The interface defined by the EFI specification includes data tables that contain platform information, and boot and runtime services that are available to the OS loader and OS. UEFI firmware provides several technical advantages over a traditional BIOS system:
> - Ability to boot a disk containing large partitions (over 2 TB) with a GUID Partition Table (GPT)
> - Flexible pre-OS environment, including network capability, GUI, multi language
> - 32-bit (for example IA-32, ARM32) or 64-bit (for example x64, AArch64) pre-OS environment
> - C language programming
> - Modular design
> - Backward and forward compatibility

https://en.wikipedia.org/wiki/UEFI
![89e5585a5b087cb3164410538844d85f.jpg](../_resources/89e5585a5b087cb3164410538844d85f.jpg)

**/boot/**
> In Linux, and other Unix-like operating systems, the /boot/ directory holds files used in booting the operating system. The usage is standardized in the Filesystem Hierarchy Standard.
> The contents are mostly Linux kernel files or boot loader files, depending on the boot loader, most commonly (on Linux) LILO or GRUB.

>**Linux**
> - `vmlinux` – the Linux kernel
> - `initrd.img` – a temporary file system, used prior to loading the kernel
> - `System.map` – a symbol lookup table

>**GRUB**
GRUB stores its files in the subdirectory grub/ (i.e. /boot/grub/). These files are mostly modules (.mod), with configuration stored in grub.cfg. 

> /boot/ is often simply a directory on the main (or only) hard drive partition. However, it may be a separate partition.

https://en.wikipedia.org/wiki//boot/
 
**vmlinuz-5.16.9**
> The engine of the Linux system. When the computer starts, the kernel is the first part of the operating system to be loaded. It detects and initializes all hardware components of the computer, then makes the components available in a file tree for the software that needs them, and transforms a single-processor machine into a multitasking machine capable of executing several programs almost simultaneously.

`grub-install /dev/sda`

```
set default=0
set timeout=5

menuentry "Alpine Linux" {
 set root=(hd0,1)
 linux /vmlinuz-virt root=UUID=e89325b9-25da-42b1-ace2-569ba4950b8d modules=sd-mod,usb-storage,ext4 quiet
 initrd /boot/initramfs-virt
}

menuentry 'GNU/Linux, with Linux 5.16.9-jle-corr' {
 insmod gzio
 insmod part_msdos
 insmod ext2
 set root='hd0,msdos6'

 echo 'Loading Linux 5.16.9-jle-corr ...'
 linux /vmlinuz-5.16.9-jle-corr root=/dev/sda5 ro
}
```
*/boot/grub/grub.cfg*

missing:
Eudev (3.1.2)  (but it's a /dev manager forked from dev, to be compatible with openRC or init systems. As I use systemd, this is useless)
Sysklogd (1.5.1)
Sysvinit (2.88dsf)  (init system, useless to me as I use systemd)
Time Zone Data (2015f) (seems to be included with glibc)
Udev-lfs Tarball (udev-lfs-20140408) (seems outdated as udev is included in systemd)

bonus:
- tree
- traceroute
- openssh
- wget
- curl
- git
- valgrind