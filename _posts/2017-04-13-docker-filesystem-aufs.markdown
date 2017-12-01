---
layout:     post
title:      " Docker存储驱动—Aufs "
date:       2017-04-13 10:51:00
author:     "ArKing"
categories: Docker
tags:
    - Docker存储驱动
---

* content
{:toc}

> docker支持的存储驱动有aufs、overlayFS、devicemapper、btrfs、zfs和vfs;
> 
> 这篇文章主要谈谈aufs；




## 使用Aufs进行联合挂载

AUFS 是一种 Union File System（联合文件系统），它可以把不同位置的目录合并mount到同一目录下。其中，最上层目录**读写**(r/w)，低层目录都为**只读**(read only)。

创建两个目录upper/以及lower/，一个联合挂载目录mnt/，在目录下创建文件，其结构如下：

```bash
chenximing@chenximing-MS-7823:~/aufs$ tree upper/ lower/ mnt/
upper/
├── file
└── file1
lower/
├── file
└── file2
mnt/
```

文件的内容如下:

```bash
chenximing@chenximing-MS-7823:~/aufs$ cat upper/file upper/file1
file in upper
file1 in upper
chenximing@chenximing-MS-7823:~/aufs$ cat lower/file lower/file2
file in lower
file2 in lower
```

通过aufs将upper/和lower/联合挂载到mnt目录下。**upper/属于读写层**，**lower/属于只读层**。aufs根据权限(如果指定了)首先mount lower/，然后mount upper/：

```bash
chenximing@chenximing-MS-7823:~/aufs$ sudo mount -t aufs -o dirs=upper/:lower/ none mnt/
chenximing@chenximing-MS-7823:~/aufs$ tree mnt
mnt
├── file
├── file1
└── file2

0 directories, 3 files
```

整个的结构看起来像下面这样：

<img src="/img/in-post/post-docker-filesystem/aufs-struct.png" style="zoom:150%" align="center"/>

图中可以反映出，联合挂载使用了一种**覆盖/合并**的机制：

* 相同文件，上层会覆盖下层的文件；
* 相同目录，会进行合并。(在上面例子中添加文件upper/dir/f1和lower/dir/f2，则联合挂载后会有一个目录dir包含f1和f2)；

所以说upper/file会覆盖lower/file，而file1则是upper/file1，file2则是lower/file2：

```bash
chenximing@chenximing-MS-7823:~/aufs$ cat mnt/file mnt/file1 mnt/file2
file in upper
file1 in upper
file2 in lower
```

### 文件读

读一个文件时，会**由上往下**进行查找，如果读的文件在upper中，则直接读取upper层中的文件。否则，继续向下层查找，直到找到或者查找完所有层为止。因此，如果使用的低层越多，如果文件存在最底层，则会带来一定的查找开销。

由于aaa/为**顶层读写层**，因此对file1和file2进行修改将会**直接修改aaa/中file1和file2内容**。不会影响bbb/中的内容。

### 文件写

首先会进行查找，对与找到的文件可能触发不同的写行为:

* 如果是一个读写层的文件：由于文件权限为读写，因此可以修改，直接对读写层的文件进行写；
* 如果是一个只读层的文件：由于文件权限为只读，所以无法修改，因此会触发copy-on-write，将文件拷贝至读写层后，对拷贝后的文件写；

所以如果修改file或file1，会直接对upper/file或upper/file1进行修改。修改file2，会先将lower/file拷贝至upper/目录，然后修改：

```bash
#1.对读写层文件写
chenximing@chenximing-MS-7823:~/aufs$ echo modify file1 > mnt/file1
chenximing@chenximing-MS-7823:~/aufs$ tree upper/ lower/
upper/
├── file
├── file1 #直接写file1
lower/
├── file
└── file2

0 directories, 4 files
chenximing@chenximing-MS-7823:~/aufs$ cat mnt/file1 upper/file1
modify file1
modify file1

#2.对只读层文件写
chenximing@chenximing-MS-7823:~/aufs$ echo modify file2 > mnt/file2
chenximing@chenximing-MS-7823:~/aufs$ tree upper/ lower/
upper/
├── file
├── file1
└── file2 #对lower/file2执行copy-on-write得到，然后写
lower/
├── file
└── file2

0 directories, 5 files
chenximing@chenximing-MS-7823:~/aufs$ cat mnt/file2 upper/file2 lower/file2
modify file2
modify file2
file2 in lower #并没有改变只读层的文件
```

