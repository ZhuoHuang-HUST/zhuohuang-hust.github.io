---
layout:     post
title:      "内核OverlayFS—目录搜索"
date:       2017-11-10 2:51:00
author:     "ArKing"
categories: Linux内核
tags:
    - 文件系统
---

* content
{:toc}
 
> **内核版本4.4.68**

本文通过strace命令结合OverlayFS内核模块中加入的调试信息，分析ls命令列举出OverlayFS联合挂载目录下文件的过程，着重分析OverlayFS部分。

我的磁盘划了3个分区：

![](/img/in-post/post-kernel-overlayfs/dir-op1-df.png)

三个分区的文件系统都是ext4，分别挂载在/,/home,/boot目录下。

目录树大概是这个样子（为了节省空间，实际上upper和lower目录下文件不止1个）：

![](/img/in-post/post-kernel-overlayfs/dir-op1-dir_tree.png)

本文主要就是分析执行命令```ls merged```时，OverlayFS做了什么。




## 从系统调用开始

strace命令可以查看一个命令调用到了哪些系统调用，执行```strace ls merged```,结果如下：

![](/img/in-post/post-kernel-overlayfs/dir-op1-strace.png)

这里只截取了关键的几个系统调用，openat()打开目录merged，返回相应的描述符。getdents()根据openat()返回的描述符对目录进行搜索。因此，overlayfs中的目录搜索主要完成两件事：

* **打开文件**
* **搜索目录**。

## 打开目录

通过openat()系统调用进入内核后，会执行下列代码：

```c
SYSCALL_DEFINE4(openat, int, dfd, const char __user *, filename, int, flags,
		umode_t, mode)
{
	if (force_o_largefile())
		flags |= O_LARGEFILE;

	return do_sys_open(dfd, filename, flags, mode);
}
```

进一步调用do\_sys\_open():

```c
long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
	struct open_flags op;
	int fd = build_open_flags(flags, mode, &op);
	struct filename *tmp;

	if (fd)
		return fd;

	tmp = getname(filename);//1.根据filename得到一个对应的struct filename变量
	if (IS_ERR(tmp))
		return PTR_ERR(tmp);

	fd = get_unused_fd_flags(flags);//2.获取一个未使用的文件描述符
	if (fd >= 0) {
		struct file *f = do_filp_open(dfd, tmp, &op);//3.打开文件
		if (IS_ERR(f)) {
			put_unused_fd(fd);
			fd = PTR_ERR(f);
		} else {
			fsnotify_open(f);
			fd_install(fd, f);
		}
	}
	putname(tmp);
	return fd;
}
```

这个函数主要完成3个工作，已经注释在代码块中，重点在第三步，通过调用do\_flip\_open函数，打开目录merged，并返回一个表示merged的file，后续对merged目录的读取和修改都将使用到这个file：

```c
struct file *do_filp_open(int dfd, struct filename *pathname,
		const struct open_flags *op)
{
	struct nameidata nd;//1.初始化一个nameidata变量
	int flags = op->lookup_flags;
	struct file *filp;

	set_nameidata(&nd, dfd, pathname);//2.保存查找信息，更新current相应字段
	filp = path_openat(&nd, op, flags | LOOKUP_RCU);//3.路径查找，打开文件
	if (unlikely(filp == ERR_PTR(-ECHILD)))
		filp = path_openat(&nd, op, flags);
	if (unlikely(filp == ERR_PTR(-ESTALE)))
		filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
	restore_nameidata();
	return filp;
}
```

为了更多的关注在overlayfs部分，这里不再继续深入函数调用栈。这里忽略一些错误检查以及复杂路径查询的处理细节，概括性的描述一下该函数完成的操作：

* **初始化一个nameidata变量nd**。
	* 这个变量在路径查找中很重要，它记录了查找信息、保存了查找起始路径。在路径的每一个分量的查找中，它会保存当前的结果。对于一般路径名查找，在查找结束时，它会包含查询结果的信息，对于父路径名查找，在查找结束时，它会包含最后一个分量所在目录的信息
