****************
rockroachDB简述
****************

Preliminary
===========

CRDB是什么
----------

    CockroacbDB致力于同时提供scalability和strong consistency，支持SQL(pgwire)的同时解决关系型数据库在cloud native下的相关问题。
    CockroachDB源于google spanner.

Spanner和TrueTime
-------------------

    与CockroachDB相比，Spanner除上述特性外，是首个提供支持外部一致性(external-consistency)分布式事务的数据库。
    Spanner通过TrueTime实现external-consistency.   
    通过调用TrueTime API: TT.now(), 返回TTinterval: [earliest, latest]代表了uncertainty的上下界，取决于时钟漂移(clock-drift)的最差预估。

.. code::

                         +  
       +----------------+| 
       |to commit Txn A || must wait until
       +----------------+| TT.after(latest) 
       ^                ^| before commit
       |                ||
       <--uncertainty -->|
       +                +|
    earliest          latest

假设有无关事务A(node1)与B(node2), Alice于终端提交A并在时间t1提交成功，Bob观测到A成功提交后立即提交B:

.. code::

                      t1           t2
                      +            +
     +--------------+ |            |
     |     TxnA     | |            |
     +--------------+ |            |
     |              | |            |
    t1e            t1l|            |
      \    7ms    /   |            |
       uncertainty    |            |
                      |            |
                   +--+-----------++
                   |     TxnB     |
                   +--------------+
                   |              |
                  t2e            t2l
                    \    7ms    /
                     uncertainty
    通过commit wait,可以确保 t1l < t1 < t2l < t2

TrueTime的局限性在于特定的环境依赖(如GCE),造成verdor lock-in.
没有TrueTime的支持，时钟漂移通常会达到百毫秒级别，CRDB不能使用commit wait.若A, B被提交至所在节点的本地时间分别为tn1和tn2, 且tn2 < tn1. 可以观察到 B HappensBefore A 的异常(anomaly).
因此，CRDB的一致性模型弱于external-consistency(linearizability),但达到了serializability, 即txn并不对外立即可见，对外可见后，所有观测者可以得到一致的顺序。
CRDB使用HLC(hybird-logical clock)管理排序和因果问题。详细请看 `HLC paper <http://www.cse.buffalo.edu/tech-reports/2014-04.pdf>`_ 


Overview
========

.. figure:: images/sql-overview.png

   sql overview (来源: pathToCockroachSrc/src/github.com/cockroachdb/cockroach/docs/tech-notes/sql/sql-overview.png)

上图展示了SQL执行的流程，涵盖了CRDB的5个layer：

1. `SQL <https://www.cockroachlabs.com/docs/stable/architecture/sql-layer.html>`_ 解析SQL，并生成可执行的flow
2. `Transactional <https://www.cockroachlabs.com/docs/stable/architecture/transaction-layer.html>`_ 确保transaction的ACID特性
3. `Distribution <https://www.cockroachlabs.com/docs/stable/architecture/distribution-layer.html>`_ 为每个节点提供storage的kv抽象层(通过distAPI使数据对上层提供整体性视图)
4. `Replication <https://www.cockroachlabs.com/docs/stable/architecture/replication-layer.html>`_ 维护各个节点replicas, 提供高可用性和伸缩性
5. `Storage <https://www.cockroachlabs.com/docs/stable/architecture/storage-layer.html>`_ 基于RocksDB, 用于数据、元数据和辅助数据的读写


重要概念
=========

- Range
    CRDB切分数据的基本单元。Range的大小维持在一个区间，默认为16MB~64MB。达到上限将触发split; 反之，则触发merge.
    Range同样是replica的基本单元，Range的多份replicas中有且仅有一个Leaseholder.
    Range中的行按升序排列。

- Leaseholder
    负责处理对应Range的所有Read和Write操作。
    对于Read操作，无需经过raft直接从MVCC读取对应版本。因此，成为Leaseholder的node必须是in-sync的。
    对于Write操作，则需通过raft进行协调。
    Leaseholder并不等同于raft leader, 但通常他们是共栖的(colocated), 这样可以降低非必要的通信开销。

- Raft Leader & Raft log
    参考 `etcd notes <../etcd/etcd.rst>`_

- Span
    表示[startKey, endKey)范围内的所有key, Span可以跨越任意Ranges.

- conflict
    有如下情况：
    1. 对于发生于t1的read, 在mvcc中遇到t2的value或intent, t1接近t2且t1 < t2, 由于跨节点时钟不同步，该read被认为发生在uncertainty window, 返回ReadWithinUncertaintyIntervalError.
    2. 对于发生于t1的read, 在mvcc中遇到t0的intent, 且t1 > t0. 

