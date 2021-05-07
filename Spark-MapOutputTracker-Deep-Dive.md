---
title: Spark MapOutputTracker Deep Dive
date: 2019-10-16 22:35:36
tags:
  - Spark
  - BigData
---

## 基础
Shuffle writer 会将中间数据保存到 Block 里面, 然后将数据的位置发送给 `MapOutputTracker`; Shuffle reader 通过向 `MapOutputTracker` 获取中间数据的位置之后, 才能读取到数据.

`MapOutputTrackerMaster` 启动在 driver 端, `MapOutputTrackerWorker` 启动在 executor 端.

ShuffleStatus 就是 `Map[Int, Array[MapStatus]]`, key 是 Shuffle 的 ID, value 数组的大小是该 ShuffleMapTask 的个数, MapStatus 会记录 stage reduce 端 task 个数的 status, 具体实现有两种: `CompressedMapStatus`/`HighlyCompressedMapStatus`, 具体实现之后分析, 当 reduce 端 task 超过 2000 (`SHUFFLE_MIN_NUM_PARTS_TO_HIGHLY_COMPRESS`) 的时候, 会使用 `HighlyCompressedMapStatus`, 看名字可以看出后一种压缩率更高. 因为其实这个 `Map[Int, Array[MapStatus]]` 会非常占据内存, 试想下, 假如我有10w 个 map 端的 task 和10w 个 reduce 端的 task, 那么这个 `Array[MapStatus]` 实际存了 100亿 task 的信息, 而且后面这些 status 还要序列化发给 executor, 又会占用更多的空间, 同时 Spark 这里代码写的也不是非常好, 导致内存占用会很高. 而 driver 端的内存大家一般不会设置的特别高, 这里就会导致 OOM, 而 driver 又是 Spark 的单点, 这是一个非常严重的稳定性问题. 之后我会给出具体的例子和修复.

## 流程:

### Write
1. `MapOutputTrackerMaster` 会 `registerShuffle` 和 `registerMapOutput`. registerShuffle 是 DAGScheduler 在创建一个 `ShuffleMapStage` 时会把这个 stage 对应的 shuffle 注册进来(`createShuffleMapStage`); `registerMapOutput` 是 在一个 `shuffleMapTask` 任务完成后(`DAGScheduler.handleTaskCompletion`)，会把 `shuffleMapTask` 输出的信息(`MapStatus`)放进来.

### Read
1. 当 shuffle read 的时候, `BlockStoreShuffleReader中`，会调用 `MapOutputTrackerWorker.getMapSizesByExecutorId` (master 端的这个方法只在 local 用)

2. 调用 `MapOutputTrackerWorker#getStatuses(shuffleID)`, Worker 有个 mapStatuses 缓存 `Map[Int, Array[MapStatus]]`, 当 Miss 的时候, 会去 fetching, 就有两个很重要的方法:
```scala
val fetchedBytes = askTracker[Array[Byte]](GetMapOutputStatuses(shuffleId))
fetchedStatuses = MapOutputTracker.deserializeMapStatuses(fetchedBytes) // 有两种模式 direct 的和 broadcast 的
```
Worker askTracker 向 `MapOutputTrackerMasterEndpoint` 要 statues, 这个 endpoint 会向 MapOutputTrackerMaster post 一个 `GetMapOutputMessage(shufflID)` 事件(放入 `LinkedBlockingQueue[GetMapOutputMessage]`), 且 master 会启动一个 `MessageLoop`, 会 take 这个阻塞队列的事件, 从 master 自己内存中维护的 `shuffleStatuses` 找到对应 shuffleID 的 ShuffleStatus(`Map[Int, Array[MapStatus]]`), 在 Write 中提过, 当 `shuffleMapTask` 完成的时候, 会通知 `DAGScheduler.handleTaskCompletion`, 所以 driver 有所有的 `MapStatues`.

