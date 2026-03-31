+++
date = '2026-03-30T13:21:40+02:00'
draft = false 
title = 'Use RBD image as rootfs'
subtitle = "Map and mount RBD images to boot Linux-based systems"
summary = ""
description = ""
author = "Lucas"

tags = ["Ceph", "RBD", "Initramfs", "Network boot"]
categories = ["Technical"]

featuredImage = "https://lucas-cauhe.github.io/blog/images/rbdroot/rbdroot-main.png"
featuredImagePreview = "https://lucas-cauhe.github.io/blog/images/rbdroot/rbdroot-main.png"

author-link = "https://github.com/lucas-cauhe/rbdroot"
+++

## Summary

Typical linux-based OS installations group the compiled kernel version, or
kernel image, the bootloader and the filesystem mounted at `/` (rootfs) in the
same hard drive. Take a look at the output of the command `fdisk -l /dev/vda`

```
Disk /dev/vda: 3 GiB, 3221225472 bytes, 6291456 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: B6811CDB-4DA2-4C54-AFBC-5BDBF59435E4

Device      Start     End Sectors  Size Type
/dev/vda1  262144 6289407 6027264  2.9G Linux root (x86-64)
/dev/vda14   2048    8191    6144    3M BIOS boot
/dev/vda15   8192  262143  253952  124M EFI System

Partition table entries are not in disk order.
```

The same block device contains the boot sector, efi and rootfs. What if the
rootfs was loaded over the network? In this blog post I will showcase the benefits
of using an RBD-backed rootfs, an example setup and further improvements.

Before explaining any further I will be using some terminology that might be
confusing sometimes, so let me explain it first.
I will refer to the `init` script located at the initramfs like `init0` and to
the one in the rootfs as `init1` for convenience.
Also the words `initrd` and `initramfs` are used interchangeably.

### Linux kernel boot process

On the early stage of the boot process, a compiled version of the kernel is
loaded into a memory filesystem from the MBR along with some initial modules,
binary files and extra configuration called initramfs (old initrd).

During this first stage, a script called `init` (`init0` from now on) loads
basic kernel modules, mounts core filesystems (such as procfs, sysfs and rootfs)
and swaps the root filesystem from RAM-based to the device specified in the kernel parameters.

This script is just a shell script, it can be freely modified and perform any
sysadmin work, as long as the required kernel modules are loaded.

Once the rootfs device has been changed, `init0` tries to find another `init`
executable (`init1` from now on), the one that brings most of the OS utilities up,
the process with pid 1.

### Network rootfs

There are two main sources to chose from for a device that contains a rootfs:
local block devices and network devices. From the latter either network
filesystems (e.g. nfs, smb, cephfs) or network block devices (e.g. iSCSI, RBD)
can be used.

