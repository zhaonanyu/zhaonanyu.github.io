---
layout: post
title: "如何编写一个Linux VFS 文件系统模块-StaticFS-3"
description: "every geek wants a blog all by himself lol"
tags: [GitHub Pages, jekyll, static site generator]
comments: true
---
继续看下面三个值，`i_atime`，`i_mtime`和`i_ctime`，这些都可以设为0，就像`romfs`一样。

~~~c
i->i_atime = i->i_mtime = i->i_ctime = 0;
~~~

看起来没有人设定`i_blkbits`，但是他们都设定了`i_blksize`，我们也这样干。`i_blocks`看起来应该指的是inode使用的块的数量，但是他们貌似都不重要（至少对于romfs和ramfs是这样的）

~~~c
i->i_blksize = PAGE_CACHE_SIZE;
i->i_blocks = 0;
~~~

我使用了`PAGE_CACHE_SIZE`因为看起来其他的文件系统都青睐它。`i_version`，我相信应该是用于记录inode是在什么时候被引用的。我想它应该是被文件夹来使用的，这样以来文件夹如果被修改过你就可以通过它知道（因为这样`i_version`就不会和全局变量`event`相吻合）。看起来也没人会去初始化`i_bytes`。`i_sem`和`i_zombie`都是`struct semaphore`，他们被VFS在需要的时候锁定inode。
终于我们找到了些好东西。`i_op`和`i_fop` 分别指向`struct inode_operations`和`struct file_operations`。我们还没有写我们自己的但是但是我们知道我们会使用它们。（**We haven't written ours, yet, but we know what we'll call them.**）
这又引出了另外一个问题。我还没有仔细了解VFS如何区分文件和文件夹，但是我想：一个inode可以被认成文件或者文件夹吗？（**can an inode be treated as a file and a directory?**）在文件系统有同时对其使用cat以及cd命令的东西吗？我可以写一个zipfs模块让.zip可以通过unzip命令访问也可以使用cd命令来把它当做一个文件夹来访问吗？这是我最后的目标，所以我希望这是可以的。我后面可能会把文件夹b既当做文件夹又当做文件，来试试看。
现在，我需要一个`inode_operations`结构传递给文件的inode，还要一个给文件夹的inode。**Or do we? romfs fails to fill in that structure for file inodes.**有没有一套已有的函数来实现这个功能呢？**Let's go wild and populate the structure, regardless of what type of inode it is!**对于`i_fop`而言我们也需要一个这样结构，和romfs一样我们也是用`generic_ro_fops()`，并假设这样就业满足我们的需要了。

~~~c
i->i_op = &staticfs_inode_operations;
        switch (ino) {  
        case 0:
        case 2: i->i_fop = &staticfs_dir_operations; break;
        case 1:
        case 3: i->i_fop = &generic_ro_fops; break;
        }
~~~

现在只剩`struct inode`了，我们需要填充`i_mapping`和`i_data`，这两个都是`struct address_space`的成员。其中`i_mapping`是一个指针而`i_data`则不是。ramfs中在`i_mapping`中设定了`i_data`，而romfs中在`i_data`中设置了`a_ops`。问啥呢？
我从内核源码以及其他VFS模块中都没找到答案，于是我上网找。这是我从Alexander Viro那里得到的答案：

i_data是这个inode读写的页
i_mapping是我该向谁去找这个页

**everything outside of individual filesystems should use the latter.**他们只在inode拥有数据时才是一样的。CODA（或者其他在本地文件系统中拥有缓存的）都将`i_mapping`指向缓存有其inode的`i_data`上。同上，对于块设备如果我们在看页缓存的时候我们应该将页缓存与`struct block_device`配合着来看，因为有很多inode有相同的 主：次 设备号。 **IOW, ->i_mapping should be pointing 
to the same place for all of them.**

从上面的答案得知如果我们的文件系统并不直接保存一些数据，我们应该使用`i_mapping`，**which is a pointer, to point to the i_data structure of some other inode -- the one that it represents.**。如果真是这样的话，那么有很多文件系统都搞错了。但是我又能说啥呢？

