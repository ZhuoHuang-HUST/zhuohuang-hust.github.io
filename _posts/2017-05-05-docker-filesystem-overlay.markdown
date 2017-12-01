---
layout:     post
title:      " Docker存储驱动—Overlay/Overlay2「译」"
date:       2017-05-05 2:57:00
author:     "ArKing"
categories: Docker
tags:
    - Docker存储驱动
---

* content
{:toc}

>该文为译文，代码为本地实际运行的结果，部分内容有修改或删除。实验环境为Ubuntu 14.04；




[原文链接](https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/#container-reads-and-writes-with-overlay)

OverlayFS是和AUFS相似的联合文件系统(union filesystem)，它有如下特点：
* *设计简洁；*
* *内核3.18开始已经并入内核主线*
* *可能更快*

因此，它在社区迅速获得广泛关注并被很多人认为是AUFS的继承。但是它任然很年轻，
因此在生产环境中使用时要谨慎；

docker的overlay存储驱动利OverlayFS的一些特征来构建以及管理镜像和容器的磁盘结构。

docker1.12后推出的overlay2在inode的利用方面比ovelay更有效，**overlay2要求内核版本大于等于4.0**。

## overlay

如下图所示，Overlay在主机上用到2个目录，这2个目录被看成是overlay的层。
upperdir为容器层、lowerdir为镜像层使用联合挂载技术将它们挂载在同一目录(merged)下，提供统一视图。

![](/img/in-post/post-docker-filesystem/overlay_constructs.jpg)

*[图片来源: Docker文档][i1]*

当容器层和镜像层拥有相同的文件时，容器层的文件可见，隐藏了镜像层相同的文件。
容器挂载目录(merged)提供了统一视图。

overlay只使用2层，意味着多层镜像不会被实现为多个OverlayFS层。每个镜像被实现为自己的目录，
这个目录在路径/var/lib/docker/overlay下。**硬链接**被用来索引和低层共享的数据，节省了空间。

当创建一个容器时，overlay驱动连接代表镜像层顶层的目录(只读)和一个代表容器层的新目录(读写)。

### 例子：镜像和容器的磁盘结构[overlay]

```shell
root@chenximing-MS-7823:/home/chenximing# tree /var/lib/docker/overlay/

/var/lib/docker/overlay/
```

一开始overlay路径下为空，用docker pull命令下载一个由5层镜像层组成的docker镜像:

```shell
chenximing@chenximing-MS-7823:~$ docker pull ubuntu

Using default tag: latest
latest: Pulling from library/ubuntu
c62795f78da9: Pull complete 
d4fceeeb758e: Pull complete 
5c9125a401ae: Pull complete 
0062f774e994: Pull complete 
6b33fd031fac: Pull complete 
Digest: sha256:c2bbf50d276508d73dd865cda7b4ee9b5243f2648647d21e3a471dd3cc4209a0
Status: Downloaded newer image for ubuntu:latests
```

此时，在路径/var/lib/docker/overlay/下，每个镜像层都有一个对应的目录，包含了该层镜像的内容，通过tree命令发现，每个镜像层下只包含一个root目录。
(因为docker1.10开始，使用基于内容的寻址，因此目录名和镜像层的id不一致)

```shell
root@chenximing-MS-7823:/home/chenximing# tree -L 2 /var/lib/docker/overlay/

/var/lib/docker/overlay/
├── 3279a41fd358cdda798d99cc2da0425b4836a489083ae9db4aedc834f426915a
│   └── root
├── 33629fe3999e692ecbeb340f855a2d05ddf4e173ea915041be217e1c9cc8a48f
│   └── root
├── 6ab876fcc7ad584dd1eebae0d9304abb22561012f6adb87828f57906d799c33b
│   └── root
├── 77806a4b8257cd5508a1131d4d47c2d4b4d51703a1cc0dd8daddf0e86a68d492
│   └── root
└── ed271f2159e3c9de18ede980e4e2ecb0ef39b96927d446ba93deab0b6ff1a8e3
    └── root
```

每一层都包含了"该层独有的文件"以及"和其低层共享的数据的硬连接"。如ls命令，每一层都使用一个硬链接来指向实际ls命令的inode号(405241)：

```shell
root@chenximing-MS-7823:/home/chenximing# ls -i /var/lib/docker/overlay/3279a41fd358cdda798d99cc2da0425b4836a489083ae9db4aedc834f426915a/root/bin/ls
    
405241 /var/lib/docker/overlay/3279a41fd358cdda798d99cc2da0425b4836a489083ae9db4aedc834f426915a/root/bin/ls
    
root@chenximing-MS-7823:/home/chenximing# ls -i /var/lib/docker/overlay/33629fe3999e692ecbeb340f855a2d05ddf4e173ea915041be217e1c9cc8a48f/root/bin/ls
    
405241 /var/lib/docker/overlay/33629fe3999e692ecbeb340f855a2d05ddf4e173ea915041be217e1c9cc8a48f/root/bin/ls
```

使用刚刚拉取的ubuntu镜像创建一个容器，用docker ps命令查看：
    
```shell
root@chenximing-MS-7823:/home/chenximing# docker ps
    
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
8963ffb422fa        ubuntu              "/bin/bash"         32 minutes ago      Up 52 seconds                           container_1
```

创建容器时，实际上是在已有的镜像层上创建了一层容器层，容器层在路径/var/lib/docker/overlay下也存在对应的目录：

```shell
root@chenximing-MS-7823:/home/chenximing# ls /var/lib/docker/overlay/
    
3279a41fd358cdda798d99cc2da0425b4836a489083ae9db4aedc834f426915a
33629fe3999e692ecbeb340f855a2d05ddf4e173ea915041be217e1c9cc8a48f
6ab876fcc7ad584dd1eebae0d9304abb22561012f6adb87828f57906d799c33b
6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d       //创建容器后新增的目录
6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d-init  //创建容器后新增的目录
77806a4b8257cd5508a1131d4d47c2d4b4d51703a1cc0dd8daddf0e86a68d492
ed271f2159e3c9de18ede980e4e2ecb0ef39b96927d446ba93deab0b6ff1a8e3
```

“6abc47......”和“6abc47.....-init”为创建容器后**新增的目录**。查看这两个目录的内容：

```shell
root@chenximing-MS-7823:/home/chenximing# tree -L 3 /var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d*/
   
/var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d/
├── lower-id
├── merged
├── upper
│   ├── dev
│   │   └── console
│   ├── etc
│   │   ├── hostname
│   │   ├── hosts
│   │   ├── mtab -> /proc/mounts
│   │   └── resolv.conf
│   └── root
└── work
    └── work
    
/var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d-init/
├── lower-id
├── merged
├── upper
│   ├── dev
│   │   └── console
│   └── etc
│       ├── hostname
│       ├── hosts
│       ├── mtab -> /proc/mounts
│       └── resolv.conf
└── work
    └── work
```

“6abc47......”为**读写层**，“6abc47.....-init”为**初始层**。
初始层中大多是初始化容器环境时，与容器相关的环境信息，
如容器主机名，主机host信息以及域名服务文件等。**所有对容器做出的改变都记录在读写层**。

文件lower-id用来索引该容器使用的镜像层，upper目录包含了容器层的内容，每当启动一个容器时，会将lower-id指向的镜像层目录以及upper目录联合挂载到merged目录，因此，容器内的视角就是merged目录下的内容。而work目录则是用来完成如copy-on_write的操作。
看看容器使用到了哪一层镜像层：

```bash
root@chenximing-MS-7823:/home/chenximing# cat /var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d/lower-id
    
ed271f2159e3c9de18ede980e4e2ecb0ef39b96927d446ba93deab0b6ff1a8e3
    
root@chenximing-MS-7823:/home/chenximing# cat /var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d-init/lower-id
    
ed271f2159e3c9de18ede980e4e2ecb0ef39b96927d446ba93deab0b6ff1a8e3
```

在刚才创建的容器中创建一个文件：

```bash
root@8963ffb422fa:/# touch file
    
root@8963ffb422fa:/# echo "hello world" > file
    
root@8963ffb422fa:/# ls
bin   dev  file  lib    media  opt   root  sbin  sys  usr
boot  etc  home  lib64  mnt    proc  run   srv   tmp  var
```

此时再观察“6abc47......”目录(读写层)：

```shell
root@chenximing-MS-7823:/home/chenximing# tree -L 3 /var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d/  
    
/var/lib/docker/overlay/6abc47fcf668b6820a2a9a8b0d0c6150f9df61ab5a323783630fac3fd282144d/
├── lower-id
├── merged
├── upper
│   ├── dev
│   │   └── console
│   ├── etc
│   │   ├── hostname
│   │   ├── hosts
│   │   ├── mtab -> /proc/mounts
│   │   └── resolv.conf
│   ├── file
│   └── root
└── work
    └── work
```

发现upper目录下多出了一个file文件，就是刚才在容器中创建的文件。

## overlay2 

和overlay为了实现“两个目录反映多层镜像“而使用硬链接不同，overlay2驱动天生支持多层。(最多128)

因此，overlay2在使用docker层相关的命令时，能提供更好的性能(如：docker build、docker commit)。而且overlay2消耗的inode节点更少。

### 例子：镜像和容器的基于磁盘的结构[overlay2]

docker pull ubuntu拉取完一个5层的Ubuntu镜像后，/var/lib/docker/overlay2下可以看到6个目录:

```shell
root@chenximing-MS-7823:/home/chenximing# tree -L 2 /var/lib/docker/overlay2
    
/var/lib/docker/overlay2
├── 1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a
│   ├── diff
│   └── link   //文件
├── 28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6
│   ├── diff
│   ├── link   //文件
│   ├── lower   //文件
│   ├── merged
│   └── work
├── 299397034eb96f22d544bf544c9ef993fa32571f466bd8a881aec0fc5a94a8df
│   ├── diff
│   ├── link   //文件
│   ├── lower   //文件
│   ├── merged
│   └── work
├── 8c0de7df4581df4787b08a28356cc8d7ba7b420c70dc36cc0615afdafb6ee15a
│   ├── diff
│   ├── link   //文件
│   ├── lower   //文件
│   ├── merged
│   └── work
├── f21a47541026214e519fccdf8b838ac2c4e11a1a2dd6a5febc061381a1972ad7
│   ├── diff
│   ├── link   //文件
│   ├── lower   //文件
│   ├── merged
│   └── work
└── l
    ├── 5DJFQNGXSA5CVOC6NA6HPUCXXB -> ../299397034eb96f22d544bf544c9ef993fa32571f466bd8a881aec0fc5a94a8df/diff
    ├── EGQNS3O24ONBL5BJSGNTX4NP6E -> ../1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a/diff
    ├── JWB63ZJDZOTK5N22OMR5BKUVMG -> ../8c0de7df4581df4787b08a28356cc8d7ba7b420c70dc36cc0615afdafb6ee15a/diff
    ├── UZHRLLTPQMODQQYBLN6NOT6N2K -> ../28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/diff
    └── X76VPZDDKIW3GLWBFUHKDFBEPE -> ../f21a47541026214e519fccdf8b838ac2c4e11a1a2dd6a5febc061381a1972ad7/diff
    
24 directories, 9 files
```

”l“目录包含一些符号链接作为缩短的层标识符. 这些缩短的标识符用来避免挂载时超出页面大小的限制。

```shell
root@chenximing-MS-7823:/home/chenximing# ls -l /var/lib/docker/overlay2/l/
    
总用量 20
lrwxrwxrwx 1 root root 72  4月 20 17:31 5DJFQNGXSA5CVOC6NA6HPUCXXB -> ../299397034eb96f22d544bf544c9ef993fa32571f466bd8a881aec0fc5a94a8df/diff
lrwxrwxrwx 1 root root 72  4月 20 17:31 EGQNS3O24ONBL5BJSGNTX4NP6E -> ../1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a/diff
lrwxrwxrwx 1 root root 72  4月 20 17:31 JWB63ZJDZOTK5N22OMR5BKUVMG -> ../8c0de7df4581df4787b08a28356cc8d7ba7b420c70dc36cc0615afdafb6ee15a/diff
lrwxrwxrwx 1 root root 72  4月 20 17:31 UZHRLLTPQMODQQYBLN6NOT6N2K -> ../28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/diff
lrwxrwxrwx 1 root root 72  4月 20 17:31 X76VPZDDKIW3GLWBFUHKDFBEPE -> ../f21a47541026214e519fccdf8b838ac2c4e11a1a2dd6a5febc061381a1972ad7/diff
```

最底层包含"link"文件(不包含lower文件，因为是最底层)，在上面的结果中“1d8e....”为最底层。
这个文件记录着作为标识符的更短的符号链接的名字、最底层还有一个"diff"目录(包含实际内容)。

```shell
root@chenximing-MS-7823:/home/chenximing# cat /var/lib/docker/overlay2/1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a/link 
    
EGQNS3O24ONBL5BJSGNTX4NP6E
    
root@chenximing-MS-7823:/home/chenximing# ls /var/lib/docker/overlay2/1d18eac18e483e5af46d52673e254cb89996041d91356c6dd1f0ac1518ba130a/diff/
    
bin   dev  home  lib64  mnt  proc  run   srv  tmp  var
boot  etc  lib   media  opt  root  sbin  sys  usr
```

从第二层开始，每层镜像层包含”lower“文件，根据这个文件可以索引构建出整个镜像的层次结构。
”diff“文件(层的内容)。还包含”merged“和"work"目录，用途和overlay一样。

```shell
root@chenximing-MS-7823:/home/chenximing# cat /var/lib/docker/overlay2/28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/link 
    
UZHRLLTPQMODQQYBLN6NOT6N2K
    
root@chenximing-MS-7823:/home/chenximing# ls /var/lib/docker/overlay2/28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/diff/
    
etc  sbin  usr  var
    
root@chenximing-MS-7823:/home/chenximing# cat /var/lib/docker/overlay2/28702e0cca9ff9a3245ce7e61326d098788e855ea599bf160465f537756a56a6/lower
    
l/EGQNS3O24ONBL5BJSGNTX4NP6E
```

再来看看容器层，容器层的文件构成和镜像层类似(这点和overlay不同)，使用刚刚拉取的ubuntu镜像创建一个容器，/var/lib/docker/overlay2目录下新增2个目录：

```shell
├── 2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19
│   ├── diff
│   ├── link
│   ├── lower
│   ├── merged
│   └── work
├── 2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19-init
│   ├── diff
│   ├── link
│   ├── lower
│   ├── merged
│   └── work
└── l
    ├── 7H7MUMX4IOAVLIJ2YPLR5MZOO5 -> ../2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19-init/diff
    ├── EBSBKJBLA2WYBHTMHKSMCEEWNP -> ../2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19/diff

    
root@chenximing-MS-7823:/home/chenximing# cat /var/lib/docker/overlay2/2558ed8dd7051c1e78ea980590a228100ec3d299c01b35e9b7fa503723593d19/lower 
    
l/7H7MUMX4IOAVLIJ2YPLR5MZOO5:l/5DJFQNGXSA5CVOC6NA6HPUCXXB:l/JWB63ZJDZOTK5N22OMR5BKUVMG:l/X76VPZDDKIW3GLWBFUHKDFBEPE:l/UZHRLLTPQMODQQYBLN6NOT6N2K:l/EGQNS3O24ONBL5BJSGNTX4NP6E
```

## 容器的读写

对于读，考虑下列3种场景：

* **读的文件不在容器层**：如果读的文件不在容器层，则从镜像层进行读
* **读的文件只存在在容器层**：直接从容器层读
* **读的文件在容器层和镜像层**：读容器层中的文件，因为容器层隐藏了镜像层同名的文件

对于写，考虑下列场景：

* **写的文件不在容器层，在镜像层**：由于文件不在容器层，因此overlay/overlay2存储驱动使用copy_up操作从镜像层拷贝文件到容器层，然后将写入的内容写入到文件新的拷贝中。
* **删除文件和目录**：删除镜像层的文件，会在容器层创建一个whiteout文件来隐藏它；删除镜像层的目录，会创建opaque目录，它和whiteout文件有相同的效果
* **重命名目录**：对一个目录调用rename(2)仅仅在资源和目的地路径都在顶层时才被允许，否则返回EXDEV。

## 配置Docker使用overlay/overlay2存储驱动

配置overlay要求内核版本大于等于3.18，且加载了overlay内核模块；  
配置overlay2要求内核版本大于等于4.0；  
docker daemon处于stopped状态；

OverlayFS能在大多数Linux文件系统上运行，但是**ext4是目前在生产环境中被推荐的**。

**1) stop Docker daemon**  

