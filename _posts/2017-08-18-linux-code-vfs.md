---
layout:     post
title:      "Linux虚拟文件系统(VFS)"
date:       2017-08-18 2:51:00
author:     "ArKing"
categories: Linux内核
tags:
    - 文件系统
---

* content
{:toc}

在Linux中，VFS为上层应用提供了访问下层文件系统的统一接口，它将文件相关的概念进行了抽象，同时提供了对这些抽象概念的操作接口。

考虑一个cp操作：

<img src="/img/in-post/post-linux-vfs/vfs.png" style="zoom:30%" align="center"/>

*[图片来源: 《深入理解Linux内核》][i1]*

VFS将文件进行了抽象，使用一个名为file的结构体来表示，这个file结构体包含了一个f\_op变量，这个变量是一个名为file\_operations的结构体指针。结构体file\_operations定义了一系列文件相关的操作。

假设/tmp/test对应的文件变量名为file\_test，/floppy/TEST对应的文件变量名为file_TEST。因此，VFS并不需要关心底层文件系统是什么。对于cp操作，它只需对file\_test调用一个读，将数据读入缓冲区。然后对file\_TEST调用一个写，将缓冲区的数据写入file\_TEST，就能完成文件的拷贝：

```c
file_test->f_op->read();
file_TEST->f_op->write();
```

具体的read和write函数由每个底层文件系统自己实现。




<br />
# VFS中的基本结构
<br />

在VFS有下列几个经常用到的结构：

| 结构 |    描述  |   文件  |    包含的操作对象     |
| :-----: | :-----------: |  :-----------: |  :-----------: |
| **super_block** | 代表一个已挂载的**文件系统** | fs.h | super_operations |
| **inode** | 代表一个特定的**文件**(包括目录) | fs.h | inode_operations |
| **dentry** | 代表一个**目录项** | dcache.h | dentry_operations |
| **file** | 代表一个和某个进程相关的**已打开的文件** | fs.h | file_operations |

“包含的操作对象”是一个包含了相关函数指针的结构体：

* **super_operations**：包含了一些内核对特定文件系统的调用，如write\_inode()和sync\_fs();
* **inode_operations**：包含了一些内核对特定文件的调用，如create(),link();
* **dentry_operations**：包含了一些内核对特定目录项的调用，如d_compare(),d_delete();
* **file_operations**：包含了一些内核对**已打开文件**的调用，如read(),write();

## VFS基本结构指向关系

除了这4个最基本的结构，还有诸如file\_system\_type，vfsmount等一些常用的结构。这些常见结构体之间的指向关系如下图：

![](/img/in-post/post-linux-vfs/relation.png)

为了后面分析每个结构在内核中的组织关系，这里不得不提一下内核中的3个链表相关的结构体，这3个结构体在内核代码中十分常见：

![](/img/in-post/post-linux-vfs/list.png)

* list_head用于组织双向链表，如果将链表的首尾节点互连还可以组织成环状双链表。
* hlist\_head为hash链表中的一个节点，这个节点是一个链表的头结点。这里需要注意的是hlist\_node的pprev成员是一个指向hlist\_node指针的指针。和list\_head中的prev成员不一样。使用指向指针的指针，可以使得头结点的后一个节点通过pprev来指向头结点(hlist_head)中的first成员，因此头结点可以节省一个成员变量的空间。当hash链表很大时，可以节省大量的内存空间。

与一般的链表的实现不同，在Linux内核中，链表节点都是作为结构体的一部分来将这些结构体前后串连起来：

```c
struct student{
	struct list_head list_node;//链表节点
	char* name;
	char* ID;
	...
};
```

而一般的链表实现则是将结构体实现为链表节点：

```c
struct student{//结构体student本身就是一个链表节点
	struct student *prev;
	struct student *next;
	char* name;
	char* ID;
}
```

由于不是这篇文章的重点，这里简单了解下就好，具体原因以及使用方法会在其它篇幅中进行介绍。

## super_block

每个super\_block可以看成是一个已挂载的文件系统实例，它被每个底层文件系统(ext2,ext3,ext4,sysfs等)实现，用来存储描述特定文件系统的信息。  
创建、管理、销毁一个super\_block对象的代码在fs/super.c中。

每当一个文件系统被挂载时，它会调用alloc\_super()方法，这个方法会去磁盘上读取superblock信息，然后用这些信息来初始化一个super_block对象。

**和super_block相关的链表有2个**：

