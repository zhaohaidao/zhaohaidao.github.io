---
layout: post
title: kafka日志复制算法ISR
---
# 摘要
xxxx 大部分共识协议都需要解决3个关键问题解决：LeaderElection、LogReplication和MemberManagement，因此，我会从这3个方面入手，描述kafka的ISR如何解决这3个问题。
# 设计目标
# LeaderElection
kafka的LeaderElection非常简单直接，kafka使用ISR动态管理副本成员，ISR里的成员均有资格被选成Leader。这是因为kafka的每条消息只有复制到ISR里的所有成员，才会被认为是committed，所以，ISR里的成员均持有所有comitted的数据，那么ISR的任意成员均有资格当选Leader。其实这里有坑，后面会提到。

实现层面，由于kafka的每个partition有多个副本，因此kafka定义了一个partition粒度的选主接口PartitionLeaderSelector，并结合具体场景分别实现了5种选主策略，这些实现虽然有所区别，但是本质上是一样的，即依据某种算法从ISR里挑选一个成员作为新的Leader。选出新的Leader后就会向新Leader发送LeaderAndISR请求。下面将分别讲解不同的选主策略以及broker对LeaderAndISR的处理。
## PartitionLeaderSelector
|  选主策略   | 触发场景  |
|  ----  | ----  |
| PreferredReplicaPartitionLeaderSelector  | 1 controller故障转移 2执行分区均衡|
| ControlledShutdownLeaderSelector  | broker优雅停机 |
| OfflinePartitionLeaderSelector  | 当broker恢复，副本状态由OfflineReplica变更为OnlineReplica时触发 |
| ReassignedPartitionLeaderSelector  | 分区均衡完成后使用 |
| NoOpLeaderSelector  | 副本状态变化时的默认选主策略，看起来没有实际场景用到 |

