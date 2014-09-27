---
layout: post
category: "read"
title:  "An critique of ANSI SQL isolation levels"
tags: [Database, Transaction]
---
### 1. Introduction

Running concurrent transactions at different isolation levels allows application designers to trade throughput for correctness. Lower isolation levels increase transaction concurrency but risk showing transactions a fuzzy or incorrect database. Surprisingly, some transactions can execute at the highest isolation level (perfect serializability) while concurrent transactions running at a lower isolation level can access states that are not yet committed or that postdate states the transaction read earlier [GLPT]. Of course, transactions running at lower isolation levels may produce invalid data. Application designers must prevent later transactions running at higher isolation levels from accessing this invalid data and propagating errors. 
以不同的隔离级别运行并发事务，可以使应用设计人员进行吞吐量和正确性的权衡。低的隔离级别提升了事务并发度，但是可能会使事务看到一个不正确的数据库。令人惊讶的，一些事务可以运行在最高的隔离级别（完美的可串行化）上，但同时在较低隔离级别上运行的并发事务可以访问尚未提交的状态，或是事务先前读取的过期数据。当然，运行在较低隔离级别的事务可能会产生无效的数据。应用设计人员必须防止较高隔离级别的事务后续访问这些无效数据。

The ANSI/ISO SQL-92 specifications [MS, ANSI] define four isolation levels: (1) READ UNCOMMITTED, (2) READ COMMITTED, (3) REPEATABLE READ, (4) SERIALIZABLE. These levels are defined with the classical serializability definition, plus three prohibited action subsequences, called phenomena: Dirty Read, Non-repeatable Read, and Phantom. The concept of a phenomenon is not explicitly defined in the ANSI specifications, but the specifications suggest that phenomena are action subsequences that may lead to anomalous (perhaps non-serializable) behavior. We refer to anomalies in what follows when suggesting additions to the set of ANSI phenomena. As shown later, there is a technical distinction between anomalies and phenomena, but this distinction is not crucial for a general understanding. 
ANSI/ISO SQL-92规范定义了四个隔离级别： (1) READ UNCOMMITTED, (2) READ COMMITTED, (3) REPEATABLE READ, (4) SERIALIZABLE。这些级别通过传统的可串行性定义来定义，以及三个禁止的动作序列（称作“现象”）：脏读，不可重复读，以及幻读。“现象”这个概念并没有在ANSI规范中显式定义，但是按照规范中的建议，“现象”是可能会引起异常行为（也许是非可串行化）的动作序列。我们定义的异常是对于ANSI“现象”的补充。正如后面要讲到的，在“异常”和“现象”之间是有技术上的差异的，但是就普通理解而言，这个差异也不是很重要。
The ANSI isolation levels are related to the behavior of lock schedulers. Some lock schedulers allow transactions to vary the scope and duration of their lock requests, thus departing from pure two-phase locking. This idea was introduced by [GLPT], which defined Degrees of Consistency in three ways: locking, data-flow graphs, and anomalies. Defining isolation levels by phenomena (anomalies) was intended to allow non-lock-based implementations of the SQL standard. 
ANSI隔离级别与锁调度器的行为相关。一些锁调度器允许事务拥有不同范围和持续时间的锁，而不仅仅是使用纯粹的两阶段锁。这一思想是 [GLPT]中提出的， [GLPT]使用三种方式定义一致性的级别：锁、数据流图、以及异常。使用现象（异常）定义隔离级别的目的是允许使用非锁的方式实现SQL标准。
This paper shows a number of weaknesses in the anomaly approach to defining isolation levels. The three ANSI phenomena are ambiguous. Even their broadest interpretations do not exclude anomalous behavior. This leads to some counter-intuitive results. In particular, lock-based isolation levels have different characteristics than their ANSI equivalents. This is disconcerting because 
commercial database systems typically use locking. Additionally, the ANSI phenomena do not distinguish among several isolation levels popular in commercial systems. 
此论文展示了使用异常来定义隔离级别的一些缺点。规范中的三个ANSI现象是有二义性的。即使它们的广义解释也不能排除异常的行为。这导致了一些反直觉的结果。特别的，基于锁的隔离级别与它们的ANSI等价者有着不同的特点。这是令人不安的，因为商用数据库系统通常都使用锁。此外，ANSI现象也不区分一些在商业系统中普遍存在的隔离级别。
Section 2 introduces basic isolation level terminology. It defines the ANSI SQL and locking isolation levels. Section 3 examines some drawbacks of the ANSI isolation levels and proposes a new phenomenon. Other popular isolation levels are also defined. The various definitions map between ANSI SQL isolation levels and the degrees of consistency defined in 1977 in [GLPT]. They also encompass Date’s definitions of Cursor Stability and Repeatable Read [DAT]. Discussing the isolation levels in a uniform framework reduces confusion. 
第2章介绍了基础的隔离级别术语。它定义可ANSI SQL和锁的隔离级别。第3章分析了ANSI隔离级别的一些缺点，并提出了一种新的现象。其他流行的隔离级别也有定义。1977年， [GLPT]给出了ANSI SQL隔离级别和一致性级别的多角度的定义映射关系。他们也对比了Date定义的游标稳定性和Repeatable Read。在统一的框架下讨论隔离级别减少了混淆。
Section 4 introduces a multiversion concurrency control mechanism, called Snapshot Isolation, that avoids the ANSI SQL phenomena, but is not serializable. Snapshot Isolation is interesting in its own right, since it provides a reduced-isolation level approach that lies between READ COMMITTED and REPEATABLE READ. A new formalism (available in the longer version of this paper [OOBBGM]) connects reduced isolation levels for multiversioned data to the classical single-version locking serializability theory. 
第4章介绍了一种多版本并发控制机制，称为快照隔离，来避免ANSI SQL的现象，但它也不是可串行化的。快照隔离位于READ COMMITTED and REPEATABLE READ之间。 
Section 5 explores some new anomalies to differentiate the isolation levels introduced in Sections 3 and 4. The extended ANSI SQL phenomena proposed here lack the power to characterize Snapshot isolation and Cursor Stability. 
第5章探讨了一些新的异常，来区分第3、4章介绍的隔离级别。这里描述的扩展的ANSI SQL现象缺乏描述快照隔离和游标稳定的能力
Section 6 presents a Summary and Conclusions. 
第6章是总结部分

### 2. Isolation Definitions

#### 2.1 Serializability Concepts（可串行化的概念）

Transactional and locking concepts are well documented in the literature [BHG, PAP, PON, GR]. The next few paragraphs review the terminology used here. 
事务和锁的概念已经在文献中详细描述了。接下来几段将重温一下这些术语

A transaction groups a set of actions that transform the database from one consistent state to another. A history models the interleaved execution of a set of transactions as a linear ordering of their actions, such as Reads and Writes (i.e., inserts, updates, and deletes) of specific data items. Two actions in a history are said to conflict if they are performed by distinct transactions on the same data item and at least one of is a Write action. Following [EGLT], this definition takes a broad interpretation of “data item”: it could be a table row, a page, an entire table, or a message on a queue. Conflicting actions can also occur on 
a set of data items, covered by a predicate lock, as well as on a single data item. 
一个事务将一系列动作组织在一起，将数据库从一个一致的状态转换到另一个。一个History描述了一系列事务的交织运行，就像一组线性排序的动作， 例如针对指定数据项的读和写（以及插入、更新和删除）。如果一个History中的两个动作是不同的事务针对同一个数据项的，而且至少有一个是写操作，则这两个动作是冲突的。按照 [EGLT]上描述的，“数据项”可以可以有很广义的定义：可以是一个表的行，一个页，整个表，或者是一个队列中的消息。冲突的动作也可以发生在一个谓词锁覆盖的一系列数据项上，就像在单个数据项上一样。
A particular history gives rise to a dependency graph defining the temporal data flow among transactions. The actions of committed transactions in the history are represented as graph nodes. If action op1 of transaction T1 conflicts with and precedes action op2 of transaction T2 in the history, then the pair becomes an edge in the dependency graph. Two histories are equivalent if they have the same committed transactions and the same depen- dency graph. A history is serializable if it is equivalent to a serial history — that is, if it has the same dependency graph (inter-transaction temporal data flow) as some 
history that executes transactions one at a time in se- quence. 
一个History代表了一个依赖图，其可以定义事务之间的数据流。History中已提交的事务的动作可以表示为图的节点。如果在History中事务T1的动作op1与事务T2的动作op2冲突，并且op1在op2之前，则成为依赖图中的一个边。如果两个History有着相同的已提交的事务以及相同的依赖图，则这两个History是等价的。如果一个History等价于一个串行History，也就是说它与一个事务接一个事务串行运行的History有着相同的依赖图，则这个History是可串行化的。