1）**全局super_block链表**：这个链表包含了系统中的所有super\_block(不止一种文件系统的super\_block)。通过一个list\_head类型的**全局变量super_blocks**来访问，内核中使用宏实现这个全局变量的初始化：

```c
static LIST_HEAD(super_blocks);
#define LIST_HEAD_INIT(name) { &(name), &(name) }//前后指针指向自身

#define LIST_HEAD(name) \
    struct list_head name = LIST_HEAD_INIT(name)
```

此后，当有新的文件系统挂载时，它会通过自身super\_block结构中list\_head类型的**成员s_list**链接到super_blocks上，形成一个全局的链表：

![](/img/in-post/post-linux-vfs/super_block-1.png)

2）**特定文件系统的super_block链表**：这个链表包含了某个特定文件系统的所有super_block。

一个文件系统类型(如ext4)，可以有多个super\_block实例。举例来说，我的Linux系统中，划分了3个分区分别挂载在/，/home，/boot目录上，挂载的文件系统都是ext4，因此，这3个ext4文件系统实例都有自己的super\_block，它们通过super\_block中hlist\_node类型的**成员s_instances**链接成“ext4的super\_block链表”。可以通过file\_system\_type的**fs_supers成员**来访问：

![](/img/in-post/post-linux-vfs/super_block-2.png)

### super_block操作

super\_operations中包含了文件系统相关的一系列函数指针。例如，如果文件系统想写它的super_block，可以通过如下方式：

```c
//sb为super_block指针
sb->s_op->write_super(sb);
```
这里给出一些关键的super_block操作：

| 操作 |    描述  |
| :-----: | :-----------: |
| struct inode \* **alloc\_inode**(struct super\_block \*sb) | 在文件系统sb下，创建并初始化一个inode |
| void **destroy\_inode**(struct inode \*inode) | 将inode释放 |
| void **dirty\_inode**(struct inode \*inode) | 当inode被修改时，日志文件系统会使用这个功能来更新日志 |
| void **write\_inode**(struct inode \*inode, int wait) | 将inode同步到磁盘，wait参数指明了操作是否应该同步 |
| void **drop\_inode**(struct inode \*inode) | 当inode失去最后一个引用时被调用，一般的Unix文件系统不定义这个功能。在这种情况下，VFS会简单的删除掉这个inode |
| void **delete\_inode**(struct inode \*inode) | 将inode从磁盘中删除 |
| void **put\_super**(struct super\_block \*sb) | 在卸载文件系统时被VFS调用，来释放sb结构，调用者必须获得相应的锁 |
| void **write\_super**(struct super\_block \*sb) | 更新sb对应磁盘中的superblock，这个操作用来同步磁盘和内存中的超级块信息。调用前必须获得相应的锁 |
| int **sync\_fs**(struct super\_block \*sb, int wait) | 同步文件系统元数据到磁盘，wait参数指明操作是否是同步的 |
| void **write\_super\_lockfs**(struct super\_block \*sb) | 防止sb被修改，同时更新sb数据到磁盘。目前被LVM使用 |
| void **unlockfs**(struct super\_block \*sb) | write\_super\_lockfs调用后被调用，此后可以修改sb |
| int **statfs**(struct super\_block \*sb, struct statfs \*statfs) | 被VFS调用获取文件系统统计信息，存入statfs中 |
| int **remount\_fs**(struct super\_block \*sb, int \*flags, char \*data) | 当文件系统被重新挂载到一个新的挂载点时被调用，必须获得相应锁 |
| void **clear\_inode**(struct inode \*inode) | 释放inode被清除任何存放相关数据的页 |


## inode

inode包含了内核操作一个文件或者目录的所有信息，这些信息从磁盘中的inode读取，并存入到内存的inode结构中。每个inode代表一个文件，但是**只有当具体文件被访问时，内存中才会构造这个文件对应的inode结构。**

一些没有inode的文件系统通常将文件的信息作为文件的一部分存储在文件中，而Unix风格的文件系统(有inode)则是将文件数据和控制信息分开存储。

inode结构中使用一个union(联合体)来指定这个inode代表的文件类型(设备文件、管道等)。

一些文件系统可能不支持inode中的一些属性。比如说，一些文件系统不记录访问时间戳(access timestamp)，这些文件系统可以使用它们觉得合适的方式来实现这些特征。例如，将对应的值(i\_atime)设置为0，让i\_atime的值和i\_mtime相等，仅在内存中更新更新i\_atime的值但是并不将其刷新到磁盘中。

### inode操作

