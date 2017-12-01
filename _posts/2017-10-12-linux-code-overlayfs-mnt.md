---
layout:     post
title:      "内核OverlayFS—注册与挂载"
date:       2017-10-12 2:51:00
author:     "ArKing"
categories: Linux内核
tags:
    - 文件系统
---

* content
{:toc}
 
>**内核版本为4.4.68**

这篇文章通过内核源码分析OverlayFS的注册与挂载过程。  

假设你对OverlayFS的层次关系已经有一个基本的了解。如果对OverlayFS的层以及联合挂载没有概念，建议先补充下相关知识，因为挂载OverlayFS时会使用到层。

分析会尽可能避开内核公有部分(每种文件系统注册和挂载过程中都会涉及的内容)，而更多的关注OverlayFS独有部分。 




## 注册

overlayfs在内核中以内核模块的形式存在，假设你已经知道在内核模块加载时，会涉及到两个关键函数——入口和出口函数。他们通过如下代码指明：

```c
module_init(ovl_init);//入口函数——ovl_init
module_exit(ovl_exit);//出口函数——ovl_exit
```

当使用命令```modprobe overlay```加载overlayfs时，会调用入口函数ovl_init：

```c
static int __init ovl_init(void)
{
	return register_filesystem(&ovl_fs_type);
}
```

因此，当加载overlay模块时，模块入口函数调用register_filesystem函数注册overlayfs：

```c
int register_filesystem(struct file_system_type * fs)
{
	int res = 0;
	struct file_system_type ** p;

	BUG_ON(strchr(fs->name, '.'));
	if (fs->next)
		return -EBUSY;
	write_lock(&file_systems_lock);
	p = find_filesystem(fs->name, strlen(fs->name));
	if (*p)
		res = -EBUSY;
	else
		*p = fs;
	write_unlock(&file_systems_lock);
	return res;
}

EXPORT_SYMBOL(register_filesystem);
```
注册一个文件系统需要用到这个文件系统的类型结构——file_system_type。这个结构中定义了文件系统的名称，挂载函数等信息。

在上面代码中，注册一个文件系统首先会检查文件系统是否已经在链表中，如果存在，则直接返回。否则调用find\_filesystem函数通过全局变量file_systems遍历全局文件系统类型链表，查看是否存在已经注册过名为overlay的文件系统，有则返回它的地址，否则返回链表结尾的地址。最后，如果没找到，则将这个文件系统链接到链表结尾。

不直接遍历全局链表查找，而是先查看fs的next指针是否为NULL的原因在于遍历链表十分耗时。

总结起来，**插入overlay模块主要工作是完成overlayfs的注册**，而注册即是将overlayfs添加到全局文件系统类型链表中。

## 挂载

每当挂载一个overlayfs时，比如下面的命令将lower和upper目录，使用overlayfs挂载到merged目录：

```bash
sudo mount -t overlay overlay -o lowerdir=lower,upperdir=upper,workdir=work merged
```

结构体变量ovl_fs_type(struct file_system_type)的mount成员指明了挂载函数为ovl_mount，它是函数mount_nodev的封装：

```c
static struct dentry *ovl_mount(struct file_system_type *fs_type, int flags,
				const char *dev_name, void *raw_data)
{
	return mount_nodev(fs_type, flags, raw_data, ovl_fill_super);
}

/* fs/super.c */
struct dentry *mount_nodev(struct file_system_type *fs_type,
	int flags, void *data,
	int (*fill_super)(struct super_block *, void *, int))
{
	int error;
	//1.创建一个superblock，新创建的这个superblock已经链接到全局superblock链表中
	struct super_block *s = sget(fs_type, NULL, set_anon_super, flags, NULL);

	if (IS_ERR(s))
		return ERR_CAST(s);

	//2.填充superblock
	error = fill_super(s, data, flags & MS_SILENT ? 1 : 0);
	if (error) {
		deactivate_locked_super(s);
		return ERR_PTR(error);
	}
	s->s_flags |= MS_ACTIVE;
	return dget(s->s_root);
}
EXPORT_SYMBOL(mount_nodev);
```

