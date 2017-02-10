---
categories:
  - "administration"
  - "virtualization"
keywords:
  - "vmware"
  - "xen"
tags:
  - "vmware"
  - "xen"
  - "ubuntu"
  - "v2v"
  - "migration"
date: "2015-10-12"
title: "Migrating old Ubuntu XEN VMs to VMware"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-vmware.png"

---

Migrations are a common part of being a Virtualization Admin. Most commonly you will be asked to perform P2V (physical to virtual), however there are times when you will have to migrate between virtual platforms. Most recently, I had to migrate a pair of VMs under the control of a third-party entity who were running some sort of xen environment.
<!--more-->

I was not given any specific details about the environment, just two image files which I was told were the root drives of their existing VMs. After personally being asked by two of my superiors to get this done ASAP due to some kind of "emergency situation", I reluctantly agreed. Near the end of the work day, a colleague and I began to work on this. This is what we found:

These images were about 1GB each in size and in .img format. They were not full disks but rather just partitions (from previous experience with xen, I am aware that you can specify partitions and not just full disks). After mounting them, we were surprised to find out that one of them was a root drive of a ancient [Ubuntu 6.06](http://old-releases.ubuntu.com/releases/6.06.0/) server! Time to roll up our sleeves and get to work.

### Migrating a XEN Ubuntu 6.06.2 LTS VM over to VMware Manually
There's no way that the [vmware converter](https://www.vmware.com/support/pubs/converter_pubs.html) can handle this job since it doesn't like the xen kernel, filesystem or bootloader. OK that’s fine, we can do this manually. Let's do this one first and prepare the destination VM; we created the VM with oldest emulated hardware available:

> **Guest OS:** Ubuntu 32-Bit<br />
> **Disk Controller:** BusLogic Parallel<br />
> **Network Adapter:** E1000

### Step 0
Great, now lets boot our VM to one of our favourite Linux Sys-Admin live ISOs… [Finnix](http://www.finnix.org/)! After enabling ssh for comfortable terminal experience we got straight to work.

### Step 1
Let’s partition and format our new 1GB drive to be similar to existing image:

```
parted -s /dev/sda mklabel msdos
parted -a optimal /dev/sda mkpart primary 0% 100%
mkfs.reiserfs /dev/sda1
```

### Step 2
Now lets mount both the xen image file provided by the 3rd-part (which we placed on an NFS share) and our new partition and copy over contents:

```
mkdir /mnt/{local,nfs,loop}
/etc/init.d/rpcbind start
mount nas:/zpool1/misc /mnt/nfs/
mount -o loop /mnt/nfs/vm_xen_ubu_root.img /mnt/loop/
mount /dev/sda1 /mnt/local/
cp -rp /mnt/loop/* /mnt/local/
```

### Step 3
OK, lets prepare to chroot in there now

```
mount -t proc proc /mnt/local/proc/
mount -t sysfs sys /mnt/local/sys/
mount -o bind /dev /mnt/local/dev/
chroot /mnt/local/
```

### Step 4
OK we are in! Lets get dns working and try apt:

```

vi /etc/resolv.conf
apt-get update

…
Failed to fetch http://security.ubuntu.com/ubuntu/dists/dapper-security/main/binary-i386/Packages.gz 404 Not Found [IP: 91.189.91.13 80]
…
Failed to fetch http://archive.ubuntu.com/ubuntu/dists/dapper/main/binary-i386/Packages.gz 404 Not Found [IP: 91.189.91.15 80]
…
```

Oh no! Looks like this release is too old, we need to find modify our apt sources to find alternative URLs

```command-line
cat /etc/*release

DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=6.06
DISTRIB_CODENAME=dapper
DISTRIB_DESCRIPTION=”Ubuntu 6.06.2 LTS”
```

OK lets replace all references to http://archives.ubuntu.com with http://old-releases.ubuntu.com and try again:

```command-line
vi /etc/apt/sources.list
apt-get update

Get:1 http://old-releases.ubuntu.com dapper Release.gpg [189B]
Get:2 http://old-releases.ubuntu.com dapper-updates Release.gpg [198B]
Get:3 http://old-releases.ubuntu.com dapper-backports Release.gpg [198B]
Get:4 http://old-releases.ubuntu.com dapper-security Release.gpg [198B]
Hit http://old-releases.ubuntu.com dapper Release
Hit http://old-releases.ubuntu.com dapper-updates Release
…
Reading package lists… Done
```

### Step 5
Success! Taking a look at the /boot/ reveals that this VM was para-virtualized and had no kernel or bootloader. Lets get that installed.

```
apt-get install lilo
apt-cache search linux
apt-get install linux-image-2.6.15-28-server

# Lets activate the partition
lilo -A /dev/sda 1
```

### Step 6
Let’s do some last minute cleanup before the grand reboot. Check `/etc/inittab` file to make sure `tty1` is set properly then check `/etc/fstab` for proper mounts (modify or comment out some lines) and reboot. Oh, you may also want disable some services from running on boot if you are missing partitions (like we were)… you maybe want to also make a backup of the shadow file and insert your own root password so that you can login after it boots. Now we can finally reboot. <br/><br />
Works! Ok lets move on to the second VM which seems to be a much more modern version of Linux.

## Migrating a XEN Ubuntu 12.04.4 LTS VM over to VMware Manually
Now i know there are better ways to migrate this more modern version, but we will use the same approach but with more a recent filesystem, bootloader and kernel. We can also bump up the disk and network driver:

>**Guest OS:** Ubuntu 64-Bit<br />
>**Disk Controller:** VMware Paravirtual<br />
>**Network Adapter:** VMXNET 3<br />

### Step 0
Lets boot our VM to Finnix again, this time selecting the 64-bit version and again enabling ssh.

### Step 1-4
Let’s zoom by these steps since they are similar to the ones above with some differences in filesystem:

```
parted -s /dev/sda mklabel msdos
parted -a optimal /dev/sda mkpart primary 0% 100%
mkfs.ext4 /dev/sda1
mkdir /mnt/{local,nfs,loop}
/etc/init.d/rpcbind start
mount nas:/zpool1/misc /mnt/nfs/
mount -o loop /mnt/nfs/vm_xen_ubu12_root.img /mnt/loop/
mount /dev/sda1 /mnt/local/
cp -rp /mnt/loop/* /mnt/local/
mount -t proc proc /mnt/local/proc/
mount -t sysfs sys /mnt/local/sys/
mount -o bind /dev /mnt/local/dev/
chroot /mnt/local/
vi /etc/resolv.conf
apt-get update

…
Reading package lists… Done
```

### Step 5
Excellent! apt sources are still good. We also have many more kernels to choose from. Lets try using one of the virtualization optimized kernels
```
apt-cache search linux | grep virtual
apt-get install linux-image-extra-virtual
```
Installing the kernel also triggered an install of grub which took care of the bootloader for us. Now all that’s left is to re-enable the `tty1` as it has been replaced with xen console hvc0. this setting is no longer in the `/etc/inittab` (since upstart) but can be found here:
```
cat /etc/init/hvc0.conf
# hvc0 – getty
#
# This service maintains a getty on hvc0 from the point the system is
# started until it is shut down again.
start on stopped rc RUNLEVEL=[2345] and (
not-container or
container CONTAINER=lxc or
container CONTAINER=lxc-libvirt)
stop on runlevel [!2345]
respawn
exec /sbin/getty -8 38400 hvc0

mv /etc/init/hvc0.conf /etc/init/tty1.conf
sed -i -e ‘s/hvc0/tty1/g’ /etc/init/tty1.conf
```

### Step 6
Like last time, remember to do some last minute cleanup before the reboot. This time we won’t have to cross our fingers too tight.
Reboot successful!