#### 2.2 ANSI SQL Isolation Levels（ANSI SQL隔离级别）

ANSI SQL Isolation designers sought a definition that would admit many different implementations, not just locking. They defined isolation with the following three phenomena: 
ANSI SQL隔离的设计者寻求一个定义，其可以允许多种不同的实现，而不仅仅是锁。他们使用了下面三种现象来定义隔离： 
P1 (Dirty Read): Transaction T1 modifies a data item. Another transaction T2 then reads that data item before T1 performs a COMMIT or ROLLBACK. If T1 then performs a ROLLBACK, T2 has read a data item that was never committed and so never really existed. 
P1（脏读）：事务T1修改了一个数据项。然后另一个事务T2在T1执行COMMIT或者ROLLBACK之前读取数据。如果T1执行了一个ROLLBACK，T2就读取了一个从未提交的数据项，

P2 (Non-repeatable or Fuzzy Read): Transaction T1 reads a data item. Another transaction T2 then modifies or deletes that data item and commits. If T1 then attempts to reread the data item, it receives a modified value or discovers that the data item has been deleted. 
P2（不可重复读）：事务T1读取一个数据项。另一个事务T2修改或者删除这个数据项并提交。如果T1尝试去重新读取数据项，它将得到一个修改过的值，或者发现这个数据项已经被删除了。
P3 (Phantom): Transaction T1 reads a set of data items satisfying some . Transaction T2 then creates data items that satisfy T1’s and commits. If T1 then repeats its read with the same , it gets a set of data items different from the first read. 
P3（幻读）：事务T1读取满足一系列满足搜索条件的数据项。事务T2而后创建满足T1的搜索条件的数据项，并提交。如果T1使用相同的搜索条件重新读取，它将得到一系列与第一次读取不同的数据项。
None of these phenomena could occur in a serial history. Therefore by the Serializability Theorem they cannot occur in a serializable history [EGLT, BHG Theorem 3.6, GR Section 7.5.8.2, PON Theorem 9.4.2]. 
这些现象中的每一个都不可能在一个串行History中出现。因此基于可串行化的理论，它们也不会在可串行化级别的的History中出现。

Histories consisting of reads, writes, commits, and aborts can be written in a shorthand notation: “w1[x]” means a write by transaction 1 on data item x (which is how a data item is “modified’), and “r2[x]” represents a read of x by transaction 2. Transaction 1 reading and writing a set of records satisfying predicate P is denoted by r1[P] and w1[P] respectively. Transaction 1’s commit and abort (ROLLBACK) are written “c1” and “a1”, respectively. 
History包括读、写、提交以及中止，可以使用缩写代替：w1[x]的意思是事务1在数据项x上运行了一个写操作，r2[x]的意思是事务2对x的一个读操作。事务1对一系列满足谓词P的记录的读写操作可以标注为r1[P]或者w1[P]。事务1的提交和中止的缩写是c1和a1.
Phenomenon P1 might be restated as disallowing the following scenario: 
现象P1可以使用下面的场景来描述

(2.1) w1[x] . . . r2[x] . . . (a1 and c2 in either order)

The English statement of P1 is ambiguous. It does not actually insist that T1 abort; it simply states that if this happens something unfortunate might occur. Some people reading P1 interpret it to mean: 
P1的英文描述是有二义性的。它并没有实际上要求T1来中止；它只是简单的说如果这样的话一些不幸的事可能会发生。一些人会把P1解释为： 
(2.2) w1[x]...r2[x]...((c1 or a1) and (c2 or a2) in any order)

Forbidding the (2.2) variant of P1 disallows any history where T1 modifies a data item x, then T2 reads the data item before T1 commits or aborts. It does not insist that T1 aborts or that T2 commits. 
禁止P1的（2.2）变体意味着任何这样的History（T1修改数据项x，然后T2在T1提交或中止之前读取该数据项）都不被允许。它并不坚持T1的中止或者T2的提交。

Definition (2.2) is a much broader interpretation of P1 than (2.1), since it prohibits all four possible commit-abort pairs by transactions T1 and T2, while (2.1) only prohibits two of the four. Interpreting (2.2) as the meaning of P1 prohibits an execution sequence if something anomalous might in the future. We call (2.2) the broad interpretation of P1, and (2.1) the strict interpretation of P1. 
对于P1的解释，定义（2.2）要比（2.1）更加宽泛，因为它禁止了T1和T2事务可能生成的所有的四个可能的提交-中止组合，而（2.1）只能禁止这四个中的两个。使用（2.2）这样的方式进行解释可以避免后续出现异常情况。我们将（2.2）称为P1的广义解释，（2.1）称为P1的狭义解释。
Interpretation (2.2) specifies a phenomenon that might lead to an anomaly, while (2.1) specifies an actual anomaly. Denote them as P1 and A1 respectively. Thus: 
（2.2）定义了一个可能会引发“异常”的“现象”，而（2.1）则定义了一个实际的“异常”。我们把他们称为P1和A1.

P1: w1[x]...r2[x]...((c1 or a1) and (c2 or a2) in any order)
A1: w1[x]...r2[x]...(a1 and c2 in any order)

Similarly, the English language phenomena P2 and P3 have strict and broad interpretations, and are denoted P2 
and P3 for broad, and A2 and A3 for strict:
相似的，英文现象P2和P3也有狭义和广义的解释，即P2，P3和A2、A3。

P2: r1[x]...w2[x]...((c1 or a1) and (c2 or a2) in any order) 
A2: r1[x]...w2[x]...c2...r1[x]...c1
P3: r1[P]...w2[y in P]...((c1 or a1) and (c2 or a2) any order)
A3: r1[P]...w2[y in P]...c2...r1[P]...c1
Section 3 analyzes these alternative interpretations after more conceptual machinery has been developed. It argues that the broad interpretation of the phenomena is required. Note that the English statement of ANSI SQL P3 just prohibits inserts to a predicate, but P3 above intentionally prohibits any write (insert, update, delete) affecting a tuple satisfying the predicate once the predicate has been read. 
在发展出更加概念性的机制后，第3章进一步分析了这些备选的解释。它提出现象的广义解释是需要的。请注意ANSI SQL P3的英文描述只是禁止了对于谓词的插入，但是上面定义的广义P3实际上是禁止了对于使用过谓词进行读取之后，而对于满足此谓词的任意元组的写入操作（插入、更新、删除）

This paper later deals with the concept of a multi-valued history (MV-history for short — see [BHG], Chapter 5). Without going into details now, multiple versions of a data item may exist at one time in a multi-version system. Any read must be explicit about which version is being read. There have been attempts to relate ANSI Isolation definitions to multi-version systems as well as more common single-version systems of a standard locking scheduler. The English language statements of the phenomena P1, P2, and P3 imply single-version histories. This is how we interpret them in the next section. 
此论文也涉及了多值历史的概念。一个数据的多个版本可以在一个多版本系统中同时存在。所有的读都必须显式的指定想要读取的版本。也曾经有尝试将ANSI隔离的定义与多版本系统进行关联，就像将ANSI隔离与基于标准的锁调度器的单版本系统进行关联一样。现象P1、P2和P3的英文描述隐含指单版本History。我们将在下一章解释他们。

