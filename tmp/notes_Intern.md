# 学习分享整理

## 技术组件

### 1. LevelDB

#### 基本概念与架构

   总结来说，LevelDB中数据的写入过程为：
   先写入内存数据结构MemTable，当MemTable大小达到阈值时转化为Immutable MemTable，后台线程将它异步Flush到磁盘，同时生成一个新的MemTable。

#### 特性思考

1. **何为“Level”？**
   LevelDB中“**Level**”的含义在于：LevelDB在磁盘中维护了多层级（Level 0～N）的文件（SSTable文件，每个文件内部的key是有序的）。在每个Level上分布着多个SSTable文件，Level 0中的SSTable文件是直接由内存中的Immutable Memtable刷盘得到的，所以Level 0中的不同SSTable之间可能存在key重复；而Level 1~N中的SSTable文件是通过**Compaction**得到的。
   我们已经知道，LevelDB的写入过程十分简单，在日志写入完成后，只需在内存数据结构MemTable中写入（删除也只需对key写入一个删除标记）即可，MemTable写满后就会持久化刷盘变成一个SSTable文件，显而易见，这些初级的SSTable文件（Level 0中）之间是可能存在重复的。
   Compaction可以对SSTable文件进行整理压缩，删除重复的KV数据。通过Compaction操作，将多个SStable文件压缩成一个，并放置在下一个Level上，如此往复，便形成了多个“Level”。
2. **为什么快？**
    官方号称LevelDB的随机写性能可以达到40万条记录每秒，随机读性能可以达到6万条记录每秒。为什么LevelDB并不是纯内存的，写性能可以如此之快，又为什么随机读的性能会比随机写的性能更低呢？
    - **写入**：对于一个插入操作Put(Key,Value)来说，完成插入操作包含两个具体步骤：
      1. 首先是写log文件，将这条KV记录以**顺序写**的方式追加到log文件末尾，尽管这是一个磁盘读写操作，但是采用文件的**顺序追加写入方式效率很高**；
      2. 如果log文件写入成功，那么将这条KV记录插入内存中的Memtable中，其底层是一个Key有序的SkipList（跳表），插入过程十分高效。完成这一步，写入记录就算完成了，所以一个插入记录操作涉及**一次磁盘文件追加写和内存插入操作**，所以LevelDB的写入性能极高。
        ![alt text](image-1.png "LevelDB写入流程")
    - **读取**：相比于写入，LevelDB中的读取过程更加复杂，数据可能存在于两个地方：内存中或磁盘中。所以在查找时，会按照以下三步进行：先查找Memtable，若没有查到则找Immutable Memtable，没查到则找SSTFiles。如果某个数据在SSTFiles中了，那么查找的速度就会比写入慢很多。具体流程如下图所示：
        ![alt text](image.png "LevelDB读取流程")
    为了加快读取速度，LevelDB引入了LRUCache来加速读取，由于SSTable文件是只读的，所以并不用关心数据一致性的问题。
3. **可靠性**
   - Redis因其高性能与易用性广受开发者青睐，但是由于Redis是纯内存数据库，在海量数据对情况下对内存占用较高，并且更多用于缓存使用，而不作为一个独立的数据库，虽然Redis提供了RDB持久化机制，但RDB实质是数据快照的形式，并不是实时保存的，不能完全保证数据安全性，而采用AOF持久化机制又会带来崩溃后恢复时间较长的问题。
   - LevelDB的数据同时存储在内存和磁盘中，数据写入时先写内存，然后异步存入磁盘中。LevelDB的内存占用情况不会像Redis一样随数据量增大而线性增大。
   - 数据再刷盘之前，是存储在内存中的（Memtable和Immutable Memtable），如果还未刷盘程序就宕机了，可以通过日志文件进行恢复。LevelDB的写入流程是先顺序写磁盘日志，再写入内存，从而能够保证数据可靠性（但如果日志文件还没有被操作系统刷盘，机器宕机了，数据仍有可能丢失）。
4. **性能瓶颈**
   - **Compaction**：Compaction操作是对磁盘中的数据进行的，当Level 0 中的文件数量达到阈值时（默认是4），就会触发一次Major Compaction，如果此时写入速度非常快，*超过了Compaction的速度*，就会造成Level 0中的SSTable文件数量一直增长。一直增长会带来两方面的主要问题：其一是读性能急剧降低，其二是占用过多的存储空间。实际上，LevelDB有两个Trigger，当0层文件数量超过`SlowdownTrigger`时，写入的速度减慢，当0层文件数量超过`PauseTrigger`时，写入暂停，直至Major Compaction完成。
   - **Compaction策略**：Compaction操作会带来大量磁盘IO开销，这可能影响写入和读取速度，所以进行Compaction操作的时机与策略至关重要。例如，如何设置各个Level的文件阈值、尽量错峰进行Compaction等。
   - **单机限制**：LevelDB是一个KV存储框架，并没有提供分布式能力，单机的通信和IO能力可能存在瓶颈，可以引入分布式架构，利用主从/集群架构实现读写分离和数据分片，可在特定场景下提高数据库性能。
