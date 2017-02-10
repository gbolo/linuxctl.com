---
categories:
  - "administration"
  - "storage"
keywords:
  - "netapp"
tags:
  - "netapp"
  - "cli"
  - "reference"
date: "2014-01-14"
title: "Netappp 7-mode - CLI Reference"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-netapp.jpg"

---
The great thing about Netapps are that they were designed with CLI in mind first. The web GUI came later. Because of that, the CLI is a joy to work with. This post is for reference purposes
<!--more-->

### The Basics
```
setup (Re-Run initial setup)
halt (Reboots controller into bootrom)
reboot (Reboots the connected controller)
sysconfig -a (Dumps the system configuration)
storage show disk (shows physical information about disks)
passwd (Changes the password for the current user)
sasadmin shelf (shows a graphical layout of your shelves with occupied disk slots)
options trusted.hosts x.x.x.x or x.x.x.x/nn (hosts that are allowed telnet, http, https and ssh admin access. x.x.x.x = ip address, /nn is network bits)
options trusted.hosts * (Allows all hosts to the above command)
```
### Check NFS clients and Traffic
```
## Enable nfs.per_client_stats.enable
options nfs.per_client_stats.enable on

## List NFS Clients connected/working
nfsstat -l

## Reset list
nfsstat -z
```
### Modifying NFS Exports
```
## Read exports:
rdfile /etc/exports

## Write new export file:
wrfile /etc/exports
** press ctrl+c when done **

## Apply exports
exportfs -a
```
### Networking
```
## rc file (startup config):
ifgrp create lacp lacptrunk1 -b ip e3a e3b  // create lacp interface using e3a and e3b
vlan create lacptrunk1 300 301 302 310 315  // trunk these vlans on this lacp interface
ifconfig lacptrunk1-315 10.131.15.12 netmask 255.255.255.0 partner lacptrunk1-315 // assign tagged IP to interface

## realtime live changes
vlan add lacptrunk1 315  // add new vlan 315 to lacp interface
ifconfig lacptrunk1-315 10.131.15.12 netmask 255.255.255.0 partner lacptrunk1-315 // assign tagged IP to interface
```
### Diagnostics
```
Press DEL at boot up during memory test followed by boot_diags and select all
priv set diags (Enter diagnostics CLI mode from the Ontap CLI)
priv set (Return to normal CLI mode from diagnostics mode)
```

### Software
```
software list (Lists software in the /etc/software directory)
software delete (Deletes software in the /etc/software directory)
software update 8.1RC2_e_image.zip -r (Install software. The -r prevents it rebooting afterwards)
```

### Aggregates
```
aggr create aggregate_name (Creates an Aggregate)
aggr destroy aggregate_name (deletes an Aggregate)
aggr offline aggregate_name (takes an Aggregate offline)
aggr online aggregate_name (brings an Aggregate online)
aggr status (shows status of all aggregates)
aggr status aggregate_name (show status of a specific Aggregate)
aggr show_space aggregate_name (shows specific aggregate space information)
```

### Volumes
```
vol create volume_name (Creates a volume)
vol status (gives the status of all volumes)
```

### Snapshots
```
snap create volume_name snapshot_name (create a snapshot)
snap list volume_name (List snapshots for a volume)
snap delete volume_name snapshot_name (delete a snapshot on a volume)
snap delete -a volume_name (Deletes all snapshots for a volume)
snap restore -s snapshot_name volume_name (Restores a snapshot on the specified volume name)
options cifs.show_snapshot on (Sets snapshot directory to be browse-able via CIFS)
options nfs.hide_snapshot off (Sets snapshot directory to be visible via NFS)
```

