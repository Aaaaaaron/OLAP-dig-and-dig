---
title: 'Hadoop MR 和 Spark 对比'
date: 2018-11-02 15:21:13
tags:
---
### 0. 启动开销
总结: Spark 计算比 MapReduce 快的根本原因在于DAG计算模型, 但 MR 真正的缺点是抽象层次太低, 大量底层逻辑需要开发者手工完成. 但是也不是说 MR 就已经没用了, 没有最好的技术, 只有合适你需求的技术.

### 1. 启动开销
Hadoop MapReduce 采用了多进程模型, 而Spark采用了多线程模型. 

Hadoop MapReduce 每个 Mpa task/Reduce Task 都是一个 JVM, 是基于进程的, task 的启动时间在秒级, 然后用完后又立即释放, 不能被其他任务重用; Spark executor 是常驻的, taks 是基于线程的, 而且由于在一个 JVM 中, 方便数据共享.

但是基于进程的好处是每个 task 都可以控制自己的资源粒度, 而线程的资源隔离并没有保证.

 MR 稳定是真的. 现在越来越觉得稳定性比性能重要很多, MR 虽然慢, 但是基本上能保证跑出结果. MR 真正的缺点是MR抽象层次太低, 大量底层逻辑需要开发者手工完成.

### 2. DAG 优化
#### 消除了冗余的 HDFS 读写
单个 MR job 和 Spark 其实可能也没啥区别, 差异主要在多个MR组成的复杂Job来和Spark比

对于一个 job, 会启动很多轮 MR 组合计算, MR 每次都会从 HDFS 读, 再写回到 HDFS, 下轮 MR 任务又要从 HDFS 读, 但是 Spark 只需要一个 job, 只读写 HDFS 一次. 中间只落本地磁盘.

#### 消除了冗余的 MapReduce 阶段

Spark lazy evaluation, 减少不必要的 stage, 可以减少 shuffle 次数, 要是没有 shuffle, Spark可以在内存中一次性完成这些操作.

### 3. Cache
Spark 可以指定 Cache 某个 RDD, 以加速后面计算.

TBD: 从更高维度剖析 Hadoop MR 和 SQL 系统的区别.