ANSI SQL defines four levels of isolation by the matrix of Table 1. Each isolation level is characterized by the phenomena that a transaction is forbidden to experience (broad or strict interpretations). However, the ANSI SQL specifications do not define the SERIALIZABLE isolation level solely in terms of these phenomena. Subclause 4.28, “SQL-transactions”, in [ANSI] notes that the 
SERIALIZABLE isolation level must provide what is “commonly known as fully serializable execution.” The prominence of the table compared to this extra proviso leads to a common misconception that disallowing the three phenomena implies serializability. Table 1 calls histories that disallow the three phenomena ANOMALY SERIALIZABLE. 
ANSI SQL使用表1定义了四个隔离级别。每个隔离级别通过一个事务禁止发生的现象来描述。但是ANSI SQL规范并没有使用现象来单独定义SERIALIZABLE 隔离级别。在ANSI规范的4.28子句“SQL-transactions”中定义了，SERIALIZABLE 隔离级别必须提供“正如完全可串行化的执行”。这样的表格可能会导致一个误解：如果不允许这三种现象，则隐含了可串行性。表1把“不允许这三种现象的History”称为ANOMALY SERIALIZABLE（异常可串行化）

The isolation levels are defined by the phenomena they are forbidden to experience. Picking a broad interpretation of a phenomenon excludes a larger set of histories than the strict interpretation. This means we are arguing for more restrictive isolation levels (more histories will be disallowed). Section 3 shows that even taking the broad interpretations of P1, P2, and P3, forbidding these phenomena does not guarantee true serializability. It would have been simpler in [ANSI] to drop P3 and just use Subclause 4.28 to define ANSI SERIALIZABLE. Note that Table 1 is not a final result; Table 3 will superseded it.
隔离级别通过禁止发生的现象来定义。使用现象的广义解释可以排除更多的History。这意味着我们使用了更加严格的隔离级别（不允许更多的History）。第3章展示了即使使用了P1、P2和P3的广义解释，禁止这些现象也无法保证真正的可串行性。如果使用4.28子句替换P3，来定义ANSI规范的ANSI SERIALIZABLE，则可以更加简单。请注意表1并不是最终结果；表3将取代它。


#### 2.3 Locking（锁）

Most SQL products use lock-based isolation. Consequently, it is useful to characterize the ANSI SQL isolation levels in terms of locking, although certain problems arise. 
大多数SQL产品使用基于锁的隔离。因此，使用锁的方式来描述ANSI SQL隔离级别是有用处的，虽然这样也存在着一些问题。

Transactions executing under a locking scheduler request Read (Share) and Write (Exclusive) locks on data items or sets of data items they read and write. Two locks by different transactions on the same item conflict if at least one of them is a Write lock. 
使用锁调度器的事务执行需要请求数据项或一组数据项上的读（共享）和写（排他）锁。如果针对相同数据项上不同事务的两个锁中，至少一个为写锁，则这两个锁是冲突的。

A Read (resp. Write) predicate lock on a given is effectively a lock on all data items satisfying the . This may be an infinite set. It includes data present in the database and also any phantom data items not currently in the database but that would satisfy the predicate if they were inserted or if current data items were updated to satisfy the . In SQL terms, a predicate lock covers all tuples that satisfy the predicate and any that an INSERT, UPDATE, or DELETE statement would cause to satisfy the predicate. Two predicate locks by different transactions conflict if one is a Write lock and if there is a (possibly phantom) data item covered by both locks. An item lock (record lock) is a predicate lock where the predicate names the specific record. 
一个读（写）谓词锁为满足给定搜索级别的所有数据项加锁。这可能是一个无限的集合。它包括了已经在数据库中的数据，以及当前还不在数据库中但在插入或更新操作后可以满足谓词的幻象数据。使用SQL的术语，一个谓词锁涵盖了所有满足谓词的元组，以及任何由INSERT、UPDATE、DELETE引起的可以满足谓词的元组。如果不同事务的两个谓词锁中一个为写锁，而且存在一个数据项被两个锁所覆盖，则这两个锁是冲突的。数据项锁可以看做是谓词为指定记录的谓词锁

A transaction has well-formed writes (reads) if it requests a Write (Read) lock on each data item or predicate before writing (reading) that data item, or set of data items defined by a predicate. The transaction is well-formed if it has well-formed writes and reads. A transaction has two- phase writes (reads) if it does not set a new Write (Read) lock on a data item after releasing a Write (Read) lock. A transaction exhibits two-phase locking if it does not request any new locks after releasing some lock. 
如果一个事务在写（读）一个数据项或者由谓词定义的一组数据项之前，为每个数据项或者谓词请求一个写（读）锁，则此事务有well-formed的写（读）。如果一个事务有well-formed的写（读），则此事务是well-formed的。如果一个事务在释放一个写（读）锁之后将不会在一个数据项上设置一个新的写（读）锁，则此事务有两阶段的写（读）。一个事务是两阶段锁的，如果它在释放锁之后不再请求新的锁。

The locks requested by a transaction are of long duration if they are held until after the transaction commits or aborts. 
Otherwise, they are of short duration. Typically, short locks are released immediately after the action completes. 
如果一个事务请求的锁在事务提交或中止之前一直持有，那此锁就是长周期的。否则它们是短周期的。典型的，短周期的锁在动作完成后立即释放。
If a transaction holds a lock, and another transaction requests a conflicting lock, then the new lock request is not
granted until the former transaction’s conflicting lock has been released. 
The fundamental serialization theorem is that well-formed two-phase locking guarantees serializability — each his-tory arising under two-phase locking is equivalent to some serial history. Conversely, if a transaction is not well-formed or two-phased then, except in degenerate cases, non-serializable execution histories are possible [EGLT].
如果一个事务持有一个锁，另一个事务请求一个冲突的锁，在前一个事务的冲突锁没有释放之前，新的锁请求不会被授予。
基础的可串行化理论定义了well-formed的两阶段锁保证了可串行性——每一个使用两阶段锁的history都等价于某个串行的history。相反的，如果一个事务不是well-formed 或者不是两阶段的，则可能会出现非可串行化的执行history

The [GLPT] paper defined four degrees of consistency, attempting to show the equivalence of locking, dependency, and anomaly-based characterizations. The anomaly defini-tions (see Definition 1) were too vague. The authors con-tinue to get criticism for that aspect of the definitions [GR]. Only the more mathematical definitions in terms of histories and dependency graphs or locking have stood the test of time.
[GLPT]论文定义了四个级别的一致性，尝试来展示锁、依赖和基于异常的特点的等价性。Definition1中的异常定义太模糊了。作者也一直因此被批评。只有针对history、依赖图或者锁的更加数学化的定义才能经受的起时间的考验。

Table 2 defines a number of isolation types in terms of lock scopes (items or predicates), modes (read or write), and their durations (short or long). We believe the isolation levels called Locking READ UNCOMMITTED, Locking READ COMMITTED, Locking REPEATABLE READ, and Locking SERIALIZABLE are the locking definitions intended by ANSI SQL Isolation levels — but as shown next they are quite different from those of Table 1. Consequently, it is necessary to differentiate isolation levels defined in terms of locks from the ANSI SQL phenomena-based isolation levels. To make this distinction, the levels in Table 2 are labeled with the “Locking” prefix, as opposed to the “ANSI” prefix of Table 1.
表2使用锁的范围（数据项或谓词）、模式（读或写）、时长（短或长）定义了一些隔离级别。我们相信称为Locking READ UNCOMMITTED, Locking READ COMMITTED, Locking REPEATABLE READ, and Locking SERIALIZABLE的隔离级别就是ANSI SQL隔离级别所期望的的基于锁的定义——但是他们和表1也有很大不同。因此，有必要区分使用锁定义的隔离级别与ANSI SQL的基于现象的隔离级别。为了实现区分，表2中的级别使用“Locking”作为前缀，而表1中的级别使用“ANSI”作为前缀。

