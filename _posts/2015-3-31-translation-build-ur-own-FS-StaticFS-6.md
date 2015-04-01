---
layout: post
title: "如何编写一个Linux VFS 文件系统模块-StaticFS-6"
description: "translation-build your own FS"
tags: [StaticFS, VFS, translation]
comments: true
---
#Files
下一步是文件，我们先来看看`struct file_operations`。

~~~c
/*
 * NOTE:
 * read, write, poll, fsync, readv, writev can be called
 *   without the big kernel lock held in all filesystems.
 */
struct file_operations {
        struct module *owner;
        loff_t (*llseek) (struct file *, loff_t, int);
        ssize_t (*read) (struct file *, char *, size_t, loff_t *);
        ssize_t (*write) (struct file *, const char *, size_t, loff_t *);
        int (*readdir) (struct file *, void *, filldir_t);
        unsigned int (*poll) (struct file *, struct poll_table_struct *);
        int (*ioctl) (struct inode *, struct file *, unsigned int, unsigned long);
        int (*mmap) (struct file *, struct vm_area_struct *);
        int (*open) (struct inode *, struct file *);
        int (*flush) (struct file *);
        int (*release) (struct inode *, struct file *);
        int (*fsync) (struct file *, struct dentry *, int datasync);
        int (*fasync) (int, struct file *, int);
        int (*lock) (struct file *, int, struct file_lock *);
        ssize_t (*readv) (struct file *, const struct iovec *, unsigned long, loff_t *);
        ssize_t (*writev) (struct file *, const struct iovec *, unsigned long, loff_t *);
        ssize_t (*sendpage) (struct file *, struct page *, int, size_t, loff_t *, int);
        unsigned long (*get_unmapped_area)(struct file *, unsigned long, unsigned long, unsigned long, unsigned l\
ong);
};
~~~

注意，我同样将该结构的注释也拷过来了。大内核锁，或称BKL，在源码中被不断提起。就我的理解而言，这个锁会放置任何其他同时发生的事情。可能是所有事情。我们目前不用去了解BKL，先把他放在一边。

如果你想的起来我们在superblock中的代码，我们为文件操作实现了两种结构——一种为文件准备另一种为文件夹准备。这个设计和ramfs比较相似，但是和romfs却不同，因为它只有一个文件夹（假设默认的设定对于romfs已经足够好了）

但是，当然我会关系默认设定到底都是些什么。ramfs和romfs中的很多值都不是在上面的结构中定义的，这说明了什么？一眼看去对于已经定义的，最多被使用的是`generic_()`。我之前说我们稍后再去看他们，或许现在先看看？

额，如果你看这玩意看的胃痛，先去看看在`/usr/src/linux/mm/filemap.c`中的`generic_file_read()`和`generic_file_write()`。我先假设他们都是管用的...别问那么多

哼，废话到此为止。我们来看看这个结构。大多数函数都对于C程序员来说应该很熟悉，因为他们的名字和文件层的名字一直。处于这个原因我们就没必要去每个都仔细看看了。**Let's just concentrate on what romfs does for not, because we're very similar to it.**

romfs只将其中的两个字段进行了填充，read和readdir。read指向了`generic_read_dir()`，但是为啥？如果它被舍我空会怎样？我们来看看`generic()`函数。

~~~c
ssize_t generic_read_dir(struct file *filp, char *buf, size_t siz, loff_t *ppos)
{
        return -EISDIR;
}
~~~

喔~这个就是当你试图去向读文件一样去读文件夹将会发生的事情。你会得到一个错误`EISADIR`。这是否说明我如果在这里设置了一个自己的处理函数，我就可以返回文件夹里面的“内容”，而且它仍然像个文件夹？这个也是我早先提出的问题，文件夹是否是可以读的也可以是可以进入的？不错~我们等会儿再过来看看这个。

我现在还有一个疑问：如果我们不把它填充进read字段？NULL代表了什么？它会不反悔任何值吗？VFS会调用其他函数来处理这个问题嘛？这个听起来很有趣，我需要先改一下ramfs模块然后使用cat一个文件夹来看看会有什么结果。我们目前先假设，“嘿，这是个文件夹”的功能必须被提供，不存在默认的处理方式。

翻译于 2015/3/31