As I mentioned before, `init0` can be modified to connect
to some of these network sources. I found the following
[patch](https://lists.debian.org/debian-kernel/2015/06/msg00163.html)
published in debian's mailing lists where a RBD device can be mapped and the FS
in it mounted to act as the rootfs.

## RBD image creation

Prior to mapping any device to the target system, you need an
RBD image with a filesystem installed in it.

**Ceph user**

First you will need to create, or use an existing, Ceph user with admin
capabilities over an RBD-enabled pool. Keep in mind that connecting to the cluster
from `init0` requires the user's name and keyring, target pool and image name.

**Export the rootfs image**

What you really want to install in the RBD image is just a filesystem, from a
specific Linux distribution. In fact, it can be anything that contains an executable
named `init` and a modules directory. This is what containers are, with some
additional configuration to isolate the filesystem's environment.

There are some easy to use command line utilities such as
[docker export](https://docs.docker.com/reference/cli/docker/container/export/)
or
[debootstrap](https://wiki.debian.org/Debootstrap) (for creating Debian systems)
that let you create rootfs images.
If you already have configured a base system and would like to export it, you
can try to use both, `fdisk` to list the partition start and end sectors of your
rootfs and `dd` to extract it into a raw image.

Consider the output of the `fdisk` command from the top of the post, you can
extract the `Linux root (x86-64)` partition by following these two steps.
First ACPI shutdown the machine, this prevents from booting an inconsistent image.
Then, if it is physically installed, boot from another device, such as a live ISO,
or if it is a virtual machine, dump its virtual block device into a file.
Next, run

```
$ dd if=/route/to/device/or/file of=/route/to/extracted-rootfs.img \
bs=$SECTOR_SIZE \
skip=$ROOTFS_START \
count=$ROOTFS_SIZE
```

where`SECTOR_SIZE`, `ROOTFS_START`and`ROOTFS_SIZE` would be 512, 262144 and
6027264, respectivelly according to the output above.

Now if you want to modify its contents, mount it locally (this will create a
loop device) and `chroot` into it.

**Update the rootfs**

There is some basic configuration to be done before booting with this image as
rootfs. The `init0` script sets up some basic networking, enough to connect to
the Ceph cluster. If, at any time, the conection is lost and the system needs
to access the RBD mapping (e.g. by swapping or loading additional modules), the
boot process will fail. For this reason, it is important to modify the rootfs
network configuration so that you make sure that the network interface that connects
to the Ceph cluster won't be updated, even if it is just to configure it as it
already is.

Another crucial step is to make sure that the uuid of the image is used for the
root mount in the `/etc/fstab` of the rootfs. Otherwise some incosistencies could
happen or in some cases, `init1` could fail.
In the following section, I will talk about another special mountpoint to consider,
but for now, make sure that the boot device, where kernel and initrd
images are installed, has a boot partition label and it is included in the `/etc/fstab`
file pointing at `/boot`.

**Import the image in Ceph**

Once the image is updated and is stable, import the image into the RBD pool you
have defined for it using the `rbd import` command and your user's keyring.

## Initramfs image creation

In order for `init0` to map an RBD image, the rbd kernel module and
a helper script have to be loaded in the initramfs image.

Linux distros provide the `initramfs-tools` package to create and update these
images with specific kernel versions.
I have also published [this GitHub repository](https://github.com/lucas-cauhe/rbdroot)
with some utility scripts that create a custom image for the purpose of mounting
an RBD-backed rootfs.

It is important to note that there might be compatibility issues between
kernel modules if the versions from initrd and rootfs don't match or are compiled
with different flags. Moreover, many distros package installation finishes with
the update of initrd, if the image that gets updated is the one from the rootfs,
changes won't take effect at all. For this purpose there needs to be a separate
`/boot` folder mounted in the rootfs that points to the device being used to boot
the system.

### Creating a custom grub image

In order to have a better control of the boot process and tweak kernel parameters
on the fly, installing the kernel and initrd images in a bootable grub partition
is a good idea.

This can also help extend the network-based boot process by telling grub to
fetch the kernel and initrd images from a network endpoint, allowing to have
a minimal (in size) and read-only local drive.

## Boot configuration

Once you have applied the patch mentioned above and installed a bootable device,
you need to pass some parameters to the kernel in order for the `init0` to
mount the rootfs.

- `root` and `boot` must be set to `/dev/rbd` (or anything that is not a local device)
  and `rbd`, in order to use `rbd` custom scripts.
- `ip` is a well documented parameter that will specify your host's network
  configuration. Consider that if you need to set addition config such as VLAN support
  you will need to use custom scripts in the `initrd` image inside the `/scripts/rbd-*`
  directories. This was my case so I added
  [this file](https://github.com/lucas-cauhe/rbdroot/blob/main/assets/rbd-top/vlan)
  as suggested [here](https://github.com/stcz/initramfs-tools-network-hook/tree/main).
- `rbdroot` is the new parameter defined in `init0` and is also well documented
  in its original patch.
- I also specify the `rootfstype` parameter because `init0` sometimes had issues to
  recognise the fstype from the RBD mapping.

## Benefits of rbdroot

One of the main benefits of this boot setup is that it is software-based, while
solutions like PXE sometimes rely on vendor-specific features or hardware
restrictions from your datacenter's architecture.
In my case, I had to go over the switch manufacturer's manual to figure out how
to deal with VLAN translation (mainly PVID configuration) and NIC partition so
that one partition could work in tagged mode and another as defined by the system.

### Prevent enterprise-level servers SD failure

Network-based boot requires little installation in the server's local drives,
that being just the grub partition.
Many enterprise-level server's system installation is done in SD cards, whose
write performance is very limited. When performed continuously, this kind of
operation damages the SD card, making it crash sooner than expected.

However, if all that is needed is to read the boot sector and rarely
perform writes for updating the initrd image, SD cards' can live longer and are
a cheaper option than hard or solid state drives.

### Centralised administration

RBD-based rootfs can help with centralising the configuration of the base system,
using a version table for example, where in the same pool you have the
\"development\" version of the rootfs image and several snapshots, one for each
version update and target server.
This way, a generic `rbd` mount script could retrieve the hostname it is
running at and mount a specific version.
Any update that the administrator plans, would be done on the \"development\" image
and once ready take a snapshot of it.

Updates could be configured either in the `rbd` scripts from the initrd image
to fetch the latest version for the server or have a local distro packages
repository that would update initrd image. Then reboot the host and the update
is installed.

I have suggested to take one snapshot for every version and host so that no two
hosts are using a shared root filesystem, which can lead to concurrency problems.
However if you expect to have a read-only system or install a shared filesystem,
it is totally okay to take just one snapshot for each version.

## Future work

This setup can be extended in many ways, below I highlight some of the extensions
I would go for next.

### Cloud-init drive

Cloud-init is a cloud project that aims to provision a system based on some variables and
scripts, defined by the user. Software installed in the system can
interpret and apply these configuration either offline or online.
This software is usually deployed as distribution packages and virtual devices
attached to the VM instance running those software packages.

Every cloud provider either on-premises or managed (public cloud providers)
has implemented this functionality for their infrastructure. Following their
naming conventions and locating where they store cloud-init drives, a physical
server could be managed from the same control plane as virtual machines.

### CephFS

Another approach to have a network-based boot would be to use a shared filesystem
such as NFS or, since I have access to a running Ceph cluster, CephFS.

This could help with sharing the same filesystem accross multiple hosts and limiting the
permissions each of these have over admin directories, hence focusing each on
the parts of the system they can modify.

This would involve updating the `rbd` script from the initrd image and implement
CephFS mount.

### Network-based kernel and initrd images

As I mentioned above, the kernel and initrd images with this setup need to be
installed in the boot partition so that grub loads the kernel successfully. This
can be improved making the entire boot media a read-only drive, if grub loads
these images from a network endpoint. This could lead to eventually making the
system entirely RAM-based.