### 删除文件

删除文件也会根据查找结果，触发不同的删除行为：

* 如果文件**只存在读写层**：由于只读层不存在同名文件，因此直接删除读写层中的文件；
* 如果文件**只存在只读层**：由于只读层文件无法修改，所以在upper层创建一个whiteout文件隐藏只读层中的文件；
* 如果文件**同时存在读写层和只读层**：首先删除读写层中的文件，然后创建whiteout文件隐藏只读层中的同名文件；

```bash
#1.文件只存在读写层
chenximing@chenximing-MS-7823:~/aufs$ rm mnt/file1
chenximing@chenximing-MS-7823:~/aufs$ tree upper/ lower/ mnt/
upper/ #upper中的file1被删除
└── file
└── file2
lower/
├── file
└── file2
mnt/ #联合挂载目录中看到file1已经被删除了
└── file
└── file2

0 directories, 6 files

#2.文件只存在只读层
chenximing@chenximing-MS-7823:~/aufs$ rm mnt/file2
chenximing@chenximing-MS-7823:~/aufs$ tree upper/ lower/ mnt/
upper/
└── file
lower/
├── file
└── file2 #只读层中的文件并未被删除
mnt/ #联合挂载目录中看到file2已经被删除了
└── file

0 directories, 4 files
#upper下的.wh.file2隐藏了lower/file2，因此mnt中对file2不可见
chenximing@chenximing-MS-7823:~/aufs$ ls -a upper/
.  ..  file  .wh.file2  .wh..wh.aufs  .wh..wh.orph  .wh..wh.plnk

#3.文件同时存在读写层和只读层
chenximing@chenximing-MS-7823:~/aufs$ rm mnt/file
chenximing@chenximing-MS-7823:~/aufs$ tree upper/ lower/ mnt/
upper/ #只读层的file被删除
lower/ 
├── file #读写层的file并未被删除
└── file2
mnt/ #联合挂载目录中看到file已经被删除了

0 directories, 2 files
chenximing@chenximing-MS-7823:~/aufs$ ls -a upper/
.  ..  .wh.file  .wh.file2  .wh..wh.aufs  .wh..wh.orph  .wh..wh.plnk
```

## Docker中的Aufs存储驱动

![](/img/in-post/post-docker-filesystem/aufs-layers.jpg)

*[图片来源: Docker文档][i1]*

上图是一个docker容器的层次化结构，docker容器由镜像生成，镜像由多层的**只读的镜像层**组成。每run一个docker容器时，会根据使用的镜像，在只读层上创建一层**读写层(容器层)**。

容器和镜像存储在/var/lib/docker/aufs/目录下，aufs目录包含了三个目录：diff、layers、mnt。

**镜像和容器层存储在diff目录下，联合挂载目录则是在mnt下**：

```bash
chenximing@chenximing-MS-7823:~/aufs$ sudo ls /var/lib/docker/aufs
diff  layers  mnt

#拉取镜像前diff目录为空，因为没有镜像层和容器层
chenximing@chenximing-MS-7823:~/aufs$ sudo ls /var/lib/docker/aufs/diff
```

### 镜像层

上面提到，镜像层在diff目录中，使用```docker pull ubuntu```命令拉取一个ubuntu镜像后再观察这个目录：

```bash
#包含5层镜像层
chenximing@chenximing-MS-7823:~/aufs$ sudo ls /var/lib/docker/aufs/diff
2fd1123336697985b55ecc4f58d471ed9c9a8d64d09b54df90d6882ef2a85f2a
6c8288af68cf7fa75239560127afbeccb5dae236173c7807a9970f91839c9269
78e6c0be52f62583e8c9b26f90548ad5b4629e0fa84d6e9430b4bc742d2cfc4e
df229100e0eced9e2fbc4dbdeb55e6ed602d917fc27aa14a29c5a11bdc1c27ba
ebc2d2b2d9ee5bcaf02263d24ca0cd016e0a812c76cad7921a49b75aa604d513

#每层镜像的内容
chenximing@chenximing-MS-7823:~/aufs$ sudo tree -L 2 /var/lib/docker/aufs/diff
/var/lib/docker/aufs/diff
├── 2fd1123336697985b55ecc4f58d471ed9c9a8d64d09b54df90d6882ef2a85f2a
│   ├── bin
│   ├── boot
│   ├── dev
│   ├── etc
│   ├── home
│   ├── lib
│   ├── lib64
│   ├── media
│   ├── mnt
│   ├── opt
│   ├── proc
│   ├── root
│   ├── run
│   ├── sbin
│   ├── srv
│   ├── sys
│   ├── tmp
│   ├── usr
│   └── var
├── 6c8288af68cf7fa75239560127afbeccb5dae236173c7807a9970f91839c9269
│   ├── etc
│   ├── sbin
│   ├── usr
│   └── var
├── 78e6c0be52f62583e8c9b26f90548ad5b4629e0fa84d6e9430b4bc742d2cfc4e
│   └── etc
├── df229100e0eced9e2fbc4dbdeb55e6ed602d917fc27aa14a29c5a11bdc1c27ba
│   └── var
└── ebc2d2b2d9ee5bcaf02263d24ca0cd016e0a812c76cad7921a49b75aa604d513
    └── run

31 directories, 0 files
```

