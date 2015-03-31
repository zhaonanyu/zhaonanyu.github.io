---
layout: post
title: "如何编写一个Linux VFS 文件系统模块-StaticFS-1"
description: "translation-build your own FS"
tags: [StaticFS, VFS, translation]
comments: true
---


标签：VFS FS StaticFS

---

#StaticFS
`StaticFS`是我制作的一个以测试为目的的无用的文件系统。这个文件系统看起来是这个样子的

~~~
/--,
    +--a
    |
    `--b
        |
        `--c
~~~

可以看到有一个根节点，根节点下面有一个文件叫做"a"（其中的内容将会是"These are the characters in file a."；同样在根节点下面还有一个文件夹"b"；在文件夹"b"下面有一个文件"c"，"c"中的内容为"These are the characters in file c.".

这个文件系统并不是可写的，是静态的，所以这个示例是为了检测我是否理解文件系统的读取过程。**I'll worry about writing in a different test filesystem, once I think I understand it a little more.**

---

#初始化
我会以我学习（go through)`ramfs`一样的流程来进行，首先是文件系统如何通过内核初始化其自身。

~~~c
static DECLARE_FSTYPE(staticfs_fs_type, "staticfs", staticfs_read_super, FS_LITTER);

static int __init init_staticfs_fs(void)
{
        return register_filesystem(&staticfs_fs_type);
}

static void __exit exit_staticfs_fs(void)
{
        unregister_filesystem(&staticfs_fs_type);
}

module_init(init_staticfs_fs)
module_exit(exit_staticfs_fs)

MODULE_LICENSE("GPL");
~~~

这基本上就是从`ramfs`模块拷贝粘贴过来的，创建一个结构来定义我们的操作系统（`FS_LITTER`在`/usr/src/linux/include/linux/fs.h`中定义的，**tells VFS that this filesystem should "litter" the dentry cache with its entries.**)

---

#超级块-Superblocks
现在我们要定义一个结构其功能是告诉`VFS`如何处理我们的`superblock`。在我们前面了解`inodes`时，我们看了`struct super_operations`的定义，同时也看到了`ramfs`同时使用了`statfs`和`put_inode`.(**and saw that ramfs implemented all of two of them, statfs and put_inode**).在了解其他支持Linux的文件系统后我了认识到几乎所有人都使用了`statfs`，所以我们来看一下`struct statfs`在`/usr/src/linux/include/asm-i386/statfs.h`中的定义：

~~~c
struct statfs {
        long f_type;
        long f_bsize;
        long f_blocks;
        long f_bfree;
        long f_bavail;
        long f_files;
        long f_ffree;
        __kernel_fsid_t f_fsid;
        long f_namelen;
        long f_spare[6];
};
~~~

`f_type`时文件系统中一个神奇的数字。我们也会有这样一个东西（We'll supply one of those.）。`f_bsize`是指块的大小，看起来所有人都有这样一个东西，那么我们也来一个。`f_blocks`可能是文件系统中块的总数，这个我不太确定。`f_bfree`指的是空闲的块数量，但是`f_bavail`是啥？许多其他的文件系统都将其设定为`f_bfree`，那么这有什么区别呢？我以后再去了解。
`f_files`是文件的数量，`f_ffree`是指剩余文件的数量（**and f_ffree the number of files available**）。`f_fsid`只有在几个现有的文件系统中被设定。他是用来做什么的？我不太清楚`。f_namelen`貌似指的是文件名最大长度。看起来所有人都设定了这个，我们也这样来。`f_spare`看起来都没有人用它，但是它的存在自然有它的道理，**providing space for a pointer to an extension structure or six.**

那么我们需要编写一个`statfs`函数。看起来大家都要来一个，而且`VFS`也需要这个东西，所以我们也随个大流。

~~~c
#define STATICFS_MAGIC 0x61626364

static int staticfs_statfs(struct super_block *sb, struct statfs *buf)
{
        buf->f_type = STATICFS_MAGIC;
        buf->f_bsize = PAGE_CACHE_SIZE;
        buf->f_namelen = 1;
        return 0;
}
~~~

我根据我们的文件系统名称（StaticFS）定义了`STATICFS_MAGIC`，其值为"abcd"。我使用了`PAGE_CACHE_SIZE`宏，因为`ramfs`也是这样的，我查了一下这个宏的大小为`4096`.我使用了`1`作为`f_namelen`因为我就是这么任性（I used 1 for f_namelen because I wanted to be different. Or difficult.）。

那么，我们已经编写了一个函数了。还剩下一个其他文件系统都使用的`put_inode`，它的功能是释放正在使用的`inode`。在`ramfs`中它使用了`force_delete()`，其他的很多文件系统也是这样干的，查看了`/usr/src/linux/fs/inode.c`后似乎设置

~~~c
inode->i_nlink = 0
~~~

就可以删除`inode`。那么为什么`romfs`不定义一个这样的值？那么是因为开发者懒得这样做就让没有用的`inode`在`VFS`里面晃来晃去？似乎其他的文件系统也都是这么做的，但是，既然我是个装逼的人我当然要定义一个这样的功能。（since only 13 of the 40ish that I'm looking at have defined a function for this.）
深入了解后，我想我知道是怎么回事了。在`ramfs`和`romfs`中有两个我没有看过的函数：`iput()`（在ramfs中)以及`iget()`（在romfs）中。

*翻译于2015/3/25*

---

`iget()`，在romfs是这样叫的，**looks for an inode in the inode cache based on a given inode number (formed in romfs from the offset of the start of the filesystem in the file used as the romfs).** 如果

