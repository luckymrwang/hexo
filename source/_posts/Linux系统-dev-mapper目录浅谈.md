title: Linux系统/dev/mapper目录浅谈
date: 2023-11-22 15:29:59
tags: [Linux]
---

Linux 系统的一般的文件系统名称类似于 `/dev/sda1` 或 `/dev/hda1`，但是今天在进行系统维护的时候，利用 `df -h` 命令敲出了 `/dev/mapper/VolGroup-lv_root` 和 `/dev/mapper/Volume-lv_home` 两个文件系统名，不解，在网上查找资料后，在此浅谈 `/dev/mapper` 目录。

<!-- more -->
## 理解 Linux 系统的 Device mapper 机制

Device mapper 是 Linux2.6 内核中提供的一种从逻辑设备到物理设备的映射机制，在该机制下，用户能够很方便的根据自己的需要实现对存储资源的管理。在具体管理时需要用到 Linux 下的逻辑卷管理器，当前比较流行的逻辑卷管理器有 LVM2（Linux Volume Manager 2 version)、EVMS(Enterprise Volume Management System)、dmraid(Device Mapper Raid Tool) 等。

   想要详细了解 Device mapper 机制，可参考博文 [http://blog.sina.com.cn/s/blog_6237dcca0100hnwb.html](http://blog.sina.com.cn/s/blog_6237dcca0100hnwb.html) ，此处不再赘述。
   
## /dev/mapper 目录的解释
 
为了方便叙述，假设一台服务器有三块硬盘分别为a，b，c，每块硬盘的容量为 1T。在安装 Linux 的时候，先根据系统及自身的需要建立基本的分区，假设对硬盘 a 进行了分区，分出去了 0.1T 的空间挂载在 /boot 目录下，其他硬盘未进行分区。系统利用 Device mapper 机制建立了一个卷组（volume group，VG），可以把 VG 当做一个资源池来看待，最后在 VG 上面再创建逻辑卷（logical volume，LV）。若要将硬盘a的剩余空间、硬盘b和硬盘c都加入到 VG 中，则硬盘a的剩余空间首先会被系统建立为一个物理卷（physical volume，PV），并且这个物理卷的大小就是 0.9T，之后硬盘a的剩余的空间、硬盘b和硬盘c以 PV 的身份加入到 VG 这个资源池中，然后你需要多大的空间，就可以从VG中划出多大的空间（当然最大不能超过VG的容量）。比如此时池中的空间就是 2.9T，此时你就可以建立一个1T以上的空间出来，而不像以前最大的容量空间只能为 1T。

`/dev/mapper/Volume-lv_root` 的意思是说有一个 VG (volume group卷组) 叫作 Volume, 这个 Volume 里面有一个 LV 叫作 lv_root。其实这个 `/dev/mapper/Volume-lv_root` 文件是一个连接文件，是连接到 `/dev/dm-0` 的，可以用命令`ll /dev/mapper/Volume-lv_root` 进行查看。

其实在系统里 `/dev/Volume/lv_root` 和 `/dev/mapper/Volume-lv_root` 以及 `/dev/dm-0` 都是一个东西，都可当作一个分区来对待。

若要了解硬盘的具体情况，可通过 `fdisk` 或者 `pvdisplay` 命令进行查看。

若想要重装系统到 `/dev/sda` 下，且安装时有些东西不想被格式化想转移到 `/dev/sdb` 下，但此时 `/dev/sda` 和 `/dev/sdb` 被放到 VG 中了，那该如何解决该问题呢？这种情况下，由于此时根本没办法确定数据在哪一个硬盘上,因为这两个硬盘就如同加到池里，被 Device mapper 管理，所以解决方案就是再建个逻辑卷出来，把数据移到新的卷里，这样就可以重装系统时只删掉之前分区里的东西，而新的卷里的东西不动，就不会丢失了。