每当VFS需要对一个inode执行一些操作时，它会调用当前文件系统实现的指定inode操作。调用方式如下：

```c
//i表示inode指针，func为具体操作
i->i_op->func(i)
```
如果在ext3下，则是调用ext3为inode实现的func，如果在ext4下，则是调用ext4为inode实现的func。

一些关键的inode操作如下:

| 操作 |    描述  |
| :-----: | :-----------: |
| int **create**(struct inode \*dir,struct dentry \*dentry,int mode) | VFS通过create和open系统调用来调用这个函数，根据指定的mode创建一个和dentry关联的inode |
| struct dentry* **lookup**(struct inode \*dir,struct dentry \*dentry) | 在目录中查找和dentry中指定文件名对应的inode |
| int **link**(struct dentry \*old_dentry,struct inode \*dir,struct dentry \*dentry) | 被系统调用link调用，为目录dir中的old_dentry创建一个硬链接dentry |
| int **unlink**(struct inode \*dir,struct dentry \*dentry) | 被系统调用unlink调用，从目录dir中移除dentry对应的inode |
| int **syslink**(struct inode \*dir,struct dentry \*dentry,const char \*sysname) | 被系统调用syslink调用，创建一个名为sysname的符号链接指向目录dir中dentry代表的文件 |
| int **mkdir**(struct inode \*dir,struct dentry \*dentry,int mode) | 被系统调用mkdir调用，根据mode创建一个新的目录 |
| int **rmdir**(struct inode \*dir,struct dentry \*dentry) | 被系统调用rmdir调用，从目录dir中删除dentry指向的目录 |
| int **mknod**(struct inode \*dir,struct dentry \*dentry,int mode,dev_t rdev) | 被系统调用mknod调用，创建一个文件，这个文件被rdev以及目录dir中的dentry引用，mode指定了初始权限 |
| int **rename**(struct inode \*old\_dir,struct dentry \*old\_dentry,struct inode \*new\_dir,struct dentry \*new\_dentry) | 将old\_dir目录下old\_dentry指定的文件重命名为new\_dentry目录下dentry指定的文件 |
| void **truncate**(struct inode \*inode) | 修改文件的大小，调用前，inode的i\_size域必须被设置为期望的新值 |
| int **permission**(struct inode \*inode, int mask) | 检查inode对应的文件是否具有mask权限，具有则返回0，否则返回一个负值。大多数文件系统将这个函数指针设为NULL而使用通用VFS的方法，简单的比较mask和inode中i_mode成员对应的位 |
| int **setattr**(struct dentry \*dentry, struct iattr \*attr) | 被notify_change调用，在inode被修改后，通知一个“改变事件” |
| int **getattr**(struct vfsmount \*mnt, struct dentry \*dentry,struct kstat \*stat) | 当通知一个inode需要从磁盘中刷新时，被VFS调用 |
| int **setxattr**(struct dentry \*dentry, const char \*name,const void \*value,size_t size, int flags) | 将dentry对应文件的name扩展属性的值设置为value |
| ssize\_t **getxattr**(struct dentry \*dentry, const char \*name,void \*value, size_t size) | 将dentry对应文件的name扩展属性的值拷贝到value中 |
| ssize\_t **listxattr**(struct dentry \*dentry, char \*list, size_t size) | 将dentry对应文件的所以属性拷贝到list指向的缓冲中 |
| int **removexattr**(struct dentry \*dentry,const char \*name) | 删除dentry对应文件的name属性 |

## dentry

dentry在内核中的主要是用于**路径查找**。和super_block以及inode不同，**它是一个内存结构，并没有对应的磁盘数据**。

在Linux执行文件打开操作时，如根据路径/root/file打开文件时，在解析过程中，如果内核中的dentry树没有这个路径。会在dentry树中生成表示这样一个路径的几个dentry，在这个例子中根目录“/”，子目录“root”，文件“file”都对应一个dentry。如果内核的dentry树中存在代表根目录/的dentry，则进一步查找root，这样一直递归，进行路径搜索：

 <img src="/img/in-post/post-linux-vfs/dentry-tree.png" style="zoom:60%" align="center"/>

一个有效的dentry可能处于下列**3种状态**中的一种：**used、unused、negative**

