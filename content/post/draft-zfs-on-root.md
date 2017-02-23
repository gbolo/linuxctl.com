---
categories:
  - "administration"
  - "storage"
keywords:
  - "zfs"
tags:
  - "zfs"
  - "ubuntu"
  - "crypto"
  - "luks"
  - "reference"
date: "2016-12-20"
title: "ZFS on Root - Laptop"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-zfs.png"

draft: true

---

After much deliberation, I decided to pick up a new [Thinkpad P50](http://www3.lenovo.com/ca/en/laptops/thinkpad/thinkpad-p/P50/p/22TP2WPWP50) Laptop. This beast has a ton of features that are rare in today's laptop landscape:

* Excellent Linux support (with newest kernel)
* User serviceable (it's made to be **easily opened up** to access components)
* Easily upgradable components:
  * 2 nvme slots
  * 1 sata 2.5-inch slot
  * 4 ddr4 slots (**64gb total capacity!**)
* 9 hour battery life, replacable battery
* 1080p screen with 4k optional
* Hybrid graphics (saves battery life)
* Very strong and durable, 8 thread core i7, nice keyboard

To save money, I ordered it without upgrades and bought my upgrades separately at my local computer shop. The upgrades included:

* 2 x 512GB intel 600p nvme
* 512GB Samsung 850PRO
* 32GB (2x16GB) DDR4 so-dimms

With a 9-hour laptop that has 8 threads and 32gb ram (with 2 slots free for future 64gb ram upgrade), I decided that it would be a great idea to use zfs for my storage needs since I have a total of 1.5tb of SSD space. My idea was to make two zpools, the first being a mirror for my root partions, and the second being a stripe for my less important data. I plan and having many docker containers and VMs running, especially for devop testing and a stripe would maximise the performance for this. However, I also wanted to encrypt the drives since a laptop this powerful should be able to handle this easily (and its 2017, everything should be encrypted!). for the mirror zpool, I would create a 200GB partition on both of my nvme drives. for the stripe zpool I would use the remaining space to create another partition.


I decided to switch from Arch to Ubuntu for this laptop, since it's been a while that I used Ubuntu. Fortunately, the zfsonlinux folks provide a [good guide](https://github.com/zfsonlinux/zfs/wiki/Ubuntu-16.10-Root-on-ZFS) to install zfs on root. However there are some things missing, since my setup is a bit different with multiple drives and requires encryption.

```
https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=820888
```