* **调用函数set_nameidata()将查找信息(查找字符串等)保存到nd中，并更新current进程的nameidata字段**；
* **调用函数get\_empty\_filp()得到一个空文件file**
* **调用函数path_init()初始化nd变量中的路径字段（设置查找起始路径），根据路径设置inode字段**：
	* 如果查找字符串以‘/’开头，说明是一个绝对路径，因此根据current进程的fs字段获取root路径并设置nd中的root字段和path字段。如果不是以'/'开头，说明是一个相对路径，则根据current进程获取当前工作路径并设置nd中的path字段。根据路径中的dentry获取根inode并设置nd的inode字段。
* **调用函数link\_path\_walk()执行路径查找，找到最后一个分量所在的目录，此时最后一个分量还未解析**:
	*  通过nd的inode字段检查当前目录的是否具有可执行权限(不然无法遍历)，不具有则返回一个错误。跳过路径字符串开头的'/'，获取查找字符串下一个'/'前的字符串，也就是得到当前查找分量字符串。对这个分量字符串执行hash操作，然后存入nd中的last字段。以path字段的dentry为父目录项，根据last字段进行目录项查找(先会查找目录项高速缓存，没有则会通过real_lookup()调用inode的lookup方法从磁盘读取目录)，得到一个和分量字符串对应的目录项。根据这个目录项和父目录下的挂载点更新nd的path路径及inode。重复上述步骤，直到nd的last字段中保存了最后一个分量的信息(此时，最后一个分量还未解析)。
* **调用函数do_last()完成最后路径分量的解析，结果会保存在nd中，使用解析结果填充第3步得到的file中的字段，打开文件**：
	*  根据nd的path字段(此时，已经记录了最后一个分量的路径)中的dentry获取文件对应的inode，然后根据inode设置file的字段，其中包括f\_op字段，得到文件操作接口。调用file->f_op->open()打开文件。

到这里，已经得到了表示merged目录（**实际上说是OverlayFS中的/目录更准确**，因为merged是OverlayFS的挂载点，也可以说是安装点，所以路径查找时会做一个转换）的文件file，并且对file执行了open操作。但是open操作到底具体做了什么？回到上面，我们知道open函数是f\_op中的一个函数接口，而f\_op又由inode的i\_fop字段得到，因此i_fop应该在创建内存inode结构时指定，也就是open操作在为OverlayFS根目录创建内存inode结构时确定，在OverlayFS中，内存inode创建由函数ovl\_new\_inode()完成：

```c
struct inode *ovl_new_inode(struct super_block *sb, umode_t mode,
			    struct ovl_entry *oe)
{
	struct inode *inode;

	inode = new_inode(sb);//在sb下分配一个inode
	if (!inode)
		return NULL;

	inode->i_ino = get_next_ino();//设置inode号
	inode->i_mode = mode;//设置mode
	inode->i_flags |= S_NOATIME | S_NOCMTIME;//设置flag

	mode &= S_IFMT;
	//根据inode类型设置inode操作
	switch (mode) {
	case S_IFDIR:
		inode->i_private = oe;
		inode->i_op = &ovl_dir_inode_operations;
		inode->i_fop = &ovl_dir_operations;//设置inode file操作
		break;

	case S_IFLNK:
		inode->i_op = &ovl_symlink_inode_operations;
		break;

	case S_IFREG:
	case S_IFSOCK:
	case S_IFBLK:
	case S_IFCHR:
	case S_IFIFO:
		inode->i_op = &ovl_file_inode_operations;
		break;

	default:
		WARN(1, "illegal file type: %i\n", mode);
		iput(inode);
		inode = NULL;
	}

	return inode;
}
```

当创建的inode是一个目录的inode时，指定了i\_fop字段，从这里可以知道，OverlayFS中，目录的读写操作，定义在ovl_dir\_operations中：

```c
const struct file_operations ovl_dir_operations = {
	.read		= generic_read_dir,
	.open		= ovl_dir_open,
	.iterate	= ovl_iterate,
	.llseek		= ovl_dir_llseek,
	.fsync		= ovl_dir_fsync,
	.release	= ovl_dir_release,
};
```

所以open接口对应的函数为ovl\_dir\_open():