* **used dentry**： 一个used dentry对应一个有效的inode(d\_inode成员指向对应的inode)，表明这个dentry有一个或多个users(d_count为正)。used dentry被VFS使用，它们指向有效的数据，因此不能被销毁；
* **unused dentry**：一个unused dentry对应一个有效的inode(d\_inode成员指向对应的inode)，但是VFS当前未使用该dentry(d_count为0)。不过，因为这些dentry仍然指向有效的对象，可能被再次使用，因此它们也被缓存。但是在内存紧张的情况下，它们也可能被销毁；
* **negative dentry**： 一个negative dentry没有相应的inode(d_inode为NULL)，这是由于相应的inode已经被删掉或者路径名本身就不正确。negative dentry也会被缓存，因为它可以加速查找速率。想象一下一个daemon程序不停的尝试打开一个不存在的文件，如果不缓存，在路径解析时，会带来巨大的开销，因此缓存是值得的。不过和unused dentry一样，在内存紧张时，它也可能被销毁；

由于路径解析过程中，一个个生成dentry，如果解析完后就扔掉则太浪费。因此，内核把dentry缓存在**dcache**(dentry cache)中。

**dcache**(dentry cache)由3部分组成：

* **used dentry list：** dentry对应inode的i\_dentry成员串连组成的链表，因为一个inode可能对应多个dentry(如硬链接),因此一个inode的i\_dentry可能包含多个dentry。
* **unused and negative dentry list：**LRU双链表，从头部插入节点，因此头部是最新的节点。当需要内存时，从尾部删除节点。
* **hash table and hash function：**用来快速的解析一个路径为多个dentry。

哈希表由数组dentry\_hashtable表示，数组的每个元素都是指向相同hash值的dentry对象组成的链表。数组的大小取决于物理RAM的大小。

实际的哈希值由d\_hash()得到。通过d\_lookup()查询哈希表，如果dentry在哈希表中则返回，否则返回NULL。

和dentry相关的inode不会被释放，因为dentry对inode的引用使得引用计数为正，这就使得缓存dentry的同时会将inode一起缓存。换而言之，只要dentry被缓存，相应的inode也会被缓存(icache)。

### dentry操作

一些关键的目录项操作如下：

| 操作 |    描述  |
| :-----: | :-----------: |
| int **d\_revalidate**(struct dentry \*dentry, struct nameidata \*) | 当VFS准备从dcache中使用dentry时，使dnetry有效。大多数文件系统将这个函数指针设为NULL，因为它们在dcache中的dentry总是有效 |
| int **d\_hash**(struct dentry \*dentry, struct qstr \*name) | 当VFS准备将一个dentry添加到哈希表时调用这个函数，为dentry生成一个哈希值 |
| int **d\_compare**(struct dentry \*dentry, struct qstr \*name1, struct qstr \*name2) | VFS调用这个函数来比较两个函数名name1和name2,函数需要dcache\_lock |
| int **d\_delete**(struct dentry \*dentry) | 当dentry的d\_count减为0时被调用，函数需要dcache\_lock以及dentry的d\_lock |
| void **d\_release**(struct dentry \*dentry) | 当dentry将要被释放时被VFS调用，默认的功能啥事也不做 |
| void **d\_iput**(struct dentry \*dentry, struct inode \*inode) | 将dentry失去关联的inode时被VFS调用，默认情况下，VFS仅仅调用iput()功能来释放inode。如果重写了这个函数，也必须在函数内调用iput() |

## file

file用来表示一个**被进程打开的文件**。进程直接处理file而不是super\_block、inode或者dentry。它是一个已打开文件的内存表示，**在open系统调用被调用时创建，在系统调用close被调用时销毁**。所以文件相关的调用定义在file操作表中。

由于file表示进程已打开的文件，不同进程可能打开相同的文件，因此磁盘中同一文件在内存中可能有多个file，所以dentry才代表实际打开的文件，同一文件的dentry和inode是唯一的。

和dentry一样，file没有相对应的磁盘结构。因此在file结构中没有flag用于表示file是否被修改而应该同步数据到磁盘中。通过file中的f_dentry索引到file对应的dentry，这个dentry又可以索引到对应的inode,则两者可以反映这个实际文件是否已经被修改。

### file操作

file操作和标准Unix系统调用十分相似，下面列举了一些关键的file操作：

