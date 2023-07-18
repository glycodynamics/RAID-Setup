# RAID Setup

RAID (Redundant Array of Independent Disks) is a popular data storage configuration that provides both performance and data redundancy. By distributing parity information across multiple drives, RAID 5 ensures data integrity and fault tolerance up to a single disk. In this step-by-step guide, we will walk you through the process of setting up RAID. I will set up raid 5 on three 16 tb HDDs but you change the RAID type and the number of disks accordingly in the specific spots. The overall idea and steps remain the same. 

First, chek if your system has already a raid setup on some other disks or not. If your machine has RAID support, it should have a file named "mdstat' inside directory ./proc (/proc/mdstat). Note that this file is one of the important ones and if you do not have that file, maybe your kernel does not have RAID support.

here are mainly 5 steps involved in setting up the RAID:

1. Varify Disk Status:
2. Partitioning 
3. Creating RAID array
4. Verifing the Changes
5. Create a filesystem and mount point

#### Verify Disk Status:
Use ```lsblk``` to list blocks and print the number of drives and how they have been installed. Make sure that disks sda, sdb, and sdc are connected to the machine and recognized by the operating system.

```
sushil@gag:~$ lsblk 
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0  14.6T  0 disk 
sdb      8:16   0  14.6T  0 disk 
sdc      8:32   0  14.6T  0 disk 
sdd      8:48   0  14.6T  0 disk 
sde      8:64   0   3.7T  0 disk 
├─sde1   8:65   0     1M  0 part 
├─sde2   8:66   0     1G  0 part /boot
├─sde3   8:67   0    32G  0 part [SWAP]
└─sde4   8:68   0   3.6T  0 part /
sdf      8:80   0  12.8T  0 disk 
└─sdf1   8:81   0  12.8T  0 part 
sdg      8:96   0  12.8T  0 disk 
sdh      8:112  0  12.8T  0 disk 
sdi      8:128  0  12.8T  0 disk 
└─sdi1   8:129  0  12.8T  0 part 
```

For example, in this machine, I have 4x 16 tb HDDs (sd[a-d]), an 8TB NVMe drive for OS (sde) and 4x 14TB HDDs. We will be building RAID 5 array on sda, sdb and sdc. 

#### 2. Partitioning the disks
HDDS over 2 TB capacity has to be partitioned using GPT (GUID Partition Table) to use the full capacity of the disk. You can do that by partitioning the disks with ```fdisk``` or ```parted```. Since gpt support in fdisk is still in the experimental stage, we will use parted here to create GPT. Open a terminal or command prompt and execute the command ```sudo parted /dev/sda``` to start the parted utility for disk sda. Replace /dev/sda with the appropriate disk identifier for sdb and sdc in subsequent steps.

Within the parted utility, run the following commands:
-  Type mklabel gpt to create a GPT partition table on the disk.
-  Give a specific partition name of leave it empty if you will have just 1 partition
-  Choose file system trype: xfs 
-  Start : 0%
-  End : 100%   - this assigns complete disk to be used in RAID. Ajust the size as needed for your specific requirements.
-  Type quit to exit the parted utility.
-  Repeat steps 3-4 for disks sdb and sdc, replacing /dev/sda with /dev/sdb and /dev/sdc, respectively.You can also use select /dev/sdb in parted to start doing operations on /dev/sdb without quitting parted after setting up dev/sda.

```
ccbrc@gag:/home/sushil$ sudo parted /dev/sda
GNU Parted 3.3
Using /dev/sda
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) mklabel gpt                                                      
(parted) mkpart                                                           
Partition name?  []?                                                      
File system type?  [ext2]? xfs                                            
Start? 0%                                                                 
End? 100%

(parted) select /dev/sdb                                                  
Using /dev/sdb
(parted) mklabel gpt                                                      
(parted) mkpart                                                           
Partition name?  []?                                                      
File system type?  [ext2]? xfs                                            
Start? 0%                                                                 
End? 100%                                                                 

(parted) select /dev/sdc                                                  
Using /dev/sdc
(parted) mklabel gpt                                                      
(parted) mkpart                                                           
Partition name?  []?                                                      
File system type?  [ext2]? xfs                                            
Start? 0%                                                                 
End? 100%                                                                 

(parted) q                                                                
Information: You may need to update /etc/fstab.

```
#### 3. Creating RAID array
If you use N devices where the smallest has size S, the size of the entire raid-5 array will be (N-1)*S for RAID 5, or (N-2)*S for raid-6. This "missing" space is used for parity (redundancy) information. Thus, if any disk fails, all the data stays intact. But if two disks fail on raid-5, or three on raid-6, all data is lost. 