```c
static int ovl_dir_open(struct inode *inode, struct file *file)
{
	struct path realpath;
	struct file *realfile;
	struct ovl_dir_file *od;
	enum ovl_path_type type;

	od = kzalloc(sizeof(struct ovl_dir_file), GFP_KERNEL);//1.分配一个ovl_dir_file变量od
	if (!od)
		return -ENOMEM;

	type = ovl_path_real(file->f_path.dentry, &realpath);//2.获取file路径类型以及真实路径
	realfile = ovl_path_open(&realpath, file->f_flags);//3.根据路径类型打开真实文件
	if (IS_ERR(realfile)) {
		kfree(od);
		return PTR_ERR(realfile);
	}
	//4.设置变量od的字段
	od->realfile = realfile;
	od->is_real = !OVL_TYPE_MERGE(type);
	od->is_upper = OVL_TYPE_UPPER(type);
	file->private_data = od;

	return 0;
}
```

函数完成的主要操作已经注释在代码中，这里提一下第2步和3步。  
所谓真实路径和真实文件，如果一个文件存在upper层，由于OverlayFS的特性，上层会覆盖下层，所以在联合挂载目录merged中，这个文件的真实路径就是对应upper层中文件的路径，真实文件就是upper层中文件的file结构。如果一个文件只存在lower层，则真实路径和真实文件就是lower层中对应文件的路径和file结构。这里file代表的是merged目录，也就是OverlayFS中的根(/)目录，所以实际上对应的upper层的文件是upper目录，因此第3步实际上是获取upper目录对应的file结构并调用file的f_op字段中的open函数(这个函数自然是ext4在为upper目录分配inode时指定)打开upper目录。

**总结起来就是，ls命令第一步通过路径查找，获取merged目录(OverlayFS中的/目录)的file内存表示，接着获取根目录的真实文件(upper目录)，打开upper目录。然后在打开操作中分配了一个ovl\_dir\_file类型的变量，这个变量记录了OverlayFS中一个文件特有的信息，最后设置file的private_data指向这个变量，以便后面利用**：

![](/img/in-post/post-kernel-overlayfs/dir-op1-open.png)

## 目录搜索

第一步已经得到了OverlayFS根目录对应的file结构，并且执行打开操作设置了文件系统特有信息，但是并没有描述怎么在该目录下显示出upper和lower层联合挂载的文件，所以这部分应该是由getdents()系统调用实现。

通过系统调用getdents()进入内核后会执行如下代码：

```c
SYSCALL_DEFINE3(getdents, unsigned int, fd,
		struct linux_dirent __user *, dirent, unsigned int, count)
{
	struct fd f;
	struct linux_dirent __user * lastdirent;
	struct getdents_callback buf = {
		.ctx.actor = filldir,
		.count = count,
		.current_dir = dirent
	};
	int error;

	if (!access_ok(VERIFY_WRITE, dirent, count))
		return -EFAULT;

	f = fdget(fd);//1.根据文件描述符号获取fd，里面包含了对应的文件file
	if (!f.file)
		return -EBADF;

	error = iterate_dir(f.file, &buf.ctx);//2.目录搜索
	if (error >= 0)
		error = buf.error;
	lastdirent = buf.previous;
	if (lastdirent) {
		if (put_user(buf.ctx.pos, &lastdirent->d_off))
			error = -EFAULT;
		else
			error = count - buf.count;
	}
	fdput(f);
	return error;
}

struct fd {
	struct file *file;
	unsigned int flags;
};
```

根据传入OverlayFS根(/)目录（后面都用“根目录”
表示）的文件描述符参数获取对应的文件file，然后调用iterate_dir()函数执行搜索：

```c
int iterate_dir(struct file *file, struct dir_context *ctx)
{
	struct inode *inode = file_inode(file);//file->f_inode，获取file对应的inode号
	int res = -ENOTDIR;
	if (!file->f_op->iterate)
		goto out;

	res = security_file_permission(file, MAY_READ);
	if (res)
		goto out;

	res = mutex_lock_killable(&inode->i_mutex);
	if (res)
		goto out;

	res = -ENOENT;
	if (!IS_DEADDIR(inode)) {
		ctx->pos = file->f_pos;//根据文件读写位置更新上下文的pos字段
		res = file->f_op->iterate(file, ctx);//调用f_op中的iterate接口
		file->f_pos = ctx->pos;//iterate会更新ctx的pos字段，使用其更新文件的读写位置
		fsnotify_access(file);
		file_accessed(file);
	}
	mutex_unlock(&inode->i_mutex);
out:
	return res;
}
EXPORT_SYMBOL(iterate_dir);
```

