---
layout: post
title: Create volume group for OpenStack
---

{{ page.title }}
================
 
<p class="meta">9 Dec 2012 - BeiJing Ring Building</p>

### I'm going to partition /dev/sdb3 to create volume group, steps are：

1. use /dev/sdb3 to create physical volume
2. use above physical volume to create volume group

But the existing system is not the type of linux LVM, an Id is needed to be changed into 8e through `fdisk`

    /dev/sdb3           13105       36947   191509504    7  HPFS/NTFS
    [root@pangpangs ~]# fdisk  /dev/sdb3
    p(查看当前分区)
    t(修改系统ID)
    3(第三块)
    8e(修改类型)
    w(保存)</pre></code>

Show the disk list:

    [root@pangpangs ~]# fdisk -l
    .....

    Device Boot      Start         End      Blocks   Id  System
    /dev/sdb1               1       13105   105263104    7  HPFS/NTFS
    /dev/sdb2           13105       36947   191509504    7  HPFS/NTFS
    /dev/sdb3           36947       60802   191611904   8e  Linux LVM
    ....

If encounterring error: *Failed to create pv*, might be have been mounted before.

    [root@pangpangs ~]# pvcreate /dev/sdb3
    Can't open /dev/sdb3 exclusively.  Mounted filesystem?

The /dev/sdb3 has been mounted
    
    [root@pangpangs ~]# df -h
    文件系统              容量  已用  可用 已用%% 挂载点
    ....
    /dev/sdb3             180G  188M  171G   1% /mnt/disk3

### unmounted /dev/sdb3
   
    [root@pangpangs ~]# umount -v /dev/sdb3
    /dev/sdb3 umounted

### create pv successfully:

    [root@pangpangs ~]# pvcreate /dev/sdb3
        Physical volume "/dev/sdb3" successfully created
    [root@pangpangs ~]# pvscan
        PV /dev/sda2   VG vg_pangpangs    lvm2 [931.02 GiB / 0    free]
        PV /dev/sdb3                      lvm2 [182.74 GiB]
        Total: 2 [1.09 TiB] / in use: 1 [931.02 GiB] / in no VG: 1 [182.74 GiB]

### create vg successfully:

    [root@pangpangs ~]# vgcreate cinder-volumes /dev/sdb3
        Volume group "cinder-volumes" successfully created

    [root@pangpangs ~]# vgdisplay
    --- Volume group ---
        VG Name               cinder-volumes
        System ID
        Format                lvm2
        Metadata Areas        1
        Metadata Sequence No  1
        VG Access             read/write
        VG Status             resizable
        MAX LV                0
        Cur LV                0
        Open LV               0
        Max PV                0
        Cur PV                1
        Act PV                1
        VG Size               182.73 GiB
        PE Size               4.00 MiB
        Total PE              46780
        Alloc PE / Size       0 / 0
        Free  PE / Size       46780 / 182.73 GiB
        VG UUID               qoV4Yf-3iZZ-DorA-SAus-1b0E-OCf2-KHK6Nt