子系统
========

closed timestamp
----------------

用于实现follower read功能( `AS OF SYSTEM TIME <https://www.cockroachlabs.com/docs/v19.1/as-of-system-time.html#select-historical-data-time-travel>`_)
若满足：对于read请求时间ts, closedts >= ts, 那么read可以在follower节点完成。

.. code::

        closed           next
          |          left | right
          |               |
          |               |
          v               v
    ---------------------------------------------------------> time
    图摘自tracker.go

cockroach周期性地尝试close next, 时间轴被分为三个区域：

1. (-∞, closed] : closed之前的状态immutable.
2. (closed, next]: left sets中包含next之前提交的proposal, 该集合不能添加新proposal
3. (next, ∞]: right sets中包含next之后提交的proposal, 可以添加新proposal

latch manager
-------------

每个range replica包含lgmanager, 内部使用interval-btree维护行锁和区间锁，使用多个btree区分读写锁。


锁的临界区包含：

- Timestamp Cache的读写
    Timestamp Cache同样以区间的形式维护

- MVCC的写入
    写入values(non-transaction或1PC)、intents(2PC)

- raft log finish application

latch manager使用copy-on-write提高自身并发访问性能，区间锁的获取过程如下：

1. manager.Lock
2. 拷贝一份btree, 作为snapshot返回
3. 插入自己要访问的一组锁lg(下一个请求需要acquire snapshot和lg的并集)
4. manager.Unlock
5. 对于snapshot和lg的overlaps, 阻塞等待直至overlaps都被释放。

timestamp Cache
---------------

tsCache内部同样区分读写cache, 它们对应的范围：

- rCache对应：
    GetRequest, ScanRequest等读操作
    ConditionalPutRequest, InitPutRequest, IncrementRequest等CAS操作
    RefreshRequest, RefreshRangeRequest(作用于读操作)
    PushTxnRequest(read/write conflict, push timestamp)

- wCache
    由于MVCC存放了timestamp信息，写操作不会更新tsCache. DeleteRangeRequest是个例外, 因为在key不存在的情况下并不会在MVCC中写入tombstone(考虑[0, ∞)会产生无数个墓碑)
    RefreshRequest, RefreshRangeRequest(作用于写操作，即DeleteRangeRequest)
    EndTransactionRequest
    PushTxnRequest(write/write conflict, push abort)

TsCache的实现有skiplist, btree和llrbtree, 与latch manager区别在于cache中的区间相互没有overlap. 
TsCache位于memory, 因此cache设置了low water mark 限制item数量，之前的timestamp不会放入cache, 旧的timestamp通过FIFO或LRU进行eviction.

Transaction之旅
================

**跟随transaction漫游CRDB的实现**

准备工作
---------

- 从源码安装CockroachDB：

.. code::

    wget -qO- https://binaries.cockroachdb.com/cockroach-v19.1.1.src.tgz | tar  xvz
    cd cockroach-v19.1.1 && make build
    # binary文件在 ./src/github.com/cockroachdb/cockroach/cockroach

- 配置一组CRDB cluster：

1. 在IDE中启动第一个node,启动参数如下：
    Environment: COCKROACH_DISTSQL_LOG_PLAN=true  (设置该参数可输出SQL执行计划URL)
    Arguments: start --insecure --listen-addr=192.168.1.101:26257 --logtostderr=INFO
2. 增加两个node：

.. code::

    cd pathToCockroachSrc/src/github.com/cockroachdb/cockroach
    # 使用 --join=XXX 加入现有cluster
    ./cockroach start --insecure --listen-addr=192.168.1.102:26257 --http-addr=192.168.1.102:8080 --store=node2 --join=192.168.1.101:26257
    ./cockroach start --insecure --listen-addr=192.168.1.102:26258 --http-addr=192.168.1.102:8081 --store=node3 --join=192.168.1.101:26257

访问任意节点gui可查看cluster状态，例：http://192.168.1.101:8080

- 建表并populate一定量数据：

.. code::

    # 使用同一binary文件连接至node
    ./cockroach sql --insecure --host=192.168.1.101:26257
    create database bihu;
    use bihu;
    create table gecko (key int primary key, value varchar(1000), gid int);
    # 写入一定量数据，省略...
    # 可以通过experimental_ranges命令观察到数据分裂成为多个ranges
    show experimental_ranges from table gecko; 
      start_key | end_key | range_id | replicas | lease_holder
    +-----------+---------+----------+----------+--------------+
      NULL      | /112821 |       62 | {1,2,3}  |            2
      /112821   | /225420 |       72 | {1,2,3}  |            1
      /225420   | NULL    |       73 | {1,2,3}  |            3