该函数进一步调用file->f\_op中的接口iterate,在前面已经知道，目录文件的f_op操作为ovl\_dir\_operations，接口iterate对应的函数为ovl\_iterate:

```c
static int ovl_iterate(struct file *file, struct dir_context *ctx)
{
	struct ovl_dir_file *od = file->private_data;//获取OverlayFS特有信息
	struct dentry *dentry = file->f_path.dentry;//文件对应的dentry
	struct ovl_cache_entry *p;

	if (!ctx->pos)
		ovl_dir_reset(file);

	if (od->is_real)//说明不是一个合并目录，当目录只存在upper层或者lower层时条件会成立
		return iterate_dir(od->realfile, ctx);

	if (!od->cache) {//在前一节我们知道这个字段为NULL,条件成立
		struct ovl_dir_cache *cache;

		cache = ovl_cache_get(dentry);//获取得到一个ovl_dir_cache
		if (IS_ERR(cache))
			return PTR_ERR(cache);

		od->cache = cache;//设置od的cache字段为前面得到的cache
		ovl_seek_cursor(od, ctx->pos);//根据上下文的位置设置od的cursor字段，指向链表对应的位置
	}
	
	//遍历，处理链表的每一项
	while (od->cursor != &od->cache->entries) {
		p = list_entry(od->cursor, struct ovl_cache_entry, l_node);
		if (!p->is_whiteout)
			if (!dir_emit(ctx, p->name, p->len, p->ino, p->type))
				break;
		od->cursor = p->l_node.next;
		ctx->pos++;
	}
	return 0;
}
```

**因此总的来说，当对OverlayFS中的一个目录进行搜索时，首先通过文件描述符得到file，进一步获得文件的OverlayFS特有信息。然后根据特有信息获取cache，如果cache为NULL，则调用函数ovl\_cache\_get先获得cache。这个cache中包含一个链表，通过对链表中的每一项进行处理，完成对目录的检索。**

所以当对根目录(联合挂载目录)进行检索时，这个链表应该记录了联合挂载目录下每一个文件的信息。分析函数ovl\_cache\_get()如何获得一个cache:

```c
static struct ovl_dir_cache *ovl_cache_get(struct dentry *dentry)
{
	int res;
	struct ovl_dir_cache *cache;

	//获取dentry目录项特有信息(dentry->d_fsdata,它指向一个ovl_entry类型的变量)中的cache
	cache = ovl_dir_cache(dentry);
	if (cache && ovl_dentry_version_get(dentry) == cache->version) {//cache存在并且有效
		cache->refcount++;
		return cache;
	}
	ovl_set_dir_cache(dentry, NULL);//设置dentry目录项特有信息中的cache为NULL

	cache = kzalloc(sizeof(struct ovl_dir_cache), GFP_KERNEL);//分配一个cache
	if (!cache)
		return ERR_PTR(-ENOMEM);

	cache->refcount = 1;
	INIT_LIST_HEAD(&cache->entries);//初始化链表头

	res = ovl_dir_read_merged(dentry, &cache->entries);
	if (res) {
		ovl_cache_free(&cache->entries);
		kfree(cache);
		return ERR_PTR(res);
	}

	cache->version = ovl_dentry_version_get(dentry);
	ovl_set_dir_cache(dentry, cache);//设置dentry目录项特有信息中的cache

	return cache;
}
```

这个函数的参数为OverlayFS根目录的目录项，函数首先通过判断是否能根据这个根目录项检索到cache，如果能并且cache有效则直接返回。如果不能则分配一个，初始化cache中的链表，并且调用函数ovl\_dir\_read\_merged向链表添加节点：