~~~c
 i->i_mapping->a_ops = &staticfs_aops;
~~~
 
上面那么多都是为了去读inode。现在我们马上可以组装我们的`super_operations`结构了，我们马上就要完成我们在superblock上面的工作了。

~~~c
static struct super_operations staticfs_ops = {
  read_inode:staticfs_read_inode,
  statfs:staticfs_statfs,
};
~~~

我们把这个函数放在哪呢？我们把它放在`s_op`的`struct super_block`里面就在返回我们的`staticfs_read_super`的时候。我们看过`struct super_block`了吗？我想还没有呢。

~~~c
struct super_block {
        struct list_head        s_list;         /* Keep this first */
        kdev_t                  s_dev;
        unsigned long           s_blocksize;
        unsigned char           s_blocksize_bits;
        unsigned char           s_dirt;
        unsigned long long      s_maxbytes;     /* Max file size */
        struct file_system_type *s_type;
        struct super_operations *s_op;
        struct dquot_operations *dq_op;
        unsigned long           s_flags;
        unsigned long           s_magic;
        struct dentry           *s_root;
        struct rw_semaphore     s_umount;
        struct semaphore        s_lock;
        int                     s_count;
        atomic_t                s_active;

        struct list_head        s_dirty;        /* dirty inodes */
        struct list_head        s_locked_inodes;/* inodes being synced */
        struct list_head        s_files;

        struct block_device     *s_bdev;
        struct list_head        s_instances;
        struct quota_info s_dquot;      /* Diskquota specific options */

        union {
    ...
        } u;
        /*
         * The next field is for VFS *only*. No filesystems have any business
         * even looking at it. You had been warned.
         */
        struct semaphore s_vfs_rename_sem;      /* Kludge */

        /* The next field is used by knfsd when converting a (inode number based)
         * file handle into a dentry. As it builds a path in the dcache tree from
         * the bottom up, there may for a time be a subpath of dentrys which is not
         * connected to the main tree.  This semaphore ensure that there is only ever
         * one such free path per filesystem.  Note that unconnected files (or other
         * non-directories) are allowed, but not unconnected diretories.
         */
        struct semaphore s_nfsd_free_path_sem;
};
~~~

就像inode结构一样，我会在u里面隐藏文件系统相关的入口。那么我们要干啥呢？
`s_list`在VFS中被用于保存一个所有超级块的列表。这个事情就交给他了。`s_dev`** is the device that this filesystem is on. **这个貌似和我们没有什么关系，我想这个值是由内核中的VFS来设定的，所以我们就不管他了。

`s_blocksize`和`s_blocsize_bit`我们和ramfs一样把他们分别设为`PAGE_CACHE_SIZE`和`PAGE_CACHE_SHIFT`。

~~~c
static struct super_block *staticfs_read_super(struct super_block *sb, void *data, int silent) {
  sb->s_blocksize = PAGE_CACHE_SIZE;
  sb->s_blocksize_bits = PAGE_CACHE_SHIFT;
}
~~~

我们现在仍然不知道`data`和`silent`是干啥的，所以我们先忽略它。`s_dirt`是用来标记一个超级块是否“脏”了或者被修改过。我们永远不会写我们的超级块所以我们也不在乎。`s_maxbytes`无足轻重，因为我们不允许写操作。另外，在看了`/usr/src/linux/fs/super.c`后发现如果`alloc_super()`被调用来创建`struct super_block`然后传递给`staticfs_read_super()`的话那么`s_maxbytes`会变为`MAX_NON_LFS`所以我们假设不理睬它也是没问题的。`s_type`指向`struct file_system_type`应该是`init_staticfs_fs()`调用`register_filesystem()`时填充的。其他的文件系统都没有设定这个值，而它是VFS处理指定文件系统时需要使用的，所以它应该是默认就被设定好了的。

翻译于 2015/3/27