[GLPT] defined Degree 0 consistency to allow both dirty reads and writes: it only required action atomicity. Degrees 1, 2, and 3 correspond to Locking READ UNCOMMITTED, READ COMMITTED, and SERIALIZABLE, respectively. No isolation degree matches the Locking REPEATABLE READ isolation level. 
[GLPT]定义了Degree0一致性，其允许脏读和脏写：它只要求动作的原子性。Degree1,2,3分别对应于Locking READ UNCOMMITTED, READ COMMITTED, and SERIALIZABLE。没有Degree与Locking REPEATABLE READ级别相对应。

Date and IBM originally used the name “Repeatable Reads” [DAT, DB2] to mean serializable or Locking SERIALIZABLE. This seemed like a more comprehensible name than the [GLPT] term “Degree 3 isolation." The ANSI SQL meaning of REPEATABLE READ is different from Date’s original definition, and we feel the terminology is unfortunate. Since anomaly P3 is specifically not ruled out by the ANSI SQL REPEATABLE READ isolation level, it is clear from the definition of P3 that reads are NOT repeatable! We repeat this misuse of the term with Locking REPEATABLE READ in Table 2, in order to parallel the ANSI definition. Similarly, Date coined the term Cursor Stability as a more comprehensible name for Degree 2 isolation augmented with protection from lost cursor updates as explained in Section 4.1 below. 
Date和IBM最初使用“Repeatable Reads”来代表可串行化或者Locking SERIALIZABLE。这看来是一个比 [GLPT]的“Degree 3 isolation"更易理解的名字。REPEATABLE READ的ANSI SQL含义与Date的原始定义是不同的，而且我们也觉得这个术语不太好。因为异常P3无法使用ANSI SQL REPEATABLE READ隔离级别来排除，但从P3的定义来看，此读取就是不可重复的。我们在表2中再次错误的使用了此术语“ Locking REPEATABLE READ”，为的是与ANSI的定义进行对齐。相似的，Date创造的术语Cursor Stability的可理解性更好，其对Degree2隔离级别提供了丢失游标更新的保护。

Definition. Isolation level L1 is weaker than isolation level L2 (or L2 is stronger than L1), denoted L1 « L2, if all non- serializable histories that obey the criteria of L2 also satisfy L1 and there is at least one non-serializable history that can occur at level L1 but not at level L2. Two isolation levels L1 and L2 are equivalent, denoted L1 == L2, if the sets of non-serializable histories satisfying L1 and L2 are identical. L1 is no stronger than L2, denoted if either L1 « L2 or L1 == L2. Two isolation levels are incomparable, denoted L1 »« L2, when each isolation level allows a non-serializable history that is disallowed by the other. 
定义：隔离级别L1比隔离级别L2弱（或者L2比L1强），记为L1 « L2，如果所有符合L2条件的非可串行化的history也都满足L1，而且至少有一个L1的非可串行化history不在L2中出现。两个隔离级别L1和L2为等价的，记为L1 == L2，如果满足L1和L2的非可串行化history是相等的。L1不强于L2，记为 ，如果 L1 « L2 或 L1 == L2。两个隔离级别是不可比较的，记为L1 »« L2，这种情况下，每个隔离级别都允许一个非可串行化history，但是另一个却不允许。
In comparing isolation levels we differentiate them only in terms of the non-serializable histories that can occur in one but not the other. Two isolation levels can also differ in terms of the serializable histories they permit, but we say Locking SERIALIZABLE == Serializable even though it is well known that a locking scheduler does not admit all possible Serializable histories. It is possible for an isolation level to be impractical because of disallowing too many serializable histories, but we do not deal with this here. 
在比较隔离级别时，我们使用非可串行化history在一个中出现而不再另一个中出现来区分它们。两个隔离级别也可以通过它们允许的可串行化history来区分，但是我们说Locking SERIALIZABLE == Serializable，即使大家知道一个锁调度器不允许所有可能的Serializable history。一个隔离级别可以是impractical 的，因为其不允许太多的可串行化history，但是我们在这里不处理这个问题。

These definitions yield the following remark.
Remark 1: Locking READ UNCOMMITTED
« Locking READ COMMITTED
« Locking REPEATABLE READ
« Locking SERIALIZABLE

In the following section, we’ll focus on comparing the ANSI and Locking definitions. 
下面的章节中，我们将聚焦于对比ANSI和锁的定义。

### 3. Analyzing ANSI SQL Isolation Levels

To start on a positive note, the locking isolation levels comply with the ANSI SQL requirements. 
以积极的方面开始，基于锁的隔离级别顺从于ANSI SQL需求

Remark 2. The locking protocols of Table 2 define locking isolation levels that are at least as strong as the corresponding phenomena-based isolation levels of Table 1. See [OOBBGM] for proof. 
相对于表1中基于现象的隔离级别，表2的锁协议定义了至少强于其的隔离级别

Hence, locking isolation levels are at least as isolated as the same-named ANSI levels. Are they more isolated? The answer is yes, even at the lowest level. Locking READ UNCOMMITTED provides long duration write locking to avoid a phenomenon called "Dirty Writes," but ANSI SQL does not exclude this anomalous behavior other than ANSI SERIALIZABLE. Dirty writes are defined as follows: 
因此，锁隔离级别至少与同名的ANSI级别隔离性相当。它们的隔离性会更强吗？答案是Yes，即使是在最低的级别。 Locking READ UNCOMMITTED提供了长周期的写锁来避免“脏写”的现象，但是ANSO SQL中除过 ANSI SERIALIZABLE没有排除这样的异常现象。脏写的定义如下：

P0 (Dirty Write): Transaction T1 modifies a data item. Another transaction T2 then further modifies that data item before T1 performs a COMMIT or ROLLBACK. If T1 or T2 then performs a ROLLBACK, it is unclear what the correct data value should be. The broad interpretation of this is: 
P0（脏写）：事务T1修改了一个数据项。另一个事务T2在T1执行Commit或者Rollback之前，进一步修改这一数据项。然后如果T1或者T2进行了Rollback后，就不清楚正确的数据值究竟是什么了。它的广义解释如下：

P0: w1[x]...w2[x]...((c1 or a1) and (c2 or a2) in any order)

One reason why Dirty Writes are bad is that they can violate database consistency. Assume there is a constraint be- tween x and y (e.g., x = y), and T1 and T2 each maintain the consistency of the constraint if run alone. However, the constraint can easily be violated if the two transactions write x and y in different orders, which can only happen if there are Dirty writes. For example consider the history w1[x] w2[x] w2[y] c2 w1[y] c1. T1"s changes to y and T2"s to x both “survive”. If T1 writes 1 in both x and y while T2 writes 2, the result will be x=2, y =1 violating x = y. 
脏写的一个问题是它可能会违反数据库的一致性。假定有一个对于x和y的约束（如x=y），T1和T2如果独立运行都可以保证这一约束。但是如果两个事务使用不同的顺序写x和y，就会违反这一约束，这只在存在脏写时出现。例如，考虑history（w1[x] w2[x] w2[y] c2 w1[y] c1.）T1对于y的修改和T2对于x的修改都将成功，如果T1对于x和y都写入1，T2写入2，则结果为x=2，y=1，违反了x=y的约束。