overlayfs的挂载使用了fs/super.c中的mount_nodev函数，该函数主要完成2个操作：

* 调用sget函数(fs/super.c)，为新挂载的overlayfs分配一个superblock：
	*  调用alloc_super函数(fs/super.c)创建并初始化一个superblock;
	*  设置这个新创建superblock的文件系统类型为overlayfs文件系统类型，然后将其插入到superblock相关的两个链表中；
* 填充这个superblock;

sget实现superblock的分配不是overlayfs的部分，因此这里不进一步深入代码，上面简单的介绍了它主要完成的工作。下面看看如何填充：

```c
static int ovl_fill_super(struct super_block *sb, void *data, int silent)
{
	struct path upperpath = { NULL, NULL };
	struct path workpath = { NULL, NULL };
	struct dentry *root_dentry;
	struct ovl_entry *oe;
	struct ovl_fs *ufs;//存储overlayfs私有信息
	struct path *stack = NULL;
	char *lowertmp;
	char *lower;
	unsigned int numlower;
	unsigned int stacklen = 0;
	unsigned int i;
	bool remote = false;
	int err;

	err = -ENOMEM;
	ufs = kzalloc(sizeof(struct ovl_fs), GFP_KERNEL);//为ufs分配内存
	if (!ufs)
		goto out;

	/*  data:"lowerdir=####,upperdir=####,workdir=work"
	 *	1.解析挂载命令中的参数，并设置ufs->config，这里主要是获取overlayfs的几个关键目录
	 */
	err = ovl_parse_opt((char *) data, &ufs->config);
	if (err)
		goto out_free_config;

	err = -EINVAL;
	if (!ufs->config.lowerdir) {//不能没有lowerdir
		pr_err("overlayfs: missing 'lowerdir'\n");
		goto out_free_config;
	}

	sb->s_stack_depth = 0;	//设置这个superblock在文件系统栈中的深度
	sb->s_maxbytes = MAX_LFS_FILESIZE;	//设置文件系统最大文件大小
	/*
	 * 2.如果有upperdir，则对upperdir和workdir进行处理：
	 */
	if (ufs->config.upperdir) {
		if (!ufs->config.workdir) {//如果有upperdir，则必须有workdir，因为workdir需要完成copy-up操作
			pr_err("overlayfs: missing 'workdir'\n");
			goto out_free_config;
		}
		
		/*查找路径upperdir，并得到相应的路径upperpath*/
		err = ovl_mount_dir(ufs->config.upperdir, &upperpath);
		if (err)
			goto out_free_config;

		if (upperpath.mnt->mnt_sb->s_flags & MS_RDONLY) {//upper文件系统不能为只读
			pr_err("overlayfs: upper fs is r/o, try multi-lower layers mount\n");
			err = -EINVAL;
			goto out_put_upperpath;
		}

		/*查找路径workdir，并得到相应的路径workpath*/
		err = ovl_mount_dir(ufs->config.workdir, &workpath);
		if (err)
			goto out_put_upperpath;

		err = -EINVAL;
		
		if (upperpath.mnt != workpath.mnt) {//upperdir和workdir必须位于同一文件系统挂载点下
			pr_err("overlayfs: workdir and upperdir must reside under the same mount\n");
			goto out_put_workpath;
		}
		
		if (!ovl_workdir_ok(workpath.dentry, upperpath.dentry)) {//upperdir和workdir不能在同一子树
			pr_err("overlayfs: workdir and upperdir must be separate subtrees\n");
			goto out_put_workpath;
		}
		//由于upperdir和workdir的挂载点相同，因此随便选择其中一个的路径访问文件系统挂载点
		sb->s_stack_depth = upperpath.mnt->mnt_sb->s_stack_depth;
	}
	err = -ENOMEM;
	lowertmp = kstrdup(ufs->config.lowerdir, GFP_KERNEL);
	if (!lowertmp)
		goto out_put_workpath;

	err = -EINVAL;
	/*处理lowerdir的字符串，得到lowerdir的数量（也就是说可以有多个lower层）*/
	stacklen = ovl_split_lowerdirs(lowertmp);
	if (stacklen > OVL_MAX_STACK) {//OVL_MAX_STACK:500
		pr_err("overlayfs: too many lower directries, limit is %d\n",
		       OVL_MAX_STACK);
		goto out_free_lowertmp;
	} else if (!ufs->config.upperdir && stacklen == 1) {//如果没有upperdir，则至少需要2个lowerdir
		pr_err("overlayfs: at least 2 lowerdir are needed while upperdir nonexistent\n");
		goto out_free_lowertmp;
	}
	
	//为lowerdir(数量=stacklen)的path栈分配内存空间
	stack = kcalloc(stacklen, sizeof(struct path), GFP_KERNEL);
	if (!stack)
		goto out_free_lowertmp;

	/*
	 * 3.处理每个lowerdir:获取相应的path
	 */
	lower = lowertmp;//此时的lowertmp的形式为：lowerdir1\0lowerdir2\0lowerdir3\0...
	for (numlower = 0; numlower < stacklen; numlower++) {
		err = ovl_lower_dir(lower, &stack[numlower],
				    &ufs->lower_namelen, &sb->s_stack_depth,
				    &remote);
		if (err)
			goto out_put_lowerpath;

		lower = strchr(lower, '\0') + 1;//接着处理下一个lowerdir
	}

	err = -EINVAL;
	sb->s_stack_depth++;
	if (sb->s_stack_depth > FILESYSTEM_MAX_STACK_DEPTH) {
		pr_err("overlayfs: maximum fs stacking depth exceeded\n");
		goto out_put_lowerpath;
	}

	/*
	 * 4：通过upperpath.mnt克隆一个私有upperdir的挂载点,存到ufs的upper_mnt成员中。
	 * 在workdir下创建work目录，得到对应的denty并存入ufs的workdir成员中
	 */
	if (ufs->config.upperdir) {
		ufs->upper_mnt = clone_private_mount(&upperpath);
		err = PTR_ERR(ufs->upper_mnt);
		if (IS_ERR(ufs->upper_mnt)) {
			pr_err("overlayfs: failed to clone upperpath\n");
			goto out_put_lowerpath;
		}

		ufs->workdir = ovl_workdir_create(ufs->upper_mnt, workpath.dentry);
		err = PTR_ERR(ufs->workdir);
		if (IS_ERR(ufs->workdir)) {
			pr_warn("overlayfs: failed to create directory %s/%s (errno: %i); mounting read-only\n",
				ufs->config.workdir, OVL_WORKDIR_NAME, -err);
			sb->s_flags |= MS_RDONLY;
			ufs->workdir = NULL;
		}
	}

	err = -ENOMEM;
	/*
	 * 5.通过stack[i].mnt克隆一个私有lowerdir的挂载点存入ufs的lower_mnt中；
	 */
	ufs->lower_mnt = kcalloc(numlower, sizeof(struct vfsmount *), GFP_KERNEL);
	if (ufs->lower_mnt == NULL)
		goto out_put_workdir;
	for (i = 0; i < numlower; i++) {
		struct vfsmount *mnt = clone_private_mount(&stack[i]);

		err = PTR_ERR(mnt);
		if (IS_ERR(mnt)) {
			pr_err("overlayfs: failed to clone lowerpath\n");
			goto out_put_lower_mnt;
		}
		/*
		 * Make lower_mnt R/O.  That way fchmod/fchown on lower file
		 * will fail instead of modifying lower fs.
		 */
		mnt->mnt_flags |= MNT_READONLY;//设置lowerdir挂载点为只读

		//每向ufs中添加一个挂载点就更新ufs中numlower的值
		ufs->lower_mnt[ufs->numlower] = mnt;
		ufs->numlower++;
	}

	/* If the upper fs is nonexistent, we mark overlayfs r/o too */
	if (!ufs->upper_mnt)
		sb->s_flags |= MS_RDONLY;

	//remote在ovl_lower_dir调用中赋值，根据remote真或假设置sb的dentry操作对象
	if (remote)
		sb->s_d_op = &ovl_reval_dentry_operations;
	else
		sb->s_d_op = &ovl_dentry_operations;

	err = -ENOMEM;
	oe = ovl_alloc_entry(numlower);//为oe分配一段内存空间，由于ovl_entry中有个数组成员lowerstack[]，因此传入参数numlower
	if (!oe)
		goto out_put_lower_mnt;

	/*
	 * 6.为文件系统创建root dentry，设置root dentry文件系统特定的数据
	 */
	root_dentry = d_make_root(ovl_new_inode(sb, S_IFDIR, oe));
	if (!root_dentry)
		goto out_free_oe;

	/*释放挂载点*/
	mntput(upperpath.mnt);
	for (i = 0; i < numlower; i++)
		mntput(stack[i].mnt);
	path_put(&workpath);
	kfree(lowertmp);

	/*将相关的dentry和path信息存入到oe(设置root dentry文件系统特定的数据)*/
	oe->__upperdentry = upperpath.dentry;
	for (i = 0; i < numlower; i++) {
		oe->lowerstack[i].dentry = stack[i].dentry;
		oe->lowerstack[i].mnt = ufs->lower_mnt[i];
	}
	kfree(stack);

	root_dentry->d_fsdata = oe;//将oe与root dentry关联

	ovl_copyattr(ovl_dentry_real(root_dentry)->d_inode,
		     root_dentry->d_inode);

	/*
	 * 7.填充superblock
	 */
	sb->s_magic = OVERLAYFS_SUPER_MAGIC;
	sb->s_op = &ovl_super_operations;//设置superblock操作对象
	sb->s_root = root_dentry;//设置superblock的根目录项
	sb->s_fs_info = ufs;//设置superblock的文件系统私有信息

	return 0;

out_free_oe:
	kfree(oe);
out_put_lower_mnt:
	for (i = 0; i < ufs->numlower; i++)
		mntput(ufs->lower_mnt[i]);
	kfree(ufs->lower_mnt);
out_put_workdir:
	dput(ufs->workdir);
	mntput(ufs->upper_mnt);
out_put_lowerpath:
	for (i = 0; i < numlower; i++)
		path_put(&stack[i]);
	kfree(stack);
out_free_lowertmp:
	kfree(lowertmp);
out_put_workpath:
	path_put(&workpath);
out_put_upperpath:
	path_put(&upperpath);
out_free_config:
	kfree(ufs->config.lowerdir);
	kfree(ufs->config.upperdir);
	kfree(ufs->config.workdir);
	kfree(ufs);
out:
	return err;
}
```