### SnapMirror
```
options snapmirror.enable on (turns on SnapMirror. Replace on with off to toggle)
vol restrict volume_name (Performed on the Destination. Makes the destination volume read only which must be done for volume based replication)
snapmirror initialize -S srcfiler:source_volume dstfiler:destination_volume (Performed on the destination. This is for full volume mirror. For example snapmirror initialize -S filer1:vol1 filer2:vol2)
snapmirror status (Shows the status of snapmirror and replicated volumes or qtree’s)
snapmirror status -l (Shows much more detail that the command above, i.e. snapshot name, bytes transferred, progress, etc)
snapmirror quiesce volume_name (Performed on Destination. Pauses the SnapMirror Replication. If you are removing the snapmirror relationship this is the first step.)
snapmirror break volume_name (Performed on Destination. Breaks or disengages the SnapMirror Replication. If you are removing the snapmirror relationship this is the second step followed by deleting the snapshot)
snapmirror resync volume_name (Performed on Destination. When data is out of date, for example working off DR site and wanting to resync back to primary, only performed  when SnapMirror relationship is broken)
snapmirror update -S srcfiler:volume_name dstfiler:volume_name (Performed on Destination. Forces a new snapshot on the source and performs a replication, only if an initial replication baseline has been already done)
snapmirror release volume_name dstfiler:volume_name (Performed on Destination. Removes a snapmirror destination)
```

### Cluster (HA Partner)
```
cf enable (enable cluster)
cf disable (disable cluster)
cf takeover (take over resources from other controller)
cf giveback (give back controller resources after a take over)
```

### Disks
```
disk show (Show disk information)
disk show -n (Show unowned disks)

## Show spares
vol status -r
```

### LUNS
```
lun setup (runs the cli lun setup wizard)
lun create -s 10g -t windows_2008 -o noreserve /vol/vol1/lun1 (creates a lun of 10GB with type Windows 2008, sets no reservation and places it in the following volume or qtree)
lun offline lun_path (takes a lun offline)
lun online lun_path (brings a lun online)
lun show -v (Verbose listing of luns)
```

### Fiber FCP
```
fcp config 0b down (Brings down fcp adapter 0b)
fcadmin config -t target 0a (Changes adapter from initiator to target)
fcadmin config (lists adapter state)
fcp start (Start the FCP service)
fcp stop (Stop the FCP service)
fcp show adapters (Displays adapter type, status, FC Nodename, FC Portname and slot number)
fcp nodename (Displays fiber channel nodename)
fcp show initiators (Show fiber channel initiators)
fcp topology show 1a (Show topology for adapter 1a)
fcp portname show -v (shows list of all avilable used/unused WWPNs)
fcp wwpn-alias set alias_name (Set a fiber channel alias name for the controller)
fcp wwpn-alias remove -a alias_name (Remove a fiber channel alias name for the controller)
igroup show (Displays initiator groups with WWN’s)
```

### CIFS
```
cifs setup (cifs setup wizard)
cifs restart (restarts cifs)
cifs shares (displays cifs shares)
cifs status (show status of cifs)
cifs domain info (Lists information about the filers connected Windows Domain)
cifs testdc ip_address (Test a specific Windows Domain Controller for connectivity)
cifs prefdc (Displays configured preferred Windows Domain Controllers)
cifs prefdc add domain address_list (Adds a preferred dc for a specific domain i.e. cifs prefdc add netapplab.local 10.10.10.1)
cifs prefdc delete domain (Delete a preferred Windows Domain Controller)
vscan on (Turns virus scanning on)
vscan off (Turns virus scanning off)
vscan reset (Resets virus scanning)
```
### SIS / Deduplication
```
sis status (Shows SIS status)
sis config (Shows SIS config)
sis on /vol/vol1 (Turns on deduplication on vol1)
sis start -s /vol/vol1 (Runs deduplication manually on vol1)
sis status -l /vol/vol1 (Displays deduplication status on vol1)
df -s vol1 (View space savings with deduplication)
sis stop /vol/vol1 (Stops deduplication on vol1)
sis off /vol/vol1 (Disables deduplication on vol1)
aggr show_space aggr0 -g
```

### DNS
```
dns flush (Flushes the DNS cache)
/etc/resolv.conf (edit this file to change your dns servers)
```