+ PreferredReplicaPartitionLeaderSelector
```scala
/**
 * 挑选AR里第一个副本作为优先副本，如果该副本所在broker是存活的且该broker存在于ISR中，那么将优先副本选为新的Leader
 * New leader = preferred (first assigned) replica (if in isr and alive);
 * New isr = current isr;
 * Replicas to receive LeaderAndIsr request = assigned replicas
 */
class PreferredReplicaPartitionLeaderSelector(controllerContext: ControllerContext) extends PartitionLeaderSelector
with Logging {
  this.logIdent = "[PreferredReplicaPartitionLeaderSelector]: "

  def selectLeader(topicAndPartition: TopicAndPartition, currentLeaderAndIsr: LeaderAndIsr): (LeaderAndIsr, Seq[Int]) = {
    val assignedReplicas = controllerContext.partitionReplicaAssignment(topicAndPartition)
    val preferredReplica = assignedReplicas.head
    // check if preferred replica is the current leader
    val currentLeader = controllerContext.partitionLeadershipInfo(topicAndPartition).leaderAndIsr.leader
    if (currentLeader == preferredReplica) {
      throw new LeaderElectionNotNeededException("Preferred replica %d is already the current leader for partition %s"
                                                   .format(preferredReplica, topicAndPartition))
    } else {
      info("Current leader %d for partition %s is not the preferred replica.".format(currentLeader, topicAndPartition) +
        " Triggering preferred replica leader election")
      // check if preferred replica is not the current leader and is alive and in the isr
      if (controllerContext.liveBrokerIds.contains(preferredReplica) && currentLeaderAndIsr.isr.contains(preferredReplica)) {
        (new LeaderAndIsr(preferredReplica, currentLeaderAndIsr.leaderEpoch + 1, currentLeaderAndIsr.isr,
          currentLeaderAndIsr.zkVersion + 1), assignedReplicas)
      } else {
        throw new StateChangeFailedException("Preferred replica %d for partition ".format(preferredReplica) +
          "%s is either not alive or not in the isr. Current leader and ISR: [%s]".format(topicAndPartition, currentLeaderAndIsr))
      }
    }
  }
}
```
+ ControlledShutdownLeaderSelector
```scala
/**
 * 从AR中挑选存活的broker组成新的ISR，并从新的ISR的随机挑选一个broker作为leader
 * 如果无法从AR中找到存活的broker，报StateChangeFailedException
 * New leader = replica in isr that's not being shutdown;
 * New isr = current isr - shutdown replica;
 * Replicas to receive LeaderAndIsr request = live assigned replicas
 */
class ControlledShutdownLeaderSelector(controllerContext: ControllerContext)
        extends PartitionLeaderSelector
        with Logging {

  this.logIdent = "[ControlledShutdownLeaderSelector]: "

  def selectLeader(topicAndPartition: TopicAndPartition, currentLeaderAndIsr: LeaderAndIsr): (LeaderAndIsr, Seq[Int]) = {
    val currentLeaderEpoch = currentLeaderAndIsr.leaderEpoch
    val currentLeaderIsrZkPathVersion = currentLeaderAndIsr.zkVersion

    val currentLeader = currentLeaderAndIsr.leader

    val assignedReplicas = controllerContext.partitionReplicaAssignment(topicAndPartition)
    val liveOrShuttingDownBrokerIds = controllerContext.liveOrShuttingDownBrokerIds
    val liveAssignedReplicas = assignedReplicas.filter(r => liveOrShuttingDownBrokerIds.contains(r))

    val newIsr = currentLeaderAndIsr.isr.filter(brokerId => !controllerContext.shuttingDownBrokerIds.contains(brokerId))
    liveAssignedReplicas.find(newIsr.contains) match {
      case Some(newLeader) =>
        debug("Partition %s : current leader = %d, new leader = %d".format(topicAndPartition, currentLeader, newLeader))
        (LeaderAndIsr(newLeader, currentLeaderEpoch + 1, newIsr, currentLeaderIsrZkPathVersion + 1), liveAssignedReplicas)
      case None =>
        throw new StateChangeFailedException(("No other replicas in ISR %s for %s besides" +
          " shutting down brokers %s").format(currentLeaderAndIsr.isr.mkString(","), topicAndPartition, controllerContext.shuttingDownBrokerIds.mkString(",")))
    }
  }
}
```
+ OfflinePartitionLeaderSelector
```scala
/**
 * 如果ISR中存在存活的副本，从ISR里存活的broker选择第一个broker作为Leader
 * 如果ISR中不存在存活的副本
 *   未开启unclean.leader.election，则抛出无在线副本异常：NoReplicaOnlineException
 *   开启unclean.leader.election，从AR中选择存活的broker作为新的leader以及新的ISR
 * 如果AR中也不存在存活的broker，则抛出无在线副本异常
 * Select the new leader, new isr and receiving replicas (for the LeaderAndIsrRequest):
 * 1. If at least one broker from the isr is alive, it picks a broker from the live isr as the new leader and the live
 *    isr as the new isr.
 * 2. Else, if unclean leader election for the topic is disabled, it throws a NoReplicaOnlineException.
 * 3. Else, it picks some alive broker from the assigned replica list as the new leader and the new isr.
 * 4. If no broker in the assigned replica list is alive, it throws a NoReplicaOnlineException
 * Replicas to receive LeaderAndIsr request = live assigned replicas
 * Once the leader is successfully registered in zookeeper, it updates the allLeaders cache
 */
class OfflinePartitionLeaderSelector(controllerContext: ControllerContext, config: KafkaConfig)
  extends PartitionLeaderSelector with Logging {
  this.logIdent = "[OfflinePartitionLeaderSelector]: "

  def selectLeader(topicAndPartition: TopicAndPartition, currentLeaderAndIsr: LeaderAndIsr): (LeaderAndIsr, Seq[Int]) = {
    controllerContext.partitionReplicaAssignment.get(topicAndPartition) match {
      case Some(assignedReplicas) =>
        val liveAssignedReplicas = assignedReplicas.filter(r => controllerContext.liveBrokerIds.contains(r))
        val liveBrokersInIsr = currentLeaderAndIsr.isr.filter(r => controllerContext.liveBrokerIds.contains(r))
        val currentLeaderEpoch = currentLeaderAndIsr.leaderEpoch
        val currentLeaderIsrZkPathVersion = currentLeaderAndIsr.zkVersion
        val newLeaderAndIsr =
          if (liveBrokersInIsr.isEmpty) {
            // Prior to electing an unclean (i.e. non-ISR) leader, ensure that doing so is not disallowed by the configuration
            // for unclean leader election.
            if (!LogConfig.fromProps(config.originals, AdminUtils.fetchEntityConfig(controllerContext.zkUtils,
              ConfigType.Topic, topicAndPartition.topic)).uncleanLeaderElectionEnable) {
              throw new NoReplicaOnlineException(("No broker in ISR for partition " +
                "%s is alive. Live brokers are: [%s],".format(topicAndPartition, controllerContext.liveBrokerIds)) +
                " ISR brokers are: [%s]".format(currentLeaderAndIsr.isr.mkString(",")))
            }
            debug("No broker in ISR is alive for %s. Pick the leader from the alive assigned replicas: %s"
              .format(topicAndPartition, liveAssignedReplicas.mkString(",")))
            if (liveAssignedReplicas.isEmpty) {
              throw new NoReplicaOnlineException(("No replica for partition " +
                "%s is alive. Live brokers are: [%s],".format(topicAndPartition, controllerContext.liveBrokerIds)) +
                " Assigned replicas are: [%s]".format(assignedReplicas))
            } else {
              ControllerStats.uncleanLeaderElectionRate.mark()
              val newLeader = liveAssignedReplicas.head
              warn("No broker in ISR is alive for %s. Elect leader %d from live brokers %s. There's potential data loss."
                .format(topicAndPartition, newLeader, liveAssignedReplicas.mkString(",")))
              new LeaderAndIsr(newLeader, currentLeaderEpoch + 1, List(newLeader), currentLeaderIsrZkPathVersion + 1)
            }
          } else {
            val liveReplicasInIsr = liveAssignedReplicas.filter(r => liveBrokersInIsr.contains(r))
            val newLeader = liveReplicasInIsr.head
            debug("Some broker in ISR is alive for %s. Select %d from ISR %s to be the leader."
              .format(topicAndPartition, newLeader, liveBrokersInIsr.mkString(",")))
            new LeaderAndIsr(newLeader, currentLeaderEpoch + 1, liveBrokersInIsr.toList, currentLeaderIsrZkPathVersion + 1)
          }
        info("Selected new leader and ISR %s for offline partition %s".format(newLeaderAndIsr.toString(), topicAndPartition))
        (newLeaderAndIsr, liveAssignedReplicas)
      case None =>
        throw new NoReplicaOnlineException("Partition %s doesn't have replicas assigned to it".format(topicAndPartition))
    }
  }
}

```
+ ReassignedPartitionLeaderSelector
```scala
/**
 * 从均衡完成后新的AR中选取存活的broker作为新的ISR
 * 如果新的ISR不为空，选择第一个broker作为新的Leader
 * 如果新的ISR为空，报无在线副本异常NoReplicaOnlineException
 * New leader = a live in-sync reassigned replica
 * New isr = current isr
 * Replicas to receive LeaderAndIsr request = reassigned replicas
 */
class ReassignedPartitionLeaderSelector(controllerContext: ControllerContext) extends PartitionLeaderSelector with Logging {
  this.logIdent = "[ReassignedPartitionLeaderSelector]: "

  /**
   * The reassigned replicas are already in the ISR when selectLeader is called.
   */
  def selectLeader(topicAndPartition: TopicAndPartition, currentLeaderAndIsr: LeaderAndIsr): (LeaderAndIsr, Seq[Int]) = {
    val reassignedInSyncReplicas = controllerContext.partitionsBeingReassigned(topicAndPartition).newReplicas
    val currentLeaderEpoch = currentLeaderAndIsr.leaderEpoch
    val currentLeaderIsrZkPathVersion = currentLeaderAndIsr.zkVersion
    val aliveReassignedInSyncReplicas = reassignedInSyncReplicas.filter(r => controllerContext.liveBrokerIds.contains(r) &&
                                                                             currentLeaderAndIsr.isr.contains(r))
    val newLeaderOpt = aliveReassignedInSyncReplicas.headOption
    newLeaderOpt match {
      case Some(newLeader) => (new LeaderAndIsr(newLeader, currentLeaderEpoch + 1, currentLeaderAndIsr.isr,
        currentLeaderIsrZkPathVersion + 1), reassignedInSyncReplicas)
      case None =>
        reassignedInSyncReplicas.size match {
          case 0 =>
            throw new NoReplicaOnlineException("List of reassigned replicas for partition " +
              " %s is empty. Current leader and ISR: [%s]".format(topicAndPartition, currentLeaderAndIsr))
          case _ =>
            throw new NoReplicaOnlineException("None of the reassigned replicas for partition " +
              "%s are in-sync with the leader. Current leader and ISR: [%s]".format(topicAndPartition, currentLeaderAndIsr))
        }
    }
  }
}
```
+ NoOpLeaderSelector
```scala
/**
 * 不做任何操作，只是返回当前的Leader、ISR和AR
 * Essentially does nothing. Returns the current leader and ISR, and the current
 * set of replicas assigned to a given topic/partition.
 */
class NoOpLeaderSelector(controllerContext: ControllerContext) extends PartitionLeaderSelector with Logging {

  this.logIdent = "[NoOpLeaderSelector]: "

  def selectLeader(topicAndPartition: TopicAndPartition, currentLeaderAndIsr: LeaderAndIsr): (LeaderAndIsr, Seq[Int]) = {
    warn("I should never have been asked to perform leader election, returning the current LeaderAndIsr and replica assignment.")
    (currentLeaderAndIsr, controllerContext.partitionReplicaAssignment(topicAndPartition))
  }
}
```

