---
title: phxpaxos源码阅读之六：完结篇
date: 2017-04-05 20:31:21

categories: 
- 分布式系统

tags:
- 分布式系统
- 一致性协议
- paxos
- 源码分析

---

### 前言

这是本系列的最后一篇，由于个人原因实在是没多余的时间写更多的篇幅了，其实 phxpaxos 还有两个比较大的亮点：内置的选主操作和成员的变更操作，还有就是 checkpoint 机制也没有详细的介绍过。以后有时间可能会更，但是不是现在，这一篇我们来做一个总结，概括一下 phxpaxos 的一些亮点。

<!--more-->

### 取消 leader 角色

将 leader 角色混淆在 paxos 中增加了不少算法的复杂度，其实这个角色并不是 paxos 所要求的，纯粹是为了解决冲突问题以及所谓的活锁问题才需要的。 

phxpaxos 完全取消了 leader 的角色，转而将把控权交给了用户，让他们自己决定是否只在一个节点上去提交，当然了就算不是一个节点提交，照样可以正常运行，只是冲突增加了。

至于冲突和活锁和冲突问题，phxpaxos 直接选用了一种随机回退的算法去避免冲突。

这样我们根本不需要去区分系统内所有节点所扮演的角色地位，因为每个节点都是平起平坐的，在某个 proposer 失效时也不用做过多的特殊处理。

### 高度抽象的状态机

phxpaxos 将所有用户的动作行为都交给了状态机，用户可以通过定义自己的状态机而使用 phxpaxos 而完成不同的任务。比如「选主操作」「kv 存取」「成员变更」，它们看上去是大相径庭的任务，但是本质上一模一样，没有必要对他们做特殊对待，全部抽象为一个状态机自己去定义具体的处理操作即可。

上面的高可控性也变向的告诉我们，要保证整个系统的一致性，必须小心定义自己的状态机，因为定义本身也会影响上层应用的一致性。

### 高并发优化

为了有效的利用多核 CPU ，phxpaxos 抽象出了 group ，每个 group 对应一个线程，并且彼此之间完全不互相干扰，至于这种「分离」标准完全交给应用去做。

我们知道分布式系统中最大的开销就是网络通信，而 paxos 消息通常是异步的，如果在等待的时间里让 CPU 干等着肯定是非常低效的， phxpaxos 引入了「消息队列」和「定时器堆」，将 paxos 消息已经各种定时时间放在上述两个结构之中，在「等待回复消息」和「定时时间未到」的过程中休眠，从而让别的 group 去使用 CPU ，大大提高了整个系统的效率。

### 取消 multi-paxos 的 NO-OP 操作

phxpaxos 的作者们认为传统的 NO-OP 操作是没必要的，这点和我的看法差不多，不但给算法添加了复杂度，实现难度也增加不少。就算法本身来讲，它并不属于 multi-paxos 的任何一环，应该单独分离开来。

当然， NO-OP 的出现是为了提高算法的效率，我们知道，相对于 raft 算法，multi-paxos 是存在 gap 的，所谓的 gap ，就是说值的 chosen 的顺序并不确定，为了能让算法继续往下走，我们就闲填入一个 NO-OP 值。但是值得注意的是就算是你填充了 NO-OP 值，还是要从别的节点习得 gap 的 value 去补上漏洞，这样的处理显得非常的繁琐。事实上完全可以让算法本身不产生 gap ，phxpaxos 的设计非常巧妙，整个 instance 都是全局有序去走的，并不会出现 gap 。当然这样 delay 会变得很高，phxpaxos 采用了另外一种做法 batch propose ，这样能获得同样的性能，因为即使填充 gap ，状态机同样要等前面的数据准备好才能继续执行下去。

### 尽可能少的写盘

对于写盘， phxpaxos 总是尽力避免写盘的操作，要么是批量写入，要么是对 sync 进行严格的控制。只要有少量冲突的出现，基本上只需要 accept 过程的一次写盘即可。当然 learn 操作是无可避免的要写盘的，另外 checkpoint 操作则直接传输文件流，非常的 heavy。

### 细节优化、参数化、可定制性

phxpaxos 对于细节上的优化比比皆是，这里就不一一列举了，目测是在使用中遇到各种坑的时候加上去的。另外给开发者提供了很多可定制性空间，比如丰富的参数定制以及非常 nice 的 breakpoint 类；当然风险与方便并存，对于了解不深的开发者，错误的定制会导致未知的错误。

### 后谈

phxpaxos 好像是我第一次完整阅读下来的开源项目，这个项目有个最大的缺点就是注释实在是太少，导致我看的还是非常痛苦的；当然不得不承认，代码的质量本身还是非常高的，虽然没有各种花样的「软件模式」，但是代码布局还是非常实用简洁的，很多地方即使没有注释也能够一目了然。

一方面理解 multi-paxos 本身就不是一件易事，另一方面在实际工程环境下需要考虑各种各样的问题，使得项目复杂度成倍的上升。所以在看完这个项目的时候还是非常有成就感的，这一切动力也源于自己最近对于分布式一致性协议的热切的好奇心。

最后还是要感谢微信后台团队带来的这个优秀的项目并且开源，毕竟 paxos 工程实现起来难度还是非常大的，phxpaxos 无论是代码风格还是架构上都是非常值得一读的，文档和作者与开发者的互动都非常的丰富，还是大力推荐一下这个项目。可惜的是自己看了3个多月，只发现了一个 typo ，提交过一个 patch 还是眼花看错了，实在是心里有愧 = = 。

最近没有多余的时间，有空会去补上「选主算法」和「成员变动」的分析，当然介于我病入膏肓的「拖延症」，建议不要抱太大的期待。

最后也如果大家也对这个项目或者是 multi-paxos 感兴趣，也欢迎评论或者发邮件与我交流。

这次是真的再见了，Bye ！

### 项目链接

我将源码分析工作的注释同步更新到了 github 的项目中，下面是项目链接：

[https://github.com/chenneal/phxpaxos-annotated.git](https://github.com/chenneal/phxpaxos-annotated.git)

欢迎大家 star 。

### 文章列表

[http://chenneal.github.io/2017/03/16/phxpaxos源码阅读之一：走马观花/](http://chenneal.github.io/2017/03/16/phxpaxos源码阅读之一：走马观花/)

[http://chenneal.github.io/2017/03/18/phxpaxos源码阅读之二：粮草先行/](http://chenneal.github.io/2017/03/18/phxpaxos源码阅读之二：粮草先行/)

[http://chenneal.github.io/2017/03/26/phxpaxos源码阅读之三：粉墨登场/](http://chenneal.github.io/2017/03/26/phxpaxos源码阅读之三：粉墨登场/)

[http://chenneal.github.io/2017/03/30/phxpaxos源码阅读之四：各个击破/](http://chenneal.github.io/2017/03/30/phxpaxos源码阅读之四：各个击破/)

[http://chenneal.github.io/2017/04/04/phxpaxos源码阅读之五：暗度陈仓/](http://chenneal.github.io/2017/04/04/phxpaxos源码阅读之五：暗度陈仓/)

[http://chenneal.github.io/2017/04/05/phxpaxos源码阅读之六：完结篇/](http://chenneal.github.io/2017/04/05/phxpaxos源码阅读之六：完结篇/)







