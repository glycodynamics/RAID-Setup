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
sk1@gag:~$ lsblk 
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
sk1@gag:/home/sk1$ sudo parted /dev/sda
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

# Creating RAID5 on three drives but adding a spare drive (lest say you have partitioned sdd to be used as a spare drive if one fails. The spare drive will be used automatically to rebuild the array once a disk fails.
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb1 /dev/sdc1 /dev/sdd1 --spare-devices=1 /dev/sdd1
```
This command creates a RAID 5 array called /dev/md0 using disks sda, sdb, and sdc. The later command with ```--spare-devices``` . Adjust the command based on the desired RAID device name and the number and names of your disks. Wait for the command to complete. The process may take some time, depending on the size of the disks.

At this point, disks will be totally crazy and start working at full capacity to reconstruct your array. Have a look in /proc/mdstat to see what's going on.

```
sk1@gag:/home/sk1$ cat /proc/mdstat 
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md0 : active raid5 sdc1[3] sdb1[1] sda1[0]
      31251490816 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [>....................]  recovery =  1.8% (288054120/15625745408) finish=1265.6min speed=201979K/sec
      bitmap: 0/117 pages [0KB], 65536KB chunk

      
unused devices: <none>

## After about 18 hours:
sk1@gag:/home/sk1$ cat /proc/mdstat 
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md0 : active raid5 sdc1[3] sdb1[1] sda1[0]
      31251490816 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [=============>.......]  recovery = 68.5% (10706726780/15625745408) finish=487.0min speed=168337K/sec
      bitmap: 0/117 pages [0KB], 65536KB chunk

unused devices: <none>
```
#### NOTE: HAVE PATIENCE AND WAIT FOR A DAY OR TWO UNTIL THIS STEP IS COMPLETE

```
### After 30 hours:
sk1@gag:/$ cat /proc/mdstat 
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md0 : active raid5 sdc1[3] sdb1[1] sda1[0]
      31251490816 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      bitmap: 0/117 pages [0KB], 65536KB chunk

unused devices: <none>
```


Remember, patience is a key virtue when it comes to RAID recovery, and taking necessary precautions will safeguard your data.
During the process of setting up RAID, one crucial aspect often overlooked is the recovery phase. It is of utmost importance to exercise patience and refrain from rebooting the machine or making any changes to the drives while the recovery is in progress. Interrupting the recovery process can lead to data corruption or failure in rebuilding the RAID array.


#### 4. Verifying the Changes
Verify which disks are involved in RAID:
```
sk1@gag:/home/sk1$ lsblk -o NAME,UUID,MOUNTPOINT
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
#### 5. Create an XFS Filesystem
```
sk1@gag:$ sudo mkfs.xfs /dev/md0
meta-data=/dev/md0               isize=512    agcount=32, agsize=244152192 blks
         =                       sectsz=4096  attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1
data     =                       bsize=4096   blocks=7812870144, imaxpct=5
         =                       sunit=128    swidth=256 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=521728, version=2
         =                       sectsz=4096  sunit=1 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
If the information is incorrect, specify stripe geometry manually with the su and sw parameters of the -d option. The su parameter specifies the RAID chunk size, and the sw parameter specifies the number of data disks in the RAID device.

#### 6. Save your created filesystem and mount point
After the array is ready, it is important to save the configuration in the proper mdadm configuration file. In Ubuntu, this is file /etc/mdadm/mdadm.conf. In some other distributions, this is file can be in /etc/mdadm.conf. Check your distribution's documentation, or look at man mdadm.conf, to see what applies to your distribution.

```
sk1@gag:/home/sk1$ sudo mdadm --detail --scan
[sudo] password for sk1: 
ARRAY /dev/md0 metadata=1.2 spares=1 name=gag.olemiss.edu:0 UUID=345d1e57:859a84ab:5aa7e5ee:062c3828


## Write to mdadm.conf

mdadm --detail --scan >> /etc/mdadm/mdadm.conf
## 


