
---
layout: post
title: "如何编写一个Linux VFS 文件系统模块-StaticFS-4"
description: "every geek wants a blog all by himself lol"
tags: [GitHub Pages, jekyll, static site generator]
comments: true
---

下面是`s_op`和`dq_op`，一个脸熟一个不是那么熟。`s_op`用于存放我们的`struct super_operations`。`dq_op`是磁盘使用的操作，看起来都没有VFS模块会自己实现一个，所以我们假设在`/usr/src/linux/fs/dquot.c `里面的这个就够用了

~~~c
sb->s_op = &staticfs_ops;
~~~

`s_type`允许我们定义mount flag，参照fs.h中，`MS_NOSUID`用于忽略suid和sgid标志位，`MS_NOEXEC`用于禁止运行存在于挂载的文件系统上的程序。我们参照romfs使用`MS_RDONLY`。设置了这个标志位后VFS应该会禁止任何试图写我们的staticfs的操作，即使我们在我们的inode上设置可写标识。
`s_magic`里面存着我们之前定义的魔法数字

~~~c
sb->s_flags = MS_RDONLY;
  sb->s_magic = STATICFS_MAGIC;
~~~

接下来是`s_root`。我们之前已经接触过这个字段了，就在前面我们试图去搞懂`put_inode()`的时候。在ramfs和romfs中都在inode中使用`d_alloc_root()`来获得`struct dentry`，我们也这样做。我们来试试使用romfs的方法来创建一个根inode，也就是使用`iget()`。我们把我们的根inode叫做0.

~~~c
sb->s_root = d_alloc_root(iget(sb,0));
  if (!sb->s_root) {
    return NULL;
  }
~~~

注意我同时也添加了几行代码用于检测`d_alloc_root()`是否返回的是非空值。
**It looks like everything else in the superblock is used for maintaining lists,**设定信号量用于保持监测用量，这些都是VFS需要解决的问题，和我们没有关系。我们现在返回我们的`superblock`然后就大功告成了！

~~~c
return sb;
}
~~~

翻译于 2015/3/29