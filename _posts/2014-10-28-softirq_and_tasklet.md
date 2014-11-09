---
layout: post
title: "软终端与tasklet"
description: "tasklet是基于软中断实现的"
tags: [tasklet, softirq, linux kernel]
comments: true
---


Folow me: <i class="icon-github icon-2x"></i>  [zhaonanyu](http://github.com/zhaonanyu)
Contact me: **zhaonanyu@gmail.com**


----------


>**下半部**
*下半部的任务就是执行与终端处理密切相关但中断处理程序本身不执行的工作。在理想的情况下，最好是中断处理程序将所有工作都交给下半部分执行，因为我们希望在终端处理程序中完成的工作越少越好（也就是越快乐好）。我们抢终端处理程序能够尽可能快地返回。*——***《Linux内核设计与实现》***

##软中断
软中断是利用[硬件中断](http://baike.baidu.com/view/486976.htm)的概念，用软件凡是进行模拟，实现宏观上的异步效果。**很多情况下，软中断和信号有些类似**，同时软中断又是和硬件中断相对应的。

**软中断**于2.3中和**tasklet**一同被引入。软中断是一组静态定义的下半部接口，有32个，**同类型的软中断可以在同多个处理器上同时执行**。这样会出现一个问题，如果**同一个**软中断在它被执行的同时**再次被触发了**，另一个处理器将执行这个触发，这样表示任何共享的数据都需要加**锁**。另外软中断处理程序执行的时候，允许相应终端，但它自己不能休眠。**也就是说如果cpu0上运行的软中断被中断时，当前处理器将不再处理别的软中断，其他的软中断仍可以在别的处理器上执行。

上面我们也了解到软中断作为下半部的一种实现方式用起来很麻烦，这也就是tasklet受青睐的原因。但是需要明确的一点是**tasklet是通过软中断实现的**。

###了解软中断
软中断是在**编译期间静态分配**的，软中断由softirq_action结构表示：

~~~c
// located in /linux/Interrupt.h

struct softirq_action
{
	void	(*action)(struct softirq_action *);
};
~~~

《Linux内核设计与实现》中说明了：

>kernel/softirq.c中定义了一个包含32个该结构体的数组。

>~~~c
static struct softirq_action softirq_vec[NR_SOFTIRQS];
>~~~


每个被注册的软终端都占据该数组的一项，因此最多可能有32个软中断。注意，这是一个定值——注册的软中断数目的最大值没法动态改变。**在当前版本的内核中，这32个项目中只会用到9个**。

**事实上**在现在的版本（3.17）中，软中断的类型已经变为10种：


~~~c 
// located in /linux/Interrupt.h

/* PLEASE, avoid to allocate new softirqs, if you need not _really_ high
   frequency threaded job scheduling. For almost all the purposes
   tasklets are more than enough. F.e. all serial device BHs et
   al. should be converted to tasklets, not to softirqs.
 */

enum
{
	HI_SOFTIRQ=0,
	TIMER_SOFTIRQ,
	NET_TX_SOFTIRQ,
	NET_RX_SOFTIRQ,
	BLOCK_SOFTIRQ,
	BLOCK_IOPOLL_SOFTIRQ,
	TASKLET_SOFTIRQ,
	SCHED_SOFTIRQ,
	HRTIMER_SOFTIRQ,
	RCU_SOFTIRQ,    /* Preferable RCU should always be the last softirq */

	NR_SOFTIRQS
};
~~~


软中断具体的调度和执行在这里就不再详细描述了，注册一个软中断后，在得到调度（或说是触发）时，软中断将会在do_softirq()这个函数中执行，do_irq()函数将会检查并执行所有待处理的软中断。


>***关于do_irq**
do_irq()函数将会根据一个**pending**局部变量来判断该软中断为何种类型，然后调用相应的处理函数（action）来处理该软中断。*


----------


##tasklet
tasklet和软中断本质上很类似，但是接口也更简单，对于**锁保护**的要求更低。
tasklet是通过软中断来实现的，tasklet实际上也是一种软中断。回过头来看一下前面提到的软中断目前的10中类型：

|软中断类型     |优先级     |描述       |
|:-------------:|:---------:|:---------:|
|HI_SOFTIRQ     |0          |高优先级tasklet|
|TIMER_SOFTIRQ  |1          |时钟       |
|NET_TX_SOFTIRQ |2          |发送网络数据包|
|NET_RX_SOFTIRQ |3          |接受网络数据包|
|BLOCK_SOFTIRQ  |4          |块设备     |
|BLOCK_IOPOLL_SOFTIRQ|5     |*新增的不知道啥意思XD*|
|TASKLET_SOFTIRQ|6          |普通优先级tasklet|
|SCHED_SOFTIRQ  |7          |调度器     |
|HRTIMER_SOFTIRQ|8          |高精度时钟 |
|RCU_SOFTIRQ    |9          |RCU锁      |

可以看到其中有两种类型的软中断和tasklet有关，或者说tasklet是一种软中断，分为**HI_SOFTIRQ和TASKLET_SOFTIRQ两种**。


----------
**总结**：
tasklet的具体实现目前还没有搞太懂，就不描述了。
另外陈老师在翻译《Linux内核设计与实现》时有个大错误XD：*p113 表8-2应该是**软中断类型表***

