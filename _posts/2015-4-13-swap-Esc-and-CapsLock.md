---
layout: post
title: "Linux下交换你的Esc和CapsLock"
description: "translation-build your own FS"
tags: [Linux, sys_config]
comments: true
---

如果你是个Vim党，那么你一定会觉得Esc键好远啊，如果你是个指法正确的少年，那么你的CapsLock肯定常年落灰。既然这样的话就把CapsLock换成Esc吧。

在家目录创建文件：.Xmodmap

```shell
remove Lock = Caps_Lock
keysym Escape = Caps_Lock
keysym Caps_Lock = Escape
add Lock = Caps_Lock
```
对于红帽用户：
修改/etc/rc.d/rc.local文件，追加一条`xmodmap ~/.Xmodmap`，重启就可以了
对于ubuntu的用户来说：
在System -> Preferences -> KeyBoard -> Layout -> Options -> Capslock behavior中也可设置交换CapsLock和ESC
*貌似没记错的话ubuntu只要在家目录中放置.Xmodmap文件就会自行加载*

**Anyway，这两个按键交换以后刚开始可能有点不习惯，不过习惯以后比原来方便很多**