5. **使用场景**
   从前文的分析，我们已经知道：LevelDB的写入性能极高，而读取性能相对较弱。所以LevelDB适用于写多读少的场景，尤其是需要高性能的持久化数据存储，但不需要过于复杂的查询和事务处理的场景。例如：
   1. 日志存储和分析系统：LevelDB适合用于存储和分析大量的日志数据。它可以高效地处理大量的写入操作，并支持按时间范围进行查询和检索。
   2. 需要存储大量数据，但查询次数相对较少的场景：例如视频网站为了实现多端续播，需要实时写入用户的观看记录（写请求很多），而用户重新观看时只需读取一次历史观看位置（写请求较少）。

#### 性能测试与对比

Todo...

### 2. 分布式协议

> 在分布式系统中，总会发生诸如机器宕机或网络异常（包括消息的延迟、丢失、重复、乱序，还有网络分区）等情况。各个节点之间要保持数据一致性，就需要一种共识算法（分布式协议）来约束各个节点。
> 分布式场景中，存在“拜占庭将军问题”（Byzantine failures）：间隔很远的每支军队依靠信使传递信息，但某些将军可能叛变。要解决拜占庭将军问题，就需要在一个可能有叛徒的非信任环境中建立战斗共识。对应到分布式场景中，叛变的将军就是恶意节点，所以按照是否容忍拜占庭容错，可分为CFT类共识算法（非拜占庭容错，不考虑恶意节点）和BFT类共识算法（拜占庭容错，考虑恶意节点）。

下文中记录的两个共识算法都是CFT类（非拜占庭容错）共识算法。

#### Paxos

**Paxos简介**：
Paxos是由Lamport大师提出的基于消息传递的分布式一致性算法，被视为当今分布式一致性算法的基石。Paxos系统中有三个角色：Proposer（提议者），Acceptor（决策者）和Learner（决策学习者）。其共识过程是Proposer提出提案来争取Acceptors的支持，当获得超过半数的支持时，则认为提案通过，发送提案结果给所有节点进行确认。在实际应用时，每个节点可以同时担任三种角色。

**Paxos流程**：
Paxos算法分为两个阶段，**Prepare**和**Propose**，具体如下：

- **阶段一**：
  **(a)** Proposer选择一个提案编号N，然后向Acceptor发送编号为N的Prepare请求。
  **(b)** 如果一个Acceptor收到一个编号为N的Prepare请求，且N大于该Acceptor已经响应过的所有Prepare请求的编号，那么它就会将它已经接受过的编号最大的提案（如果有的话）作为响应反馈给Proposer，同时该Acceptor承诺不再接受任何编号小于N的提案。
- **阶段二**：
  **(a)** 如果Proposer收到**半数以上**Acceptor对其发出的编号为N的Prepare请求的响应，那么它就会发送一个针对[N,V]提案的Accept请求给半数以上的Acceptor。注意：V就是收到的响应中编号最大的提案的value，如果响应中不包含任何提案，那么V就由Proposer自己决定。
  **(b)** 如果Acceptor收到一个针对编号为N的提案的Accept请求，只要该Acceptor没有对编号大于N的Prepare请求做出过响应，它就接受该提案。

    ![alt text](image-2.png "Paxos算法流程")

Acceptor在收到Prepare请求时的两个承诺和一个应答：

1. 承诺不再接受Proposal ID小于等于当前请求的Prepare请求；
2. 承诺不再接受Proposal ID小于当前请求的Propose请求；
3. 应答已经Accept过的Propose中ID最大的那个提案的Value和ID，没有则返回空值。

**活锁问题**：
由于Paxos允许同时存在多个Proposer，在极端情况下，可能存在活锁问题：
![alt text](image-3.png)

**Multi-Paxos**：
上文提到，Paxos可能存在活锁问题，会导致系统进入死循环，其原因在于Basic Paxos允许同时存在多个Proposer。并且Basic Paxos只能对一个提案值形成决议，决议的形成至少需要两次网络来回（Preapare和Propose两个阶段）。在实际工程应用中，使用更多的是**Multi-Paxos**。
Multi-Paxos基于Basic Paxos做了以下改进：
在所有的Proposers中选举一个Leader，每次由Leader唯一地提交Proposal。这样由于没有Proposer竞争，避免了活锁问题，同时可以省略Prepare操作，提高效率。
选取Leader的阶段也是一次决议的形成，这一阶段需要运行一次Basic Paxos。Leader选取完成后，由于系统中只有一个Proposer，所以可以跳过Preapare阶段。

Chubby、ZooKeeper等框架的一致性算法都是基于Multi-Paxos实现的。

#### Raft