| 操作 |    描述  |
| :-----: | :-----------: |
| loff\_t **llseek**(struct file \*file,loff\_t offset, int origin) | 根据offset更新文件指针file |
| ssize\_t **read**(struct file \*file,char \*buf, size_t count,loff_t \*offset) | 从文件file偏移量offset处，读取count字节的数据到buf中 |
| ssize\_t **aio\_read**(struct kiocb \*iocb,char \*buf, size_t count,loff_t offset) | 被系统调用aio_read调用，对iocb中描述的文件执行一个异步读操作 |
| ssize\_t **write**(struct file \*file,const char \*buf, size\_t count,loff\_t \*offset)) | 被系统调用write调用，从buf中读取数据，以file的offset为起始位置，写count字节数据 |
| ssize_t **aio\_write**(struct kiocb \*iocb, const char \*buf,size\_t count, loff\_t offset) | 发起一个异步写操作 |
| int **readdir**(struct file \*file, void \*dirent,filldir\_t filldir) | 被系统调用readdir调用，返回目录列表中的下一个目录 |
| unsigned int **poll**(struct file \*file,struct poll\_table\_struct \*poll\_table) | 被系统调用poll调用，等待file的一个活动 |
| int **ioctl**(struct inode \*inode, struct file \*file,unsigned int cmd,unsigned long arg) | 被系统调用ioctl调用，将一个命令和一系列参数发送给一个设备。当file是一个打开的设备节点时会使用此函数，调用者必须持有BKL(Big Kernel Lock) |
| int **unlocked\_ioctl**(struct file \*file, unsigned int cmd,unsigned long arg) | 和ioctl功能相同，只是不需要调用者持有BKL |
| int **mmap**(struct file \*file,struct vm\_area\_struct \*vma) | 被系统调用mmap调用，将file映射到指定的地址空间 |
| int **open**(struct inode \*inode, struct file \*file) | 被系统调用open调用，生产一个file，并将其连接到inode |
| int **flush**(struct file \*file) | 当file的引用计数减少时被调用，其意图取决于具体文件系统 |
| int **release**(struct inode \*inode, struct file \*file) | 当file最后一个引用被销毁时调用(比如，最后一个进程执行close操作)，其意图取决于文件系统 |
| int **fsync**(struct file \*file, struct dentry \*dentry,int datasync) | 将file所有缓存的数据写入到磁盘 |
| int **aio\_fsync**(struct kiocb \*iocb, int datasync) | 将iocb指定file的所有缓存的数据写入到磁盘 |
| int **fasync**(int fd, struct file \*file, int on) | 关掉或者开启异步I/O的信号通知 |
| int **lock**(struct file \*file, int cmd, struct file\_lock \*lock) | 给file上锁 |
| ssize\_t **readv**(struct file \*file,const struct iovec \*vector,unsigned long count,loff\_t \*offset) | 被系统调用readv调用，从file中读取数据到iovec指定的buffer中 |
| ssize\_t **writev**(struct file \*file,const struct iovec \*vector,unsigned long count,loff\_t \*offset) | 被系统调用writev调用，将iovec指定的buffer中的数据写入到file中 |
| ssize\_t **sendfile**(struct file \*file, loff\_t \*offset,size\_t size,read\_actor\_t actor,void \*target) | 被系统调用sendfile调用，从一个文件向另一个文件拷贝数据，数据拷贝完全在内核中进行，没有内核与用户空间之间数据拷贝的开销 |
| ssize\_t **sendpage**(struct file \*file, struct page \*page,int offset, size\_t size,loff\_t \*pos, int more) | 将一个文件中的数据发送到另一个文件中 |

<br />
<br />
# 内核源码

**内核版本：4.4.68**

## super_block

