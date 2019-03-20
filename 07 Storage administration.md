## Overview

This section provides some reference and describes some tricks to manage the storage devices in a Rocks cluster. These include the hard drives in each compute node, the hard drives in the head node (frontend), and the hard drives in the storage node (NAS appliance).

## Hard drive selection

If you are building a NAS for heavy I/O work (since each compute node will need to access this NAS, and all user home directory is stored here), make sure to use enterprise grade drives, such as Seagate Constellation drives, and make sure to use SAS drives rather than SATA drives (the price difference is very minor today). If you are building a NAS for occasional data back up and storage, then a drive designed for NAS can be used, such as WD Red drives. Just do not buy any drive for desktop applications, such as Seagate Burracuda.

For compute node, it does not really matter that much, so a cheap (but large-capacity) drive is probably preferred when budget is a concern; otherwise, use enterprise drive. The approximate capacity really depends on your application: for genomics application that often requires processing 100G of raw sequence data and generate even 500G temporary files, it makes the most sense to use $TMP in a compute node to store these temporary files (rather than using the NAS appliance itself), so a large drive (such as 3T) in compute node makes more sense. 

For head node, make sure to use enterprise drives, and use a RAID 1. Size does not matter much, yet it is more important to have redundancy.

## NAS storage administration

The NAS is a basically a compute box with many hard drives. Typical set up are 36 drives within a 4U enclosure, or 18 drives within a 2U enclosure. These drives are connected to a SAS RAID controller and a SAS expander (typically when you buy the computer box, these components are already there so you do not need to do anything). LSI is the most popular RAID controller, but some manufacturers such as Dell has their own controllers. My preference is to use LSI; even when I have access to a Dell machine with H700 controllers, I still unplug it and install a new LSI card. The reason is that manufacturers such as Dell/IBM/HP make things difficult for users to customize so that users can only buy from them with high premium; for example, the SAS cable for H700 is NOT compatible with a real SAS cable that everybody else uses, even though they have identical appearance and have identical name and identical interface. It drived me crazy once, and I cannot imagine how other users can possibly figure this out.

The instructions below refer to LSI controllers. A general overview of the procedure is described in the figure below.