As discussed in [GLPT, BHG] and elsewhere, automatic transaction rollback is another pressing reason why P0 is important. Without protection from P0, the system can’t undo updates by restoring before images. Consider the history: w1[x] w2[x] a1. You don’t want to undo w1[x] by restoring its before-image of x, because that would wipe out w2’s update. But if you don’t restore its before-image, and transaction T2 later aborts, you can’t undo w2[x] by restoring its before-image either! Even the weakest locking systems hold long duration write locks. Otherwise, their recovery systems would fail. So we conclude Remark 3: 
Remark 3: ANSI SQL isolation should be modified to re- quire P0 for all isolation levels. 
P0之所以重要的另一个原因是自动事务回滚。如果没有对于P0的保护，系统将无法通过恢复之前的值来undo做过的更新。考虑这样的history： w1[x] w2[x] a1. 你不会想通过恢复x之前的值来undo w1[x]，因为这将擦去w2的更新。但是如果你不恢复它之前的值，而且T2后续也中止了，你也不能通过恢复之前的值来undo w2[x]。即使是最弱的锁系统也会长期的持有写锁。否则它们的恢复系统就会失败。因此我们有Remark 3: 
Remark 3: ANSI SQL隔离应被修改成所有隔离级别都需要P0（换言之，对于脏写的保护是每个隔离级别都需要的）

We now argue that a broad interpretation of the three ANSI phenomena is required. Recall the strict interpretations are: 
我们现在来证明三种ANSI现象的广义解释是需要的。回顾一下狭义解释：

A1: w1[x]...r2[x]...(a1 and c2 in either order) (Dirty Read) 
A2: r1[x]...w2[x]...c2...r1[x]...c1 (Fuzzy or Non-Repeatable Read) 
A3: r1[P]...w2[y in P]...c2....r1[P]...c1 (Phantom)

By Table 1, histories under READ COMMITTED isolation forbid anomaly A1, REPEATABLE READ isolation forbids anomalies A1 and A2, and SERIALIZABLE isolation forbids anomalies A1, A2, and A3. Consider history H1, involving a $40 transfer between bank balance rows x and y: 
如表1中定义，READ COMMITTED隔离下的history禁止了异常A1，REPEATABLE READ隔离禁止了A1和A2，SERIALIZABLE 隔离禁止了异常A1、A2和A3。考虑history H1，涉及银行余额行x和y之间的一个40元的转账：

H1: r1[x=50] w1[x=10] r2[x=10] r2[y=50] c2 r1[y=50] w1[y=90] c1 （A1、A2、A3都无法捕获这一非可串行化的History）

H1 is non-serializable, the classical inconsistent analysis problem where transaction T1 is transferring a quantity 40 from x to y, maintaining a total balance of 100, but T2 reads an inconsistent state where the total balance is 60. The history H1 does not violate any of the anomalies A1, A2, or A3. In the case of A1, one of the two transactions would have to abort; for A2, a data item would have to be read by the same transaction for a second time; A3 re- quires a phantom value. None of these things happen in H1. Consider instead taking the broad interpretation of A1, the phenomenon P1: 
H1是非可串行化的，典型的不一致分析问题。T1将数量40从x转到y，以维持总数量为100，但是T2却读到了一个不一致的状态，其总数量为60。H1并没有违反异常A1、A2、A3中的任何一个。作为A1而言，两个事务之一需要中止；对于A2，一个数据项需要被同一个事务第二次读；A3需要一个幻象值。所有这些都没有在H1发生。考虑一下A1的广义解释P1

P1: w1[x]...r2[x]...((c1 or a1) and (c2 or a2) in any order)

H1 indeed violates P1. Thus, we should take the interpretation P1 for what was intended by ANSI rather than A1. 
The Broad interpretation is the correct one.
H1确实违反了P1（H1中的片段w1[x=10] r2[x=10]）。因此我们应该使用P1，而不是A1。广义解释是正确的那一个

Similar arguments show that P2 should be taken as the ANSI intention rather than A2. A history that discriminates these two interpretations is: 
相似的，P2也应该作为A2的替代。一个可以分辨这两种解释的history是：

H2: r1[x=50] r2[x=50] w2[x=10] r2[y=50] w2[y=90] c2 r1[y=90] c1 

H2 is non-serializable — it is another inconsistent analysis, where T1 sees a total balance of 140. This time nei- ther transaction reads dirty (i.e. uncommitted) data. Thus P1 is satisfied. Once again, no data item is read twice nor is any relevant predicate evaluation changed. The problem with H2 is that by the time T1 reads y, the value for x is out of date. If T2 were to read x again, it would have been changed; but since T2 doesn"t do that, A2 doesn"t apply. Replacing A2 with P2, the broader interpretation, solves this problem. 
H2是非可串行化的，另一个不一致分析。T1看到的总数量为140。这次两个事务都没有读脏数据。因此P1是满足的。并且也没有数据项被读取两次，也没有任何的相关谓词求值的变动。H2的问题是当T1读y的时候，x的值已经过时了。如果T2再次读取x，x的值就将发生变化。将A2用P2替代，可以解决这个问题
P2: r1[x]...w2[x]...((c1 or a1) and (c2 or a2) any order)

H2 would now be disqualified when w2[x=20] occurs to overwrite r1[x=50]. Finally, consider A3 and history H3: 
现在当w2[x=20]（此处应为w2[x=10]）来覆盖 r1[x=50]时，H2就违反了P2。最后来考虑A3和History H3

A3: r1[P]...w2[y in P]...c2...r1[P]...c1 (Phantom)

H3: r1[P] w2[insert y to P] r2[z] w2[z] c2 r1[z] c1

Here T1 performs a to find the list of active employees. Then T2 performs an insert of a new active employee and then updates z, the count of em- ployees in the company. Following this, T1 reads the count of active employees as a check and sees a discrepancy. This history is clearly not serializable, but is allowed by A3 since no predicate is evaluated twice. Again, the Broad interpretation solves the problem. 
此处T1执行一个<搜索条件>来得到有效雇员的列表。然后T2执行一个新的有效雇员的插入，然后再更新z（公司内的雇员数量）。在此之后，T1也读取雇员数量，此时得到一个不一致的值。很明显这个History不是可串行化的，但是它却不违反A3，因为不会两次执行谓词。于是广义定义又一次解决了这一问题。

P3: r1[P]...w2[y in P]...((c1 or a1) and (c2 or a2) any order)

If P3 is forbidden, history H3 is invalid. This is clearly what ANSI intended. The foregoing discussion demon- strates the following results. 
如果P3被禁止，H3也就无效了。这完全就是ANSI想要达到的目的。

Remark 4. Strict interpretations A1, A2, and A3 have unintended weaknesses. The correct interpretations are the Broad ones. We assume in what follows that ANSI meant to define P1, P2, and P3. 
Remark 4：A1、A2、A3这样的狭义解释存在缺陷。广义解释才是正确的。

Remark 5. ANSI SQL isolation phenomena are incomplete. There are a number of anomalies that still can arise. 
New phenomena must be defined to complete the definition of locking. Also, P3 must be restated. In the following definitions, we drop references to (c2 or a2) that do not restrict histories. 
Remark 5：ANSI SQL隔离现象是不完整的。仍然有一些异常可能出现。必须定义新的现象来使锁的定义变得完整。同时P3需要被重新描述。此外在下面的定义中，我们也去掉了 (c2 or a2)的部分，这不会造成什么影响

P0: w1[x]...w2[x]...(c1 or a1) (Dirty Write)
P1: w1[x]...r2[x]...(c1 or a1) (Dirty Read)
P2: r1[x]...w2[x]...(c1 or a1) (Fuzzy or Non-Repeatable Read) 
P3: r1[P]...w2[y in P]...(c1 or a1) (Phantom) 

One important note is that ANSI SQL P3 only prohibits inserts (and updates, according to some interpretations) to a predicate whereas the definition of P3 above prohibits any write satisfying the predicate once the predicate has been read — the write could be an insert, update, or delete. 
重要的一点是ANSI SQL P3只禁止了对于一个谓词的插入（按照某些解释，也覆盖更新），然而上述P3的定义禁止了所有满足谓词的写操作，一旦此谓词已被读取过——写操作可能是插入、更新或者删除。