#### 7. Mount the filesystem

Check the UUID of md0 attay:
sk1@gag:/scratch$ lsblk -o NAME,UUID
NAME    UUID                                 MOUNTPOINT
sda                                          
└─sda1  345d1e57-859a-84ab-5aa7-e5ee062c3828 
  └─md0 4b857035-2469-4ba9-9c83-960125cd75ed
sdb                                          
└─sdb1  345d1e57-859a-84ab-5aa7-e5ee062c3828 
  └─md0 4b857035-2469-4ba9-9c83-960125cd75ed
sdc                                          
└─sdc1  345d1e57-859a-84ab-5aa7-e5ee062c3828 
  └─md0 4b857035-2469-4ba9-9c83-960125cd75ed

put the raid array UUID and mount point in /etc/fstab:

# RAID5 /dev/md0 on sda1, sdb1 and sdc1
UUID=4b857035-2469-4ba9-9c83-960125cd75ed /scratch                 xfs     defaults,usrquota,grpquota        1 2

#Then do:
sk1@gag:~$ sudo mount /scratch

sk1@gag:~$ sudo df -H
Filesystem      Size  Used Avail Use% Mounted on
/dev/md0         32T  224G   32T   1% /scratch
```
All set! The usrquota,grpquota keywords are needed only when you want to enable user quota or disk quote on the system. Reboot the machine once to sure everything starts itself, and the raid is visible.

####  Time for running as shots as the troubleshooting (which I really wish you dont have to do ever!) needs a strong heart when there is data on disks. 


### Troubleshooting 

### Stopping and Running RAID:

Stopping a running RAID device is easy:

```
sk1@gag:~$ sudo mdadm --stop /dev/md0
mdadm: stopped /dev/md0
```
Running is complicated and ```mdadm --run /dev/md0``` will NOT work. Do instead:

```
sk1@gag:~$ sudo mdadm --assemble --scan 
mdadm: /dev/md0 has been started with 3 drives.

or

mdadm --assemble /dev/md0 /dev/sda1 /dev/sdb1 /dev/sdc1
```

### RAID Recovery

During a disk failure, RAID-5 read performance slows down because each time data from the failed drive is needed, the parity algorithm must reconstruct the lost data. Writes during a disk failure do not take a performance hit and will actually be slightly faster. Once a failed disk is replaced, data reconstruction begins either automatically or after a system administrator intervenes, depending on the hardware. 
Here we are going to cover a situation where we had RAID 5 set up on 4x 14TB HDDs, and upon reboot, RAID 5 is now inactive, and the drive cannot be mounted. We will follow a few steps to diagnose the issue and fix the RAID array, ensuring we can recover the data. 


1. Findout what's going wrong
   
- Check ``lsblk``` and mdstat