We will be creating RAID5 on this example. Execute the command:
```
# creating RAID 5 on three rives:
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sda1 /dev/sdb1 /dev/sdc1

# Creating RAID5 on three drives but adding a spare drive (lest say you have partitioned sdd to be used as spare drive if one fails. Spare drive will be used automatically to rebuild the array once a disk fails.
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb1 /dev/sdc1 /dev/sdd1 --spare-devices=1 /dev/sdd1
```
This command creates a RAID 5 array called /dev/md0 using disks sda, sdb, and sdc. The later command with ```--spare-devices``` . Adjust the command based on the desired RAID device name and the number and names of your disks. Wait for the command to complete. The process may take some time depending on the size of the disks.

At this point disks will be totally crazy and start working in full capacity to reconstruct of your array. Have a look in /proc/mdstat to see what's going on.

```
ccbrc@gag:/home/sushil$ cat /proc/mdstat 
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md0 : active raid5 sdc1[3] sdb1[1] sda1[0]
      31251490816 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [>....................]  recovery =  1.8% (288054120/15625745408) finish=1265.6min speed=201979K/sec
      bitmap: 0/117 pages [0KB], 65536KB chunk

      
unused devices: <none>

## After about 18 hours:
ccbrc@gag:/home/sushil$ cat /proc/mdstat 
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md0 : active raid5 sdc1[3] sdb1[1] sda1[0]
      31251490816 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [=============>.......]  recovery = 68.5% (10706726780/15625745408) finish=487.0min speed=168337K/sec
      bitmap: 0/117 pages [0KB], 65536KB chunk

unused devices: <none>

```
HAVE PATIENCE AND WAIT FOR A DAY OR TWO UNTIL THIS STEP IS COMPLETE
Remember, patience is a key virtue when it comes to RAID recovery, and taking necessary precautions will safeguard your data.
During the process of setting up RAID, one crucial aspect often overlooked is the recovery phase. It is of utmost importance to exercise patience and refrain from rebooting the machine or making any changes to the drives while the recovery is in progress. Interrupting the recovery process can lead to data corruption or failure in rebuilding the RAID array.


#### 4. Verifying the Changes
Verify which disks are involved in RAID:
```
ccbrc@gag:/home/sushil$ lsblk -o NAME,UUID,MOUNTPOINT
NAME    UUID                                 MOUNTPOINT
sda                                          
└─sda1  345d1e57-859a-84ab-5aa7-e5ee062c3828 
  └─md0                                      
sdb                                          
└─sdb1  345d1e57-859a-84ab-5aa7-e5ee062c3828 
  └─md0                                      
sdc                                          
└─sdc1  345d1e57-859a-84ab-5aa7-e5ee062c3828 
  └─md0                                      
sdd                                          
sde                                          
├─sde1                                       
├─sde2  a24f589a-8113-4e31-bc00-bad36d8271fa /boot
├─sde3  af84dd67-c733-4e45-a2b3-df63a5a83ede [SWAP]
└─sde4  72c80aee-79b6-4070-8f0b-2eef4421ff0c /
```

Have a look in /proc/mdstat. You should see that the array is ready, and running. It will show one U for each drive, UUU in our case here.

#### 5. Save your Create filesystem and mount point
After the array is ready, it is important to save the configuration in the proper mdadm configuration file. In Ubuntu, this is file /etc/mdadm/mdadm.conf. In some other distributions, this is file can be in /etc/mdadm.conf. Check your distribution's documentation, or look at man mdadm.conf, to see what applies to your distribution.