```c
struct super_block {
	struct list_head	s_list;		/* Keep this first */
	dev_t			s_dev;		/* search index; _not_ kdev_t */
	unsigned char		s_blocksize_bits;	//块大小(bits)
	unsigned long		s_blocksize;	//块大小(bytes)
	loff_t			s_maxbytes;		//最大文件大小
	struct file_system_type	*s_type;	//文件系统类型
	const struct super_operations	*s_op;
	const struct dquot_operations	*dq_op;
	const struct quotactl_ops	*s_qcop;
	const struct export_operations *s_export_op;
	unsigned long		s_flags;
	unsigned long		s_iflags;	/* internal SB_I_* flags */
	unsigned long		s_magic;
	struct dentry		*s_root;	//目录挂载点
	struct rw_semaphore	s_umount;
	int			s_count;	//引用计数
	atomic_t		s_active;
#ifdef CONFIG_SECURITY
	void                    *s_security;
#endif
	const struct xattr_handler **s_xattr;

	struct hlist_bl_head	s_anon;		/* anonymous dentries for (nfs) exporting */
	struct list_head	s_mounts;	/* list of mounts; _not_ for fs use */
	struct block_device	*s_bdev;	//指向文件系统存在的块设备指针
	struct backing_dev_info *s_bdi;
	struct mtd_info		*s_mtd;
	struct hlist_node	s_instances;
	unsigned int		s_quota_types;	/* Bitmask of supported quota types */
	struct quota_info	s_dquot;	/* Diskquota specific options */

	struct sb_writers	s_writers;

	char s_id[32];				/* Informational name */
	u8 s_uuid[16];				/* UUID */

	void 			*s_fs_info;	/* Filesystem private info */
	unsigned int		s_max_links;
	fmode_t			s_mode;

	/* Granularity of c/m/atime in ns.
	   Cannot be worse than a second */
	u32		   s_time_gran;

	/*
	 * The next field is for VFS *only*. No filesystems have any business
	 * even looking at it. You had been warned.
	 */
	struct mutex s_vfs_rename_mutex;	/* Kludge */

	/*
	 * Filesystem subtype.  If non-empty the filesystem type field
	 * in /proc/mounts will be "type.subtype"
	 */
	char *s_subtype;

	/*
	 * Saved mount options for lazy filesystems using
	 * generic_show_options()
	 */
	char __rcu *s_options;
	const struct dentry_operations *s_d_op; /* default d_op for dentries */

	/*
	 * Saved pool identifier for cleancache (-1 means none)
	 */
	int cleancache_poolid;

	struct shrinker s_shrink;	/* per-sb shrinker handle */

	/* Number of inodes with nlink == 0 but still referenced */
	atomic_long_t s_remove_count;

	/* Being remounted read-only */
	int s_readonly_remount;

	/* AIO completions deferred from interrupt context */
	struct workqueue_struct *s_dio_done_wq;
	struct hlist_head s_pins;

	/*
	 * Keep the lru lists last in the structure so they always sit on their
	 * own individual cachelines.
	 */
	struct list_lru		s_dentry_lru ____cacheline_aligned_in_smp;
	struct list_lru		s_inode_lru ____cacheline_aligned_in_smp;
	struct rcu_head		rcu;
	struct work_struct	destroy_work;

	struct mutex		s_sync_lock;	/* sync serialisation lock */

	/*
	 * Indicates how deep in a filesystem stack this SB is
	 */
	int s_stack_depth;

	/* s_inode_list_lock protects s_inodes */
	spinlock_t		s_inode_list_lock ____cacheline_aligned_in_smp;
	struct list_head	s_inodes;	/* all inodes */
};
```

## super_operations

```c
struct super_operations {
   	struct inode *(*alloc_inode)(struct super_block *sb);
	void (*destroy_inode)(struct inode *);

   	void (*dirty_inode) (struct inode *, int flags);
	int (*write_inode) (struct inode *, struct writeback_control *wbc);
	int (*drop_inode) (struct inode *);
	void (*evict_inode) (struct inode *);
	void (*put_super) (struct super_block *);
	int (*sync_fs)(struct super_block *sb, int wait);
	int (*freeze_super) (struct super_block *);
	int (*freeze_fs) (struct super_block *);
	int (*thaw_super) (struct super_block *);
	int (*unfreeze_fs) (struct super_block *);
	int (*statfs) (struct dentry *, struct kstatfs *);
	int (*remount_fs) (struct super_block *, int *, char *);
	void (*umount_begin) (struct super_block *);

	int (*show_options)(struct seq_file *, struct dentry *);
	int (*show_devname)(struct seq_file *, struct dentry *);
	int (*show_path)(struct seq_file *, struct dentry *);
	int (*show_stats)(struct seq_file *, struct dentry *);
#ifdef CONFIG_QUOTA
	ssize_t (*quota_read)(struct super_block *, int, char *, size_t, loff_t);
	ssize_t (*quota_write)(struct super_block *, int, const char *, size_t, loff_t);
	struct dquot **(*get_dquots)(struct inode *);
#endif
	int (*bdev_try_to_free_page)(struct super_block*, struct page*, gfp_t);
	long (*nr_cached_objects)(struct super_block *,
				  struct shrink_control *);
	long (*free_cached_objects)(struct super_block *,
				    struct shrink_control *);
};
```

## inode

