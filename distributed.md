# 分布式

## 共识算法

设想有一台游戏服务器或者一台负责交易撮合的服务器，如果当前的状态只在一台服务器，那么当服务器宕机时，这个时候服务就会不可用，需要重启服务器，重启时还要加载磁盘的状态以恢复内存的状态。为了做高可用，引入了共识算法，简而言之，就是做服务状态的副本，设置N台服务器，但是从外部看来，仍然满足一定的一致性，比较严格的一致性就是线性一致性了。

线性一致性：就是在外部看来，对N台服务器的读写，就像对一台服务器的读写一样。也就是说如果写入了最新值，就不会读到旧的值。

### Raft算法

https://blog.csdn.net/z69183787/article/details/112168120

https://ms2008.github.io/2019/12/04/etcd-rumor/

Raft协议分选主(Leader Election)和日志复制(Log Replication)两部分。

**选主**

Raft为集群中的节点定义了三种状态，分别是Follower，Candidate，Leader。

选主步骤如下：

* Candidate状态的节点会向其他节点发送vote_request，参数需要带上当前节点的term，term表示第几轮选举，每当Candidate发起新一轮的投票请求，term就会加1，带有过期term的消息不会被处理(如heartbeat，vote_request，vote)。
* Follower节点收到vote_request，如果Follower节点检测Leader故障(**election timeout**时间内没收到heartbeat)，且在该term并未投过票，则投票给Candidate。
* 某Candidate收到超过半数的投票，成为Leader。
* Leader节点每隔**heartbeat timeout**时间就会给其他节点发送心跳。**election timeout**的时间是150~300毫秒之间的随机值。

**日志复制**

Raft协议规定只能通过Leader来写，可以从任意节点读。Leader通过日志复制的方式将数据同步到其他节点。Leader需要更新的数据封装在Append Entries消息中，通过heartbeat传输给其他节点。写数据一共分两个阶段：

* 第一阶段：写日志，不提交。Append Entries={在日志中的编号，要更新的数据，不提交}。当Follower收到Append Entries消息后，如果日志不冲突，将Append Entries插入到日志中，回应Leader可写。当Leader收到超过半数的节点回应时，发起第二阶段写。
* 第二阶段：提交。Append Entries={在日志中的编号，提交}，Leader回应客户端写成功（**数据已不可逆转**）。当Follower收到Append Entries消息后，在日志中提交数据。

**状态机**

Raft节点收到提交的log后，会将其应用到自己的状态机（内存、磁盘等），当收到某条commit的log，记为commit index，当某条log被应用到状态机，记为apply index。如果各个节点的初始状态相同，应用相同的log后，最终各个节点会达到相同的状态。

#### Aeron Cluster

Aeron Cluster实现了Raft算法，下面是Aeron集群核心方法：

```java
public interface ClusteredService
{
    // Start event for the service where the service can perform any initialisation required and load snapshot state.
    void onStart(Cluster cluster, Image snapshotImage);

    // A session has been opened for a client to the cluster.
    void onSessionOpen(ClientSession session, long timestamp);

    // A session has been closed for a client to the cluster.
    void onSessionClose(ClientSession session, long timestamp, CloseReason closeReason);

    // A message has been received to be processed by a clustered service.
    void onSessionMessage;

    // The service should take a snapshot and store its state to the provided archive
    void onTakeSnapshot(ExclusivePublication snapshotPublication);

    // Notify that the cluster node has changed role.
    void onRoleChange(Cluster.Role newRole);

    // An election has been successful and a leader has entered a new term.
    default void onNewLeadershipTermEvent;
}
```











