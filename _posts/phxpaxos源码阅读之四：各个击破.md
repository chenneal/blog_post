---
title: phxpaxos源码阅读之四：各个击破
date: 2017-03-30 16:27:21

categories: 
- 分布式系统

tags:
- 分布式系统
- 一致性协议
- paxos
- 源码分析

---

### 前言

本篇终于进入到了主题，即 paxos 算法本身，paxos 本身十分晦涩难懂，我花了好久的时间才理解了这个协议本身，不过如果我能提前知道它的设计哲学的话，似乎能更早的理解它。我已经忘记是从哪里看到的下面这句话，不过对于 paxos 来讲真的是过于贴切了：

** 与其预测未来，不如限制未来。 ** 

<!--more-->

### paxos 的设计哲学

很多人能够轻易的掌握 paxos 的执行流程，但是对于其设计原因常常会感到莫名其妙。比如下面问题就新手就很难去回答：

1. 为何一半以上就能代表大多数？

2. 为何要两阶段？

其实只要你读懂了那篇论文就能够回答，paxos 算法本身是自然而然的一个逆推的过程，采取措施的哲学就是「暴力」，下面我来给你理一理 single-paxos 的思路：

假设你已经知道了必须要采取多轮的 Ballot 才能达成共识，那么我们就必须要保证所有从不同的 node 发起的 Ballot 一旦被 accept ，就肯定是相同的值。

那么要如何保证？我们保证 BallotID 更大的 proposal 能学习到前面已经 accept 的值就行了啊。

那万一后面杀出一个 BallotID 更小的值怎么办，分布式系统什么没有可能？很容易，直接拒绝。喵喵喵？？？你再说一遍？没错，就是直接拒绝。

那如果有部分 node 搞独立 accept 了其它 value 怎么办，没事，只要满足大多数节点的同意，因为任意 accept 两个 Ballot 的赞成节点就肯定有交集，再怎么独立你也能看到我，因为你也要有大多数的赞成票才能 accept 。  

那么上面2个问题的答案就是很明显了，为什么需要大多数是一半以上？因为两个一半以上就是合集。为什么需要2阶段，因为我需要看到是否已经有人抢先我确定了某个值，我要和他保持一致。有先来的人（Ballot更小）躲在暗处想后来出来搞事怎么办？一看到就直接枪毙即可。

所以 paxos 总是采取最暴力简洁的方案去解决问题。

### multi-paxos

single-paxos 是没有任何用处的，除非你能忍受一个 value 3 个 RTT 的代价，这在工程实践里是不可能的。multi-paxos 就是为了寻找每一个 paxos 实例之间的关系所诞生的，我们看看 phxpaxos 是如何做的：

phxpaxos 将一轮 paxos 算法成为一个实例，放在 Instance 类中，这个 Instance 有独立的 proposer acceptor 和 learner，每个 Instance 之间是互相不干扰的。

和 single-paxos 不同的是，proposalID 即 BallotID 跳出了单个 Instance 的束缚，成为了一个全局有序的 ID 。首先，既然是全局有序，那么 BallotID 必须要唯一，phxpaxos 的做法是将 BallotID 做成了一个复合键 (NodeID, ProposalID)。其次在算法层面上也做了优化：那就是同一个节点的不同 InstanceID 共享一个 BallotID ，只有在 acceptor 没有收到冲突的消息，这个 BallotID 就可以一直不变，直到冲突了再说。这时候第一阶段的 prepare 是不需要的，因为既然没有别的节点去干扰我的 proposal，我何必再需要 promise ？

### phxpaxos 的 multi-paxos

proposal 的发起提议：

```C++ 
int Proposer :: NewValue(const std::string & sValue)
{
    if (m_oProposerState.GetValue().size() == 0)
    {
        m_oProposerState.SetValue(sValue);
    }

    m_iLastPrepareTimeoutMs = START_PREPARE_TIMEOUTMS;
    m_iLastAcceptTimeoutMs = START_ACCEPT_TIMEOUTMS;

    // 这里直接做了 multi-paxos 的优化，去掉了 single-paxos 的第一阶段。
    if (m_bCanSkipPrepare && !m_bWasRejectBySomeone)
    {
        PLGHead("skip prepare, directly start accept");
        Accept();
    }
    // 如果冲突了，要重新执行 prepare 阶段。
    else
    {
        //if not reject by someone, no need to increase ballot
        Prepare(m_bWasRejectBySomeone);
    }
    return 0;
}
```

