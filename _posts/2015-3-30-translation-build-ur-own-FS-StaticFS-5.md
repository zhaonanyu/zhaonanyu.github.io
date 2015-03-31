---
layout: post
title: "如何编写一个Linux VFS 文件系统模块-StaticFS-5"
description: "translation-build your own FS"
tags: [StaticFS, VFS, translation]
comments: true
---

#Inodes
是时候编写一些inode的功能了。我们再来看看fs.h中的`struct inode_operations`结构：

~~~c
struct inode_operations {
        int (*create) (struct inode *,struct dentry *,int);
        struct dentry * (*lookup) (struct inode *,struct dentry *);
        int (*link) (struct dentry *,struct inode *,struct dentry *);
        int (*unlink) (struct inode *,struct dentry *);
        int (*symlink) (struct inode *,struct dentry *,const char *);
        int (*mkdir) (struct inode *,struct dentry *,int);
        int (*rmdir) (struct inode *,struct dentry *);
        int (*mknod) (struct inode *,struct dentry *,int,int);
        int (*rename) (struct inode *, struct dentry *,
                        struct inode *, struct dentry *);
        int (*readlink) (struct dentry *, char *,int);
        int (*follow_link) (struct dentry *, struct nameidata *);
        void (*truncate) (struct inode *);
        int (*permission) (struct inode *, int);
        int (*revalidate) (struct dentry *);
        int (*setattr) (struct dentry *, struct iattr *);
        int (*getattr) (struct dentry *, struct iattr *);
};
~~~

考虑到我们这个是一个只读文件系统，接下来的工作应该不难。在romfs中只实现了上面之中的一个函数——`lookup`，但是我们为了保险起见还是都来看一下。

`create`，`mkdir`，`link`，`symlink`和`mknod`都是创建的功能，这些在我们的文件系统中都是不允许的，所以我们就让他们空着。`unlink`和`rmdir`是移除inode，这也是我们的文件系统不允许的。`rename`会修改或者移除inode，这个我们也是不允许的。

那么来看看我们还没有看过的字段。`readlink`和`follow_link`是为了符号链接准备的，我们不需要他们。`truncate`？这个我不太清楚是什么。`permission`貌似是用于决定用户是否可以读取inode的。好吧，这个我们也不太了解。现在我们先跳过他们，因为ramfs和romfs中也没有管它们。`revalidate`应该是用于询问VFS模块确保inode是正确的——这个对于一个通过网络传输的文件系统是很有用给的。最后是`setattr`和`getattr`这两个我也不太清楚，虽然对于我们以后要做的VFS模块是很有用的，但是至少目前我们用不上。

现在剩下什么？只有`lookup`函数了。在romfs和ramfs都是这样做的，不过我目前仍然不是太明白。我们来看看ramfs里面：

~~~c
 d_add(dentry, NULL);
return NULL;
~~~

看来是dentry把inode传递过来，添加到dentry cache中，然后返回一个NULL。继续阅读romfs的代码，它又多干了一些事情。** It gets the name of the filename we're interested in from the dentry object passed in, at it checks for it in the directory specified by the inode passed in. **如果它在文件夹中找到了入口的名字，他会将入口的inode以及dentry加入dentry cache中。在romfs代码的注释中我们还可以了解到如果没有找到入口，应该调用`d_add()`返回一个NULL inode。我们遵循这个。

~~~c
static struct dentry *staticfs_lookup(struct inode *dir, struct dentry *dentry) {
  int ino;
  struct inode *inode;
  const char *name;

  ino = -1;
  inode = NULL;
  name = dentry->d_name.name;

  switch (dir->i_ino) {
  case 0:
    if (strcmp(name,"a")==0) ino=1;
    if (strcmp(name,"b")==0) ino=2;
    break;
  case 2:
    if (strcmp(name,"c")==0) ino=3;
    break;
  }

  if (ino>=0) {
    inode=iget(dir->i_sb,ino);
  }

  d_add(dentry,inode);
  return NULL;
}
~~~

喔，这还真是不少的代码！我假设传入的inode已经被确认为文件夹，所以我们就不检查他了。这也就是说他如果不是inode 0也就是根文件夹，他就是inode 2也就是b文件夹。鉴于我们的文件夹入口是静态的，我们还没有是么可以用来比对的记录，我们就只有一个定制的开关。如果正确的名字出现在正确的地方的话，我们就把`ino`指向inode号所对应的文件。如果开关返回的是-1我们就将inode设定为NULL。

重新阅读一遍ramfs和romfs中的注释，我想我终于搞懂了。romfs中：

~~~c
 /*
         * it's a bit funky, _lookup needs to return an error code
         * (negative) or a NULL, both as a dentry.  ENOENT should not
         * be returned, instead we need to create a negative dentry by
         * d_add(dentry, NULL); and return 0 as no error.
         * (Although as I see, it only matters on writable file
         * systems).
         */
~~~

ramfs中：

~~~c
/*
 * Lookup the data. This is trivial - if the dentry didn't already
 * exist, we know it is negative.
 */
~~~

其中的"negative"解决了我的疑惑。一个negative dentry。ramfs中说“如果它现在不存在”，说明他就不存在与dentry cache中。dentry cache是ramfs的一个快照，如果它不存在于dentry cache中，说明他并不是来自于这个session中的，如果它来自于这个session中，那么他就也会存在于dentry cache中。**This means it can just take any dentry given and throw it away, **如果VFS发出请求，他也不在那。romfs需要多做一些工作，因为貌似VFS并不涉及一个已存在的文件。

既然我们inode相关所有的函数都写好了（实际上就一个），我们现在就来组装我们的`struct inode_operations`。

~~~c
static struct inode_operations staticfs_inode_operations = {
  lookup:staticfs_lookup,
};
~~~

inode也大功告成了！

翻译于 2015/3/30

---