## HandleLeaderAndIsr
+ 核心功能：经历选主操作后，目标broker上的部分partition replica可能会晋升成Leader，而部分partition replica可能会由Leader退化成Follower，broker需要正确处理这两种场景。
+ 流程图
+ 流程讲解
  + 屏蔽stale请求-拒绝epoch小于当前broker的请求
  + becomeLeaders
    + 1 停掉这些partitions的fetchers
    + 2 更新缓存中的partition metadaata
      + 2.1 重置AR中其他replicas的LEO
        + 该broker上次担任Leader时可能存在旧的LEO
      + 2.2 设置新的Leader和新的ISR
      + 2.3 更新HW
        + 疑问：ISR长度为1时，直接使用LEO作为HW？
    + 3 将这批partitions加入leader partitions set
  + becomeFollowers
    + 1 将这批partitions从leader partitions set中移除
    + 2 将这批replicas标记为follower，以便生产者客户端无法写入更多的数据
      + 2.1 设置新的Leader
      + 2.2 将ISR设置为EMPTY
      + 注：只有在leader存活时，才标记为follower
    + 3 停掉这批partitions的的fetchers，以便ReplicaFetcherThread无法添加更多数据。
    + 4 对这批partitions的log截断到HW，并对其执行checkpoint
    + 5 清理purgatory中的produce和fetch请求
    + 6 如果这个broker并未执行下线，为这批partition新的Leaders添加fetchers
# LogReplication
# MemberManagement
# 和paxos算法的异同