在拉取一个ubuntu镜像后，diff中创建了5个目录，对于ubuntu镜像的5层镜像层，上面我们可以看到每层的内容。再看看联合挂载目录mnt：

```bash
chenximing@chenximing-MS-7823:~/aufs$ sudo tree -L 2 /var/lib/docker/aufs/mnt
/var/lib/docker/aufs/mnt
├── 2fd1123336697985b55ecc4f58d471ed9c9a8d64d09b54df90d6882ef2a85f2a
├── 6c8288af68cf7fa75239560127afbeccb5dae236173c7807a9970f91839c9269
├── 78e6c0be52f62583e8c9b26f90548ad5b4629e0fa84d6e9430b4bc742d2cfc4e
├── df229100e0eced9e2fbc4dbdeb55e6ed602d917fc27aa14a29c5a11bdc1c27ba
└── ebc2d2b2d9ee5bcaf02263d24ca0cd016e0a812c76cad7921a49b75aa604d513

5 directories, 0 files
```

### 容器层

使用命令```docker create -it --name ct ubuntu /bin/bash```创建一个容器，看看有什么变化：

```bash
root@chenximing-MS-7823:~/aufs$ ls /var/lib/docker/aufs/diff
2fd1123336697985b55ecc4f58d471ed9c9a8d64d09b54df90d6882ef2a85f2a
6c8288af68cf7fa75239560127afbeccb5dae236173c7807a9970f91839c9269
78e6c0be52f62583e8c9b26f90548ad5b4629e0fa84d6e9430b4bc742d2cfc4e
b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2
b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2-init
df229100e0eced9e2fbc4dbdeb55e6ed602d917fc27aa14a29c5a11bdc1c27ba
ebc2d2b2d9ee5bcaf02263d24ca0cd016e0a812c76cad7921a49b75aa604d513

root@chenximing-MS-7823:/home/chenximing/aufs$ tree -L 2 /var/lib/docker/aufs/diff/b45*
/var/lib/docker/aufs/diff/b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2
/var/lib/docker/aufs/diff/b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2-init
├── dev
│   └── console
└── etc
    ├── hostname
    ├── hosts
    ├── mtab -> /proc/mounts
    └── resolv.conf

2 directories, 5 files
```

从上面可以看出，**创建一个容器，实际上就是创建了一层读写层(容器层)**,也就是上面的“b45...9e2”目录。

初始化层“b5..9e2-init”用来初始化容器环境，由于hostname这些信息每个容器独有，而容器层可能会被提交成镜像，从而成为一个镜像层，所以这些文件不适合放在容器层中。

mnt目录下也创建了对应的联合挂载目录，目录同样为空：

```bash
chenximing@chenximing-MS-7823:~/aufs$ sudo tree -L 2 /var/lib/docker/aufs/mnt
/var/lib/docker/aufs/mnt
├── 2fd1123336697985b55ecc4f58d471ed9c9a8d64d09b54df90d6882ef2a85f2a
├── 6c8288af68cf7fa75239560127afbeccb5dae236173c7807a9970f91839c9269
├── 78e6c0be52f62583e8c9b26f90548ad5b4629e0fa84d6e9430b4bc742d2cfc4e
├── b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2 
├── b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2-init
├── df229100e0eced9e2fbc4dbdeb55e6ed602d917fc27aa14a29c5a11bdc1c27ba
└── ebc2d2b2d9ee5bcaf02263d24ca0cd016e0a812c76cad7921a49b75aa604d513
```