The definition of proposed ANSI isolation levels in terms of these phenomena is given in Table 3. 


For single version histories, it turns out that the P0, P1, P2, P3 phenomena are disguised versions of locking. For example, prohibiting P0 precludes a second transaction writing an item after the first transaction has written it, equivalent to saying that long-term Write locks are held on data items (and predicates). Thus Dirty Writes are impossible at all levels. Similarly, prohibiting P1 is equiv- alent to having well-formed reads on data items. Prohibiting P2 means long-term Read locks on data items. Finally, Prohibiting P3 means long-term Read predicate locks. Thus the isolation levels of Table 3 defined by these phenomena provide the same behavior as the Locking isolation levels of Table 2.
对于单版本history而言，P0、P1、P2、P3就是锁的另一个变种版本。例如禁止P0就是在第一个事务写数据后，防止第二个事务写相同的数据，与之等价的说法是在数据项上加长周期的写锁。因此脏写在所有的级别上都是不可能的。相似的，禁止P1就等价于在数据项上有well-formed的读。禁止P2意味着在数据项上的长周期的读锁。最终，禁止P3意味着长周期的读谓词锁。因此，表3中通过这些现象定义的隔离级别提供了表2中基于锁的隔离级别相同的行为。

Remark 6. The locking isolation levels of Table 2 and the phenomenological definitions of Table 3 are equivalent. Put another way, P0, P1, P2, and P3 are disguised redefinition’s of locking behavior. 
Remark 6：表2的锁隔离级别和表3的现象定义是等价的。

In what follows, we will refer to the isolation levels listed in Table 3 by the names in Table 3, equivalent to the Locking versions of these isolation levels of Table 2. When we refer to ANSI READ UNCOMMITTED, ANSI READ COMMITTED, ANSI REPEATABLE READ, and ANOMALY SERIALIZABLE, we are referring to the ANSI definition of Table 1 (inadequate, since it did not include 
P0).
接下来，我们将使用表3中的名字来引用表3中的隔离级别，与表2中的这些隔离级别的锁版本等价。但当我们说ANSI READ UNCOMMITTED, ANSI READ COMMITTED, ANSI REPEATABLE READ, and ANOMALY SERIALIZABLE时，我们事实上是想说表1的ANSI定义

The next section shows that a number of commercially available isolation implementations provide isolation levels that fall between READ COMMITTED and REPEATABLE READ. To achieve meaningful isolation levels that distinguish these implementations, we will assume P0 and P1 as a basis and then add distinguishing new phenomena. 
下一章展示了一些商业系统中的隔离实现，其提供的隔离级别在READ COMMITTED和REPEATABLE READ之间。为了更好的区分这些实现，我们将假定P0、P1（Read Committed）是基础，并基于此来增加新的现象。


### 4. Other Isolation Types
#### 4.1 Cursor Stability（游标稳定性）

Cursor Stability is designed to prevent the lost update phenomenon. 
游标稳定性是设计用于防止“丢失更新”的现象
P4 (Lost Update): The lost update anomaly occurs when transaction T1 reads a data item and then T2 updates the data item (possibly based on a previous read), then T1 (based on its earlier read value) updates the data item and commits. In terms of histories, this is: 
P4（丢失更新）：当事务T1读一个数据项，然后T2更新这一数据项（可能是基于前一次读的结果），然后T1（基于它前一次读取的结果）更新这一数据项，而后提交。以history的方式描述如下：

P4: r1[x]...w2[x]...w1[x]...c1 (Lost Update)

The problem, as illustrated in history H4, is that even if T2 commits, T2"s update will be lost. 
这个问题就像在H4 history中表现的那样，即使T2提交了，T2的更新也将丢失。

H4: r1[x=100] r2[x=100] w2[x=120] c2 w1[x=130] c1

The final value of x contains only the increment of 30 added by T1. P4 is possible at the READ COMMITTED isolation level, since H4 is allowed when forbidding P0 (a commit of the transaction performing the first write action precedes the second write) or P1 (which would require a read after a write). However, forbidding P2 also precludes P4, since w2[x] comes after r1[x] and before T1 commits or aborts. Therefore the anomaly P4 is useful in distinguishing isolation levels intermediate in strength be- tween READ COMMITTED and REPEATABLE READ. 
x最终的值只包含了由T1加上的30。P4在READ COMMITTED级别也是可能出现的，因为禁止P0（在提交之前两次写，此处w2[x=120]之后就提交了，因此不算脏写）或P1（在一个写后面跟一个读），H4还是被允许的。但是，禁止P2就防止了P4（此处应为H4），因为w2[x]在r1[x]之后出现，且在T1提交或中止之前。

The Cursor Stability isolation level extends READ COMMITTED locking behavior for SQL cursors by adding a new read action for FETCH from a cursor and requiring that a lock be held on the current item of the cursor. The lock is held until the cursor moves or is closed, possibly by a commit. Naturally, the Fetching transaction can update the row, and in that case a write lock will be held on the  row until the transaction commits, even after the cursor moves on with a subsequent Fetch. The notation is extended to include, rc, meaning read cursor, and wc, meaning write the current record of the cursor. A rc1[x] and a later wc1[x] precludes an intervening w2[x]. Phenomenon P4, renamed P4C, is prevented in this case. 
Cursor Stability隔离级别为SQL游标扩展了READ COMMITTED的锁的行为，其要求从一个游标FETCH数据时，对游标指向的这一数据项加锁。这个锁将保持到游标移动或者被关闭。很自然的，Fetch数据的事务也可以更新行，此时需要为此行加写锁，直至事务提交，即使是下一个Fetch移动了游标之后。此处使用rc来代表read cursor，wc来代表write当前的cursor。一个rc1[x]以及后面的wc1[x]防止了中间插入的w2[x]。现象P4重命名为P4C，来防止这样的情况。

P4C: rc1[x]...w2[x]...w1[x]...c1 (Lost Update)

Remark 7:
READ COMMITTED « Cursor Stability « REPEATABLE READ

Cursor Stability is widely implemented by SQL systems to prevent lost updates for rows read via a cursor. READ COMMITTED, in some systems, is actually the stronger Cursor Stability. The ANSI standard allows this.
Cursor Stability在SQL系统中被广泛实现，用于防止通过游标读取行时的丢失更新。在有些系统中，READ COMMITTED实际是更强的Cursor Stability。ANSI标准是允许这样的。

The technique of putting a cursor on an item to hold its value stable can be used for multiple items, at the cost of using multiple cursors. Thus the programmer can parlay Cursor Stability to effective Locking REPEATABLE READ isolation for any transaction accessing a small, fixed number of data items. However this method is inconvenient and not at all general. Thus there are always histories fitting the P4 (and of course the more general P2) phenomenon that are not precluded by Cursor Stability. 
在一个数据项上加游标来保持其值的稳定性的这一做法也可以用于多数据项，但需要使用多个游标。因此对于任意访问一个小的、固定数目的数据项的事务而言，程序员可以利用Cursor Stability来达到相对于Locking REPEATABLE READ更高的效率。但是这一方法不很方便也不通用。因此有可能符合P4现象（当然也符合更通用的P2）的History无法被Cursor Stability所防止。（Cursor Stability防护的是P0、P1和P4C）

### 4.2 Snapshot Isolation（快照隔离）

These discussions naturally suggest an isolation level, called Snapshot Isolation, in which each transaction reads reads data from a snapshot of the (committed) data as of the time the transaction started, called its Start-Timestamp. This time may be any time before the transaction’s first Read. A transaction running in Snapshot Isolation is never blocked attempting a read as long as the snapshot data from its Start-Timestamp can be maintained. The transaction"s writes (updates, inserts, and deletes) will also 
be reflected in this snapshot, to be read again if the transaction accesses (i.e., reads or updates) the data a second time. Updates by other transactions active after the transaction Start-Timestamp are invisible to the transaction. 
快照隔离：每个事物从事物开始时的一个数据快照中读取数据，此时间成为起始时间戳。这一时间可以是此事务的首个读之前的任一时间。在快照隔离下，只要维护了起始时间戳之后的快照数据，一个事务在尝试读的时候就不会被阻塞。事务的写操作（更新、插入、删除）也同样体现在此快照中，如果事务再次访问此数据，就会得到新的数据值。但事务启动时间戳之后发生的其他事务的更新是不可见的。