```c
struct inode {
	umode_t			i_mode;		//访问权限
	unsigned short		i_opflags;
	kuid_t			i_uid;		//拥有者的用户ID
	kgid_t			i_gid;		//拥有者的组ID
	unsigned int		i_flags;

#ifdef CONFIG_FS_POSIX_ACL
	struct posix_acl	*i_acl;
	struct posix_acl	*i_default_acl;
#endif

	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;

#ifdef CONFIG_SECURITY
	void			*i_security;
#endif

	/* Stat data, not accessed from path walking */
	unsigned long		i_ino;	//inode号
	/*
	 * Filesystems may only read i_nlink directly.  They shall use the
	 * following functions for modification:
	 *
	 *    (set|clear|inc|drop)_nlink
	 *    inode_(inc|dec)_link_count
	 */
	union {
		const unsigned int i_nlink;
		unsigned int __i_nlink;
	};
	dev_t			i_rdev;
	loff_t			i_size;		//文件大小(byte)
	struct timespec		i_atime;
	struct timespec		i_mtime;
	struct timespec		i_ctime;
	spinlock_t		i_lock;	/* i_blocks, i_bytes, maybe i_size */
	unsigned short          i_bytes;
	unsigned int		i_blkbits;		//block大小(bit)
	blkcnt_t		i_blocks;			//文件大小(block)

#ifdef __NEED_I_SIZE_ORDERED
	seqcount_t		i_size_seqcount;
#endif

	/* Misc */
	unsigned long		i_state;
	struct mutex		i_mutex;

	unsigned long		dirtied_when;	/* jiffies of first dirtying */
	unsigned long		dirtied_time_when;

	struct hlist_node	i_hash;
	struct list_head	i_io_list;	/* backing dev IO list */
#ifdef CONFIG_CGROUP_WRITEBACK
	struct bdi_writeback	*i_wb;		/* the associated cgroup wb */

	/* foreign inode detection, see wbc_detach_inode() */
	int			i_wb_frn_winner;
	u16			i_wb_frn_avg_time;
	u16			i_wb_frn_history;
#endif
	struct list_head	i_lru;		/* inode LRU list */
	struct list_head	i_sb_list;
	union {
		struct hlist_head	i_dentry;
		struct rcu_head		i_rcu;
	};
	u64			i_version;
	atomic_t		i_count;		//引用数
	atomic_t		i_dio_count;
	atomic_t		i_writecount;
#ifdef CONFIG_IMA
	atomic_t		i_readcount; /* struct files open RO */
#endif
	const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
	struct file_lock_context	*i_flctx;
	struct address_space	i_data;
	struct list_head	i_devices;
	union {
		struct pipe_inode_info	*i_pipe;
		struct block_device	*i_bdev;
		struct cdev		*i_cdev;
		char			*i_link;
	};

	__u32			i_generation;

#ifdef CONFIG_FSNOTIFY
	__u32			i_fsnotify_mask; /* all events this inode cares about */
	struct hlist_head	i_fsnotify_marks;
#endif

	void			*i_private; /* fs or device private pointer */
};
```

## inode_operations

```c
struct inode_operations {
	struct dentry * (*lookup) (struct inode *,struct dentry *, unsigned int);
	const char * (*follow_link) (struct dentry *, void **);
	int (*permission) (struct inode *, int);
	struct posix_acl * (*get_acl)(struct inode *, int);

	int (*readlink) (struct dentry *, char __user *,int);
	void (*put_link) (struct inode *, void *);

	int (*create) (struct inode *,struct dentry *, umode_t, bool);
	int (*link) (struct dentry *,struct inode *,struct dentry *);
	int (*unlink) (struct inode *,struct dentry *);
	int (*symlink) (struct inode *,struct dentry *,const char *);
	int (*mkdir) (struct inode *,struct dentry *,umode_t);
	int (*rmdir) (struct inode *,struct dentry *);
	int (*mknod) (struct inode *,struct dentry *,umode_t,dev_t);
	int (*rename) (struct inode *, struct dentry *,
			struct inode *, struct dentry *);
	int (*rename2) (struct inode *, struct dentry *,
			struct inode *, struct dentry *, unsigned int);
	int (*setattr) (struct dentry *, struct iattr *);
	int (*getattr) (struct vfsmount *mnt, struct dentry *, struct kstat *);
	int (*setxattr) (struct dentry *, const char *,const void *,size_t,int);
	ssize_t (*getxattr) (struct dentry *, const char *, void *, size_t);
	ssize_t (*listxattr) (struct dentry *, char *, size_t);
	int (*removexattr) (struct dentry *, const char *);
	int (*fiemap)(struct inode *, struct fiemap_extent_info *, u64 start,
		      u64 len);
	int (*update_time)(struct inode *, struct timespec *, int);
	int (*atomic_open)(struct inode *, struct dentry *,
			   struct file *, unsigned open_flag,
			   umode_t create_mode, int *opened);
	int (*tmpfile) (struct inode *, struct dentry *, umode_t);
	int (*set_acl)(struct inode *, struct posix_acl *, int);
} ____cacheline_aligned;
```