### 启动容器

为什么联合挂载目录在拉取一个镜像和创建一个容器后都是空？原因在于容器并没有启动，因此何来联合挂载。**容器启动就是将diff目录下的镜像层与容器层联合挂载到mnt目录下**，使用```docker start ct```启动容器，再次观察mnt目录：

```bash
root@chenximing-MS-7823:/home/chenximing/aufs$ tree -L 2 /var/lib/docker/aufs/mnt
/var/lib/docker/aufs/mnt
├── 2fd1123336697985b55ecc4f58d471ed9c9a8d64d09b54df90d6882ef2a85f2a
├── 6c8288af68cf7fa75239560127afbeccb5dae236173c7807a9970f91839c9269
├── 78e6c0be52f62583e8c9b26f90548ad5b4629e0fa84d6e9430b4bc742d2cfc4e
├── b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2
│   ├── bin
│   ├── boot
│   ├── dev
│   ├── etc
│   ├── home
│   ├── lib
│   ├── lib64
│   ├── media
│   ├── mnt
│   ├── opt
│   ├── proc
│   ├── root
│   ├── run
│   ├── sbin
│   ├── srv
│   ├── sys
│   ├── tmp
│   ├── usr
│   └── var
├── b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2-init
├── df229100e0eced9e2fbc4dbdeb55e6ed602d917fc27aa14a29c5a11bdc1c27ba
└── ebc2d2b2d9ee5bcaf02263d24ca0cd016e0a812c76cad7921a49b75aa604d513

26 directories, 0 files
```

容器层对应的联合挂载目录此时以及不为空了，目录下的内容就是我们在容器内使用```ls /```看到的文件。

### 查看容器/镜像层次关系

现在我们知道一个镜像通常对应多次镜像层，如果你想知道一个容器的镜像层和容器层的上下关系，可以通过下面的方法查看：

```bash
root@chenximing-MS-7823:/var/lib/docker/aufs$ cat /sys/fs/aufs/si_5124fe8368e572f5/br[0-6]*
/var/lib/docker/aufs/diff/b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2=rw
/var/lib/docker/aufs/diff/b451c296ac5ac40e64a9bff28cf1409ea2b47c41be05e87430d46d3958f179e2-init=ro+wh
/var/lib/docker/aufs/diff/ebc2d2b2d9ee5bcaf02263d24ca0cd016e0a812c76cad7921a49b75aa604d513=ro+wh
/var/lib/docker/aufs/diff/78e6c0be52f62583e8c9b26f90548ad5b4629e0fa84d6e9430b4bc742d2cfc4e=ro+wh
/var/lib/docker/aufs/diff/df229100e0eced9e2fbc4dbdeb55e6ed602d917fc27aa14a29c5a11bdc1c27ba=ro+wh
/var/lib/docker/aufs/diff/6c8288af68cf7fa75239560127afbeccb5dae236173c7807a9970f91839c9269=ro+wh
/var/lib/docker/aufs/diff/2fd1123336697985b55ecc4f58d471ed9c9a8d64d09b54df90d6882ef2a85f2a=ro+wh
```


## 结语

实际上**docker的aufs存储驱动本质上是利用了aufs文件系统来实现镜像和容器的层次管理**，使用aufs存储驱动时，容器内的读写最终都会由aufs文件系统来处理。

docker使用aufs作为存储驱动时,通过将**只读的镜像层**和**读写的容器层**联合挂载到同一目录下，将多层合并成文件系统的单层表示。所有的变化都只是记录在容器层，因此它的优势很明显：

* 能够实现镜像共享，使用相同镜像的容器只是在镜像上创建了一个容器层(读写层)。因此能够节省存储资源，同时加速容器的创建和启动时间；
* 同时，由于aufs出现较早，因此比较稳定，在大量生产中实践过，有较强的社区支持；

但是它也存在缺陷：

* 没有并入Linux内核，只支持ubuntu；
* 它是文件级的存储，对只读层大文件的修改，引发的copy-on-write会带来很大的I/O开销；

## 参考

* *[Docker存储-Aufs](http://www.cnblogs.com/sammyliu/p/5931383.html)*
* *[Docker五种存储驱动原理及应用场景和性能测试对比](http://dockone.io/article/1513)*

[i1]: https://docs.docker.com/engine/userguide/storagedriver/images/aufs_layers.jpg