3. driver 拿到对应的 shuffleStatuses 之后, 需要把它 reply 回 请求的发起方, 也就是 executor, 这是最耗费内存的一步操作, 也是外面后期性能优化的点: `context.reply(shuffleStatus.serializedMapStatus(broadcastManager, isLocal, minSizeForBroadcast))`, 这个方法会调用 `MapOutputTracker.serializeMapStatuses`. 这个方法会使用 Java 的序列化机制(ObjectOutputStream)来序列化一个 `Array[MapStatus]` (对应一个 shuffle 的所有 MapStatus 输出), 并使用 gzip 压缩, 当序列化完之后, 会有有两种通知给 executor 的模式: 当序列化后的 byte 数组大小小于 minBroadcastSize(512K) 时, 会直接返回 Array[Byte], 后续使用 Spark 的 RPC 模式返回给 executor, 否则则用 Broadcast 机制返回给 executor.

## 问题

1. 每个 executor 拿的对应 shuffle 的 `Array[MapStatus]` 都是全量的. 这其实没有必要, 最好每个 executor 只拿自己 task 需要的 map statues 就可以了, 但是这个实现不容易
2. 当 reduce 端 task 非常多的时候, 会使用 `HighlyCompressedMapStatus`, 这里面会用一个 RoaringBitmap 存 emptyBlocks, 但是其实当 reduce 特别多的时候, 存有 block 的反而更少 
3. 序列化的时候, 使用 `ByteArrayOutputStream`, 且没有设置初始化大小, 导致一直在 grow, 不断的发生 array copy. 而且 `ByteArrayOutputStream` 比较坑爹, toByteArray 还会进行一次 array copy.
4. 当满足一定条件会进行 broadcast, toByteArray又生成一个 array. 且要是进行 broadcast 的话, 上面的序列化就根本没有必要, 因为 broadcast 还会进行一次序列化.

## 模拟
测试数据集为 tpch 50, 使用 Spark SQL 测试, 测试查询为`select count(*) from lineitem group by l_comment `, 启动 20w Map 端 task, 5w Reduce 端 task, 4g driver memory.

发生 oom
```
java.lang.OutOfMemoryError: Java heap space
    at java.util.Arrays.copyOf(Arrays.java:3236)
    at java.io.ByteArrayOutputStream.grow(ByteArrayOutputStream.java:118)
    at java.io.ByteArrayOutputStream.ensureCapacity(ByteArrayOutputStream.java:93)
    at java.io.ByteArrayOutputStream.write(ByteArrayOutputStream.java:153)
    at java.util.zip.DeflaterOutputStream.deflate(DeflaterOutputStream.java:253)
    at java.util.zip.DeflaterOutputStream.write(DeflaterOutputStream.java:211)
    at java.util.zip.GZIPOutputStream.write(GZIPOutputStream.java:145)
    at java.io.ObjectOutputStream$BlockDataOutputStream.writeBlockHeader(ObjectOutputStream.java:1894)
    at java.io.ObjectOutputStream$BlockDataOutputStream.drain(ObjectOutputStream.java:1875)
    at java.io.ObjectOutputStream$BlockDataOutputStream.flush(ObjectOutputStream.java:1822)
    at java.io.ObjectOutputStream.flush(ObjectOutputStream.java:719)
    at java.io.ObjectOutputStream.close(ObjectOutputStream.java:740)
    at org.apache.spark.MapOutputTracker$$anonfun$serializeMapStatuses$2.apply$mcV$sp(MapOutputTracker.scala:804)
    at org.apache.spark.util.Utils$.tryWithSafeFinally(Utils.scala:1369)
    at org.apache.spark.MapOutputTracker$.serializeMapStatuses(MapOutputTracker.scala:803)
    at org.apache.spark.ShuffleStatus.serializedMapStatus(MapOutputTracker.scala:174)
    at org.apache.spark.MapOutputTrackerMaster$MessageLoop.run(MapOutputTracker.scala:397)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)
```

查看 dump:

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20191016230210.png)


## 解决
TODO