上面的函数主要完成以下几件事：

* 对挂载信息进行解析，获取lowerdir，upperdir和workdir的字符串数据，存入一个ovl_config结构中（ufs->config）；
* 对上面3个目录进行路径搜索，得到相应的路径upperpath、workpath以及stack[]（lowerdir可能不止一个）
* **设置overlayfs的文件系统私有信息ufs（struct ovl_fs）**：
	- 通过upperpath，克隆得到overlayfs upperdir的挂载点upper_mnt；
	- 在workdir下搜索创建work目录；
	- 通过stack[]，克隆得到overlayfs lowerdir的挂载点数组lower_mnt[]；
* **为overlayfs创建根目录项（root_dentry），并且设置root_dentry的文件系统特定数据oe**（struct ovl_entry）：
	- 设置oe的__upperdentry为upperdir对应的dentry；
	- 设置oe的lowerdir路径栈，路径的dentry为每个lowerdir对应的dentry，路径的挂载点为ufs中lower_mnt数组中相应的挂载点；
* 填充superblock（对应overlayfs文件系统）：
	- 设置superblock的文件系统私有信息为ufs；
	- 设置superblock的根目录项为root_dentry；
	- 设置superblock的其它信息(这点在函数的其它部分也存在)；

### 挂载实例分析