![biocluster-storage](https://cloud.githubusercontent.com/assets/5926328/14517371/70889e5e-01c1-11e6-8ee9-50866258d0da.png)

## Creating virtual drives (VD)

To illustrate this by a simple example, suppose I get a 36-drive 4U storage server, with an external SAS cable connecting to a 45-drive 4U JBOD. So in total I have 81 drives. Due to the limitations of RAID6 (you cannot have too many drives in a RAID 6), the 81 drives needs to be split into 3 drive groups, so we can do a 27+26+26+2. We will use 2 drives as global hot spares, so that whenever a drive fails, the RAID will automatically rebuild using one of the hot spares.

Depending on BIOS settings in your motherboard, you can either perform the DG/VD creation directly within motherboard BIOS (only for very new motherboards), or perfrom the creation after LSI's own BIOS screen. Essentially you create 3 drive groups and specify the number of drives within each group. Remember that each drive group must use RAID6. Do not use RAID5!!!

(It requires some explanation why RAID5 cannot be used: the main reason is that drives tend to have silent data corruption, so if a drive in RAID5 fails and if you rebuild the array, the rebuilding process may not be successful. RAID6 is more tolerant to this situation. Typically, the RAID card will do surveillance of all drives once in a while to detect these types of drive errors)

One slight complication is that we want to install Rocks in a separate and smalll virtual drives, so that we can keep re-install the operating system in the future without damaging any data stored in the NAS array. To accomplish this, within the first drive group, we will create a logical drive of 100GB, and another logical drive that takes the rest of the space. So in the end, we have `/dev/sda`, `/dev/sdb`, `/dev/sdc`, `/dev/sdd`. The first VD is 100G, yet the other three VDs have large storage capacity.

## MegaRaid Hardware information

System status information can be collected using lsiget command, which is available at  http://mycusthelp.info/LSI/_cs/AnswerDetail.aspx?sSessionID=&aid=8264.

## MegaCli

* MegaCli installation
    Download the program from http://www.lsi.com/support/Pages/download-results.aspx?keyword=megacli. By default, MegaCli is installed at /opt/MegaRAID/MegaCli/MegaCli64

* MegaCli command line reference

    Use the following command to print complete help message: `/opt/MegaRAID/MegaCli/MegaCli64 -h`

    The following link has more detailed description: http://mycusthelp.info/LSI/_cs/AnswerPreview.aspx?sSessionID=&inc=8040

* MegaCli cheat sheet

    http://erikimh.com/megacli-cheatsheet/
    http://genuxation.com/wiki/index.php/MegaCLI
    https://supportforums.cisco.com/docs/DOC-16309
    lsi.sh script https://calomel.org/megacli_lsi_commands.html 

## Essential commands to use in a new system
```
[root@nas-0-1 ~]$ /opt/MegaRAID/MegaCli/MegaCli64  -AdpAllInfo -aALL | fgrep Failed
Security Key Failed              : No
  Failed Disks    : 2 
Deny Force Failed                       : No
```

This immediately tells us that two LD has failed!!!

To solve this, I cleared foreign configuration, and then manually stopped an ongoing ‘Rebuild’, then assigned the two ‘bad drives’ to ‘global hot spare’, then the rebuild starts automatically. I stopped an ongoing ‘Rebuild’ because it is rebuilt from a different enclosure.


show system informatoin:
```
MegaCli -AdpAllInfo -aALL
MegaCli -CfgDsply -aALL
MegaCli -adpeventlog -getevents -f lsi-events.log -a0 -nolog
```

Show adapter information:
```
/opt/MegaRAID/MegaCli/MegaCli64 -AdpAllInfo -a0
```

show enclosure information:
```
/opt/MegaRAID/MegaCli/MegaCli64 -EncInfo -aALL
```

show virtual dive information:
```
/opt/MegaRAID/MegaCli/MegaCli64 -LDInfo -Lall -aALL
```

show physical drive information
```
/opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL
```

Identify discs that may fail in the future
```
/opt/MegaRAID/MegaCli/MegaCli64 -PDList -aALL | fgrep S.M.A.R.T
```

show BBU infomration:
```
/opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -aALL
```

Manual relearn battery
```
/opt/MegaRAID/MegaCli/MegaCli64 -AdpBbuCmd -BbuLearn -a0 
```

use WB(write-back) even if BBU is bad
```
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp CachedBadBBU -Lall -aAll
```

Reverse above procedure
```
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp NoCachedBadBBU -Lall -aAll
```

Set virtual drive to writeback
```
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp WB -LALL -aALL
```

Set virtual drive to readahead (ReadAdaptive)
```
/opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp RA -LALL -aALL
```

Identify all hot spares
```
/opt/MegaRAID/MegaCli/MegaCli64 -PDlist -a0 | fgrep Hotspare -B 10
```

set global hot spare
```
/opt/MegaRAID/MegaCli/MegaCli64 -PDHSP -Set -PhysDrv [E:S] -aN
```

Perform diagnosis on adapter
```
/opt/MegaRAID/MegaCli/MegaCli64 -AdpDiag -a0
```

Start, stop, suspend, resume, and show the progress of a consistency check operation
```
/opt/MegaRAID/MegaCli/MegaCli64 -LDCC -Start|-Abort|-Suspend|-Resume|-ShowProg|-ProgDsply -Lx|-L0,1,2|-LALL -aN|-a0,1,2|-aALL
```

## MegaRaid Storage Manager

Install: 
Download the package, then “./install.csh -a” for complete installation
However, every time I install it, I got the following message. It works for a while but after a while, the nas-0-0 can no longer be accessed from biocluster.
```
Can not find snmptrap in /usr/bin
Can not continue installation of LSI SAS-IR Agent
Please install the net-snmp agent
warning: %post(sas_ir_snmp-13.08-0401.x86_64) scriptlet failed, exit status 1
```

I solved the problem by installing net-snmp-utils (first search for packages that provides snmptrap, then install it):
```
[root@nas-0-0 disk]# yum provides */snmptrap  
[root@nas-0-0 disk]# yum install net-snmp-utils
```

Uninstall: 
```
/usr/local/MegaRAID\ Storage\ Manager/uninstaller.sh
```
or
```
/usr/local/MegaRAID\ Storage\ Manager/__uninstaller.sh 
```
if the former does not work.

In one server, also found that when I install latest version (v15) of MSM, I cannot connect to remote servers ("megaraid storage manager servers could not be found because server may be down"), and other version such as v13 does not work either. In that case, I have to roll back to version 12 (10/8/2012), and then can use MSM correctly. This may be due to the firmware version of the LSI card.

Sometimes the framework stops running so one cannot remotely control/monitor the status in a server. In this case, try restart MSM Framework service by `/etc/init.d/vivaldiframeworkd restart`.


## Creating a new XFS volume

Once you set up RAID and created several hardware-level logical drives, next we should use LVM to create software-level physical and logical volumes. These are VERY DIFFERENT concepts!!! The drive group, logical drive and physical drive in RAID card setting are all hardware level concepts, and once they are completed, what's presented to the user are merely things like `/dev/sda`, `/dev/sdb`, `/dev/sdc` and so on. Next, we need to make software-level LVM to manage these drives and combine them together.

Typically, you want to install Rocks in `/dev/sda`, so this partition is pretty small, in fact 100Gb is more than enough (note that Rocks will partition this to `/dev/sda1`, `/dev/sda2` and so on to install operating system on this logical drive). The `/dev/sdb` and `/dev/sdc` and so on could be dozens or hundreds of terabytes, and they are used as `/export` in our NAS.

To create a new volume, follow the procedure:

1. do `parted /dev/sdb`, use `mklabel gpt` to make a GPT partition table (default is MSDOS which does not handle >4TB partition). use `print` to view it, then use `mkpart primary 0% 100%` to automatically create a partition that is optimal (if I use `mkpart primary 0 100%` it will create mis-aligned partitions). An example is given below:

```
(parted) print                                                            
Model: AVAGO MR9361-8i (scsi)
Disk /dev/sdb: 99.9TB
Sector size (logical/physical): 512B/512B
Partition Table: msdos

Number  Start  End  Size  Type  File system  Flags

(parted) mklabel gpt                                                      
Warning: The existing disk label on /dev/sdb will be destroyed and all data on this disk will be lost. Do you want to continue?
Yes/No? yes                                                               
(parted) print                                                            
Model: AVAGO MR9361-8i (scsi)
Disk /dev/sdb: 99.9TB
Sector size (logical/physical): 512B/512B
Partition Table: gpt

Number  Start  End  Size  File system  Name  Flags
(parted) mkpart primary 0% 100%   
(parted) print                                                            
Model: AVAGO MR9361-8i (scsi)
Disk /dev/sdb: 99.9TB
Sector size (logical/physical): 512B/512B
Partition Table: gpt

Number  Start   End     Size    File system  Name     Flags
 1      1049kB  99.9TB  99.9TB               primary
```

2. create a physical volume which takes up all spaces in `/dev/sdb`. If pvcreate is not available, do a `yum install lvm2` first.

```
[root@nas-0-0 ~]# pvcreate /dev/sdb1 
  Writing physical volume data to disk "/dev/sdb1"
  Physical volume "/dev/sdb1" successfully created
[root@nas-0-0 ~]# pvscan 
  PV /dev/sdb1                      lvm2 [90.86 TiB]
  Total: 1 [90.86 TiB] / in use: 0 [0   ] / in no VG: 1 [90.86 TiB]
```

3. If you have multiple logical drives, do this for `/dev/sdc`, `/dev/sdd` and so on.
 
 ```
[root@nas-0-7 ~]# pvscan 
  PV /dev/sdb1                      lvm2 [90.86 TiB]
  PV /dev/sdc1                      lvm2 [87.32 TiB]
  PV /dev/sdd1                      lvm2 [87.32 TiB]
  Total: 3 [265.49 TiB] / in use: 0 [0   ] / in no VG: 3 [265.49 TiB]
```

4. Create a volume group, which may have one or more physical volumes

```
[root@nas-0-0 ~]# vgcreate nas-0-0 /dev/sdb1 /dev/sdc1 /dev/sdd1
  Volume group "nas-0-0" successfully created
[root@nas-0-0 ~]# vgscan
  Reading all physical volumes.  This may take a while...
  Found volume group "nas-0-0" using metadata type lvm2
```

4. Create a logical volume called `export` that takes up all space in the nas-0-0 volume group.

```
[root@nas-0-0 ~]# lvcreate -l 100%FREE -n export nas-0-0
  Logical volume "export" created
```

5. Create XFS in the `/export` (if `mkfs.xfs` is not available, do a `yum install xfsprogs` first):

```
[root@nas-0-0 ~]# mkfs.xfs /dev/mapper/nas--0--0-export
meta-data=/dev/mapper/nas--0--0-export isize=256    agcount=266, agsize=268435455 blks
         =                       sectsz=512   attr=2, projid32bit=0
data     =                       bsize=4096   blocks=71266857984, imaxpct=1
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0
log      =internal log           bsize=4096   blocks=521728, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```

    Note that default xfs block size is 4096 (bsize=4096 in the screen when running the above command).

6. After nas-0-0 installation is done, add `/dev/mapper/nas--0--0-export /export            xfs     defaults        0 0` to `/etc/fstab`, so that the LVM is mounted to `/export` every time nas-0-0 is started. Additionally, add `/export 10.1.1.1/255.255.0.0(fsid=0,rw,async,insecure,no_root_squash)` to /etc/exports, so that the “/export” directory can be shared to local ib network.

```
[root@nas-0-0 ~]# moutn /export
[root@nas-0-0 ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/sda1              58G  3.5G   52G   7% /
tmpfs                  64G     0   64G   0% /dev/shm
/dev/mapper/nas--0--0-export
                      266T   41M  266T   1% /export
```

    Now the directory is accessible as `/export` in the local storage server.

## Extending the XFS volume by additional JBOD

Suppose that in the future, you want to add additional JBOD to the system, yet extending the size of the `/export` directory. This can be accomplished by the following procedure: First, create new drive groups in WEBBIOS or configuration utility, and suppose the system will treat them as `/dev/sdd` and `/dev/sde`, respectively. Now go to Linux, type `parted /dev/sdd`, then `mklabel gpt`, then `mkpart primary 0% 100%`, then “quit”. Then `mkfs.xfs /dev/sdd1` (this step is most likely not needed). Do the same for `/dev/sde1`. Now `pvcreate /dev/sdd1` and `pvcreate /dev/sde1`. Use “pvscan” to confirm two physical volumes created. use “vgdisplay” to confirm FREE PE=0. Then `vgextend nas-0-1 /dev/sdd1` and `vgextend nas-0-1 /dev/sde1`, then “vgdisplay” again. Check the FREE size. Now `lvextend -l +100%FREE /dev/nas-0-1/export`, type “df -h” to check size. Then `xfs_growfs /dev/nas-0-1/export`, and type `df -h` again to check if the size has changed to the correct size. 



## Clear RAID information from hard drive

If you insert an old hard drive into a compute node, yet the hard drive may previously be part of a RAID array, Rocks will refuse to install operating system into the drive. There are no commands in Rocks that can override this decision.

If you encounter this situation, that is, during Rocks installation in a compute node, a screen shows up telling you that the drive has RAID information, you can use the following procedure to wipe out RAID information from the hard drive:

First, just press Ctrl+Alt+F5 (or maybe F6 or F4) to switch to a text terminal from GUI terminal, then do

`blockdev --getsz /dev/sda` to get the size of drive. You can also do `cat /proc/partitions` to get the size of the drive in blocks.

Now we need to wipe the first 20 blocks and last 20 blocks from the drive:

```
dd if=/dev/zero of=/dev/sda count=20
dd if=/dev/zero of=/dev/sda seek=xxxx count=20
```

The command seeks to 20 blocks before the total block size (so you'll need to do some simple math).

Then use Ctrl+Alt+F1 (or maybe F2 or F3) to switch to GUI and press enter. Then Rocks will install.

## A note on XFS versus ZFS

Recent version of Rocks begin to incorporate ZFS as an optional roll that you can install and use. ZFS is a file system originally developed on Solaris but recently adapted to Linux systems, and it has lots of theoretical advantages. XFS is a very old file system but it is very robust and can handle very large partitions well. (Around 2010-2011 timeframe, one of my co-workers was having a lot of complaints on using ZFS but I am sure that ZFS has matured to a point where it can be reliably used in Rocks now, such that Rocks developers decided to actually include a ZFS roll).

The key feature of ZFS is block-level CRC, which is supposed to detect silent data corruption. In addition, ZFS somewhat serves as if it is a software RAID as well as volume manager (rather than a simple file system), and can take care of data redundancy and rebuilding by itself. In other words, you can think of ZFS as a LSI card plus LVM2 plus file system. An excellent background description on ZFS can be found [here](http://www.datamation.com/data-center/the-zfs-story-clearing-up-the-confusion-1.html).

It is your own choice whether to use XFS or ZFS; as long as you do not use any other file systems, things will be probably fine for a small bioinformatics lab either way.

## Performance benchmarking

There are several ways to measure performance

```
[root@biocluster ~]# hdparm -tT /dev/sda
/dev/sda:
 Timing cached reads:   19416 MB in  2.00 seconds = 9717.68 MB/sec
 Timing buffered disk reads: 382 MB in  3.01 seconds = 126.78 MB/sec

 [root@nas-0-0 ~]# hdparm -tT /dev/sda 
/dev/sda:
 Timing cached reads:   21334 MB in  2.00 seconds = 10677.39 MB/sec
 Timing buffered disk reads: 4764 MB in  3.00 seconds = 1587.54 MB/sec
```

The biocluster used a typical hard drive, but nas-0-0 used a RAID6 array with much improved performance.

Similarly, we can use `dd` command to evaluate performance

```
[root@biocluster ~]# dd if=/dev/zero of=testfile bs=8k count=1000k
1024000+0 records in
1024000+0 records out
8388608000 bytes (8.4 GB) copied, 64.8641 s, 129 MB/s
[root@biocluster ~]# dd if=testfile of=/dev/null bs=8k
1024000+0 records in
1024000+0 records out
8388608000 bytes (8.4 GB) copied, 1.45163 s, 5.8 GB/s

[root@nas-0-0 ~]# dd if=/dev/zero of=testfile bs=8k count=1000k
1024000+0 records in
1024000+0 records out
8388608000 bytes (8.4 GB) copied, 5.3096 s, 1.6 GB/s
[root@nas-0-0 ~]# dd if=testfile of=/dev/null bs=8k
1024000+0 records in
1024000+0 records out
8388608000 bytes (8.4 GB) copied, 1.1867 s, 7.1 GB/s
```

The read performance is very good because the files are cached. You may want to increase the count to test the read performance in a more realistic fashion.

Finally, we can use professional software such as IOZone to perform the benchmarking. Download source code from http://www.iozone.org/ and then unpack, then build the executable as `make linux-AMD64`. The automatic testing (`-a` argument) is not so informative as the max file size used is only 512MB, so we should use different sets of commands and test processing multiple files together.

First go to NAS and test local performance of the RAID array:

```
[root@nas-0-0 BACKUP]# ~/admin/iozone3_434/src/current/iozone -R -l 5 -u 5 -r 64k -s 64g -F file1 file2 file3 file4 file5 -i 0 -i 1
"  Initial write " 2622417.06 
"        Rewrite " 2562444.69 
"           Read " 1722868.50 
"        Re-read " 1718255.59 
```

Next go to a compute node and cd to `/share/datasets` and test the NFS performance:

```
[root@compute-0-0 datasets]#  ~/admin/iozone3_434/src/current/iozone -R -l 5 -u 5 -r 64k -s 64g -F file1 file2 file3 file4 file5 -i 0 -i 1
"  Initial write " 1054813.91 
"        Rewrite " 1077901.91 
"           Read "  491567.32 
"        Re-read "  495062.38 
```

# The scariest thing: handling unexpected emergencies

The scariest thing that we may encounter is complete file system loss (as opposed to losing one or two drives in a drive group, which can be rebuilt by RAID arrays). I have encountered a lot of these types of scenarios, and below I describe general strategies to handle them.

## XFS volume repair

This usually happens when there is unclean mount/unmount, but it may also happen when you are using the system. For example, when the system shows IO error, one cannot read/write from the /export directory any more. Umounting and re-mounting shows "mount: strcuture needs cleaning" message. Restarting the computer did not help either. Now, we can use xfs_repair to fix this problem. The command replays the journal log to fix any inconsistencies that might have resulted from the file system not being cleanly unmounted. 

If the journal log itself has become corrupted, you can reset the log by specifying the -L option to xfs_repair. However, this may result in some data loss, so be careful with it.

If you just want to check the file system status, xfs_check can be used instead. In any case, you must unmount the file system first, before using these commands.

```
[root@nas-0-1 ~]# xfs_repair /dev/mapper/nas--0--1-export
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ERROR: The filesystem has valuable metadata changes in a log which needs to
be replayed.  Mount the filesystem to replay the log, and unmount it before
re-running xfs_repair.  If you are unable to mount the filesystem, then use
the -L option to destroy the log and attempt a repair.
Note that destroying the log may cause corruption -- please attempt a mount
of the filesystem before doing this.
[root@nas-0-1 ~]# mount /export/
mount: Structure needs cleaning
[root@nas-0-1 ~]# umount /export 
mount: Structure needs cleaning
[root@nas-0-1 ~]# xfs_repair -L /dev/mapper/nas--0--1-export
Phase 1 - find and verify superblock...
Phase 2 - using internal log
        - zero log...
ALERT: The filesystem has valuable metadata changes in a log which is being
destroyed because the -L option was used.
        - scan filesystem freespace and inode maps...
agi unlinked bucket 6 is 26403654 in ag 0 (inode=26403654)
sb_icount 17162816, counted 34859264
sb_ifree 125, counted 1683
sb_fdblocks 69126412932, counted 60959141713
        - found root inode chunk
Phase 3 - for each AG...
        - scan and clear agi unlinked lists...
        - process known inodes and perform inode discovery...
        - agno = 0
data fork in ino 2425892 claims free block 6063805942
data fork in ino 2425892 claims free block 6063805943
```

The XFS FAQ provides excellent guide: http://xfs.org/index.php/XFS_FAQ

When doing `xfs_repair`, sometimes you may find that you get stuck in phase 6:

```
Phase 6 - check inode connectivity...
        - resetting contents of realtime bitmap and summary inodes
        - traversing filesystem ...
```

This is usually caused by prefetching. Just add the `-P` argument to xfs_repair.

## LSI self-check does not pass

When you are starting up the storage server, the LSI/AVAGO BIOS will show something like 'F/W initializing devices x%', where the x changes from 0 to 100. It may take a few seconds or a minute. Additional information, such as the battery status, the PCI slot number, and the physical drives, will be shown.

Sometimes the screen freezes at some point, such as the 'F/W initializing step' and cannot continue. If you encounter this, try to power cycle the whole system (unplugging power and wait 30 seconds). Try this a few times, and perhaps it will work the second or third time that you try this.

Then you should perhaps consider replacing the RAID card since it is likely defective or at least unstable. Buy a new card. Then just replace the old card by the new card, plug the cable, then everything should just work immediately. The reason is that RAID configurations are stored in each hard drive, so that a new card can still read and import these configurations and start working immediately after the replacement.


## Foreign configuration found

Sometimes when you start up the storage server, the LSI/AVAGO BIOS will show a message about 'foreign configuration found', and ask you what to do. You did not change anything in the system, why does this message show up? It could be due to a variety of reasons, say for example, one hard drive suddently becomes bad and is dropped from the RAID array, and LSI raid immediately detects so it starts rebuilding using a hot spare (so the configuration changed automatically). Now this hard drive recovered and becomes live again, the configuration on it is now "foreign" as it does not belong to the current array. In this case, you can probably just configure it as a hot spare without importing foreign configuration, while at the same time buy another drive to replace it soon since it becomes unstable.

## Entire virtual drive or drive group is gone!

This may occur for all drives in a drive group, say for example, you connect JBOD to a storage server but somehow the power for JBOD gets disconnected for a second. Now the system will show I/O error as the JBOD is not synchronized with the main storage server. If you restart the storage server, the volume can no longer be mounted, and in fact the LSI BIOS will show error such as '3 virtual drives found on the host adapter, 1 virtual drive offline'. Do not panic, there are ways to address that.

First make sure whether there are problems with the physical cable or the SAS expander. You can just count the number of drives found by the RAID card, and see whether all drives are indeed found by the card. If so, then it is probably just a configuration problem to recover the lost virtual drive.

Go to LSI/AVAGO BIOS, then check what is wrong with the drives in the virtual drive. It is likely that they are all marked as 'frn-bad' (foreign bad) or 'unconfigured bad'. For each of the drive, manually change them to 'good'. Once all the changes are done, scan the adapter again, and if it ask whether to import foreign configuration (remember that once some drives are out of the array even just for one second, they are considered foreign and their membership gets lost in the array), say yes. Then the old virtual drive will show up now, and the system should have returned normal.

Make sure to do a `xfs_check` to see whether there is anything wrong with the system, and if so, repair it.

## Some handy commands

- `lsblk` can be used to check all block devices in a system quickly. You can do `fdisk -l` as well but the output is messy.

```
[root@phoenix ~]# lsblk 
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 223.6G  0 disk 
|-sda1   8:1    0 195.3G  0 part /
`-sda2   8:2    0  28.3G  0 part [SWAP]
sdb      8:16   0 223.6G  0 disk 
`-sdb1   8:17   0 223.6G  0 part /state/partition1
```

## diagnosing high I/O wait

Sometimes a system may become extremely slow and unreponsive. Running `top` shows that I/O wait value is very high, suggesting that storage is the main reason. It is not straightforward to diagnose what is causing the problem, but there are a few things that you may try.

First in `top`, press `M`, to sort all jobs by the memory usage. It is possible that a specific job is using a very large amount of memory so that it begins to use up swap space, resulting in slow performance of the system. If so, kill this job.

Next, do `iostat -x -d 2`, to check the I/O status for different devices in the system. A consistently high %util indicates potential problems with the volume.

```
[root@nas-0-1 /]$ iostat -x -d 1
Linux 2.6.32-504.16.2.el6.x86_64 (nas-0-1.local)        05/25/2017      _x86_64_        (8 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda               0.05     6.34    0.14    0.42    23.21    54.11   139.23     0.02   28.45   5.17   0.29
sdb               0.02     0.03    4.42    2.92   360.25  1501.07   253.36     0.22   29.39   2.35   1.73
sdd               0.14     0.01    6.63    6.39  1831.61  3420.39   403.34     0.16   12.53   1.98   2.58
sde               0.08     0.01    5.05    5.94  1368.72  3181.95   413.95     0.25   22.62   1.41   1.55
sdc               0.00     0.00    0.90    1.48   238.15   775.68   426.22     0.14   58.12   1.32   0.31
sdg               0.07     0.01    4.81    6.57  1304.04  3520.98   423.93     0.07    6.55   1.51   1.72
sdf               0.11     0.02    5.00    7.03  1375.41  3767.08   427.66     0.10    8.29   1.59   1.91
dm-0              0.00     0.00   27.24   30.20  6478.19 16167.15   394.21     0.11    1.84   0.98   5.63

Device:         rrqm/s   wrqm/s     r/s     w/s   rsec/s   wsec/s avgrq-sz avgqu-sz   await  svctm  %util
sda               0.00     0.00    1.00    0.00    16.00     0.00    16.00     0.03   25.00  25.00   2.50
sdb               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
sdd               3.00     0.00    5.00  210.00  2048.00 114800.00   543.48   147.54  662.59   4.53  97.50
sde               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
sdc               0.00     0.00    0.00    0.00     0.00     0.00     0.00     0.00    0.00   0.00   0.00
sdg               0.00     0.00   15.00    0.00  4096.00     0.00   273.07     0.31   20.73   8.60  12.90
sdf               0.00     0.00    2.00    0.00   512.00     0.00   256.00     0.04   33.50  11.00   2.20
dm-0              0.00     0.00   23.00  194.00  6144.00 106040.00   516.98   149.87  666.91   4.49  97.40
```

Usually you can use `dmsetup info /dev/dm-0` and `lsblk` to check exactly how the LVM partitions are used. In this case, there are 7 drives that are combined into the dm-0.

```
[root@nas-0-1 /]$ dmsetup info /dev/dm-0 
Name:              nas--0--1-export
State:             ACTIVE
Read Ahead:        256
Tables present:    LIVE
Open count:        1
Event number:      0
Major, minor:      253, 0
Number of targets: 6
UUID: LVM-GPKU7vxQUCQnWpkSqkU2zyM23bhARXQbTwDgtk942TUELdbHhVlGXAlzJzZsvXgT
[root@nas-0-1 /]$ lsblk 
NAME                        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda                           8:0    0   100G  0 disk 
|-sda1                        8:1    0  78.1G  0 part /
`-sda2                        8:2    0  21.9G  0 part [SWAP]
sdb                           8:16   0  57.2T  0 disk 
`-sdb1                        8:17   0  57.2T  0 part 
  `-nas--0--1-export (dm-0) 253:0    0 372.8T  0 lvm  /export
sdd                           8:48   0  76.4T  0 disk 
`-sdd1                        8:49   0  76.4T  0 part 
  `-nas--0--1-export (dm-0) 253:0    0 372.8T  0 lvm  /export
sde                           8:64   0  69.1T  0 disk 
`-sde1                        8:65   0  69.1T  0 part 
  `-nas--0--1-export (dm-0) 253:0    0 372.8T  0 lvm  /export
sdc                           8:32   0  24.6T  0 disk 
`-sdc1                        8:33   0  24.6T  0 part 
  `-nas--0--1-export (dm-0) 253:0    0 372.8T  0 lvm  /export
sdg                           8:96   0  65.5T  0 disk 
`-sdg1                        8:97   0  65.5T  0 part 
  `-nas--0--1-export (dm-0) 253:0    0 372.8T  0 lvm  /export
sdf                           8:80   0    80T  0 disk 
`-sdf1                        8:81   0    80T  0 part 
  `-nas--0--1-export (dm-0) 253:0    0 372.8T  0 lvm  /export
  ```

Next, use the `ps` command to check what programs are running that may consume too much I/O resources. This is borrowed from [this post](http://bencane.com/2012/08/06/troubleshooting-high-io-wait-in-linux/). 

```
 # for x in `seq 1 1 10`; do ps -eo state,pid,cmd | grep "^D"; echo "----"; sleep 5; done
```

If a program shows up consistently in D state, it means that it is always waiting I/O. Now you can use `lsof -p <pid>` to find out what files are being written or read.



















