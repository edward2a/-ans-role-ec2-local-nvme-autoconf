# ans-role-ec2-local-nvme-autoconf
Auto configuration of local NVMe drives in EC2 instances

## Usage
Add this role in your playbook to automatically setup and mount NVMe instance store on boot.<br />
You can configure your application's init script to depend on this via SystemD.

```
---
- hosts: all
  remote_user: ec2-user
  become: true
  roles:
  - local-nvme-autoconf
  vars:
    volume:
      name: dbdata-vol
      mountpoint: /var/dbdata
      post_mount_mkdir: 'binlog,data'
      fs: xfs
      fs_owner: nosql # 
```

|Key|Info|
|:--|:---|
|volume.name|Name for the file system label and device name if RAID (/dev/md/\<volume.name\>). Bear in mind the file system label limit (xfs: 12 chars)|
|volume.mountpoint|Path to mount the volume (will be created if non-existent)|
|volume.post_mount_mkdir|Directories to create in the volume after mounted|
|volume.fs|File system to format the volume|
|volume.fs_owner|File system owner user (chown \<user\>) for the volume and the created directories|

## What will this do?
The installed components on this role will:
- Execute at boot
- Inspect the system for local NVMe disks
- If more than 1 disk, create a RAID0 array with mdadm (conditional)
- Format the disk or array with the desired filesystem (conditional)
- Mount the volume at the desired mount point (always)
- Create directories inside the mount poitn (conditional)

## Why this way?
Rationale behind the design of this script is the dynamic nature of instance store disks.<br />
The target is to simplify the usage as much as possible, so in all situations a single volume is created, formatted, mounted and given access to a specific user.<br />
As this runs as a service in SystemD, any application installed that depends on the local storage can establish a dependency (see [systemd.unit/Requires](https://www.freedesktop.org/software/systemd/man/systemd.unit.html#Requires=))<br />

Disk considerations:
- New instance: local disks are empty
- Stop/Start: local disks are empty
- Reboot: local disks are same as before restart
- Instance type change: disks are empty and quantity may vary (1-8 disks on i3 instances)

Config considerations:
- Using fstab can render the system unbootable (specially with SystemD), as on stop/start the disks change and the volume to mount is no longer available until re-created
- Using noauto option on fstab could accommodate the mounting part, but I prefer to leave fstab alone
- Need to accommodate to disks being empty at boot and topology changes (see Instance type change in disk considerations)
- Need to consider the possibility of directories needing to be created after the volume is mounted
- Need to set access on the volume and the directories inside it