为了更直观的分析，考虑下面这样一个例子：

![](/img/in-post/post-kernel-overlayfs/mount_tree.png) 

在我的ubuntu上，有两个分区，挂载点分别为/和/home，文件系统都是ext4。每个文件系统都有对应的挂载点和superblock。

现在在/home/username/路径下，创建了4个overlayfs需要用到的目录：merged、upper、work、lower。upper和lower下分别创建了文件file1和file2。

可以通过这一节开头提到的挂载命令将upper和lower联合挂载到merged目录中，这个联合挂载的文件系统就是一个overlayfs，挂载点就是merged。

**填充super_block_3主要包括以下步骤：**

* 首先得到upper，lower，work的路径：

![](/img/in-post/post-kernel-overlayfs/mount_step_1.png) 

* 然后根据上一步得到的路径，设置overlayfs的文件系统私有信息ufs（struct ovl_fs）以及创建work目录；  
接着为overlayfs创建根目录项（root_dentry），并且设置root_dentry的文件系统特定数据oe;  
最后将这些信息存入到overlayfs的superblock中：

![](/img/in-post/post-kernel-overlayfs/mount_step_2.png) 



## 相关结构体和变量

### 变量ovl_fs_type

```c
/*overlayfs文件系统*/
static struct file_system_type ovl_fs_type = {
	.owner		= THIS_MODULE,
	.name		= "overlay",
	.mount		= ovl_mount,//挂载函数
	.kill_sb	= kill_anon_super,
};

/*文件系统类型结构体*/
struct file_system_type {
	const char *name;
	int fs_flags;
#define FS_REQUIRES_DEV		1 
#define FS_BINARY_MOUNTDATA	2
#define FS_HAS_SUBTYPE		4
#define FS_USERNS_MOUNT		8	/* Can be mounted by userns root */
#define FS_USERNS_DEV_MOUNT	16 /* A userns mount does not imply MNT_NODEV */
#define FS_USERNS_VISIBLE	32	/* FS must already be visible */
#define FS_RENAME_DOES_D_MOVE	32768	/* FS will handle d_move() during rename() internally. */
	struct dentry *(*mount) (struct file_system_type *, int,
		       const char *, void *);
	void (*kill_sb) (struct super_block *);
	struct module *owner;
	struct file_system_type * next;
	struct hlist_head fs_supers;

	struct lock_class_key s_lock_key;
	struct lock_class_key s_umount_key;
	struct lock_class_key s_vfs_rename_key;
	struct lock_class_key s_writers_key[SB_FREEZE_LEVELS];

	struct lock_class_key i_lock_key;
	struct lock_class_key i_mutex_key;
	struct lock_class_key i_mutex_dir_key;
};
```

### 结构体ovl_fs

```c
/* private information held for overlayfs's superblock */
struct ovl_fs {
	struct vfsmount *upper_mnt;
	unsigned numlower;
	struct vfsmount **lower_mnt;
	struct dentry *workdir;
	long lower_namelen;
	/* pathnames of lower and upper dirs, for show_options */
	struct ovl_config config;
};
```

### 结构体ovl_config

```c
struct ovl_config {
	char *lowerdir;
	char *upperdir;
	char *workdir;
};
```

### 结构体ovl_entry

```c
struct ovl_entry {
	struct dentry *__upperdentry;
	struct ovl_dir_cache *cache;
	union {
		struct {
			u64 version;
			bool opaque;
		};
		struct rcu_head rcu;
	};
	unsigned numlower;
	struct path lowerstack[];
};
```