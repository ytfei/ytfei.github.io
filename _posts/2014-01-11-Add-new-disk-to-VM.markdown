---
layout: post
title: 为虚拟机新增磁盘
categories: [android, linux]
date: 2014-01-11 14:52:00 +0800
---

下面主要是关于虚拟磁盘添加到虚拟机之后，如何分区和格式化的过程。

## 磁盘分区

{% highlight bash %}
evans@master:/dev$ sudo fdisk -l

Disk /dev/sda: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders, total 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000b5cfd

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    81788927    40893440   83  Linux
/dev/sda2        81790974    83884031     1046529    5  Extended
/dev/sda5        81790976    83884031     1046528   82  Linux swap / Solaris

Disk /dev/sdb: 64.4 GB, 64424509440 bytes
255 heads, 63 sectors/track, 7832 cylinders, total 125829120 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000

Disk /dev/sdb doesn't contain a valid partition table

{% endhighlight %}


`/dev/sdb` 就是新增的磁盘，上面还没有任何分区信息，下面来着手分区。由于是第二块磁盘，且我的需求也比较简单，直接在上面建立一个分区就可以了。

{% highlight bash %}
evans@master:/dev$ sudo fdisk /dev/sdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0xf1dfadb5.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won't be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

# 查看已经的分区信息，现在还没有，所以下面的列表也是空的
Command (m for help): p

Disk /dev/sdb: 64.4 GB, 64424509440 bytes
255 heads, 63 sectors/track, 7832 cylinders, total 125829120 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xf1dfadb5

   Device Boot      Start         End      Blocks   Id  System

# 建立新分区
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended

# 先了一个主分区，开始和结束的扇区用的都是默认值，直接回车就好了
Select (default p): p
Partition number (1-4, default 1): 1
First sector (2048-125829119, default 2048): 
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-125829119, default 125829119): 
Using default value 125829119

# 分区建立好后，再次查看分区信息，列表中就多了一项 /dev/sdb1
Command (m for help): p

Disk /dev/sdb: 64.4 GB, 64424509440 bytes
255 heads, 63 sectors/track, 7832 cylinders, total 125829120 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xf1dfadb5

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   125829119    62913536   83  Linux

# 将分区信息写入磁盘。如果不做这一步就直接退出的话，以上操作将全部无效。
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.

{% endhighlight %}



完成分区操作后，我们再来看看磁盘上的变化：

{% highlight bash %}
evans@master:/dev$ sudo fdisk -l

Disk /dev/sda: 42.9 GB, 42949672960 bytes
255 heads, 63 sectors/track, 5221 cylinders, total 83886080 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x000b5cfd

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *        2048    81788927    40893440   83  Linux
/dev/sda2        81790974    83884031     1046529    5  Extended
/dev/sda5        81790976    83884031     1046528   82  Linux swap / Solaris

Disk /dev/sdb: 64.4 GB, 64424509440 bytes
128 heads, 39 sectors/track, 25206 cylinders, total 125829120 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0xf1dfadb5

   Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048   125829119    62913536   83  Linux

{% endhighlight %}

最后一行，表明分区信息已经写入成功。第一将使用 `fdisk -l` 时，显示的信息表示 `/dev/sdb` 上没有有效的信息，这次有了。分区完成后，接着就开始格式化磁盘。



## 磁盘格式化

查看已有磁盘的格式，为了保持一致，新磁盘也采用同样的格式

{% highlight bash %}
evans@master:/dev$ df -T
Filesystem     Type     1K-blocks     Used Available Use% Mounted on
/dev/sda1      ext4      40251776 30332612   7874492  80% /
udev           devtmpfs   2053336        4   2053332   1% /dev
tmpfs          tmpfs       824260      840    823420   1% /run
none           tmpfs         5120        0      5120   0% /run/lock
none           tmpfs      2060648     1412   2059236   1% /run/shm
{% endhighlight %}

已有磁盘 `/dev/sda1` 用的是 `ext4` ,新磁盘也采用同样的格式。

{% highlight bash %}
sudo mkfs -t ext4 /dev/sdb1
{% endhighlight %}

格式化完成后，磁盘就准备就绪了。但是这个磁盘必需“加载”后，才能够被使用。感觉就像是，在一个孤岛上建立一个房子，但人们却无法去使用。加载的过程，就如果是将孤岛与大陆连接起来。

## 加载磁盘

先临时加载磁盘，非常简单

{% highlight bash %}
sudo mkdir /android
sudo mount /dev/sdb1 /android
{% endhighlight %}

好了，加载完成。现在，在 `/android` 目录下建立的所有文件，都将被存储在新的磁盘上。但是一旦系统重启，你又得重新加载一次了。如果觉得麻烦，那么就往下看看怎么自动加载。

### 自动加载磁盘

自动加载是指在系统启动后，由操作系统来自动完成磁盘的加载。你的第一块磁盘 `/dev/sda1` 就是这样被加载的，安装操作系统时，这一步已经由安装程序顺便做了，现在我们自己来做。当然，过程也是简单的，只需要修改一下 `/etc/fstab` 。

查看磁盘的[UUID](https://help.ubuntu.com/community/UsingUUID)

{% highlight bash %}
evans@master:~$ sudo df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        39G   29G  7.6G  80% /
udev            2.0G   12K  2.0G   1% /dev
tmpfs           805M  844K  805M   1% /run
none            5.0M     0  5.0M   0% /run/lock
none            2.0G  1.4M  2.0G   1% /run/shm
/dev/sdb1        60G  180M   56G   1% /home/evans/codebase/android   ## 记下路径

evans@master:~$ sudo blkid
/dev/sda1: UUID="6314c61b-4ba2-43b6-933a-8575f69f93b5" TYPE="ext4" 
/dev/sda5: UUID="5609fe6e-3a96-4fa5-924e-be18161f439c" TYPE="swap" 
/dev/sdb1: UUID="842afa50-6a3c-4910-862d-c662c37bf081" TYPE="ext4"   ## 记下UUID

evans@master:~$ sudo vi /etc/fstab

  9 # / was on /dev/sda1 during installation
 10 UUID=6314c61b-4ba2-43b6-933a-8575f69f93b5 /               ext4    errors=remount-ro 0       1

# 将第10行复制然后再粘贴，再对其进行修改。
# 需要修改的内容有两处
# 一处是UUID，用上面的UUID进行替换
# 二是路径，将 / 替换为上面的自己的路径

  9 # / was on /dev/sda1 during installation
 10 UUID=6314c61b-4ba2-43b6-933a-8575f69f93b5 /               ext4    errors=remount-ro 0       1
 11 
 12 # /home/evans/codebase/android on /dev/sdb1 during installation
 13 UUID=842afa50-6a3c-4910-862d-c662c37bf081 /home/evans/codebase/android               ext4    errors=remount-ro 0       1

{% endhighlight %}
