---
layout: post
title: "如何编写一个Linux VFS 文件系统模块-StaticFS-6"
description: "translation-build your own FS"
tags: [StaticFS, VFS, translation]
comments: true
---

另一个字段`readdir`，有一个自制的函数`romfs_readdir()`：

~~~c
static int
romfs_readdir(struct file *filp, void *dirent, filldir_t filldir)
{
        struct inode *i = filp->f_dentry->d_inode;
        struct romfs_inode ri;
        unsigned long offset, maxoff;
        int j, ino, nextfh;
        int stored = 0;
        char fsname[ROMFS_MAXFN];       /* XXX dynamic? */

        maxoff = i->i_sb->u.romfs_sb.s_maxsize;

        offset = filp->f_pos;
        if (!offset) {
                offset = i->i_ino & ROMFH_MASK;
                if (romfs_copyfrom(i, &ri, offset, ROMFH_SIZE) <= 0)
                        return stored;
                offset = ntohl(ri.spec) & ROMFH_MASK;
        }

        /* Not really failsafe, but we are read-only... */
        for(;;) {
                if (!offset || offset >= maxoff) {
                        offset = maxoff;
                        filp->f_pos = offset;
                        return stored;
                }
                filp->f_pos = offset;

                /* Fetch inode info */
                if (romfs_copyfrom(i, &ri, offset, ROMFH_SIZE) <= 0)
                        return stored;

                j = romfs_strnlen(i, offset+ROMFH_SIZE, sizeof(fsname)-1);
                if (j < 0)
                        return stored;

                fsname[j]=0;
                romfs_copyfrom(i, fsname, offset+ROMFH_SIZE, j);

                ino = offset;
                nextfh = ntohl(ri.next);
                if ((nextfh & ROMFH_TYPE) == ROMFH_HRD)
                        ino = ntohl(ri.spec);
                if (filldir(dirent, fsname, j, offset, ino,
                            romfs_dtype_table[nextfh & ROMFH_TYPE]) < 0) {
                        return stored;
                }
                stored++;
                offset = nextfh & ROMFH_MASK;
        }
}
~~~

这个函数应该是在VFS中目前我们见过最大工作量的玩意了。这里我们需要填充一个`dirent`结构，通过filldir()传入一个空指针，然后传入一个`filldir_t`类型。有趣的是需要被填充的结构以及它填充的函数并不是静态的。如我之前的假设。我想VFS是否有很多结构和函数都是为了以后使用而现在没有被启用。这个我以后再回来看看。

来看看这个函数，它获得了他需要的inode还有文件的偏移指针。这个偏移量貌似是依赖于不同魔族的，所以你可以使用byte的数量作为进入文件夹的偏移量，或者像我一样，我会使用索引，因为我其实没啥文件可以读。至少，我是希望这能管用。

看起来我还可以处理`filldir()`函数用来返回小于零，也就是代表“我满了”的错误信息。这也就是为什么我们需要支持一个带有偏移量的调用——防止我们需要停止正在传输的内容。这个看起来对于我们来说不是什么问题，因为我们只有俩文件夹。

让我们开始写。

~~~c
static int staticfs_readdir(struct file *fp, void *dirent, filldir_t filldir) {
  struct inode *i=fp->f_dentry->d_inode;
  unsigned long offset=fp->f_pos;
  int ino=i->i_ino;
  int stored = 0;
  unsigned char ftype;
  char fsname[2];

  for (;;) {
    fsname[0]='\0';

    switch (ino) {
    case 0:
      switch (offset) {
      case 0:fsname[0]='a';fsname[1]='\0';ftype=DT_REG;break;
      case 1:fsname[0]='b';fsname[1]='\0';ftype=DT_DIR;break;
      };break;
    case 2:
      switch (offset) {
      case 0:fsname[0]='c';fsname[1]='\0';ftype=DT_REG;break;
      };break;
    }

    if (fsname[0]=='\0') {
      return stored;
    }

    if (filldir(dirent,fsname,1,offset,ino,ftype)<0) {
      return stored;
    }
    stored++;
    offset++;
  }
}
~~~

这个比romfs的实现略微简易一些，特别是因为我们的staticfs是静态的~我们的函数已经写好了，现在可以填充结构体了。

~~~c
static struct file_operations staticfs_dir_operations = {
  read:generic_read_dir,
  readdir:staticfs_readdir,
};
~~~

最后我们需要考虑的事情就是地址空间了。

翻译于 2015/4/1