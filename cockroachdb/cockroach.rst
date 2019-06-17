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

   图1:sql overview (来源: pathToCockroachSrc/src/github.com/cockroachdb/cockroach/docs/tech-notes/sql/sql-overview.png)

图1展示了SQL执行的流程，涵盖了CRDB的5个layer：

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

- Leaseholder
    负责处理对应Range的所有Read和Write操作。
    对于Read操作，无需经过raft直接从MVCC读取对应版本。因此，成为Leaseholder的node必须是in-sync的。
    对于Write操作，则需通过raft进行协调。
    Leaseholder并不等同于raft leader, 但通常他们是共栖的(colocated), 这样可以降低非必要的通信开销。

- Raft Leader & Raft log
    参考 `etcd notes <../etcd/etcd.rst>`_


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