```c
static int ovl_dir_read_merged(struct dentry *dentry, struct list_head *list)
{
	int err;
	struct path realpath;
	struct ovl_readdir_data rdd = {
		.ctx.actor = ovl_fill_merge,
		.list = list,
		.root = RB_ROOT,
		.is_merge = false,
	};
	int idx, next;

	for (idx = 0; idx != -1; idx = next) {
		next = ovl_path_next(idx, dentry, &realpath);

		if (next != -1) {
			err = ovl_dir_read(&realpath, &rdd);
			if (err)
				break;
		} else {
			/*
			 * Insert lowest layer entries before upper ones, this
			 * allows offsets to be reasonably constant
			 */
			list_add(&rdd.middle, rdd.list);
			rdd.is_merge = true;
			err = ovl_dir_read(&realpath, &rdd);
			list_del(&rdd.middle);
		}
	}
	return err;
}
```

这个函数创建一个ovl\_readdir\_data类型的变量rdd，然后通过一个for循环遍历，获取根目录每一层的路径（在这里实际上就是获取upper目录的路径和lower目录的路径），然后对每一层调用一个ovl\_dir\_read函数，传入rdd作为参数。由于rdd的list字段指向链表，所以函数ovl\_dir\_read()实现了每一层向链表中添加节点：

```c
static inline int ovl_dir_read(struct path *realpath,
			       struct ovl_readdir_data *rdd)
{
	struct file *realfile;
	int err;

	realfile = ovl_path_open(realpath, O_RDONLY | O_DIRECTORY);//通过该层路径获得该层的文件
	if (IS_ERR(realfile))
		return PTR_ERR(realfile);

	rdd->first_maybe_whiteout = NULL;
	rdd->ctx.pos = 0;
	do {
		rdd->count = 0;
		rdd->err = 0;
		err = iterate_dir(realfile, &rdd->ctx);//转为对该层进行检索
		if (err >= 0)
			err = rdd->err;
	} while (!err && rdd->count);

	if (!err && rdd->first_maybe_whiteout)
		err = ovl_check_whiteouts(realpath->dentry, rdd);

	fput(realfile);

	return err;
}
```

如果你还能大概回忆起这一节开始到这的大概步骤，应该会记得在对OverlayFS根目录(merged目录)检索时，就调用了iterate_dir函数。刚刚我们又提到，函数ovl\_dir\_read()实现每一层向链表中添加节点。因此这个函数先是得到该层的文件(即upper目录的file，lower目录的file)，然后转为对该层文件进行检索。但是观察本文开头的目录树，实际上upper和lower都是ext4下的目录，调用ext4的目录检索接口为何能修改一个OverlayFS结构中的链表？这就是rdd的用途了，在上一个函数中，设置了rdd->ctx->actor为ovl\_fill\_merge。所以在ext4目录检索的过程中，通过ctx->actor实现了对OverlayFS中函数的调用，进一步实现向链表中添加节点：

```c
static int ovl_fill_merge(struct dir_context *ctx, const char *name,
			  int namelen, loff_t offset, u64 ino,
			  unsigned int d_type)
{
	struct ovl_readdir_data *rdd =
		container_of(ctx, struct ovl_readdir_data, ctx);

	rdd->count++;
	if (!rdd->is_merge)
		return ovl_cache_entry_add_rb(rdd, name, namelen, ino, d_type);
	else
		return ovl_fill_lower(rdd, name, namelen, offset, ino, d_type);
}
```

这个函数的逻辑还是很清晰的，首先根据ctx还原得到rdd，由于能够还原rdd，因此能通过rdd的list字段访问链表。如果不是最后一层，则调用ovl\_cache\_entry\_add\_rb()向链表中添加节点，否则调用ovl\_fill\_lower()添加。需要提一点的是，rdd的root字段指向一颗红黑树的根节点，函数ovl\_cache\_entry\_add\_rb()在想链表中添加节点的同时会向红黑树中添加节点，而函数ovl\_fill\_lower()只会向链表中添加节点。感兴趣可以具体去看看源码，这里就不再细致分析这个添加过程了，直接看图吧：

![](/img/in-post/post-kernel-overlayfs/dir-op1-iterator.png)

联合挂载到merged目录下的每一个文件都有一个对应的ovl\_cache\_entry。通过l\_node字段添加到链表中。通过访问链表，可以还原这个ovl\_cache\_entry结构，访问文件的大小，inode号，名称等信息。

**总的来说，整个搜索过程看似很复杂，其实也很简单。就是围绕着cache和cache中的链表进行。如果没有cache，则分配，然后将检索转化为对每一层的检索。检索过程中借助rdd完成向链表中添加节点。**