---
title: phxpaxos源码阅读之五：暗度陈仓
date: 2017-04-04 18:22:21

categories: 
- 分布式系统

tags:
- 分布式系统
- 一致性协议
- paxos
- 源码分析

---

### 前言

上一篇重点分析了 paxos 的核心成员 proposer 和 acceptor 的工作流程，那这一篇我们重点分析 learner 这个角色。

在大多数讲述 paxos 算法的文章中，这个角色是最简明易懂的，然而在实际的工程实践中，learner 往往是最复杂的，我们从 phxpaxos 项目中 learner 的消息种类就可以看出其复杂程度。由于 learner 需要根据当前 node 的运行进度来判断来执行不同的动作，并且与整个系统的系能有着千丝万缕的联系，我们将分情况去考察。

<!--more-->

### 各节点匀速提交

这个时候我们会发现通常这个时候只有一个 proposer ，这个 proposal 你可以当成所谓的 leader ，当然在 phxpaxos 中其实它并不存在。由于没有其它的节点干扰，所以它总是能够提交成功。这个时候我们看一下在 phxpaxos 中会发生什么。

首先需要讲一下在没有节点失联已经宕机的情况下，phxpaxos 是如何保持所有的节点的进度在同一水平线上的。在每次提交一个新值之前，phxpaxos 会通过下面一个函数判断自己的进度：

```C++
const bool Learner :: IsIMLatest() 
{
    return (GetInstanceID() + 1) >= m_llHighestSeenInstanceID;
} 
```
如果这个函数判断是返回 false ，那么当前的 node 会暂时暂停工作转而去等待直到 learner 拉小与别的节点差距。

我们假设每个节点在提交一个新值开始的时候都能维持在最新的进度，因为只要能 touch 到大部分的节点， m_llHighestSeenInstanceID 的值基本上就是正确的。当然这里我们只有一个 proposer 提交的时候，只要能够正常工作，就不会出现本节点落后的情况，对方节点会被动的去 learn 自己的值。如果我们能够准确的联系到对方，那么假设我们提议的值已经被大多数节点成功 accept 了，那么本节点的 proposer 会调用 ProposerSendSuccess 函数，代码如下：

```C++
void Learner :: ProposerSendSuccess(
        const uint64_t llLearnInstanceID,
        const uint64_t llProposalID)
{
    BP->GetLearnerBP()->ProposerSendSuccess();

    PaxosMsg oPaxosMsg;

    // 对方的 learner 收到这个类型的消息会去学习已经被 accept 的值。
    oPaxosMsg.set_msgtype(MsgType_PaxosLearner_ProposerSendSuccess);
    oPaxosMsg.set_instanceid(llLearnInstanceID);
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_proposalid(llProposalID);
    oPaxosMsg.set_lastchecksum(GetLastChecksum());

    //run self first
    // 广播这个消息。
    BroadcastMessage(oPaxosMsg, BroadcastMessage_Type_RunSelf_First);
}
```

这个消息会广播给 group 中所有的节点，所有的节点的 learner 收到这个消息后就会主动的去学习这个值。

在最好的情况下，每次这个 leader 节点都会将自己被 accept 的值发给所有的节点学习，并且所有的节点都会学习成功，但是不幸的是有些节点会由于各种原因没有收到这个消息导致没有学到这个值而无法进入下一个 instance ，这就导致某些节点会有滞后的行为。 

### 不匀速的情况

其实这个状态才是常态，很多情况下，所有节点是不可能匀速的。举个例子，就算只有一个 proposer 能够提交，只要收到大多数节点的 accept 就会立马去 learn 自己的值并执行状态机，这个时候假设有些节点 accept reply 的消息十分的慢以至于 leader 的下一个节点已经开始下一轮的 instance 了，这可这么办？我们发现 phxpaxos 针对于 落后一个 instance 的情况做了自己的优化，但是落后 2 个甚至 2 个以上的 instance 的情况并没有在 proposer 和 acceptor 的消息系统中去处理。那可咋办？答案就是 learner ， phxpaxos learner 的设计最主要的优化就是为每一个 learner 配了一个 learner_sender ，顾名思义，它会一直在后台观察各节点的进度状态，一发现有差距，就会立马去学习弥补以便跟上进度。这个 learner_sender 的 loop 如下：