```
sk1@gag:/home/sk1$ cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md127 : inactive sdg[1](S) sdh[2](S)
      27344500736 blocks super 1.2
       
unused devices: <none>

sk1@gag:/home/sk1$ sudo lsblk 
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
sk1@gag:/home/sk1$ sudo  sg_map -x
/dev/sg0  0 0 0 0  0  /dev/sda
/dev/sg1  1 0 0 0  0  /dev/sdb
/dev/sg2  2 0 0 0  0  /dev/sdc
/dev/sg3  3 0 0 0  0  /dev/sdd
/dev/sg4  6 0 0 0  0  /dev/sde
/dev/sg5  7 0 0 0  0  /dev/sdf
/dev/sg6  8 0 0 0  0  /dev/sdg
/dev/sg7  9 0 0 0  0  /dev/sdh
/dev/sg8  10 0 0 0  0  /dev/sdi

sk1@gag:/home/sk1$ sudo cat /etc/mdadm/mdadm.conf 
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
sk1@gag:/home/sk1$ sudo fdisk -l /dev/sdf
Disk /dev/sdf: 12.75 TiB, 14000519643136 bytes, 27344764928 sectors
Disk model: WDC  WUH721414AL
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
Disklabel type: gpt
Disk identifier: 3F87EE4D-5583-45FD-BD8F-45D8600076FF

Device     Start         End     Sectors  Size Type
/dev/sdf1   2048 27344762879 27344760832 12.8T Linux filesystem


sk1@gag:/home/sk1$ sudo fdisk -l /dev/sdg
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


sk1@gag:/home/sk1$ sudo fdisk -l /dev/sdh
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


sk1@gag:/home/sk1$ sudo fdisk -l /dev/sdi
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
sk1@gag:/home/sk1$ sudo mdadm --query /dev/md127
/dev/md127:  (null) 0 devices, 2 spares. Use mdadm --detail for more detail.

sk1@gag:/home/sk1$ sudo mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
        Raid Level : raid0
     Total Devices : 2
       Persistence : Superblock is persistent

             State : inactive
   Working Devices : 2

              Name : user.domain.com:100
              UUID : f9883684:3b9d073e:ef83584f:911f6a4b
            Events : 122729

    Number   Major   Minor   RaidDevice

       -       8      112        -        /dev/sdh
       -       8       96        -        /dev/sdg

```
Now it's clear that sdf and adi are, for some reason, not part of md127 anymore, and strangely, it shows raid0 while it was actually raid5. I think RAID is just guessing raid0 here, as there are only two disks as part of the raid array. 

Now examine all four disks:
```
sk1@gag:/home/sk1$ sudo mdadm --examine /dev/sdf
/dev/sdf:
   MBR Magic : aa55
Partition[0] :   4294967295 sectors at            1 (type ee)



sk1@gag:/home/sk1$ sudo mdadm --examine /dev/sdg
/dev/sdg:
          Magic : a92b4efc
        Version : 1.2
    Feature Map : 0x1
     Array UUID : f9883684:3b9d073e:ef83584f:911f6a4b
           Name : user.domain.com:100
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



sk1@gag:/home/sk1$ sudo mdadm --examine /dev/sdh
/dev/sdh:
          Magic : a92b4efc
        Version : 1.2
    Feature Map : 0x1
     Array UUID : f9883684:3b9d073e:ef83584f:911f6a4b
           Name : user.domain.com:100
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


sk1@gag:/home/sk1$ sudo mdadm --examine /dev/sdi
/dev/sdi:
   MBR Magic : aa55
Partition[0] :   4294967295 sectors at            1 (type ee)

```

Now it's clear that It looks like the partition table was changed on sdf and sdi. You can follow the following steps at your own risk on loosing the data:

### Fix

1. Comment out /data in fstab so it will not automatically be fs checked or mounted during boot.

2. Check the current state: lsblk --fs
```
sk1@gag:/home/sk1$ lsblk --fs
NAME    FSTYPE            LABEL                 UUID                                 FSAVAIL FSUSE% MOUNTPOINT
sde                                                                                                 
├─sde1                                                                                              
├─sde2  ext4                                    a24f589a-8113-4e31-bc00-bad36d8271fa  581.9M    33% /boot
├─sde3  swap                                    af84dd67-c733-4e45-a2b3-df63a5a83ede                [SWAP]
└─sde4  ext4                                    72c80aee-79b6-4070-8f0b-2eef4421ff0c    2.8T    16% /
sdf                                                                                                 
└─sdf1                                                                                              
sdg     linux_raid_member user.domain.com:100 f9883684-3b9d-073e-ef83-584f911f6a4b                
sdh     linux_raid_member user.domain.com:100 f9883684-3b9d-073e-ef83-584f911f6a4b                
sdi                                                                                                 
└─sdi1                                                                                    
   ```
3. Shows sdg and sdh as raid members. If you see a file system detected on sdf1 or sdi1, this will likely not work or be considerably more likely to run into trouble.

Also, possibly check cat /proc/partitions

3. Clear GPT label and partition on sdf and sdi:

