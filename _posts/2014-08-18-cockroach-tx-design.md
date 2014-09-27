---
layout: post
category: "read"
title:  "Cockroach design(transaction)"
tags: [Cockroach, Transaction]
---
### Lock-Free Distributed Transaction
Cockroach提供免锁的分布式事务。Cockroach事务支持两种隔离级别：Snapshot Isolation（SI）和Serializable Snapshot Isolation（SSI）。SI实现起来比较容易，性能也高，在大多数场景下能保证正确性（例外情况包括Write Skew）。SSI的实现难度较大，同样也能做到高性能（在冲突较少的情况下），正确性也没有问题。Cockroach的SSI实现是基于文献内容的，以及一些可能比较新颖的想法。

SSI是缺省的隔离级别，SI为开发人员提供了足够高的性能，但同时需要容忍Write Skew问题。在轻度冲突的系统中，我们的SSI实现不需要锁或者额外的写操作，和SI的性能基本相同。在有冲突的情况下，我们的SSI实现同样也不需要锁，但是会中止更多的事务。Cockroach的SI和SSI实现防止了任意的长事务出现饥饿的场景。

Cahill的论文中提出了一种可能的SSI实现。另有一篇论文也不错（Calvin）。关于使用防止读写冲突的方式来实现SSI（而不是去检测它们，称之为write-snapshot isolation）的讨论，可以参考Yabandeh的论文（a critique of si），此文章给了Cockroach的SSI许多灵感。

每个Cockroach事务在启动时都被分配了一个随机的优先级，以及一个暂定的时间戳。暂定时间戳预计了事务将在何时提交。在多个分布的节点之间组织事务的过程中，此时间戳可能会增加，但是绝对不会减少。时间戳是物理时钟和逻辑时钟的组合，用于支持单调递增，而不会出现退化场景以导致时间戳偏离实际时钟，参考Hybrid Logical Clock的论文。

事务分两个阶段执行：

第一阶段：
在事务写的每个数据中都加一个"intent"值。这些都是普通的MVCC值，附加一个特别的标签（Intent）来指示这个值可能后面要提交，如果事务自己提交的话（？？）。此外，事务ID（唯一、在事务启动时由客户端获取）也存储在Intent值中。事务ID用于在事务表格中进行查找，以解决冲突，或在两个相等的时间戳之间进行排序（时间戳会相等？？）。每个节点都返回写操作所用的时间戳；客户端选择最大的一个做为最终提交的时间戳。

每个Range维护一个小的（最新10秒钟的读时间戳）、内存中的 Cache，Cache的内容是key到最近读此key的时间戳的映射。这个latest-read-cache在每次写的时候都会用到。如果写操作的备选时间戳早于cache的低水位线（最后被移出的时间戳）或者将要写的key的读时间戳比备选时间戳要晚，较晚的时间戳值将和写操作一同返回。cache的条目中将先移出较早的时间戳，并相应的更新低水位线。如果选了一个新的range副本leader，它将低水位线设置为（当前wall time ＋ 时钟偏差的99％）


第二阶段：
通过在系统事务表格中写一个条目（key的前缀是\0tx）来提交事务。提交条目的值中包含了备选时间戳（在必要时进行增加来适应任何最新的读时间戳）。需要注意的是，此时刻事务就被认为是完全提交了，控制权可以返回给客户端

在SI事务的情况下，如果一个提交时间戳进行增加来适应并发的读事务，这是完全可以接受的，而且提交也可以继续。对于SSI事务，暂定时间戳和提交时间戳不同会造成事务重启（重启和中止是不同的）

此外同时，所有已写的值会更新以删除之前的"intent"标签。事务在这一步之前就认为是完全提交了，并不会等待以返回控制权给协调者。

如果没有冲突的话，这样就结束了。不需要做其他的事情以保证系统的正确性。但当系统出现冲突时，情况就变的有趣了。有这样几种可能：

