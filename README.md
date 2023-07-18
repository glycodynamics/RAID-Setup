# RAID Setup

RAID (Redundant Array of Independent Disks) is a popular data storage configuration that provides both performance and data redundancy. By distributing parity information across multiple drives, RAID 5 ensures data integrity and fault tolerance upto a single disk. In this step-by-step guide, we will walk you through the process of setting up RAID. I will setup raid 5 on three 16 tb HDDs but you change the RAID type and number of disks accordingly in the specific spetps. The overall idea and steps remisn same. 

First chek if your system has already a raid setup on some otehr disks or not. If your machine has RAID support, it should have a file named "mdstat' inside direcotry ./proc (/proc/mdstat). Note that htis file is one of the important ones and if you do not have that file, maybe your kernel does not have RAID support.

here are mainly 5 steps involved in setting up the RAID:

1. Varify Disk Status:
2. Partitioning 
3. Creting RAID array
4. Verifing the Changes
5. Create filesystem and mount point

#### Varify Disk Status:e:
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

For example in this machine I have 4x 16 tb HDDs (sd[a-d]), an 8TB NVMe drive for OS (sde) and 4x 14TB HDDs. Here were will be building RAID 5 arraon on sda, sdb and sdc. 

#### 2. Partitioning the disks
HDDS over 2 tb capacity has to be partitioned using GPT (GUID Partition Table) to use the full capacity of the disk. You can do that by partitioning the disks with ```fdisk``` or ```parted```. Since gpt support in fdisk is still in experimental stage, we will use parted here to creat GPT. Open a terminal or command prompt and xecute the command ```sudo parted /dev/sda``` to start the parted utility for disk sda. Replace /dev/sda with the appropriate disk identifier for sdb and sdc in subsequent steps.

Within the parted utility, run the following commands:
-  Type mklabel gpt to create a GPT partition table on the disk.
-  Give a specific partition name of leave it empty if you will have just 1 partition
-  Choose file system trype: xfs 
-  Start : 0%
-  End : 100%   - this assigns complete disk to be used in RAID. Ajust the size as needed for your specific requirements.
-  Type quit to exit the parted utility.
-  Repeat steps 3-4 for disks sdb and sdc, replacing /dev/sda with /dev/sdb and /dev/sdc respectively.You can also use select /dev/sdb in parted to star doing operation on sdb with quiting it after setting up dev/sda.

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
#### 3. Creting RAID array
If you use N devices where the smallest has size S, the size of the entire raid-5 array will be (N-1)*S for RAID 5, or (N-2)*S for raid-6. This "missing" space is used for parity (redundancy) information. Thus, if any disk fails, all the data stays intact. But if two disks fail on raid-5, or three on raid-6, all data is lost. 

We will be creating RAID5 on this example. Execute the command:
```
# creating RAID 5 on three rives:
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sda1 /dev/sdb1 /dev/sdc1

# Creating RAID5 on three drives but adding a spare drive (lest say you have partitioned sdd to be used as spare drive if one fails. Spare drive will be used autometically to rebuil the array one a disk fails.
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb1 /dev/sdc1 /dev/sdd1 --spare-devices=1 /dev/sdd1
```
This command creates a RAID 5 array called /dev/md0 using disks sda, sdb, and sdc. The later command with ```--spare-devices``` . Adjust the command based on the desired RAID device name and the number and names of your disks. Wait for the command to complete. The process may take some time depending on the size of the disks.

At this point disks will ter totally crazy and start working in full capacity to reconstruction of your array. Have a look in /proc/mdstat to see what's going on.

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


#### 4. Verifing the Changes
Varify which disks are involved in RAID:
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


During a disk failure, RAID-5 read performance slows down because each time data from the failed drive is needed, the parity algorithm must reconstruct the lost data. Writes during a disk failure do not take a performance hit and will actually be slightly faster. Once a failed disk is replaced, data reconstruction begins either automatically or after a system administrator intervenes, depending on the hardware.


