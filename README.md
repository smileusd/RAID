# RAID
RAID是用于增加数据存储的性能和可靠性的技术。全程为Redundant Array of Inexpensive Disks（廉价磁盘冗余阵列）

* RAID 0 - strping
* RAID 1 - mirroring
* RAID 5 - striping with parity
* RAID 6 - striping with double parity
* RAID 10 - combining mirroring and striping

#### 准备
mdadm目前是Linux标准RAID管理工具, 能够支持多种模式，可以对RAID进行创建，管理，添加和移除设备等，还能查看RAID设备的详细信息。功能非常强大。
```
$yum install mdadm -y
$git clone git://neil.brown.name/mdadm
```
devicemapper 也能创建RAID设备.

用dd创建4个镜像作为loop设备
```
$dd if=/dev/zero of=disk1.img bs=512 count=4096000
$losetup /dev/loop2 disk1.img
...
```

### RAID level 0 – Striping
![image](http://www.prepressure.com/images/raid-level-0-striping.svg)

Raid 0 将数据进行条带化，同时向多个磁盘(至少2个)写数据.不做数据冗余，理想的情况，每个磁盘使用单独的控制器管理。
  
#### 优点
* 读写性能好，没有数据冗余。
* 所有磁盘都有用.
* 容易实现.
#### 缺点
* 没有容错机制
#### mdadm

```
mdadm --create --verbose /dev/md0 --level=stripe --raid-devices=2 /dev/loop2 /dev/loop3
mdadm --manage --stop /dev/md0
```
查看结果
```
$fdisk -l
...
Disk /dev/md0: 4192 MB, 4192206848 bytes, 8187904 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 524288 bytes / 1048576 bytes

$lsblk
NAME  MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
...
loop2  7:2    0     2G  0 loop  
└─md0  9:0    0   3.9G  0 raid0 
loop3  7:3    0     2G  0 loop  
└─md0  9:0    0   3.9G  0 raid0
```

#### devicemapper
```
// 创建4设备组成的raid0设备，16384000是4个设备总的Sector数量， 0 是每个设备的偏移量
$dmsetup create test-raid0 --table '0 16384000 striped 4 128 /dev/loop4 0 /dev/loop5 0 /dev/loop6 0 /dev/loop7 0'
```
查看结果
```
$lsblk
loop4         7:4    0     2G  0 loop  
└─test-raid0  253:23   0   7.8G  0 dm    
loop5         7:5    0     2G  0 loop  
└─test-raid0  53:23   0   7.8G  0 dm    
loop6         7:6    0     2G  0 loop  
└─test-raid0  253:23   0   7.8G  0 dm    
loop7         7:7    0     2G  0 loop  
└─test-raid0  253:23   0   7.8G  0 dm

$fdisk -l
...
Disk /dev/mapper/test-raid0: 8388 MB, 8388608000 bytes, 16384000 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 65536 bytes / 262144 bytes

$dmsetup status test-raid0
0 16384000 striped 4 7:4 7:5 7:6 7:7 1 AAAA
```

### RAID level 1 – Mirroring
![image](http://www.prepressure.com/images/raid-level-1-mirroring.svg)

RAID 1 数据存储两次，如果一个drive挂了，控制器会使用另外一个drive或者直接拷贝另一个drive的数据给它。

RAID 1的优缺点很明显，值得一提的的是，它不能保证热交换磁盘，也就是说，当一块盘坏了之后，需要将计算机停机，然后才能更换磁盘。

#### mdadm
```
$mdadm --create --verbose /dev/md0 --level=mirror --raid-devices=2 /dev/loop2 /dev/loop3
```
#### devicemapper
devicemapper中mirror与raid1是有差别的
```
// 4096000是单个设备的sector数量，因为mirror没有扩大容量，2是设备数量， - 表示没有metadata 设备
$dmsetup create test-raid1 --table '0 4096000 raid raid1 3 0 region_size 1024 2 - /dev/loop4 - /dev/loop5'
//下面的是有metadata设备的raid1创建
$dmsetup create test-raid1 --table '0 4096000 raid raid1 3 0 region_size 1024 2 /dev/loop4 /dev/loop5 /dev/loop6 /dev/loop7'

$dmsetup status test-raid1
0 4096000 raid raid1 2 AA 4096000/4096000 idle 0
```

### RAID level 5
最为普遍的RAID方式。要求至少3个drive，其中一个drive的一个磁盘作为奇偶校验盘，其他块做striping。所有奇偶校验盘会广泛分布在所有drive上。当有某个盘挂掉后，可以利用其他块以及奇偶校验盘来恢复数据，但如果同一个块中有两个设备挂掉，那么整个RAID就挂了。也就是说，RAID5能够支持单drive失败。

![image](http://www.prepressure.com/images/raid-level-5-striping-with-parity.svg)

RAID5的优点是兼顾了读写性能和安全性，能够支持单drive的失败情况。缺点在于实现比较复杂，恢复数据比较慢，如果在恢复的过程中其他drive也发生故障，那么整个RAID就挂了。

#### mdadm
```
// spare device指定备用磁盘，创建3块盘的RAID5
$mdadm --create --verbose /dev/md1 --level=5 --raid-devices=3 /dev/loop4 /dev/loop5 /dev/loop6 --spare-devices=1 /dev/loop7
```
查看信息
```
$mdadm --detail /dev/md1
/dev/md1:
        Version : 1.2
  Creation Time : Tue Mar  7 16:57:39 2017
     Raid Level : raid5
     Array Size : 4093952 (3.90 GiB 4.19 GB)
  Used Dev Size : 2046976 (1999.34 MiB 2096.10 MB)
   Raid Devices : 3
  Total Devices : 4
    Persistence : Superblock is persistent

    Update Time : Tue Mar  7 16:57:57 2017
          State : clean 
 Active Devices : 3
Working Devices : 4
 Failed Devices : 0
  Spare Devices : 1

         Layout : left-symmetric
     Chunk Size : 512K

           Name : st-integrat-node00:1  (local to host st-integrat-node00)
           UUID : 071d2e75:2029e9a2:2e427dd9:d2ab804f
         Events : 18

    Number   Major   Minor   RaidDevice State
       0       7        4        0      active sync   /dev/loop4
       1       7        5        1      active sync   /dev/loop5
       4       7        6        2      active sync   /dev/loop6

       3       7        7        -      spare   /dev/loop7
```

#### device-mapper
```
// 这里创建了没有metadata设备的4个设备的raid5，因此大小是3倍磁盘扇区大小，有一个作为奇偶校验。 
$dmsetup create test-raid5 --table '0 12288000 raid raid5_ls 3 64 region_size 1024 4 - /dev/loop4 - /dev/loop5 - /dev/loop6 - /dev/loop7'
// metadata设备的创建过程和前面RAID1的类似，这里略过。
//devicemapper还可以创建degraded RAID5,只需将最后一个设备置为 - 即可。
$dmsetup create test-raid5 --table '0 12288000 raid raid5_ls 3 64 region_size 1024 4 - /dev/loop4 - /dev/loop5 - - -
```

查看结果
```
// resync 表示正在同步， 4096000/4096000表示全部同步完成。
$dmsetup status test-raid5
0 12288000 raid raid5_ls 4 aaaa 2048064/4096000 resync 0
0 12288000 raid raid5_ls 4 AAAA 4096000/4096000 idle 0
//同步过程中也是可以删除设备的
```

### RAID level 6 – Striping with double parity
![image](http://www.prepressure.com/images/raid-level-6-striping-with-dual-parity.svg)

RAID6改进了RAID5，使用了两块奇偶校验盘，
RAID6的创建过程和RAID5相似.

### RAID level 10 – combining RAID 1 & RAID 0
RAID10可以看做是RAID1和RAID0的结合。

![image](http://www.prepressure.com/images/raid-level-1-and-0-striping-mirroring.svg)

RAID10的优点在于恢复数据速度很快，但相比与RAID5和6，使用设备的代价更大。

#### mdadm
```
$mdadm --create --verbose /dev/md1 --level=10 --raid-devices=4 /dev/loop4 /dev/loop5 /dev/loop6 /dev/loop7
mdadm: layout defaults to n2
mdadm: layout defaults to n2
mdadm: chunk size defaults to 512K
mdadm: size set to 2046976K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md1 started.
```
查看结果
```
$mdadm --detail /dev/md1
/dev/md1:
        Version : 1.2
  Creation Time : Tue Mar  7 17:44:43 2017
     Raid Level : raid10
     Array Size : 4093952 (3.90 GiB 4.19 GB)
  Used Dev Size : 2046976 (1999.34 MiB 2096.10 MB)
   Raid Devices : 4
  Total Devices : 4
    Persistence : Superblock is persistent

    Update Time : Tue Mar  7 17:45:04 2017
          State : clean 
 Active Devices : 4
Working Devices : 4
 Failed Devices : 0
  Spare Devices : 0

         Layout : near=2
     Chunk Size : 512K

           Name : st-integrat-node00:1  (local to host st-integrat-node00)
           UUID : 643b371a:eeadc82d:7d4effee:cf00411c
         Events : 17

    Number   Major   Minor   RaidDevice State
       0       7        4        0      active sync set-A   /dev/loop4
       1       7        5        1      active sync set-B   /dev/loop5
       2       7        6        2      active sync set-A   /dev/loop6
       3       7        7        3      active sync set-B   /dev/loop7
```

#### device-mapper
```
// 创建4个drive的RAID10，一半做镜像，所以大小为2倍drive sector大小。
$dmsetup create test-raid10 --table '0 8192000 raid raid10 3 64 region_size 1024 4 - /dev/loop4 - /dev/loop5 - /dev/loop6 - /dev/loop7'
```
查看结果
```
$dmsetup status test-raid10
0 8192000 raid raid10 4 AAAA 8192000/8192000 idle 0
```





### 参考文献
* [1] RAID, https://www.prepressure.com/library/technology/raid/
* [2] Device-mapper, https://wiki.gentoo.org/wiki/Device-mapper
* [3] RAID-Setup, https://raid.wiki.kernel.org/index.php/RAID_setup
* [4] dm-raid, https://www.kernel.org/doc/Documentation/device-mapper/dm-raid.txt
* [5] How To Create RAID Arrays with mdadm on Ubuntu 16.04, https://www.digitalocean.com/community/tutorials/how-to-create-raid-arrays-with-mdadm-on-ubuntu-16-04
