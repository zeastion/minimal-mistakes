---
title: "分布式学习手记04 - ZooKeeper 及 ZAB 协议简介"
categories:
  - 分布式
date: 2018-06-13
toc: true
toc_label: ""
toc_icon: "align-left"
header:
  teaser: /assets/images/fbs_teaser.gif
---

> ZooKeeper 是由雅虎创建的开源分布式协调服务，源于谷歌 Chubby，使用 ZAB (ZooKeeper Atomic Broadcast) 协议。

## 简介

ZooKeeper 是一个典型的分布式数据一致性的解决方案，分布式应用程序可以基于它实现数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master 选举、分布式锁和分布式队列等功能。

### 设计目标

ZooKeeper 致力于提供一个高性能、高可用、具有严格的顺序访问控制能力的分布式协调服务。

- 目标 1 简单的数据模型

  通过一个共享的、树形结构的名字空间进行协调，并将全量数据存储在内存里，以此提高服务器吞吐、减少延迟；

- 目标 2 可以构建集群

  过半的节点正常工作，集群就可以正常提供服务，保证高可用；

- 目标 3 顺序访问

  使用全局唯一的递增编号，达到顺序访问控制，实现复杂的同步原语；

- 目标 4 高性能

  尤其适用读操作为主的应用场景。

### 概念名词

ZooKeeper 的几个核心概念

- 集群角色

  Leader - 为客户端提供读和写服务；

  Follwer - 参与写操作的 “过半写成功” 策略，提供读服务，参与 Leader 的选举过程；

  Observer - 提供读服务。

- Session

  客户端会话，客户端和服务器端会创建一个 TCP 长连接，通过该连接，客户端可以通过心跳检测与服务器保持有效的会话，向其发送请求并接受响应。

- ZNode

  数据模型中的数据单元，ZooKeeper 的数据模型是一棵树 (ZNode Tree)，由 / 进行分割的路径就是一个 ZNode。每个 ZNode 上会保存自己的数据内容及相关属性信息。

- 版本

  对应于每个 ZNode，ZooKeeper 都会为其维护一个叫 Stat 的数据结构，Stat 中记录该 ZNode 的三个数据版本：

  version - 当前 ZNode 的版本；

  cversion - 当前 ZNode 子节点的版本；

  aversion - 当前 ZNode 的 ACL 版本。

- Watcher

  事件监听器，ZooKeeper 允许用户在指定节点上注册一些 Watcher，并在特殊事件触发时，由 ZooKeeper 服务端将事件通知推送给目标客户端。

- ACL

  ZooKeeper 采用 ACL (Access Control Lists) 策略进行权限控制，包括 CREATE、READ、WRITE、DELETE、ADMIN 五种权限。

## ZAB 协议

ZooKeeper Atomic Broadcast 是特别为 ZooKeeper 设计的崩溃可恢复的原子消息广播算法。

### 模式

ZAB 协议包括崩溃恢复和消息广播两种模式：

- 崩溃恢复

  当整个服务框架启动时，或当 Leader 节点发生故障，ZAB 协议就会进入崩溃恢复模式：选举新的 Leader，当集群中超过半数机器与新 Leader 完成状态同步（数据同步），ZAB 协议就退出崩溃恢复，进入消息广播模式；

- 消息广播

  原子广播的过程类似二阶段提交：针对客户端的事务请求，Leader 节点为其生成对应的事务提议 (Proposal) ，并将其发送给集群中其余所有机器，再收集选票，决定是否进行事务提交。此外，广播协议基于具有 FIFO 特性的 TCP 协议进行网络通信，从而保证消息接收与发送的顺序性。

### 算法描述

ZAB 协议包括消息广播和崩溃恢复两个过程，细分为三个阶段：发现 -> 同步 -> 广播。

参与协议的每一个分布式进程，会循环执行这三个阶段，一个循环称为一个主进程周期。

#### 阶段 1 - 发现

发现阶段即 Leader 选举过程，在多个分布式进程中选举出主进程。

1.1 Follower 将自己最后接受的事务主进程周期 epoch 值发送给准 Leader L<sub>准</sub>；

  - 事务标识包括两个部分：主进程周期 epoch（高 32 位） + 主进程周期内的事务计数 counter（低 32 位，从 0 递增）

1.2 接收到超过半数 Follower 的 epoch 消息后，L<sub>准</sub> 生成新的 epoch<sub>new</sub> ，发送给这些 Follower；

  - epoch<sub>new</sub> 是 L<sub>准</sub> 收到的所有 epoch 中最大值加一

1.3 Follower 收到 L<sub>准</sub> 发送的 epoch<sub>new</sub> ，更新自己的 epoch 值并向 L<sub>准</sub> 反馈 Ack 消息；

  - 这个 Ack 消息包含当前 Follower 的 epoch<sub>new</sub> 和该 Follower 的历史事务提议集合 I<sub>提</sub>

#### 阶段 2 - 同步

2.1 L<sub>准</sub> 将 epoch<sub>new</sub> 和 I<sub>提</sub> 打包成 NewLeader(e,I) 消息发送给所有 Follower；

2.2 Follwer 接收到 NewLeader(e,I) 消息后：若 epoch<sub>new</sub> 和该 Follower 本地的 epoch 不匹配，则该 Follower 无法参与本轮同步，进入下一轮循环；若能匹配，则 Follower 执行事务的应用操作，完成后向 L<sub>准</sub> 发送反馈；

2.3 L<sub>准</sub> 接收到过半 Follower 对 NewLeader(e,I) 的反馈后，向所有 Follower 发送 Commit 消息；

2.4 Follower 收到 L<sub>new</sub> 的 Commit 消息后，依次处理并提交所有前期事务集合；

#### 阶段 3 - 广播

完成数据同步后，崩溃恢复阶段结束，ZAB 协议开始接收客户端新的事务请求，进行消息广播流程

3.1 L<sub>new</sub> 接收到客户端新的事务请求后，生成对应的事务 Proposal，向所有 Follower 发送提案；

3.2 Follower 根据先后次序处理收到的提案，向 L<sub>new</sub> 发送反馈 Ack；

3.3 L<sub>new</sub> 收到过半 Follower 对事务的 Ack 消息，发送 Commit 给所有 Follower，要求它们进行事务的提交；

3.4 Follower 收到 L<sub>new</sub> 的 Commit 消息后，提交该事务。

### 运行状态

整个框架运行过程中，每一个进程可能处于三种状态之一

- LOOKING - Leader 选举阶段

  框架启动阶段，所有进程的初始化状态，或 Leader 崩溃时，Follower 会转换到该状态。处于 LOOKING 状态的进程都会试图选举一个新的 Leader。

- FOLLOWING - Follower 节点和 Leader 保持同步状态

  产生新的 Leader 后，Follower 处于该状态，与唯一的 Leader 保持同步。

- LEADING - Leader 服务器作为主进程领导状态

  选举完成后，新的 Leader 处于该状态。