```
sk1@gag:/scratch$ sudo dd if=/dev/zero of=/dev/sdf bs=512 count=34
[sudo] password for sk1: 
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 1.00613 s, 17.3 kB/s


sk1@gag:/scratch$ sudo dd if=/dev/zero of=/dev/sdi bs=512 count=34
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 1.02758 s, 16.9 kB/s

sk1@gag:/scratch$ sudo dd if=/dev/zero of=/dev/sdf bs=512 count=34 seek=$((`sudo blockdev --getsz /dev/sdf` - 34))
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 0.00206208 s, 8.4 MB/s

sk1@gag:/scratch$ sudo dd if=/dev/zero of=/dev/sdi bs=512 count=34 seek=$((`sudo blockdev --getsz /dev/sdi` - 34))
34+0 records in
34+0 records out
17408 bytes (17 kB, 17 KiB) copied, 0.00205267 s, 8.5 MB/s

sk1@gag:/scratch$ lsblk --fs
NAME    FSTYPE            LABEL                 UUID                                 FSAVAIL FSUSE% MOUNTPOINT
loop0   squashfs                                                                           0   100% /snap/bare/5
loop1   squashfs                                                                           0   100% /snap/chromium/2497
loop2   squashfs                                                                           0   100% /snap/core18/2751
loop3   squashfs                                                                           0   100% /snap/lxd/25086
loop4   squashfs                                                                           0   100% /snap/lxd/25112
loop5   squashfs                                                                           0   100% /snap/chromium/2529
loop6   squashfs                                                                           0   100% /snap/core18/2785
loop7   squashfs                                                                           0   100% /snap/core20/1974
loop8   squashfs                                                                           0   100% /snap/core22/806
loop9   squashfs                                                                           0   100% /snap/gnome-42-2204/111
loop10  squashfs                                                                           0   100% /snap/gnome-3-38-2004/143
loop11  squashfs                                                                           0   100% /snap/gtk-common-themes/1535
loop12  squashfs                                                                           0   100% /snap/snapd/19361
loop13  squashfs                                                                           0   100% /snap/gnome-42-2204/120
loop14  squashfs                                                                           0   100% /snap/core20/1950
loop15  squashfs                                                                           0   100% /snap/cups/980
loop16  squashfs                                                                           0   100% /snap/gtk-common-themes/1534
loop17  squashfs                                                                           0   100% /snap/snapd/19457
loop18  squashfs                                                                           0   100% /snap/cups/974
loop19  squashfs                                                                           0   100% /snap/core22/817
loop20  squashfs                                                                           0   100% /snap/gnome-3-38-2004/140
sda                                                                                                 
└─sda1  linux_raid_member gag.olemiss.edu:0     345d1e57-859a-84ab-5aa7-e5ee062c3828                
  └─md0 xfs                                     4b857035-2469-4ba9-9c83-960125cd75ed   28.9T     1% /scratch
sdb                                                                                                 
└─sdb1  linux_raid_member gag.olemiss.edu:0     345d1e57-859a-84ab-5aa7-e5ee062c3828                
  └─md0 xfs                                     4b857035-2469-4ba9-9c83-960125cd75ed   28.9T     1% /scratch
sdc                                                                                                 
└─sdc1  linux_raid_member gag.olemiss.edu:0     345d1e57-859a-84ab-5aa7-e5ee062c3828                
  └─md0 xfs                                     4b857035-2469-4ba9-9c83-960125cd75ed   28.9T     1% /scratch
sdd                                                                                                 
sde                                                                                                 
├─sde1                                                                                              
├─sde2  ext4                                    a24f589a-8113-4e31-bc00-bad36d8271fa  581.9M    33% /boot
├─sde3  swap                                    af84dd67-c733-4e45-a2b3-df63a5a83ede                [SWAP]
└─sde4  ext4                                    72c80aee-79b6-4070-8f0b-2eef4421ff0c    2.8T    16% /
sdf                                                                                                 
sdg     linux_raid_member user.domain.com:100 f9883684-3b9d-073e-ef83-584f911f6a4b                
sdh     linux_raid_member user.domain.com:100 f9883684-3b9d-073e-ef83-584f911f6a4b                
sdi                                                                                                 


sk1@gag:/scratch$ sudo mdadm -S /dev/md127
mdadm: stopped /dev/md127


sk1@gag:/scratch$ lsblk --fs
NAME    FSTYPE            LABEL             UUID                                 FSAVAIL FSUSE% MOUNTPOINT
loop0   squashfs                                                                       0   100% /snap/bare/5
loop1   squashfs                                                                       0   100% /snap/chromium/2497
loop2   squashfs                                                                       0   100% /snap/core18/2751
loop3   squashfs                                                                       0   100% /snap/lxd/25086
loop4   squashfs                                                                       0   100% /snap/lxd/25112
loop5   squashfs                                                                       0   100% /snap/chromium/2529
loop6   squashfs                                                                       0   100% /snap/core18/2785
loop7   squashfs                                                                       0   100% /snap/core20/1974
loop8   squashfs                                                                       0   100% /snap/core22/806
loop9   squashfs                                                                       0   100% /snap/gnome-42-2204/111
loop10  squashfs                                                                       0   100% /snap/gnome-3-38-2004/143
loop11  squashfs                                                                       0   100% /snap/gtk-common-themes/1535
loop12  squashfs                                                                       0   100% /snap/snapd/19361
loop13  squashfs                                                                       0   100% /snap/gnome-42-2204/120
loop14  squashfs                                                                       0   100% /snap/core20/1950
loop15  squashfs                                                                       0   100% /snap/cups/980
loop16  squashfs                                                                       0   100% /snap/gtk-common-themes/1534
loop17  squashfs                                                                       0   100% /snap/snapd/19457
loop18  squashfs                                                                       0   100% /snap/cups/974
loop19  squashfs                                                                       0   100% /snap/core22/817
loop20  squashfs                                                                       0   100% /snap/gnome-3-38-2004/140
sda                                                                                             
└─sda1  linux_raid_member gag.olemiss.edu:0 345d1e57-859a-84ab-5aa7-e5ee062c3828                
  └─md0 xfs                                 4b857035-2469-4ba9-9c83-960125cd75ed   28.9T     1% /scratch
sdb                                                                                             
└─sdb1  linux_raid_member gag.olemiss.edu:0 345d1e57-859a-84ab-5aa7-e5ee062c3828                
  └─md0 xfs                                 4b857035-2469-4ba9-9c83-960125cd75ed   28.9T     1% /scratch
sdc                                                                                             
└─sdc1  linux_raid_member gag.olemiss.edu:0 345d1e57-859a-84ab-5aa7-e5ee062c3828                
  └─md0 xfs                                 4b857035-2469-4ba9-9c83-960125cd75ed   28.9T     1% /scratch
sdd                                                                                             
sde                                                                                             
├─sde1                                                                                          
├─sde2  ext4                                a24f589a-8113-4e31-bc00-bad36d8271fa  581.9M    33% /boot
├─sde3  swap                                af84dd67-c733-4e45-a2b3-df63a5a83ede                [SWAP]
└─sde4  ext4                                72c80aee-79b6-4070-8f0b-2eef4421ff0c    2.8T    16% /
sdf                                                                                             
sdg                                                                                             
sdh                                                                                             
sdi                                                                                       


sk1@gag:/scratch$ sudo mdadm --create --verbose /dev/md127 --level=5  --raid-devices=4 /dev/sdf /dev/sdg /dev/sdh /dev/sdi
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: partition table exists on /dev/sdg
mdadm: partition table exists on /dev/sdg but will be lost or
       meaningless after creating array
mdadm: partition table exists on /dev/sdh
mdadm: partition table exists on /dev/sdh but will be lost or
       meaningless after creating array
mdadm: size set to 13672250368K
mdadm: automatically enabling write-intent bitmap on large array
Continue creating array? no
mdadm: create aborted.
```
If successful, this is indeed encouraging. 


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
sk1@gag:~$ sudo mdadm --create --assume-clean --verbose /dev/md127 --level=5  --raid-devices=4 /dev/sdf /dev/sdg /dev/sdh /dev/sdi    
[sudo] password for sk1: 
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: partition table exists on /dev/sdg
mdadm: partition table exists on /dev/sdg but will be lost or
       meaningless after creating array