```
ccbrc@gag:/home/sushil$ sudo mdadm --detail --scan
[sudo] password for ccbrc: 
ARRAY /dev/md0 metadata=1.2 spares=1 name=gag.olemiss.edu:0 UUID=345d1e57:859a84ab:5aa7e5ee:062c3828


## Write to mdadm.conf

mdadm --detail --scan >> /etc/mdadm/mdadm.conf
```


### RAID Recovery

During a disk failure, RAID-5 read performance slows down because each time data from the failed drive is needed, the parity algorithm must reconstruct the lost data. Writes during a disk failure do not take a performance hit and will actually be slightly faster. Once a failed disk is replaced, data reconstruction begins either automatically or after a system administrator intervenes, depending on the hardware. 
Here we are going to cover a situation where we had RAID 5 set up on 4x 14TB HDDs, and upon reboot, RAID 5 is now inactive, and the drive cannot be mounted. We will follow a few steps to diagnose the issue and fix the RAID array, ensuring we can recover the data. 


1. Findout what's going wrong
   
- Check ``lsblk``` and mdstat

```
ccbrc@gag:/home/sushil$ cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md127 : inactive sdg[1](S) sdh[2](S)
      27344500736 blocks super 1.2
       
unused devices: <none>

ccbrc@gag:/home/sushil$ sudo lsblk 
NAME    MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sde       8:64   0   3.7T  0 disk  
├─sde1    8:65   0     1M  0 part  
├─sde2    8:66   0     1G  0 part  /boot
├─sde3    8:67   0    32G  0 part  [SWAP]
└─sde4    8:68   0   3.6T  0 part  /
sdf       8:80   0  12.8T  0 disk  
└─sdf1    8:81   0  12.8T  0 part  
sdg       8:96   0  12.8T  0 disk  
sdh       8:112  0  12.8T  0 disk  
sdi       8:128  0  12.8T  0 disk  
└─sdi1    8:129  0  12.8T  0 part  

````
We had sd[f-i] in RAID 5, but something went wrong, and we can no longer access the data from RAID. Disk sd[f-i] is also not showing any raid type partition. The mdstst shows md127 is inactive and only two disks sdg and sdh are part of it. There is something very strange and likely that two disks failed and system administratior did not notice it. Let's try some more things: 


```
ccbrc@gag:/home/sushil$ sudo  sg_map -x
/dev/sg0  0 0 0 0  0  /dev/sda
/dev/sg1  1 0 0 0  0  /dev/sdb
/dev/sg2  2 0 0 0  0  /dev/sdc
/dev/sg3  3 0 0 0  0  /dev/sdd
/dev/sg4  6 0 0 0  0  /dev/sde
/dev/sg5  7 0 0 0  0  /dev/sdf
/dev/sg6  8 0 0 0  0  /dev/sdg
/dev/sg7  9 0 0 0  0  /dev/sdh
/dev/sg8  10 0 0 0  0  /dev/sdi

ccbrc@gag:/home/sushil$ sudo cat /etc/mdadm/mdadm.conf 
# mdadm.conf
#
# !NB! Run update-initramfs -u after updating this file.
# !NB! This will ensure that initramfs has an up-to-date copy.
#
# Please refer to mdadm.conf(5) for information about this file.
#

# by default (built-in), scan all partitions (/proc/partitions) and all
# containers for MD superblocks. Alternatively, specify devices to scan, using
# wildcards if desired.
#DEVICE partitions containers

# automatically tag new arrays as belonging to the local system
HOMEHOST <system>

# instruct the monitoring daemon where to send mail alerts
MAILADDR root

# Definitions of existing MD arrays

# This configuration was auto-generated on Thu, 23 Apr 2020 07:34:28 +0000 by mkconf
```


```cat /etc/mdadm/mdadm.conf``` shows no RAID information has been saved in this file. That can also be the reason that the raid could not be assembled after rebooting the machine. Check "kernel" messages from /var/log/messages or the system journal from a known good boot and a bad or suspect boot to verify partitioning has not changed on the disk(s) underlying the raid. Also would indicate any hardware/media errors.

Check each disk now:

