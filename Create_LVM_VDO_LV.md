LVM (Logical Volume Manager)
============================
LVM is a flexible storage management system that allows you to create, resize, and manage logical storage volumes on top of physical storage devices (like disks or partitions).
It abstracts the physical disks and gives you logical control.

Basic Components:

PV (Physical Volume) – a physical disk or partition (can also be a loop device).

VG (Volume Group) – a pool of storage created by combining one or more PVs.

LV (Logical Volume) – a usable storage volume created from the VG (you format and mount this).



VDO (Virtual Data Optimizer)
============================
VDO adds data reduction capabilities — like:

    + Deduplication – removes duplicate data blocks.

    + Compression – reduces the size of stored data.

    + Thin provisioning – allocates space only when data is written.

In short, VDO helps save disk space by storing data more efficiently.


Note: 

Loop Devices:
-------------
VDO volumes (LV) can't be created directly on the loop devices; however, loop devices can be part of a volume group (VG).


Practical process:
==================
Currently I don't have an extra physical disk to practice LVM and VDO on, I am simulating a disk using empty file.

1. Create an empty file for disk simulation
    
    ![Disk Size before File creation](./imgs/LVM_VDO_1.png)
    
````bash
    sudo fallocate -l10G /root/disk1
````

    ![Disk Size after File creation](./imgs/LVM_VDO_2.png)

    note:
        I am creating a 10 GB sparse file. This file simulates a real disk, which is great for labs. It contains no data yet; space is just reserved logically on the filesystem.
    
        In a production setup, this step would correspond to using a real disk (like /dev/sdb).


2. Attach it as a loop device

    sudo losetup -f /root/disk1 --show

    ![First Free Loop Device](./imgs/LVM_VDO_3.png)

    note:
        -f = Finds the first free loop device
        --show = prints the first free loop device

        From now on, /dev/loopxy behaves just like a physical disk to the Linux kernel

        For checking
            lsblk
            losetup -a


3. Create a Volume Group (VG)

    vgcreate myvg /dev/loopxy

    note:
        vgcreate first creates a Physical Volume (PV) on /dev/loopxy. Then it groups that PV into a VG called myvg

    ![Volume Group and Loop device act as physical volume](./imgs/LVM_VDO_4.png)


4. Create LVM-based VDO volume

    sudo lvcreate --type vdo --name myvdolv --extents 100%FREE --virtualsize 50G myvg

    note:
        Under the hood, LVM creates two related volumes:

        1. A VDO data LV (stores actual data).

        2. A VDO metadata LV (stores deduplication/compression metadata).

        Because VDO supports **thin provisioning**, you can make the logical size (50G) larger than the physical space (10 GB loop device) — the VDO engine compresses and deduplicates data to fit it efficiently.

    ![LVM-based VDO volume](./imgs/LVM_VDO_5.png)    


5. Format the new logical volume with the XFS filesystem (recommended for VDO)

    sudo mkfs.xfs -K /dev/myvg/myvdolv

    ![XFS filesystem](./imgs/LVM_VDO_6.png) 

    ![List of block devices](./imgs/LVM_VDO_7.png) 


Monitor LV VDO
==============

lvs -a -o +devices

By Default, both compression and data-deduplication is enabled

    lvs -o+vdo_compression,vdo_deduplication

You can enable/disable it at pool level

    lvchange --compression y|n myvg/myvdolv

    lvchange --deduplication y|n myvg/myvdolv

![Example](./imgs/LVM_VDO_8.png) 

To view the space

    vdostats --human-readable


Example:

Mount VDO LV to temporary area
    mount /dev/myvg/myvdolv /mnt

    note:
        Before:
            df -h /mnt
            vdostats --human-readable

![Compare the filesystem used size and VDO LV used size, before for loop](./imgs/LVM_VDO_9.png) 

        for i in {1..9} ; do cp /boot/initramfs-0-rescue-415fa3778d2f45a2818864c6eeafd591.img /mnt/f$i; done


        After:
            df -h /mnt
            vdostats --human-readable

![Compare the filesystem used size and VDO LV used size, After for loop](./imgs/LVM_VDO_10.png) 


Cleanup
=======
    umount /mnt
    lvremove /dev/myvg/myvdolv -y
    vgremove myvg -y
    losetup -d /dev/loop0
    rm -f /root/disk1
