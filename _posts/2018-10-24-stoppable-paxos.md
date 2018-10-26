---
layout: post
title: Stoppable Paxos解读
---
# 摘要
在Reconfiguring a State Machine一文中，lamport将reconfiguration分解为3个正交的问题：停止当前的状态机、选择下一个状态机的成员组和组合一系列状态机。本文将详细解读实现停止当前状态机的算法：即Stoppable Paxos。
# 再论Paxos
本章简要回忆了经典Paxos算法，此处不再赘述。不过本章的几个重要概念值得注意，会在后续章节用到。
+ phase2a proposer进行第二阶段消息的提交（更明确一点，就是将command放入待发送消息队列）
+ maxbal2a(i, b, Q) 即一阶段的返回值里最大的ballot number（小于b）
+ val2a(b, Q) 即paxos一阶段的返回值里最大的ballot number（小于b）对应的command，如果不为空，说明这个instance之前执行过phase2a，因此才有可能被观察到
# Stoppable Paxos算法
与普通Paxos算法一样，新的Leader以合适的Ballot执行Phase1a。但当Leader发现stop command有可能被选择（chosen）时，Stoppable Paxos的处理方式与普通Paxos不同。
    
+ 编号小于i的实例，正常执行phase2a
+ 编号大于i的实例
    + 如果编号i执行的命令不是stop command，则编号大于i的实例正常执行phase2a
    + 反之，Leader不会对大于i的实例执行任何操作

因此，问题的关键就在于，当val2a(i,b,Q)==stp时，且任意大于i的j，val2a(j,b,Q)是任意值时，Leader应该怎么做？问题取决于mbal2a(i,b,Q) 和mbal2a(j,b,Q)。

+ 当mbal2a(j,b,Q)大于mbal2a(i,b,Q)时，stp一定没有被选择，因此这个stop command实际上是无效的。这意味着Leader看起来不受stp的约束，可以选择任何值
+ 否则，Leader需要执行phase2a，尝试让stp被选择生效；并且对任意大于i的j的实例，不做任何操作。

当Leader对实例i的phase2a执行stop command时，不会对任何大于i的实例执行phase2a，否则，Leader就会执行常规Paxos。Stoppable Paxos在并发执行上的表现和Paxos相同。

接着，作者开始正式介绍Stoppable Paxos。从上述描述可知，Stoppable Paxos真正区别于普通Paxos的地方仅仅在于phase2a的启动条件。这里以E1-E6分别标记。

+ E1-E2和普通Paxos要求一样
+ E3：如果sval2a不为空，则当前实例执行的command等于sval2a。
    + 在普通Paxos中，因为实例i提交的命令有可能以较低的bollot number提交并被选择，如果val2a不为空，则v=val2a(i, b, Q)
    + 我们定义sval2a：当val2a是一个无效的stop command时，sval2a=空，否则，sval2a和val2a相等
+ E4：当且仅当满足以下2个条件时，phase2a才能执行stop command
    + E4a：没有任意大于i的实例执行过phase2a。也就是说任意大于i的实例j，其val2a(j, b, Q)等于空，也就说明sval2a(j, b, Q)等于空
    + E4b：如果Leader没有被要求（by sval2a的值要求）执行stop command，Leader也就不能被要求对任意大于i的实例执行任意command
        + 有点绕口，讲人话就是，如果sval2a(i, b, Q)等于空，sval2a(j, b, Q)一定也必须等于空。为什么会这样我们分情况分析
            + 如果sval2a(i, b, Q)是stop command，那么sval2a(j, b, Q)一定是空（E4a保证）
            + 如果sval2a(i, b, Q)是空，意味着sval2a可以选择任意值，那么此时要选择stop command的话，必须保证sval2a(j, b, Q)等于空，否则可能会打破E3和E4a
                + E3要求sval2a不为空时，必须以之前获取的值作为提交的命令（有可能以lowered number ballot提交并被选择），如果此时sval2a(j, b, Q)不为空又不执行phase2a，会打破Paxos的语义：一个命令被多数派选择，却最终无效！
+ E5：Leader不能在小于i的实例中，以stop command执行过phase2a
+ E6：需要保证任意小于i的实例j，其sval2a(j, b, Q)不为空

从以上的约束来看，Leader被要求对实例i执行stop command，并且对任意大于i的实例j执行一些命令是不可能发生的：

+ 如果Leader对实例i执行了stop command，E5会阻止任意j的phase2a的执行
+ 如果实例j的phase2a先执行，E4a会阻止实例i执行stop command
+ sval2a的定义保证了实例j的phase1b消息会导致实例i的stop command无效

和普通Paxos不同，在Stoppable Paxos中，单独的实例间逻辑不是独立的。E4-E6以及sval2a的值受到其他实例phase2a执行结果以及以当前bollot number 接收到的phase1b消息的影响。然而，这并非意味着引入额外的消息以及延迟。

+ 和普通的Paxos一样，只需要发送一次即可获取所有phase1b消息
+ Stoppable Paxos并没有要求其他实例先于当前实例执行phase2a
+ 启动条件仅仅是要求某些操作（phase2a）还并未执行（not have been done）

实际上，Leader可以以任意顺序提交命令，这同样适用于stop command。由于stop command常常被用于reconfiguration，如果希望reconfiguration快速生效时，可以将i作为下一个可用的命令编号；或者可以选用一个更大的值惰性生效。

# Correctness
略
# Proof
略（有机会更新，跟普通Paxos比，证明过程复杂繁琐许多）
# 总结
和初读Paxos made simple的感受类似，lamport的论文惜字如金，有的语句不是那么容易明白，但是一旦理解，又会觉得作者一个多余的字，一句废话都没有。
Stoppable paxos实际上是paxos的一个变种，作者找到问题的切入点（phase2a的启动条件），通过巧妙增加约束的方式实现了停止状态机，不仅没有引入额外消息以及延迟等副作用，而且并发能力和paxos相当（任意并发提交多个实例），思路值得学习。