Reader碰见有较新时间戳的write intent：reader可以继续运行；它将读到该值的一个较老的版本，因此不会有冲突。回忆一下，write intent可能会使用比备选时间戳更晚的时间戳来进行提交；它永远不会使用更早的时间戳进行提交。另外注意：如果reader发现了自身事务写入的一个有着较新时间戳的intent，reader就会返回那个值。
Reader碰见有较早时间戳的write intent：reader必须在事务表格中查询intent的事务ID的状态。如果事务已经提交了，reader可以读取这个值。如果写事务尚未提交，则reader有两个选择。如果写冲突是来自一个SI事务，reader可以将那个写事务的提交时间戳向后移（push into the future）。这样做起来很简单：reader直接去更新事务的提交时间戳，来指明当事务提交时，它的时间戳至少不能小于此时间戳。但是当写冲突是来自于一个SSI事务，reader需要比较优先级。如果它有着较高的优先级，则它就像SI事务一样后移写事务的提交时间戳，这样写事务就可以发现自己的时间戳已经被后移了，将会重启。如果它的优先级较低，它就将自己中止，然后使用一个新的优先级来重试，新的优先级为（新的随机优先级，冲突事务的优先级－1，两者之间的最大值）
Writer碰见有着较低优先级的未提交的write intent：writer将中止冲突的事务。
Writer碰见有着较高优先级的未提交的write intent或者新提交的值：writer将中止自身，接着使用一个新优先级（新的随机优先级，冲突事务的优先级－1，两者之间的最大值）来重试。重试将在一个较短的，随机的间隔后开始。
Writer碰见有着较低优先级的已提交的write intent或者新提交的值：writer不会中止，但是会重启事务。重启时将使用相同的优先级，但是备选时间戳将会前移到提交时间戳。提交时间戳必须比碰见的已提交的写的时间戳大。

客户端维护了一系列的事务之前准备写的key。当事务重启时，它将这些key去掉，将它们重写一次。剩下的key。





### Transaction Management
事务是由Client Proxy管理的（或者SQL A zure parlance中的gateway）。和Spanner不同，写操作是不会缓存的，会直接发送给所有涉及的Range。这样可以使得在出现写冲突的时候，事务可以快速中止。Client Proxy需要跟踪所有的写key，以在事务结束时删除写入的intent。

如果一个事务成功结束了，所有的intent被升级成committed。当事务中止时，所有的intent被删除。client proxy不保证会清除intent；但是dangling intent会在后续的reader和writer碰见时进行升级和删除，而且系统不依赖于及时删除intent而保证正确性。

当client proxy在"运行事务结束之前"重新启动的场景下，dangling事务会继续存在于事务表格中，直到被其他事务中止。缺省情况下，事务每5秒钟会对事务表格发送一次心跳。reader和writer碰见的有dangling intent的事务，如果在规定间隔内没有心跳的话，会将其中止。

对于中止事务的重试和中止的详细研究，可以参考文献「」


### Transaction Table
隔离级别定义：SI＝1；SSI＝2
事务状态定义：正在运行＝1；提交＝2；中止＝3
事务详细：事务ID、备选时间戳、心跳时间戳、优先级、隔离级别、状态


#### 优点：

不需要可靠的代码执行，以防止stalled 2PC协议
SI的语义下，读不会阻塞；SSI的语义下，读可能会中止
比传统2PC时延更低，因为第二阶段只需要一次向事务表格中的写入，而不是向所有参与者的同步交互
优先级避免了任意长度事务的饥饿，
写操作不在客户端缓存，写操作失效恢复较快
对于SSI而言，没有读锁的开销（与其他SSI实现相比）
精心选择的优先级可以为任意事务设置大概的时延保证（例如：让OLTP事务中止的可能性比低优先级事务低10倍，如异步调度任务）

#### 缺点：
来自非副本Leader的读仍然需要向Leader发送一个ping，以更新last-read-cache
中止的事务可能会在心跳周期内对冲突的writer进行阻塞，虽然平均的等待时间不长。相对于检测和重启2PC，以释放读写锁的方式，这样可能会性能更好
和其他SI实现不同：不是first writer win，较短的事务不会永远都是快速的结束。这和以往的OLTP系统特征不同。
与2PL相比，在有冲突的系统中的中止将会降低吞吐量。中止和重试将增加读写流量，造成时延增加以及吞吐量的降低。