mdadm: partition table exists on /dev/sdh
mdadm: partition table exists on /dev/sdh but will be lost or
       meaningless after creating array
mdadm: size set to 13672250368K
mdadm: automatically enabling write-intent bitmap on large array
Continue creating array? yes
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md127 started.
```
Now check mdstat:
```
sk1@gag:~$ cat /proc/mdstat 
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10] 
md127 : active raid5 sdi[3] sdh[2] sdg[1] sdf[0]
      41016751104 blocks super 1.2 level 5, 512k chunk, algorithm 2 [4/4] [UUUU]
      bitmap: 102/102 pages [408KB], 65536KB chunk

md0 : active raid5 sda1[0] sdc1[3] sdb1[1]
      31251490816 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/3] [UUU]
      bitmap: 0/117 pages [0KB], 65536KB chunk

unused devices: <none>
```
looks good :)

```
sk1@gag:/home/sk1$ sudo mdadm --query /dev/md127
[sudo] password for sk1:
/dev/md127:  raid5 4 devices, 0 spares. Use mdadm --detail for more detail.
sk1@gag:/home/sk1$ sudo mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
     Creation Time : Tue May 24 11:37:00 2022
        Raid Level : raid5
     Used Dev Size : 18446744073709551615
      Raid Devices : 4
     Total Devices : 2
       Persistence : Superblock is persistent

       Update Time : Tue Jul  4 08:29:36 2023
             State : active, FAILED, Not Started
    Active Devices : 2
   Working Devices : 2
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : unknown

              Name : user.domain.com:100
              UUID : f9883684:3b9d073e:ef83584f:911f6a4b
            Events : 122729

    Number   Major   Minor   RaidDevice State
       -       0        0        0      removed
       -       0        0        1      removed
       -       0        0        2      removed
       -       0        0        3      removed

       -       8       32        1      sync   /dev/sdc
       -       8       48        2      sync   /dev/sdd