上面的代码在 Proposal 类用两个标志位来表示是否已经发生了冲突，如果有重新发起 proposal 。下面分别看看两种情况的处理：

```C++ 
void Proposer :: Prepare(const bool bNeedNewBallot)
{
    PLGHead("START Now.InstanceID %lu MyNodeID %lu State.ProposalID %lu State.ValueLen %zu",
            GetInstanceID(), m_poConfig->GetMyNodeID(), m_oProposerState.GetProposalID(),
            m_oProposerState.GetValue().size());

    m_oTimeStat.Point();
    ExitAccept();
    m_bIsPreparing = true;
    // 进入了这个函数，代表不能跳过一阶段。
    m_bCanSkipPrepare = false;
    m_bWasRejectBySomeone = false;
    // 重置最大值选项，方便冲突后新的 ballotID 的选定。
    m_oProposerState.ResetHighestOtherPreAcceptBallot();
    if (bNeedNewBallot)
    {
        m_oProposerState.NewPrepare();
    }
    PaxosMsg oPaxosMsg;
    oPaxosMsg.set_msgtype(MsgType_PaxosPrepare);
    oPaxosMsg.set_instanceid(GetInstanceID());
    oPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oPaxosMsg.set_proposalid(m_oProposerState.GetProposalID());
    // 不管 accept 成功还是失败，我们都开始新的一轮计数。
    m_oMsgCounter.StartNewRound();
    // 将当前的 prepare 加入到定时器当中去。
    AddPrepareTimer();
    PLGHead("END OK");
    // 广播给所有的节点尝试 prepare 。
    BroadcastMessage(oPaxosMsg);
}
```

Accept() 和 上面的逻辑相似，但是没有重新决定 BallotID 的过程，那么我们看看 acceptor 是如何处理 proposer 的消息的：

```C++
int Acceptor :: OnPrepare(const PaxosMsg & oPaxosMsg)
{
    ....

    PaxosMsg oReplyPaxosMsg;
    oReplyPaxosMsg.set_instanceid(GetInstanceID());
    oReplyPaxosMsg.set_nodeid(m_poConfig->GetMyNodeID());
    oReplyPaxosMsg.set_proposalid(oPaxosMsg.proposalid());
    oReplyPaxosMsg.set_msgtype(MsgType_PaxosPrepareReply);
    oBallot(oPaxosMsg.proposalid(), \oPaxosMsg.nodeid());

    if (oBallot >= m_oAcceptorState.GetPromiseBallot())
    {
        ....

        // 回传消息里包括已经 accept 的最大的 ballot id 的值。
        oReplyPaxosMsg.set_preacceptid(m_oAcceptorState.GetAcceptedBallot().m_llProposalID);
        oReplyPaxosMsg.set_preacceptnodeid(m_oAcceptorState.GetAcceptedBallot().m_llNodeID);

        // 如果此前没有任何 accept 的值，由 proposer 自己决定。 
        if (m_oAcceptorState.GetAcceptedBallot().m_llProposalID > 0)
        {
            oReplyPaxosMsg.set_value(m_oAcceptorState.GetAcceptedValue());
        }

        // 更新 promise proposeid 的值。
        m_oAcceptorState.SetPromiseBallot(oBallot);

        // 记住已经 promise 的最大值。
        int ret = m_oAcceptorState.Persist(GetInstanceID(), GetLastChecksum());
        if (ret != 0)
        {
            PLGErr("Persist fail, Now.InstanceID %lu ret %d",
                    GetInstanceID(), ret);
            return -1;
        }
    }
    else
    {
        ....
        
        oReplyPaxosMsg.set_rejectbypromiseid(m_oAcceptorState.GetPromiseBallot()\
                                                             .m_llProposalID);
    }
    nodeid_t iReplyNodeID = oPaxosMsg.nodeid();
    PLGHead("END Now.InstanceID %lu ReplyNodeID %lu",
            GetInstanceID(), oPaxosMsg.nodeid());;
    SendMessage(iReplyNodeID, oReplyPaxosMsg);
    return 0;
}
```