```
ccbrc@gag:/home/sushil$ sudo fdisk -l /dev/sdf
Disk /dev/sdf: 12.75 TiB, 14000519643136 bytes, 27344764928 sectors
Disk model: WDC  WUH721414AL
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 3F87EE4D-5583-45FD-BD8F-45D8600076FF

Device     Start         End     Sectors  Size Type
/dev/sdf1   2048 27344762879 27344760832 12.8T Linux filesystem


ccbrc@gag:/home/sushil$ sudo fdisk -l /dev/sdg
Disk /dev/sdg: 12.75 TiB, 14000519643136 bytes, 27344764928 sectors
Disk model: WDC  WUH721414AL
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot Start        End    Sectors Size Id Type
/dev/sdg1           1 4294967295 4294967295   2T ee GPT

Partition 1 does not start on physical sector boundary.


ccbrc@gag:/home/sushil$ sudo fdisk -l /dev/sdh
Disk /dev/sdh: 12.75 TiB, 14000519643136 bytes, 27344764928 sectors
Disk model: WDC  WUH721414AL
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: dos
Disk identifier: 0x00000000

Device     Boot Start        End    Sectors Size Id Type
/dev/sdh1           1 4294967295 4294967295   2T ee GPT

Partition 1 does not start on physical sector boundary.


ccbrc@gag:/home/sushil$ sudo fdisk -l /dev/sdi
Disk /dev/sdi: 12.75 TiB, 14000519643136 bytes, 27344764928 sectors
Disk model: WDC  WUH721414AL
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: F57637FE-ACC5-4E2B-9803-176AF573552A

Device     Start         End     Sectors  Size Type
/dev/sdi1   2048 27344762879 27344760832 12.8T Linux filesystem
```

Check md127 status using 'mdadm' 
```
ccbrc@gag:/home/sushil$ sudo mdadm --query /dev/md127
/dev/md127:  (null) 0 devices, 2 spares. Use mdadm --detail for more detail.

ccbrc@gag:/home/sushil$ sudo mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
        Raid Level : raid0
     Total Devices : 2
       Persistence : Superblock is persistent

             State : inactive
   Working Devices : 2

              Name : ccbrc.olemiss.edu:100
              UUID : f9883684:3b9d073e:ef83584f:911f6a4b
            Events : 122729

    Number   Major   Minor   RaidDevice

       -       8      112        -        /dev/sdh
       -       8       96        -        /dev/sdg

```
Now its clear that sdf and adi are for some reason not part of md127 anymore and strangely it shows Raule level raid0 while it was actually raid5. I think RAID is just guessing raid0 here as having only two disks as part of the raid array. 

Now examine all four disks:
```
ccbrc@gag:/home/sushil$ sudo mdadm --examine /dev/sdf
/dev/sdf:
   MBR Magic : aa55
Partition[0] :   4294967295 sectors at            1 (type ee)
ccbrc@gag:/home/sushil$ sudo mdadm --examine /dev/sdg
/dev/sdg:
          Magic : a92b4efc
        Version : 1.2
    Feature Map : 0x1
     Array UUID : f9883684:3b9d073e:ef83584f:911f6a4b
           Name : ccbrc.olemiss.edu:100
  Creation Time : Tue May 24 11:37:00 2022
     Raid Level : raid5
   Raid Devices : 4

 Avail Dev Size : 27344500736 (13038.87 GiB 14000.38 GB)
     Array Size : 41016751104 (39116.62 GiB 42001.15 GB)
    Data Offset : 264192 sectors
   Super Offset : 8 sectors
   Unused Space : before=264112 sectors, after=0 sectors
          State : clean
    Device UUID : 8efccfed:a07a0038:51d813bd:96645ecc

Internal Bitmap : 8 sectors from superblock
    Update Time : Tue Jul  4 08:29:36 2023
  Bad Block Log : 512 entries available at offset 64 sectors
       Checksum : 1f3639d6 - correct
         Events : 122729

         Layout : left-symmetric
     Chunk Size : 512K

   Device Role : Active device 1
   Array State : AAAA ('A' == active, '.' == missing, 'R' == replacing)
ccbrc@gag:/home/sushil$ sudo mdadm --examine /dev/sdh
/dev/sdh:
          Magic : a92b4efc
        Version : 1.2
    Feature Map : 0x1
     Array UUID : f9883684:3b9d073e:ef83584f:911f6a4b
           Name : ccbrc.olemiss.edu:100
  Creation Time : Tue May 24 11:37:00 2022
     Raid Level : raid5
   Raid Devices : 4

 Avail Dev Size : 27344500736 (13038.87 GiB 14000.38 GB)
     Array Size : 41016751104 (39116.62 GiB 42001.15 GB)
    Data Offset : 264192 sectors
   Super Offset : 8 sectors
   Unused Space : before=264112 sectors, after=0 sectors
          State : clean
    Device UUID : b2ae5a2e:42c2c398:cda588f6:e980ba60

Internal Bitmap : 8 sectors from superblock
    Update Time : Tue Jul  4 08:29:36 2023
  Bad Block Log : 512 entries available at offset 64 sectors
       Checksum : 8e551d6b - correct
         Events : 122729

         Layout : left-symmetric
     Chunk Size : 512K

   Device Role : Active device 2
   Array State : AAAA ('A' == active, '.' == missing, 'R' == replacing)
ccbrc@gag:/home/sushil$ sudo mdadm --examine /dev/sdi
/dev/sdi:
   MBR Magic : aa55
Partition[0] :   4294967295 sectors at            1 (type ee)

```

