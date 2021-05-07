---
title: Spark SQL Join Deep Dive
date: 2019-03-29 18:20:19
tags:
  - Spark
  - BigData
---
# Join 基础
## Nested Loop Join
- 本质就是嵌套 for 循环
- 适用于被连接的数据子集较小
- Nested Loop先扫描外表, 每读取一条记录，就去去另一张表(内表, 一般是带索引的大表)里查找. 若没有索引的话一般就不会选择 Nested Loop Join(针对传统数据库).

## Hash Join
- 大数据集连接时的常用方式, 可以减少一次 for 循环
- 优化器使用两个表中较小（相对较小）的表做 hash 表, 然后扫描较大的表并探测该 hash 表，找出与匹配的行。
- 适用于没有索引且较小的表完全可以放于内存中的情况
- Hash Join只能应用于等值连接

## Merge Join
- for 循环次数同 Hash Join
- 排序两个表, 然后遍历第一个表, 在第二表中找对应的 key, 由于是排序的, 查找的复杂度会小于 O(n)
- 主要开销在排序, 如果表的 key 已经是排序的话, 开销比较小

# Spark Join 框架
## 基本执行框架
- 参与Join操作的两张表分别被称为流式表（StreamTable）和构建表（BuildTable）, 一般来说系统会默认将大表设定为流式表，将小表设定为构建表

![Spark SQL 内核剖析](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190630105331.png)

### 无 Shuffle 的 join
#### BroadcastJoinExec
通过将小表 broadcast 到每个 executor 节点上, 从而避免大表产生 shuffle.

![Spark SQL 内核剖析](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190630164046.png)

![Spark SQL 内核剖析](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190630164058.png)

##### 选择条件
1. 首先看有没有 hints, 如果有 hints, 直接选用 BHJ
2. 如果能够广播非构建(build) 表, `JoinSelection#canBroadcast{ plan.stats.sizeInBytes <= conf.autoBroadcastJoinThreshold }`, 默认 broadcast 的 threshold 是10Mb, 也会选用 BHJ.

##### 缺点
1. 每个 executor 上都会有一份表的数据, 有冗余
2. 进行 broadcast 会拉数据到 driver 端, 对 driver 内存造成压力

### 有 Shuffle 的 join
在 Spark SQL 中, 最直观的进行 join 的操作如下:
1. 对两个表分别对应 Join Key 进行 shuffle, 这样两个表上相同的 join key 的记录会在一个分区上, 方便进行 join 操作
2. 对每个分区的记录进行 join(有Hash/Sort 两种方式, 下面介绍).

#### ShuffleHashJoinExec
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190630164357.png)

在 shuffle 过后, 对每个分区中的小表构造出一张 hash 表

##### 选择条件
1. 当不 preferSortMergeJoin 时, 才会看下面的条件, 不然直接会用 SMJ
2. 一边需要 `canBuildLocalHashMap`: `plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions`
3. 小表的数据量要比大表小很多`muchSmaller`: `a.stats.sizeInBytes * 3 <= b.stats.sizeInBytes`
4. join cond 的key 没有排序

#### SortMergeJoinExec

##### 选择条件

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190805222307.png)

当两个表都较大时, 会选用这种 SMJ.

# Spark Join Selection Code

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190805222901.png)

```scala
    def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {

      // --- BroadcastHashJoin --------------------------------------------------------------------

      // broadcast hints were specified
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastByHints(joinType, left, right) =>
        val buildSide = broadcastSideByHints(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))

      // broadcast hints were not specified, so need to infer it from size and configuration.
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastBySizes(joinType, left, right) =>
        val buildSide = broadcastSideBySizes(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))

      // --- ShuffledHashJoin ---------------------------------------------------------------------

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
         if !conf.preferSortMergeJoin && canBuildRight(joinType) && canBuildLocalHashMap(right)
           && muchSmaller(right, left) ||
           !RowOrdering.isOrderable(leftKeys) =>
        Seq(joins.ShuffledHashJoinExec(
          leftKeys, rightKeys, joinType, BuildRight, condition, planLater(left), planLater(right)))

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
         if !conf.preferSortMergeJoin && canBuildLeft(joinType) && canBuildLocalHashMap(left)
           && muchSmaller(left, right) ||
           !RowOrdering.isOrderable(leftKeys) =>
        Seq(joins.ShuffledHashJoinExec(
          leftKeys, rightKeys, joinType, BuildLeft, condition, planLater(left), planLater(right)))

      // --- SortMergeJoin ------------------------------------------------------------

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if RowOrdering.isOrderable(leftKeys) =>
        joins.SortMergeJoinExec(
          leftKeys, rightKeys, joinType, condition, planLater(left), planLater(right)) :: Nil

      // --- Without joining keys ------------------------------------------------------------

      // Pick BroadcastNestedLoopJoin if one side could be broadcast
      case j @ logical.Join(left, right, joinType, condition)
          if canBroadcastByHints(joinType, left, right) =>
        val buildSide = broadcastSideByHints(joinType, left, right)
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      case j @ logical.Join(left, right, joinType, condition)
          if canBroadcastBySizes(joinType, left, right) =>
        val buildSide = broadcastSideBySizes(joinType, left, right)
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      // Pick CartesianProduct for InnerJoin
      case logical.Join(left, right, _: InnerLike, condition) =>
        joins.CartesianProductExec(planLater(left), planLater(right), condition) :: Nil

      case logical.Join(left, right, joinType, condition) =>
        val buildSide = broadcastSide(
          left.stats.hints.broadcast, right.stats.hints.broadcast, left, right)
        // This join could be very slow or OOM
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      // --- Cases where this strategy does not apply ---------------------------------------------

      case _ => Nil
    }

```