Snapshot Isolation is a type of multiversion concurrency control. It extends the Multiversion Mixed Method de- scribed in [BHG], which allowed snapshot reads by read- only transactions. 
快照隔离是一种多版本并发控制。它扩展了[BHG]中定义的多版本混合方法，其允许只读事务读取快照。

When the transaction T1 is ready to commit, it gets a Commit-Timestamp, which is larger than any existing Start-Timestamp or Commit-Timestamp. The transaction successfully commits only if no other transaction T2 with a Commit-Timestamp in T1’s execution interval [Start- Timestamp, Commit-Timestamp] wrote data that T1 also wrote. Otherwise, T1 will abort. This feature, called First- committer-wins prevents lost updates (phenomenon P4). When T1 commits, its changes become visible to all 
transactions whose Start-Timestamps are larger than T1‘s Commit-Timestamp. 
当事务T1准备好提交时，它获取一个提交时间戳，其比任何已存的起始时间戳或者提交时间戳要大。只有在没有其他事务T2的提交时间戳在T1的执行时间间隔[Start- Timestamp, Commit-Timestamp] 内，并且T2写了T1也写了的数据，这样T1才能提交成功。否则T1会中止。这一特性称为First- committer-wins，可以避免丢失更新（现象P4）。当T1提交时，它的改变对于所有启动时间戳大于T1提交时间戳的事务可见

Snapshot Isolation is a multi-version (MV) method, so single-valued (SV) histories do not properly reflect the temporal action sequences. At any time, each data item might have multiple versions, created by active and committed transactions. Reads by a transaction must choose the appropriate version. Consider history H1 at the beginning of Section 3, which shows the need for P1 in a single valued execution. Under Snapshot Isolation, the same sequence of actions would lead to the multi-valued history: 
快照隔离是一个多版本方法，因此单值(SV)的history不能很好地体现基于时间的动作序列。在任意时间，每个数据可能会有多个版本，由活动的或已提交的事务创建。事务的读取必须选择适当的版本。考虑第3章的H1，其展示了在单值环境下需要P1。在Snapshot Isolation下，同样的动作序列可能会导致多值的history：

H1.SI: r1[x0=50] w1[x1=10] r2[x0=50] r2[y0=50] c2
r1[y0=50] w1[y1=90] c1

H1.SI has the dataflows of a serializable execution. In [OOBBGM], we show that all Snapshot Isolation histories can be mapped to single-valued histories while preserving dataflow dependencies (the MV histories are said to be View Equivalent with the SV histories, an approach covered in [BHG], Chapter 5). For example the MV history H1.SI would map to the serializable SV history: 
H1.SI

H1.SI.SV: r1[x=50] r1[y=50] r2[x=50] r2[y=50] c2
w1[x=10] w1[y=90] c1

Mapping of MV histories to SV histories is the only rigor- ous touchstone needed to place Snapshot Isolation in the Isolation Hierarchy. 

Snapshot Isolation is non-serializable because a transaction’s Reads come at one instant and the Writes at another. For example, consider the single-value history: 
Snapshot Isolation不是可串行化的，因为一个事务的读操作

H5: r1[x=50] r1[y=50] r2[x=50] r2[y=50] w1[y=-40]
w2[x=-40] c1 c2

H5 is non-serializable and has the same inter-transactional dataflows as could occur under Snapshot Isolation (there is no choice of versions read by the transactions). Here we assume that each transaction that writes a new value for x and y is expected to maintain the constraint that x + y should be positive, and while T1 and T2 both act properly in isolation, the constraint fails to hold in H5. 

Constraint violation is a generic and important type of concurrency anomaly. Individual databases satisfy con- straints over multiple data items (e.g., uniqueness of keys, referential integrity, replication of rows in two tables, etc.). Together they form the database invariant constraint predi- cate, C(DB). The invariant is TRUE if the database state DB is consistent with the constraints and is FALSE otherwise. Transactions must preserve the constraint predicate to maintain consistency: if the database is consistent when the transaction starts, the database will be consistent when the transaction commits. If a transaction reads a database state that violates the constraint predicate, then the transaction suffers from a constraint violation concurrency 
anomaly. Such constraint violations are called inconsistent analysis in [DAT]. 

A5 (Data Item Constraint Violation). Suppose C() is a database constraint between two data items x and y in the 
database. Here are two anomalies arising from constraint violation. 
假设 C()是一个在数据库中关于数据项x和y的约束，下面是两个违反约束的异常

A5A Read Skew Suppose transaction T1 reads x, and then a second transaction T2 updates x and y to new values and 
commits. If now T1 reads y, it may see an inconsistent state, and therefore produce an inconsistent state as output. In terms of histories, we have the anomaly: 
假定事务T1读取x，然后第二个事务T2更新x和y到新的值并提交。如果现在T1读取y，它将看到一个不一致的状态，因此产生一个不一致的输出： 

A5A: r1[x]...w2[x]...w2[y]...c2...r1[y]...(c1 or a1) (Read Skew) 

A5B Write Skew Suppose T1 reads x and y, which are consistent with C(), and then a T2 reads x and y, writes x, and commits. Then T1 writes y. If there were a constraint between x and y, it might be violated. In terms of histo- ries: 
A5B: r1[x]...r2[y]...w1[y]...w2[x]...(c1 and c2 occur) (Write Skew) 


Fuzzy Reads (P2) is a degenerate form of Read Skew where x=y. More typically, a transaction reads two dif- ferent but related items (e.g., referential integrity). Write Skew (A5B) could arise from a constraint at a bank, where account balances are allowed to go negative as long as the sum of commonly held balances remains non-negative, with an anomaly arising as in history H5. 

Clearly neither A5A nor A5B could arise in histories where P2 is precluded, since both A5A and A5B have T2 write a data item that has been previously read by an un- committed T1. Thus, phenomena A5A and A5B are only useful for distinguishing isolation levels that are below REPEATABLE READ in strength. 

The ANSI SQL definition of REPEATABLE READ, in its strict interpretation, captures a degenerate form of row constraints, but misses the general concept. To be specific, Locking REPEATABLE READ of Table 2 provides protection from Row Constraint Violations but the ANSI SQL definition of Table 1, forbidding anomalies A1 and A2, does not. 

Returning now to Snapshot Isolation, it is surprisingly strong, even stronger than READ COMMITTED. 
现在回到Snapshot Isolation来，令人惊讶的，它比READ COMMITTED要强。

Remark 8. READ COMMITTED « Snapshot Isolation

Proof. In Snapshot Isolation, first-committer-wins precludes P0 (dirty writes), and the timestamp mechanism prevents P1 (dirty reads), so Snapshot Isolation is no weaker than READ COMMITTED. In addition, A5A is possible under READ COMMITTED, but not under the Snapshot Isolation timestamp mechanism. Therefore READ COMMITTED « Snapshot Isolation. 
证明：在 Snapshot Isolation下，first-committer-wins规则防止了P0（脏写），时间戳机制防止了P1（脏读），因此Snapshot Isolation不会比READ COMMITTED要弱。此外在READ COMMITTED下可能会发生A5A，但是在 Snapshot Isolation时间戳机制下却不会发生。因此READ COMMITTED « Snapshot Isolation.