sk1@gag:~$ sudo mdadm --detail /dev/md127
/dev/md127:
           Version : 1.2
     Creation Time : Wed Jul 19 15:28:41 2023
        Raid Level : raid5
        Array Size : 41016751104 (39116.62 GiB 42001.15 GB)
     Used Dev Size : 13672250368 (13038.87 GiB 14000.38 GB)
      Raid Devices : 4
     Total Devices : 4
       Persistence : Superblock is persistent

     Intent Bitmap : Internal

       Update Time : Wed Jul 19 15:29:39 2023
             State : clean 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : bitmap

              Name : gag.olemiss.edu:127  (local to host gag.olemiss.edu)
              UUID : 8dec2ecc:e24c1a04:e2e682e6:cfcfdf5e
            Events : 2

    Number   Major   Minor   RaidDevice State
       0       8       80        0      active sync   /dev/sdf
       1       8       96        1      active sync   /dev/sdg
       2       8      112        2      active sync   /dev/sdh
       3       8      128        3      active sync   /dev/sdi

```
Whoa! Did you expect it?

7. Attempt to mount read-only
```
sk1@gag:~$ sudo mount -o ro -t xfs /dev/md127 /data
sk1@gag:~$ cd /data/
sk1@gag:/data$ ls

THIS IS INDEED YOUR DATA, AND NOW YOU CAN BACK IT UP AND GO FOR RUNNING! Note that you are supposed to create RAID `correctly` again on these drives to make it work upon reboot.
```

### Acknowledgement
Thanks to Ken and Gabe from the Mississippi Center of Supercomputing for showing how to delete partition tables. 

Other resources:
https://raid.wiki.kernel.org/index.php/RAID_setup

