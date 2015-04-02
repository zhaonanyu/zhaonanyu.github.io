---
layout: post
title: "如何编写一个Linux VFS 文件系统模块-StaticFS-2"
description: "translation-build your own FS"
tags: [StaticFS, VFS, translation]
comments: true
---

`iget()`，在romfs是这样叫的，**looks for an inode in the inode cache based on a given inode number (formed in romfs from the offset of the start of the filesystem in the file used as the romfs).** 如果没有找到这样一个inode，那么就会创建一个新的inode，**incrementing usage inside it if necessary.**。这个功能是由`romfs_read_super()`函数在文件系统中的超级块被初始化时完成。
`iput()`，在`ramfs`中，和`iget()`是在同一个地方。`ramfs`相比`romfs`通过另一种方式获得inode；他会调用`ramfs_get_inode()`然后在其中调用`new_inode()`（其调用了`get_empty_inode()`而这个函数又调用了`alloc_inode()`）。在`romfs`中则是调用了`iget()`，`iget()`调用了`iget4()`然后调用了`get_new_inode()`然后又调用了`alloc_inode()`。**iput() is called if**，处于什么原因，这个inode不能被指定为跟i-节点。
看起来挺有趣，但是这并没有帮助我们了解作为`put_inode`的`force_delete()`——`struct super_operations`结构的入口，而在`romfs`中他们并没有这样做。这是因为`romfs`的inode 节点是内部创建的，作为`iget()`的成果，那么`ramfs`到底是什么时候创建的自己的`new_inode()`呢？嘛，总之如上文中描述的`romfs ramfs`都用同样的方法创建了inode。
那么到这里我不准备对付`put_inode`。因为这对于`romfs`已经足够了，对于我们的`StaticFS`也足够。

继续往下看`struct super_operations`，我们还需要实现什么？`dirty_inode`，`write_inode`，`delete_inode`，`write_super`以及`write_super_lockfs`这些都是用来进行写操作的，但是我们是个静态系统文件系统（所以不用理睬）。`clear_inode`还有`unlockfs`这个我不清楚。`put_super`看起来是保存在`u`中的用于清理任何模块相关的数据。`umount_begin`现在只被应用于NFS了，而且大概也不适用了。我们会忽略掉那些`knfsd`的方法，以及其他的杂牌方法。到了这里我相信我们只用实现`read_inode`了。
我们还从来没有见过一个inode读取函数，所以我们不太清楚它到底需要做些什么。鉴于他是一个超级块的函数，所以我们应该以超级块的视角来寻找我们所需要的信息。我们可以获得到一个inode作为参数，所以我们不用创建一个。这同时也代表了这个inode中会包含了一些信息，** so we know which inode the caller is interested in**。我会通过查看`romfs`的代码来一窥究竟。
哼~。那么他到底是做什么的呢？`i->ino`中包含了我们所感兴趣的inode编号。其他的都位于`struct inode`中，那么我们先来看一下：

~~~c
struct inode {
        struct list_head        i_hash;
        struct list_head        i_list;
        struct list_head        i_dentry;

        struct list_head        i_dirty_buffers;
        struct list_head        i_dirty_data_buffers;

        unsigned long           i_ino;
        atomic_t                i_count;
        kdev_t                  i_dev;
        umode_t                 i_mode;
        nlink_t                 i_nlink;
        uid_t                   i_uid;
        gid_t                   i_gid;
        kdev_t                  i_rdev;
        loff_t                  i_size;
        time_t                  i_atime;
        time_t                  i_mtime;
        time_t                  i_ctime;
        unsigned int            i_blkbits;
        unsigned long           i_blksize;
        unsigned long           i_blocks;
        unsigned long           i_version;
        unsigned short          i_bytes;
        struct semaphore        i_sem;
        struct semaphore        i_zombie;
        struct inode_operations *i_op;
        struct file_operations  *i_fop; /* former ->i_op->default_file_ops */
        struct super_block      *i_sb;
        wait_queue_head_t       i_wait;
        struct file_lock        *i_flock;
        struct address_space    *i_mapping;
        struct address_space    i_data;
        struct dquot            *i_dquot[MAXQUOTAS];
        /* These three should probably be a union */
        struct list_head        i_devices;
        struct pipe_inode_info  *i_pipe;
        struct block_device     *i_bdev;
        struct char_device      *i_cdev;

        unsigned long           i_dnotify_mask; /* Directory notify events */
        struct dnotify_struct   *i_dnotify; /* for directory notifications */

        unsigned long           i_state;

        unsigned int            i_flags;
        unsigned char           i_sock;

        atomic_t                i_writecount;
        unsigned int            i_attr_flags;
        __u32                   i_generation;
    union {
        ...
    } u;
};
~~~

我贴上了`u union`中的与文件系统相关的内容。我想我们可以忽略开头的五个入口，他们都是`struct list_head`变量，我相信他们都是VFS所需要的东西而不是我们需要的。我们回台再来看。

那么，我的得到的`i_ino`在初始化时包含了一个inode number。而inode cache使用了`i_count`（我们在前面的`iput()`以及`iget()`中都见过了）。`i_dev`？许多VFS模块都将这个值设定为`sb->s_dev`，** so this is probably the device on which the filesystem is found.**。不同的inode可能存在于不同的设备中甚至是他们没有被使用，特别是对于`/dev`文件系统而言。好吧，我迟一些会看看`kdev_t`，来看看其中的值。
`i_mode`是inode上面的权限。终于我们看到了我们想看的东西！我想我要把我们的staticfs的所有文件以及文件夹的权限都设定为444/555，所以所有人都可已去读他们，当然他们不能去写它。所有的文件夹也都可以导航。那么如果我们的文件以及文件夹出现write flags即使我们根本没有实现任何写功能会出现什么事情？我们应该去试试吗？当然！我们来把我们的文件看起来可写，然后试试看。
我们来开始写一个功能。

~~~c
static void staticfs_read_inode(struct inode *i) {
        int ino;

        ino = i->i_ino;
        i->i_mode = S_IRWXUGO;
~~~

`S_IRWXUGO`是一个在`/usr/src/linux/include/linux/stat.h`中的掩码，他会将flags设定为777。我们来将`i_nlink`设定为1，因为我们的每个inode都只有一个引用。我们再来将`i_iuid`以及`i_igid`设定为0这样其属主就是root了。为啥不呢？

~~~c
    i->i_nlink = 1; 
        i->i_uid = 0;
        i->i_gid = 0;
~~~

我们来看看，`i_rdev`应该是和`i_dev`绑定的，所以我们保留它。我们文件系统中的两个文件都包含了35个字符。**The directories, well, it's up to us to say what a directory size represents.**。ext2返回字节的数目，显然是以4k的块为单位并用于存储文件。

目前看来还挺有意思。那么"."以及".."是怎么工作的呢？我应该实现他们作为我的文件系统的入口？还是他们是一种特别的文件夹存在于别的地方？我不清楚。**我们就不要他们了**，来看看会有什么情况。这样一来根文件夹的大小为2，a的大小为35，b的大小为1，c的大小为35。为了达到这一点，我们需要确定如何定位一个文件。通常这都是由我们的文件系统来完成的比如`romfs`使用inode的偏移量来确定一个文件。所以我们设定我们的“根”为0，a的inode为1，b的inode为2，c的inode则为3.

~~~c
 switch (ino) {  
        case 0: i->i_size = 2; break;
        case 1: i->i_size = 35; break;
        case 2: i->i_size = 1; break;
        case 3: i->i_size = 35; break;
        default: i->i_size = 0; break;
        }
~~~

*翻译于 2015/3/26*