Note that it is difficult to picture how Snapshot Isolation histories can disobey phenomenon P2 in the single-valued interpretation. Anomaly A2 cannot occur, since a transac- tion under Snapshot Isolation will read the same value of a 
data item even after a temporally intervening update by an- other transaction. However, Write Skew (A5B) obviously 
can occur in a Snapshot Isolation history (e.g., H5), and in the Single Valued history interpretation we"ve been reason- 
ing about, forbidding P2 also precludes A5B. Therefore Snapshot Isolation admits history anomalies that REPEATABLE READ does not. 

Snapshot Isolation cannot experience the A3 anomaly. A transaction rereading a predicate after an update by another will always see the same old set of data items. But the REPEATABLE READ isolation level can experience A3 anomalies. Snapshot Isolation histories prohibit histories with anomaly A3, but allow A5B, while REPEATABLE READ does the opposite. Therefore: 

Remark 9. REPEATABLE READ »« Snapshot Isolation.

However, Snapshot Isolation does not preclude P3. Consider a constraint that says a set of job tasks determined by a predicate cannot have a sum of hours greater than 8. T1 reads this predicate, determines the sum is only 7 hours and adds a new task of 1 hour duration, while a concurrent transaction T2 does the same thing. Since the two transactions are inserting different data items (and different index entries as well, if any), this scenario is not precluded by First-Committer-Wins and can occur in 
Snapshot Isolation. But in any equivalent serial history, the phenomenon P3 would arise under this scenario. 
但是 Snapshot Isolation不能防止P3。考虑这样一个约束“”

Perhaps most remarkable of all, Snapshot Isolation has no phantoms (in the strict sense of the ANSI definitions A3). 
Each transaction never sees the updates of concurrent transactions. So, one can state the following surprising re- sult (recall that section Table 1 defined ANOMALY SERIALIZABLE as ANSI SQL definition of SERIALIZABLE) without the extra restriction in Subclause 4.28 in [ANSI]: 

Remark 10. Snapshot Isolation histories preclude anomalies A1, A2 and A3. Therefore, in the anomaly interpretation of ANOMALY SERIALIZABLE of Table 1: 
ANOMALY SERIALIZABLE « SNAPSHOT ISOLATION.

Snapshot Isolation gives the freedom to run transactions with very old timestamps, thereby allowing them to do 
time travel — taking a historical perspective of the database — while never blocking or being blocked by writes. Of course, update transactions with very old timestamps would abort if they tried to update any data item that had been updated by more recent transactions. 

Snapshot Isolation admits a simple implementation mod- eled on the work of Reed [REE]. There are several com- mercial implementations of such multi-version databases. Borland’s InterBase 4 [THA] and the engine underlying Microsoft’s Exchange System both provide Snapshot Isolation with the First-committer-wins feature. First- committer-wins requires the system to remember all up- dates (write locks) belonging to any transaction that commits after the Start-Timestamp of each active transac- tion. It aborts the transaction if its updates conflict with remembered updates by others. 

Snapshot Isolation’s "optimistic" approach to concurrency control has a clear concurrency advantage for read-only 
transactions, but its benefits for update transactions is still debated. It probably isn’t good for long-running update transactions competing with high-contention short transac-tions, since the long-running transactions are unlikely to be the first writer of everything they write, and so will probably be aborted. (Note that this scenario would cause a real problem in locking implementations as well, and if the solution is to not allow long-running update transactions that would hold up short transaction locks, Snapshot Isolation would also be acceptable.) Certainly in cases where short update transactions conflict minimally and long-running transactions are likely to be read only, Snapshot Isolation should give good results. In regimes
where there is high contention among transactions of comparable length, Snapshot Isolation offers a classical optimistic approach, and there are differences of opinion as to the value of this.

#### 4.3 Other Multi-Version Systems

There are other models of multi-versioning. Some com-mercial products maintain versions of objects but restrict Snapshot Isolation to read-only transactions (e.g., SQL-92, Rdb, and SET TRANSACTION READ ONLY in some other databases [MS, HOB, ORA]; Postgres and Illustra [STO, ILL] maintain such versions long-term and provide time-travel queries). Others allow update transactions but do not provide first-committer-wins protection (e.g., Oracle Read Consistency isolation [ORA]).
也有其他的多版本模型。一些商业产品维护了对象的版本，但是将快照隔离限制在只读事务；Postgres和Illustra长期维护了这样的版本，并提供了time-travel查询。其他的允许更新事务，但是不提供first-committer-wins的保护（例如Oracle的Read Consistency隔离）

Oracle Read Consistency isolation gives each SQL state-ment the most recent committed database value at the time the statement began. It is as if the start-timestamp of the transaction is advanced at each SQL statement. The members of a cursor set are as of the time of the Open Cursor. The underlying mechanism recomputes the ap-propriate version of the row as of the statement timestamp. Row inserts, updates, and deletes are covered by Write locks to give a first-writer-wins rather than a first-committer-wins policy. Read Consistency is stronger than READ COMMITTED (it disallows cursor lost updates (P4C)) but allows non-repeatable reads (P3), general lost updates (P4), and read skew (A5A). Snapshot Isolation does not permit P4 or A5A.
Oracle的Read Consistency隔离为每个SQL语句提供了语句开始时最新的已提交的数据库值。这就像是

If one looks carefully at the SQL standard, it defines each statement as atomic. It has a serializable sub-transaction (or timestamp) at the start of each statement. One can imagine a hierarchy of isolation levels defined by assign-ing timestamps to statements in interesting ways (e.g., in Oracle, a cursor fetch has the timestamp of the cursor open).
如果仔细的研究SQL标准，它定义了每个语句都是原子的。

### 5. Summary and Conclusions

In summary, there are serious problems with the original ANSI SQL definition of isolation levels (as explained in Section 3). The English language definitions are ambigu-ous and incomplete. Dirty Writes (P0) are not precluded. Remark 5 is our recommendation for cleaning up the ANSI Isolation levels to equate to the locking isolation levels of [GLPT]. 
总之，最初的ANSI SQL的隔离级别定义有着严重的问题。其英文描述有二义性也不完整。脏写（P0）未排除。Remark 5是我们的建议

ANSI SQL intended to define REPEATABLE READ isolation to exclude all anomalies except Phantom. The anomaly definition of Table 1 does not achieve this goal, but the locking definition of Table 2 does. ANSI’s choice of the term Repeatable Read is doubly unfortunate: (1) repeatable reads do not give repeatable results, and (2) the industry had already used the term to mean exactly that: repeatable reads mean serializable in several products. We recommend that another term be found for this. 
ANSI SQL打算通过定义REPEATABLE READ隔离来排除除过幻读之外的所有异常。Table1的异常定义没有达到目标，但是Table2的锁定义达到了。ANSI对于术语REPEATABLE READ的选择是不幸的：（1）可重复读没有提供可重复的结果，（2）工业界已经用了这个术语：一些产品中，可重复读意味着可串行化。我们推荐去为它找另一个术语

A number of commercially-popular isolation levels, falling between the REPEATABLE READ and SERIALIZABLE levels of Table 3 in strength, have been characterized with some new phenomena and anomalies in Section 4. All the isolation levels named here have been characterized as shown in Figure 2 and Table 4. Isolation levels at higher levels in Figure 2 are higher in strength (see the Definition at the beginning of Section 4.1) and the connecting lines are labeled with the phenomena and anomalies that 
differentiate them. 
一些商业流行的隔离级别，都在REPEATABLE READ和SERIALIZABLE级别之间，以处理第4章描述的新现象和异常为特点。这里描述的所有隔离级别都在图2和表4中体现了。图2中更高的隔离级别
On a positive note, reduced isolation levels for multi-ver- sion systems have never been characterized before — de- spite being implemented in several products. Many appli- cations avoid lock contention by using Cursor Stability or Oracle"s Read Consistency isolation. Such applications will find Snapshot Isolation better behaved than either: it avoids the lost update anomaly, some phantom anomalies (e.g., the one defined by ANSI SQL), it never blocks read- only transactions, and readers do not block updates. 