OnAccept 的处理类似，甚至判断条件都是一样的，所谓的两阶段做的动作基本上无异，只是一阶段多一次机会去调整自己的 BallotID ，这个时候你会发现我们所有 accept 的值 BallotID 肯定是全局有序的。

我们再看看 proposer 在收到 reply 时是如何做的：

```C++
void Proposer :: OnPrepareReply(const PaxosMsg & oPaxosMsg)
{
    ....

    // 收到消息时发现已经不在 prepare 阶段了，直接忽略这个消息。
    if (!m_bIsPreparing)
    {
        BP->GetProposerBP()->OnPrepareReplyButNotPreparing();
        //PLGErr("Not preparing, skip this msg");
        return;
    }

    // 虽然正在 prepare 阶段，但是 proposeID 不一致，同样忽略。
    if (oPaxosMsg.proposalid() != m_oProposerState.GetProposalID())
    {
        BP->GetProposerBP()->OnPrepareReplyNotSameProposalIDMsg();
        //PLGErr("ProposalID not same, skip this msg");
        return;
    }

    // 统计回复的节点数量。
    m_oMsgCounter.AddReceive(oPaxosMsg.nodeid());

    if (oPaxosMsg.rejectbypromiseid() == 0)
    {
        oBallot(oPaxosMsg.preacceptid(), oPaxosMsg.preacceptnodeid());
        ....
        // 统计赞成的节点数量。
        m_oMsgCounter.AddPromiseOrAccept(oPaxosMsg.nodeid());
        m_oProposerState.AddPreAcceptValue(oBallot, oPaxosMsg.value());
    }
    else
    {
        ....
        // 统计拒绝的节点数量。
        m_oMsgCounter.AddReject(oPaxosMsg.nodeid());
        m_bWasRejectBySomeone = true;
        m_oProposerState.SetOtherProposalID(oPaxosMsg.rejectbypromiseid());
    }

    // 超过半数赞同意味着本次 prepare 阶段成功。
    if (m_oMsgCounter.IsPassedOnThisRound())
    {
        int iUseTimeMs = m_oTimeStat.Point();
        ....
        // 3.21 : 下次再次运行 proposer 时，不需要再进行 prepare 阶段了。
        // 可能有人会问为什么要这样，因为在等待 accept 回复的过程中，
        // 当前线程会重新扔进 loop 中，再次唤醒需要一个标志位判断。
        m_bCanSkipPrepare = true;
        Accept();
    }
    // 3.21 : 收到大多数节点 reject 的消息或者已经收到了收到了所有节点的消息。
    // 设立一个随机的定时器，为的是与别的节点避免冲突。
    else if (m_oMsgCounter.IsRejectedOnThisRound()
            || m_oMsgCounter.IsAllReceiveOnThisRound())
    {
        BP->GetProposerBP()->PrepareNotPass();
        PLGImp("[Not Pass] wait 30ms and restart prepare");
        AddPrepareTimer(OtherUtils::FastRand() % 30 + 10);
    }

    PLGHead("END");
}
```

OnAcceptReply 也是类似的处理，只是在接到消息后需要去 learn 那个值并且广播给每个节点。这个时候我发现了在 learn 的时候并未真正的写入，容我以后再分析。

### 总结

phxpaxos 基本上是按照经典的 multi-paxos 算法去实现的，在协议上没有做什么手脚，原汁原味。但是在性能优化上做了不少的工作，特别是在 learner 上下足了功夫，使得原本简单的角色变得相当复杂，当然和算法本身就没有关系了。以后我会注重从性能上去分析 phxpaxos ，特别是有关 learner 和 checkpoint 机制，bye ！

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








 



