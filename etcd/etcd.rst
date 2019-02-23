**************
etcd 学习笔记
**************

关于etcd
========
etcd是Raft一致性算法的一种基于golang的实现。

本文基于版本3.3.X, v3 api

官方github：`etcd-io/etcd <https://github.com/etcd-io/etcd>`_

Raft一致性算法论文： [1]_

etcd目标
--------

* 通过强一致性、可靠性的KV store(**reliable capacity: several gigabytes**)，提供关键元数据存储。
* 提供分布式锁、选举等协调服务。

etcd的一致性guarantees
----------------------

* CAP语义下：发生网络分区时，etcd牺牲可用性确保一致性，属于CP模型。
* 一致性模型：所有操作遵循sequential consistency / serializability(txn).
* 只读操作：在默认配置下，额外提供serializable read.

etcd提供的一致性仅弱于linearizability / strict serializability. 在实践中，Google Spanner达到了strict serializability(也称为external consistency).


Raft算法guarantees
------------------


.. _Election Safety:

* **Election Safety**: 同一任期只能有一个leader当选

.. _Leader Append-Only: 

* **Leader Append-Only**: leader不会覆盖和删除既有日志条目；只能增加新条目

.. _Log Matching: 

* **Log Matching**: 如果两日志都包含一个具有相同index和term的条目，那么可以认为这两个日志从头到该条目为止是完全一致的

.. _Leader Completeness: 

* **Leader Completeness**: 如果一个日志条目在给定term被提交，那么这条日志一定会被包含在所有任期大于给定term的leader日志中

.. _State Machine Safety: 

* **State Machine Safety**: 如果某一server将一个日志条目apply到状态机中，那么其他server不会apply一个具有相同index的不同日志条目

以上摘自raft paper [1]_

etcd实现
========

消息流
------

.. figure:: images/follower-put.png
   
   图1:用户通过client连接follower进行put操作

.. figure:: images/main-loop.png

   图2:三个routine通过channel推动raft状态机

raft状态机主要通过三个go routine推动：

#. step-routine: 位于raft/node.Node.run. Event loop, 接收并处理事件，并维护mailbox(包含raft updates与待发msgs)

#. raft-routine: 位于etcdserver/raft.raftNode.start. 触发tick事件，从step-routine接收就绪的raft updates并持久化到wal，发送待发msgs到对应的peers，派发apply entries到apply-routine，并通过advance channel通知step-routine已完成

#. apply-routine: 位于etcdserver/server.EtcdServer.run. 接收raft-routine发送的apply entries，schedule apply到fifo队列中完成

**step-routine, raft-routine, apply-routine并非官方正式命名**

图1中的流程为：

#. follower接收client请求，通过对应API入口，异步注册wait(等待proposal成功apply到状态机)，并proposal到自身状态机(MsgProp -> stepFollower)

#. follower将MsgProp转发给当前leader

#. leader将MsgProp中的entries写入wal，并同时下发包含entries的MsgApp给所有follower

#. 所有follower将MsgApp中的entries写入wal，并通过MsgAppResp向leader报告

#. leader接收maintain所有members的progress，称为prs，当该条日志在quorum中被报告完成(条目index低于或等于quorum progress)，leader正式提交quorum progress之前的条目

#. leader将quorum committed index放入下一条MsgApp中向所有followers广播

#. follower根据MsgApp中的m.Commit, 将可执行的entries apply到状态机中，如果当前follower在已注册的wait中发现对应client请求，那么通知wait向client返回结果


etcd read
---------

- 如果在range request中将serializable设置为true(default false)，follower接受到请求后直接进行本地查询后返回

  * 相较于默认配置，性能有一定提升
  * 违背了一致性承诺，有stale reads现象

- 默认情况下，


TODO

Reference
=========

.. [1] `In Search of an Understandable Consensus Algorithm <https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf>`_