#### 选择一个时间戳
在时间偏移的分布式系统中读取数据的一个关键挑战是：如何选择一个时间戳（绝对时间），以保证其大于所有已经提交事务的最新时间戳。如果系统无法读取到已提交的数据，则无法声称自己的一致性。

如果事务只访问单个节点是比较简单的。事务指定时间戳为0。表明节点应该使用它自己的当前时间（节点以HLC的形式维护时间，wall time ＋ logical time）。这保证了节点上已经提交的数据有着较早的时间戳。

对于访问多个节点的情况，将使用节点"t"。此外，最大时间戳"t+e"将为已提交的数据提供时间戳的上限（e是最大的时间偏移）。当事务运行时，任何读到的数据，如果其时间戳位于（t，t+e）区间内，则会导致事务中止，并使用冲突时间戳tc进行重试，其中tc > t。最大时间戳t+e还是维持原样。

我们假定重试是很少的，但是如果重试变的有问题，则需要重新考虑这一假定。需要注意的是，对于historical read则不存在这个问题。一个不需要重试的可选方案是预先对所有涉及节点进行一次轮询，并选择节点返回的最高的wall time做为时间戳。但是预先获知将访问哪些节点是很困难的。Cockroach也可以潜在的使用一个全局时钟（Google通过Pinax实现），对于较小的、geographically-proximate的集群而言，这也是可行的。

#### Linearizability
先看一下Spanner。通过。，Spanner提供了任意两个Non-Overlapping事务（绝对时间）的全局顺序，其时延约为14ms。换句话说，Spanner保证了如果事务T1的提交早于事务T2的启动，则为T1分配的提交时间戳小于T2的。使用原子钟以及GPS接收器，Spanner将时钟偏移的不确定性控制在10ms以内。为了满足一致性的保证，事务必须使用两倍的时钟漂移量来提交。论文（Spanner"s Concurrency Control）中有详细描述。

在没有特殊硬件的情况下，Cockroach也可以做出相同的保证，但其代价是更长的等待时间。如果集群只使用NTP，事务等待时间可能会超过150ms。对于广域网而言，跨数据中心的连接延迟会使得此问题看起来不那么严重。但如果时钟可以更加精确，提交延迟将可以得到改善。

然而，让我们回过头看一下，是否 Spanner的外部一致性保证值得去进行自动的提交等待。首先，如果提交等待被完全省略，。但是在时钟偏移的情况下，。换句话说，如果没有全局顺序，可能会出现下面的场景：
启动事务T1以修改值x，提交时间为s1
在T1提交时，启动T2以修改值y，提交时间为s2
读取x和y，发现s1 > s2


Spanner保证的外部一致性被称为linearizability。Spanner的保证可以被形式化为：任意两个进程，其时钟偏移处于某个区间内，可以独立的记录它们的wall time（T1的结束为Tend1，T2的开始为Tstart2），如果后续比较得到Tend1 < Tstart2，则提交时间s1 < s2。这样的保证已经可以覆盖所有显式因果性的场景，以及可以想象到的隐式因果性的场景。

我们的论点是：从单个客户端或一系列连续的客户端而言，因果性是首要重要的。正如此，Cockroach提供了两个机制，来为没有精确控制时间偏移的系统提供linearizability，同时避免大量的事务提交等待：

1）客户端提供最高的事务提交时间戳，。

新启动的客户端从进程启动到开始它的首个事务，至少等待2＊e的时间。

Cockroach中所有因果相关的事件都用于维护linearizability。例如消息队列，保证了接收的时间戳晚于发送的时间戳，已送达的消息

2）已提交的事务


在提交等待时使用这些机制，Cockroach的保证可以被形式化为：事务T2的启动时间Tstart2晚于事务T1的结束时间Tend1，说明事务的提交时间戳s1 < s2