> Raft的作用跟Multi-Paxos相同，但结构更加简单容易理解，是对Paxos的简化实现。
> "RAFT"四个字母是对Reliable, Replicated, Redundant, And Fault-Tolerant对缩写，利用[日志连续性]对Paxos做了很好的简化。

Raft系统中有三种角色：**Leader**、**Follower**和**Candidate**，其中Candidate在选举时才会出现。并且**同一时刻至多只有一个Leader**，Leader会负责所有外部的请求，如果Follower收到了请求，则会将请求路由到Leader。

**Raft流程**：
Raft通过分解问题，将一致性目标分解成了若干子问题，总结起来可分为这三个方面：Leader Election（主节点选举）、Log Replication（日志同步）、Safety（保证以上目标的约束条件）。

- **Leader Election**：关于主节点选举，这里直接列出Raft论文中使用状态机描述的图片，对节点角色的转变有很清晰 的描述：
  ![alt text](image-4.png)
  需要补充的两点是：在选取Leader时，总是会选取term（任期）最大、数据最全的那个节点；为了避免总是有大量节点同时发起投票，Raft引入了随机Time Out的机制来错开不同Candidate的投票过程。
- **Log Replication**：系统选出Leader后，就可以对外提供服务了，客户端只能直接和Leader进行交互。在Leader接受请求后，这个请求是`uncommited`状态，Leader将请求封装后通过RPC发送给Follower，Follower接收后返回ACK；在Leader收到大部分Follower的回复后，会将这条请求的状态设置为`commited`，并对客户端返回成功，此时这条消息就视为写入成功了，后续Leader会继续对Follower发送状态变更信息。
- **Safety**：在实际的场景中，可能会面临各种宕机、网络连接等问题，下面分析几种可能的宕机的情况：
  1. **Follower宕机**：Follower宕机后恢复的情况较为简单，恢复后直接重新加入集群并接收来自Leader的消息，同步数据即可。
  2. **Leader宕机**：Leader宕机的情况相对复杂，具体宕机时间节点可分为7种情况和1种网络分区情况，Raft协议可以很好地处理这些情况，并不会出现一致性问题，具体细节将在[崩溃恢复](#崩溃恢复)部分讨论。

<span id="崩溃恢复"></span>

**崩溃恢复**：
**Follower宕机**：Follower宕机后恢复的情况较为简单，恢复后直接重新加入集群并接收来自Leader的消息，同步数据即可。
**Leader宕机**：具体分为7种Leader宕机情况和1种网络分区情况：

1. 数据到达Leader前：此时Leader还未接收数据，不影响一致性；
2. 数据到达Leader，但还未同步到Follower：Follower上没有该数据，Follower重新选主后集群也没有数据，旧Leader恢复后成为Follower，同步数据后删除`uncommited`的数据，不存在一致性问题。
3. 数据到达Leader，同步到了所有Follower，但没有向Leader回复：重新选出的主节点具有该数据，会`commit`这个数据，不存在一致性问题。客户端都不会收到成功的响应，可进行重试，需要在工程实现时考虑**幂等**。
4. 数据到达Leader，同步到了大多数Follower，但没有向Leader回复：由于Raft要求节点只能投票给数据比自己更新的Candidate，并要求Candidate获取集群中过半数节点的选票，所以新的Leader是含有这条数据的。新的Leader会`commit`这条数据并同步给其他节点。这种情况下客户端同样不会感知到数据请求成功，同样需要实现幂等。
5. **数据到达Leader，同步到了少数Follower，并且没有向Leader回复**：此时会出现两种情况,1:已经同步了该数据的节点被选为Leader，会commit这条数据并同步到其他节点；2:由于大多数节点并没有收到这条数据，没同步该数据的节点也可能被选为Leader，那么这条数据信息就会丢失。
6. 数据到达Leader，成功复制到大多数/所有Follower，Leader已提交这个数据，但Follower还未提交：此时客户端已接收到成功，重新选主后新Leader一定包含这条数据，然后`commit`这条数据，不存在一致性问题。
7. 数据到达Leader，成功复制到大多数/所有Follower且已提交，还未响应客户端：重新选主后数据是一致的，注意幂等即可。
8. 网络分区导致脑裂，出现双Leader：旧Leader仍然接受客户端请求，但由于不能和Follower通信，所以数据永远不会被`commit`，新Leader选出后正常接受数据，旧Leader网络恢复后会变为Follower，然后同步数据，不会出现数据不一致。

**与Paxos对比**：

- 日志提交顺序：Raft按顺序提交日志，而Paxos允许不按序提交，Paxos需要额外的协议来填补可能出现的日志问题。
- 日志信息：Raft中所有日志的副本都具有相同索引、任期和命令，而Paxos中可能会有所不同。
- Leader选举过程：Raft选出具有多数日志的节点作为Leader，无需重新同步日志。Multi-Paxos可能会选举出一个宕机了很久的节点作为Leader，从而带来大量的日志同步开销。

### 3. 分布式锁

- #### Chubby

- #### Redis

- #### Zookeeper

## 业务系统
