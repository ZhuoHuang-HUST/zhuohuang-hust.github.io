---
layout:     post
title:      " Linux存储设备的UUID"
date:       2017-04-09 10:51:00
author:     "ArKing"
categories: Linux管理
tags:
    - 磁盘与文件系统管理
---

* content
{:toc}

>“通用唯一识别码(Universally Unique identifier，简称UUID)；
其目的，是让分布式系统中的所有元素，都能有唯一的辨识信息，而不需要通过中央控制端来做辨识...”




## 存储设备的UUID

在Linux系统中，每个存储设备有一个对应的UUID，**通过UUID可以定位唯一的存储设备，
但是通过设备名可能定位到不同的存储设备**。

通常情况下，Linux的一个存储设备对应某块磁盘上的一个分区，比如/dev/sda1:

* 第一部分——“sda”表示磁盘名;
* 第二部分——“1”表示这块磁盘上的分区1;

### IDE接口的设备

IDE接口的**磁盘设备命名为hd[a-d][1-63]**。一台有2个IDE接口(一个IDE接口可接2个磁盘：主设备master，从设备slave)的主机上，第一部分命名规则：

| 接口 |    Master  |     Slave     |
| :-----: | :-----------: |  :-----------: |
| IDE1 | /dev/hda | /dev/hdb |
| IDE2 | /dev/hdc | /dev/hdd |

因此，当只有1块磁盘时，会根据磁盘接到的**物理位置**来决定磁盘标识。所以**通过设备名能定位到唯一的存储设备**。

### SATA/USB/SCSI接口的设备

SATA/USB/SCSI接口的磁盘使用SCSI模块来驱动，磁盘设备命名为/dev/sd[a-p][1-15]。但是，和IDE接口的磁盘设备不同，**第一部分的命名与磁盘的物理位置无关**，而是由**内核检测到的顺序决定**。

假设有一块磁盘A接到SATA接口2时，它的磁盘标识为sda。如果在SATA接口1插入一块磁盘B，磁盘A的标识会变成sdb，因此通过sda1就无法准确定位到正确的磁盘设备。

<br>
所以**要定位到准确的设备，就需要用到存储设备的UUID**。

## 查看存储设备的UUID

可以通过如下方式查看存储设备的UUID：

```bash
#方法一:
ls -l /dev/disk/by-uuid/
#方法二:
sudo blkid
```

在我的机器上结果如下：

```bash
chenximing@chenximing-MS-7823:~$ sudo blkid
/dev/sda1: UUID="b55cd2cf-4b3b-4d41-b37e-b8234ef52ee3" TYPE="ext4"
/dev/sda2: UUID="92b967d0-677d-4a3b-8340-ef2f65f6e793" TYPE="ext4"
/dev/sda3: UUID="1ed3c68a-6c7c-457a-afdd-57b82498e02e" TYPE="ext4"
/dev/sda4: UUID="a65c4955-f14b-4f3b-be3e-9833d3e72817" TYPE="swap"
/dev/sdb1: UUID="3458b96b-fe27-4289-928e-01e172b1e9da" TYPE="ext4"
/dev/sdb2: UUID="cb160486-f34f-41b4-871d-0b642778c583" TYPE="ext4"
/dev/sdb3: UUID="860e7c32-918c-4b5c-9bf4-abc949efae8b" TYPE="ext4"
```

## 参考资料

* *[维基百科](https://zh.wikipedia.org/wiki/%E9%80%9A%E7%94%A8%E5%94%AF%E4%B8%80%E8%AF%86%E5%88%AB%E7%A0%81)*
* *[Linux磁盘分区UUID的获取及其UUID的作用](http://www.cnblogs.com/xia/archive/2011/01/30/1947706.html)*