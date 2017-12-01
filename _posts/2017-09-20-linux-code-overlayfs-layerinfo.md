---
layout:     post
title:      "内核OverlayFS—文件层次信息"
date:       2017-09-20 2:51:00
author:     "ArKing"
categories: Linux内核
tags:
    - 文件系统
---

* content
{:toc}

>**内核版本为4.4.68**

我们知道，在OverlayFS中，读写操作和文件所在的层息息相关。尤其是写一个只读层的文件时，由于只读层不能修改，因此会触发copy-up操作。那么内核怎么知道一个文件到底是一个upper层（读写层）的文件还是一个lower（只读层）的文件？也就是说，在OverlayFS中如何记录文件的层次信息？

**OverlayFS通过一个名为ovl\_entry的结构体来记录文件的层次信息**。在OverlayFS中，会为每个文件分配一个ovl\_entry变量，通过**文件dentry的d_fsdata字段**指向这个分配的ovl\_entry变量。d_fsdata是一个void类型的指针，记录文件系统特有的信息。

对于OverlayFS中的每个文件，内核首先得到它的dentry，然后通过下面的方式访问文件的层次信息：

```c
struct ovl_entry *oe = dentry->d_fsdata;
```

因此，**只要知道了文件的dentry，就能得到文件的层次信息**。




## OverlayFS的本质

OverlayFS之所以需要一个结构来记录文件的层次信息是因为**OverlayFS下的文件（联合挂载目录下的文件）并不是upper层或者lower层文件的拷贝。在OverlayFS下访问一个文件，实际上都会转化成对upper层或者lower层真实文件的访问。**因此，就需要一个结构来记录每个文件到底源自哪层。

## 文件层次信息

```c
struct ovl_entry {
	struct dentry *__upperdentry;//记录upper层dentry
	struct ovl_dir_cache *cache;
	union {
		struct {
			u64 version;
			bool opaque;
		};
		struct rcu_head rcu;
	};
	unsigned numlower;//lower层数
	struct path lowerstack[];//记录lower层路径
};
```

ovl\_entry结构中的\_\_upperdentry记录了upper层信息，lowerstack字段记录了lower层信息。由于可以有多个lower层，所以lower层信息使用一个数组记录，数组的每个元素对应每一层lower层的信息：

 * 如果一个文件只存在upper层，则lower层信息为空；
 * 如果一个文件只存在lower层，则\_\_upperdentry字段为NULL；
 * 如果文件同时存在upper层和lower层，则两个字段都不为空；

因此，只要得到了文件的dentry，就可以通过ovl\_entry访问upper层或lower层文件的dentry，通过upper层或lower层文件的dentry可以进一步访问upper层或lower层文件的inode，实现对upper层和lower层的访问。

比如说在下图中，upper目录作为upper层，lower目录作为lower层，联合挂载到merged目录下：

![](/img/in-post/post-kernel-overlayfs/dir-op1-dir_tree.png)

对于OverlayFS中的file1，其dentry对应的ovl\_entry结构的\_\_upperdentry字段，就指向upper/file1文件的dentry；
对于OverlayFS中的file2，其dentry对应的ovl\_entry结构的lowerstack[0].dentry，就是lower/file2文件的dentry；
**对于OverlayFS的根目录/，其dentry对应的ovl\_entry结构的\_\_upperdentry字段，指向upper目录的dentry，lowerstack[0].dentry就是lower目录的dentry;**

## 文件路径类型

OverlayFS中文件路径类型由枚举类型ovl\_path\_type表示:

```c
enum ovl_path_type {
	__OVL_PATH_PURE		= (1 << 0),//1
	__OVL_PATH_UPPER	= (1 << 1),//2
	__OVL_PATH_MERGE	= (1 << 2),//4
};
```

根据文件的dentry，可以调用函数ovl\_path\_type()得到文件路径类型：

```c
enum ovl_path_type ovl_path_type(struct dentry *dentry)
{
	struct ovl_entry *oe = dentry->d_fsdata;
	enum ovl_path_type type = 0;

	if (oe->__upperdentry) {//说明这个文件存在upper层
		type = __OVL_PATH_UPPER;//__OVL_PATH_UPPER (2)
		/*
		 * Non-dir dentry can hold lower dentry from previous
		 * location. Its purity depends only on opaque flag.
		 */
		if (oe->numlower && S_ISDIR(dentry->d_inode->i_mode))
			//__OVL_PATH_UPPER |  __OVL_PATH_MERGE (6)
			type |= __OVL_PATH_MERGE;
		else if (!oe->opaque)
			//__OVL_PATH_UPPER | __OVL_PATH_PURE (3)
			type |= __OVL_PATH_PURE;
	} else {
		if (oe->numlower > 1)
			//__OVL_PATH_MERGE (4)
			type |= __OVL_PATH_MERGE;
	}
	return type;
}
```

这里**假设只有1层lower层**，给出下列几种常见文件及其文件路径类型：

| 描述 |    文件路径类型(值)  |
| :-----: | :-----------: |
| 只在lower层中存在的一般**文件或目录** | (0) |
| 只在upper层中存在的一般**文件或目录** | \_\_OVL\_PATH\_UPPER \| \_\_OVL\_PATH\_PURE (3) |
| 同时存在lower和upper层的**文件** | \_\_OVL\_PATH\_UPPER (2) |
| 同时存在lower和upper层的**目录** | \_\_OVL\_PATH\_UPPER \| \_\_OVL\_PATH\_MERGE (6) |

同时存在lower和upper层的文件和目录的文件路径类型不一样的原因在于，同名文件上层会覆盖下层，而同名目录则是会进行合并。

所以文件层次信息中的opaque字段被设置时，可以理解成一种“覆盖隐藏”，也就是说由于下层信息被覆盖，不会展示在联合目录下，所以不需要再对下层的信息进行分析。删除文件和目录也是使用了whiteout文件实现这种“覆盖隐藏”。

## 总结

结构体ovl_entry记录了OverlayFS中文件的层次信息，通过这个结构体，内核可以根据一个OverlayFS文件的dentry来实现对相应upper层和lower层的文件访问。由于OverlayFS的本质是将对文件的操作转化为对底层文件系统upper层或lower层文件的操作。因此在OverlayFS中，会大量涉及到对文件层次信息的访问。理解这个结构有助于理解OverlayFS如何实现操作转化。因此这个结构很重要。 