```bash
$ sudo service docker stop
```

**2) 确认内核版本以及overlay内核模块已经加载;**

```bash
#查看内核版本
$ uname -r  

#查看内核是否已加载overlay模块 
$ lsmod | grep overlay  
overlay
```
如果未加载overlay模块，假设你的内核已经编译了overlay模块，则可以通过下列命令进行加载，或者在启动docker服务时，会自动加载overlay模块：

```bash
$ modprobe overlay
```

**3) 启动docker daemon（使用overlay/overlay2存储驱动）**

**方法一(推荐)**：
在/etc/docker/路径下创建daemon.json文件，并添加下列内容：

```bash
{
  "storage-driver": "overlay2"
}
```
然后启动docker服务:

```bash
$ sudo service docker start
```

**方法二**:

```shell
$ dockerd --storage-driver=overlay &  

[1] 29403  
root@ip-10-0-0-174:/home/ubuntu# INFO[0000] Listening for HTTP on unix (/var/run/docker.sock)  
INFO[0000] Option DefaultDriver: bridge  
INFO[0000] Option DefaultNetwork: bridge
```

**4) 核实daemon已经在使用overlay/overlay2**

```bash
$ docker info
    
Containers: 0  
Images: 0  
Storage Driver: overlay  
Backing Filesystem: extfs
...
```

