---
categories:
  - "administration"
  - "containers"
keywords:
  - "docker"
tags:
  - "docker"
  - "storage"
  - "devicemapper"
  - "lvm"
date: "2016-04-18"
title: "Docker Storage Driver - Devicemapper"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-docker.jpg"

---

Docker relies on storage engines to layer images. The default storage driver depends on who packaged docker for your OS. Fortunately, it's not too difficult to change; However you may lose your images and containers so it's best to decide on a driver when you begin.
<!--more-->

Currently (docker 1.11) has the following engines available:


<table style="height: 202px;" width="285">
<tbody>
<tr>
<td><strong>Technology</strong></td>
<td><strong>Driver Name</strong></td>
</tr>
<tr>
<td>OverlayFS</td>
<td>overlay</td>
</tr>
<tr>
<td>AUFS</td>
<td>aufs</td>
</tr>
<tr>
<td>Btrfs</td>
<td>btrfs</td>
</tr>
<tr>
<td>Device Mapper</td>
<td>devicemapper</td>
</tr>
<tr>
<td>VFS*</td>
<td>vfs</td>
</tr>
<tr>
<td>ZFS</td>
<td>zfs</td>
</tr>
</tbody>
</table>

```command-line
docker info | grep -i "storage driver"
Storage Driver: devicemapper
```

Choosing the right storage engine requires some reading. Personally, I went with `devicemapper direct-lvm` since I have experience working with lvm and it seems to be well supported by docker. Here is how I got it configured on a fresh docker host running centOS 7:

```language-bash
# stop docker
root@c7-1 ~ $ systemctl stop docker

# create your vg,lv
root@c7-1 ~ $ lvcreate --wipesignatures y -n thinpool vg-docker -l 95%VG
Logical volume "thinpool" created.

root@c7-1 ~ $ lvcreate --wipesignatures y -n thinpoolmeta vg-docker -l 1%VG
Logical volume "thinpoolmeta" created.

root@c7-1 ~ $ lvconvert -y --zero n -c 512K --thinpool vg-docker/thinpool --poolmetadata vg-docker/thinpoolmeta
WARNING: Converting logical volume vg-docker/thinpool and vg-docker/thinpoolmeta to pool's data and metadata volumes.
THIS WILL DESTROY CONTENT OF LOGICAL VOLUME (filesystem etc.)
Converted vg-docker/thinpool to thin pool.

# create this lvm profile
root@c7-1 ~ $ cat /etc/lvm/profile/docker-thinpool.profile
activation {
     thin_pool_autoextend_threshold=80
     thin_pool_autoextend_percent=20
}

# apply the above profile, and verify that your lv is being monitored
root@c7-1 ~ $ lvchange --metadataprofile docker-thinpool vg-docker/thinpool
Logical volume "thinpool" changed.

root@c7-1 ~ $ lvs -o+seg_monitor
LV VG Attr LSize Pool Origin Data% Meta% Move Log Cpy%Sync Convert Monitor
thinpool vg-docker twi-a-t--- 28.50g 0.00 0.02 monitored

# configure docker deamon
root@c7-1 ~ $ cat /etc/docker/daemon.json
{
         "storage-driver": "devicemapper",
         "storage-opts": [
                 "dm.thinpooldev=/dev/mapper/vg--docker-thinpool",
                 "dm.use_deferred_removal=true"
         ]
}

# This will delete all your images and containers!
root@c7-1 ~ $ rm -rf /var/lib/docker/*

# start docker
root@c7-1 ~ $ systemctl start docker

# verify
...
Server Version: 1.11.0
Storage Driver: devicemapper
Pool Name: vg--docker-thinpool
Pool Blocksize: 524.3 kB
Base Device Size: 10.74 GB
Backing Filesystem: xfs
Data file:
Metadata file:
Data Space Used: 320.3 MB
Data Space Total: 30.6 GB
Data Space Available: 30.28 GB
Metadata Space Used: 180.2 kB
Metadata Space Total: 318.8 MB
Metadata Space Available: 318.6 MB
Udev Sync Supported: true
Deferred Removal Enabled: true
Deferred Deletion Enabled: false
Deferred Deleted Device Count: 0
Library Version: 1.02.107-RHEL7 (2015-12-01)
...

```