```C++
void LearnerSender :: run()
{
    m_bIsStart = true;
    while (true)
    {
        WaitToSend();
        if (m_bIsEnd)
        {
            PLGHead("Learner.Sender [END]");
            return;
        }
        SendLearnedValue(m_llBeginInstanceID, m_iSendToNodeID);
        SendDone();
    }
}

void LearnerSender :: WaitToSend()
{
    m_oLock.Lock();
    // 所谓的确认就是要等待对方知道自己的 chosen 信息之后已经确定要从自己
    // 的节点去学习，这样才能发送自己的数据到那个已经确认过的节点。
    while (!m_bIsComfirmed)
    {
        // 最长等待 1000ms 。
        m_oLock.WaitTime(1000);
        if (m_bIsEnd)
        {
            break;
        }
    }
    m_oLock.UnLock();
}
```

上面这 2 个函数即使不需要注释就知道它们的用途， phxpaxos 单独开辟了一个线程，并以一个 confirm 开关作为 learn 的触发点，那么这个触发机制是怎样的，由于太过啰嗦，我直接理一下它的线路。

1.首先有一个关键的函数会去作为这个 learn 的起始点，这个函数如下：

```C++
void Learner :: Reset_AskforLearn_Noop(const int iTimeout)
{
    if (m_iAskforlearn_noopTimerID > 0)
    {
        m_poIOLoop->RemoveTimer(m_iAskforlearn_noopTimerID);
    }

    m_poIOLoop->AddTimer(iTimeout, Timer_Learner_Askforlearn_noop, \
                         m_iAskforlearn_noopTimerID);
}
```

这个函数的命名非常有迷惑性，会误导人联想到 multi-paxos 的 NO-OP 操作，看下来似乎并没有太大的联系，这个函数调用非常的频繁，它在 instance 初始化的时候第一次调用，然后就一发不可收拾，那是因为加入这个定时器之后，以后每次处理这个超时事件都会重新更新一个新的定时器事件，也就是说并没有真正的 remove timer 的操作。想一想这是理所应当的， learner 必须要不断地工作才能够保证 paxos 算法正确的运行下去。

2.到了一定的时间之后取出定时器事件调用 AskforLearn_Noop 函数，这个函数重置了定时器之后又调用了 AskforLearn 函数。

3.AskforLearn 函数广播了 MsgType_PaxosLearner_AskforLearn 消息等待有节点接自己的活，如果发现有人接手了，接手的那个节点会调用 SendNowInstanceID 函数发送它自己当前的进度通知自己。

4.在本节点收到对方当前的进度之后会调用 ComfirmAskForLearn() 去向对方确认是否启动 learn 工作。

5.对方确认之后会置 confirm 标志位唤醒 LearnerSender ，此后不断地做流式学习就行。

### 落后太多怎么办

因为 paxos log 是无限增长的，不可能全部存储下来，所以落后太多肯定是没法子的，那怎么办？作者很幽默的说了句凉拌。哈哈，其实 phxpaxos 为了解决这个问题，引入了 checkpoint 机制。每次会设定一个 instanceID 的临界点作为 checkpoint 的触发点，如果某个节点落后太多，也就是小于某个节点的 minchoseninstanceid ，都会触发 checkpoint 拉取的操作，这个操作直接传输数据文件，所以非常的重，因此作者建议尽量多的保存 paxoslog 的数据。

### 总结

phxpaxos 把 learn 设计成这样很大一部分是为了效率，比如流式学习，一次性可以提速好几个 instance 而不用在重新去进行 paxos 算法流程，还可以发现项目中处处可以发现各种各样小的优化，比如落差只有一个 instance 的情况，比如还会不断的去优化 proposalID 的值尽量避免冲突。

phxpaxos 并没有要求所有节点作严格的对齐，而是通过后台的 learn_sender 线程去拉平差距。在落后差距非常大或者重新加入节点的情况，则会通过 checkpoint 这个非常重的操作去拉平。不过只要节点成员不发生变动，只要保留足够多的 paxoslog ，就可以避免发生 checkpoint 的操作。

这篇就到这，bye ！

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