## OverlayFS and Docker 性能

一般来说，overlay/overlay2很快，几乎肯定比aufs和devicemapper快。在某些特定场景下，还可能比btrfs快。
此外，还有几点overlay/overlay2驱动性能相关的注意事项：

* **页缓存**：OverlayFS支持页缓存共享，意味着多个容器访问相同的文件能够共享一个单一的page cache entry。
使得overlay/overlay2驱动能高效使用内存，是PaaS以及其它高密度场景一个好的选择。

* **copy_up**：对镜像层大文件进行写操作时，copy-on-write会给写操作带来大量延迟。

* **inode 限制**：使用overlay会引起过度的inode消耗，消耗会随着主机上的镜像和容器的增加而增加。
拥有大量镜像的主机在大量容器启动和停止时可能会耗尽inodes。
不幸的是你只能在文件系统创建时指定inode数，因此你可能需要考虑将/var/lib/docker放在另一个独立的设备上，
或者在创建文件系统时手动修改inode值。而overlay2则没有这样的问题。

* **RPM和Yum**：OverlayFS仅实现了POSIX标准的一部分，某些操作还会违反POSIX标准，copy_up操作就是其中一个。

下面是提升OverlayFS驱动性能的最佳实践.

* **SSD**：为了获得最佳性能，一个通常的想法是使用诸如SSD这类更快的存储设备；

* **使用数据卷**： 数据卷提供了最好的以及最可预见的性能。
因为绕过了存储驱动，因此不会存在瘦供给和copy-on-write带来的潜在性能开销。
因此，写操作较频繁的数据应该放在数据卷上。

[i1]: https://docs.docker.com/engine/userguide/storagedriver/images/overlay_constructs.jpg