## dentry

```c
struct dentry {
	/* RCU lookup touched fields */
	unsigned int d_flags;		/* protected by d_lock */
	seqcount_t d_seq;		/* per dentry seqlock */
	struct hlist_bl_node d_hash;	/* lookup hash list */
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */
	unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

	/* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */
	const struct dentry_operations *d_op;
	struct super_block *d_sb;	/* The root of the dentry tree */
	unsigned long d_time;		/* used by d_revalidate */
	void *d_fsdata;			/* fs-specific data */

	struct list_head d_lru;		/* LRU list */
	struct list_head d_child;	/* child of parent list */
	struct list_head d_subdirs;	/* our children */
	/*
	 * d_alias and d_rcu can share memory
	 */
	union {
		struct hlist_node d_alias;	/* inode alias list */
	 	struct rcu_head d_rcu;
	} d_u;
};
```

## dentry_operations

```c
struct dentry_operations {
	int (*d_revalidate)(struct dentry *, unsigned int);
	int (*d_weak_revalidate)(struct dentry *, unsigned int);
	int (*d_hash)(const struct dentry *, struct qstr *);
	int (*d_compare)(const struct dentry *, const struct dentry *,
			unsigned int, const char *, const struct qstr *);
	int (*d_delete)(const struct dentry *);
	void (*d_release)(struct dentry *);
	void (*d_prune)(struct dentry *);
	void (*d_iput)(struct dentry *, struct inode *);
	char *(*d_dname)(struct dentry *, char *, int);
	struct vfsmount *(*d_automount)(struct path *);
	int (*d_manage)(struct dentry *, bool);
	struct inode *(*d_select_inode)(struct dentry *, unsigned);
	struct dentry *(*d_real)(struct dentry *, struct inode *);
} ____cacheline_aligned;
```

## file

```c
struct file {
	union {
		struct llist_node	fu_llist;
		struct rcu_head 	fu_rcuhead;
	} f_u;
	struct path		f_path;		//包含了对应的dentry
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	/*
	 * Protects f_ep_links, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	atomic_long_t		f_count;
	unsigned int 		f_flags;
	fmode_t			f_mode;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct list_head	f_ep_links;
	struct list_head	f_tfile_llink;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
} __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
```

## file_operations

```c
struct file_operations {
	struct module *owner;
	loff_t (*llseek) (struct file *, loff_t, int);
	ssize_t (*read) (struct file *, char __user *, size_t, loff_t *);
	ssize_t (*write) (struct file *, const char __user *, size_t, loff_t *);
	ssize_t (*read_iter) (struct kiocb *, struct iov_iter *);
	ssize_t (*write_iter) (struct kiocb *, struct iov_iter *);
	int (*iterate) (struct file *, struct dir_context *);
	unsigned int (*poll) (struct file *, struct poll_table_struct *);
	long (*unlocked_ioctl) (struct file *, unsigned int, unsigned long);
	long (*compat_ioctl) (struct file *, unsigned int, unsigned long);
	int (*mmap) (struct file *, struct vm_area_struct *);
	int (*open) (struct inode *, struct file *);
	int (*flush) (struct file *, fl_owner_t id);
	int (*release) (struct inode *, struct file *);
	int (*fsync) (struct file *, loff_t, loff_t, int datasync);
	int (*aio_fsync) (struct kiocb *, int datasync);
	int (*fasync) (int, struct file *, int);
	int (*lock) (struct file *, int, struct file_lock *);
	ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
	unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned long);
	int (*check_flags)(int);
	int (*flock) (struct file *, int, struct file_lock *);
	ssize_t (*splice_write)(struct pipe_inode_info *, struct file *, loff_t *, size_t, unsigned int);
	ssize_t (*splice_read)(struct file *, loff_t *, struct pipe_inode_info *, size_t, unsigned int);
	int (*setlease)(struct file *, long, struct file_lock **, void **);
	long (*fallocate)(struct file *file, int mode, loff_t offset,
			  loff_t len);
	void (*show_fdinfo)(struct seq_file *m, struct file *f);
#ifndef CONFIG_MMU
	unsigned (*mmap_capabilities)(struct file *);
#endif
};
```

[i1]: https://book.douban.com/subject/2287506/