Now it's clear that It looks like the partition table was changed on sdf and sdi. You can follow the following steps at your own risk on loosing the data:

### Fix

1. Comment out /data in fstab so it will not be fs checked or mounted during boot automatically.

2. Check the current state: lsblk --fs
```
ccbrc@gag:/home/sushil$ lsblk --fs
NAME    FSTYPE            LABEL                 UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sde                                                                                                 
├─sde1                                                                                              
├─sde2  ext4                                    a24f589a-8113-4e31-bc00-bad36d8271fa  581.9M    33% /boot
├─sde3  swap                                    af84dd67-c733-4e45-a2b3-df63a5a83ede                [SWAP]
└─sde4  ext4                                    72c80aee-79b6-4070-8f0b-2eef4421ff0c    2.8T    16% /
sdf                                                                                                 
└─sdf1                                                                                              
sdg     linux_raid_member ccbrc.olemiss.edu:100 f9883684-3b9d-073e-ef83-584f911f6a4b                
sdh     linux_raid_member ccbrc.olemiss.edu:100 f9883684-3b9d-073e-ef83-584f911f6a4b                
sdi                                                                                                 
└─sdi1                                                                                    
   ```
3. Shows sdg and sdh as raid members. If you see a file system detected on sdf1 or sdi1, this will likely not work or be considerably more likely to run into trouble.

```

Also, possibly check cat /proc/partitions

3. Clear gpt label and partition on sdf and sdi:
```
# dd if=/dev/zero of=/dev/sdf bs=512 count=34
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 0.0426989 s, 408 kB/s
# dd if=/dev/zero of=/dev/sdi bs=512 count=34
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 0.0367204 s, 474 kB/s
# dd if=/dev/zero of=/dev/sdf bs=512 count=34 seek=$((`blockdev --getsz /dev/sdf` - 34))
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 0.0363717 s, 479 kB/s
# dd if=/dev/zero of=/dev/sdi bs=512 count=34 seek=$((`blockdev --getsz /dev/sdi` - 34))
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 0.0376497 s, 462 kB/s
```
4. Check again using lsblk --fs and/or cat /proc/partitions. If the sdb1 and sdd1 are not gone then you need to reboot at this point or make the OS rescan until /proc/partitions is correct.

5. Stop
```
mdadm -S /dev/md127 (this should stop mdadm on the current raid device)
```

6. Clear raid metadata info if it shows up using lsblk --fs
```
mdadm --zero-superblock /dev/sdg
mdadm --zero-superblock /dev/sdh
```

6. Create with same structure using whole disk

```
mdadm --create --verbose /dev/md127 --level=5  --raid-devices=4 /dev/sdf /dev/sdg /dev/sdh /dev/sdi
```
7. Attempt to mount read-only

```
mount -o ro -t xfs /dev/md127 /data
```
