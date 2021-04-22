---
title: VM-MT论文阅读
date: 2021-04-20 01:22:42
tags:
- 论文阅读
cover_picture: https://raw.githubusercontent.com/Sunny-Island/Resources/main/images/vm-ft.jpg
---

## The design of a practical system for fault-tolerant virtual machines

### replica 备份方式

- 传内存块过去：state transfer，以保证state的相同

- 传操作过去：replicated state machine，使用state transfer

Primary/Backup

### 机器级别vs应用程序级别

该论文是在机器指令级别实现主从备份，现在的分布式程序往往在应用程序之间实现备份。

核心思想是让Backup像Primary一样运行，CPU指令级别上的。然后通过对于一些确定的输入，可以确定有相同的运行结果，但是对于一些不确定的事件，比如多核交互（本文只使用单核），指令中断，获取随机数，Backup需要做一些特殊处理，比如拦截中断的指令序列号让其在自己上也在特定的位置中断。

为什么指令中断会影响CPU，因为中断往往来自CPU外部，比如IO设备，时间片到期等等。

中断怎么被复制到Backup上？通过Intel CPU特定的硬件完成。

不能让从机快于主机。

由于VMM会监视输入输出，所以对于Backup机器，其输出不会被返回到客户端，只有在主机失效的时候，根据FT-protocol会切换主备份



### FT-Protocol

![test](/image/vm-ft.jpg )

主机记录自己进行的指令操作，然后发给从机等待执行。从机会放到buffer中去等待执行，这样就防止了从机比主机快。

如果主机fail了，从机直接对外输出

##### Output requirement

如果从机接管了主机，那么从机中已经被传进来的主机指令会继续执行。这样客户端就不会察觉到异常，服务会继续提供。

这条规则是为了保证指令的一致性，如果主机挂掉了，从机会保持和主机一致的行为。

##### Output rule

如果主机执行完就输出了，然后挂掉了，但还没给从机发自己执行过的指令操作，这样也会导致不一致。

因此要求主机先不急着输出自己的执行结果，先把自己的执行指令发给从机并收到了从机的回复，然后再输出结果。

