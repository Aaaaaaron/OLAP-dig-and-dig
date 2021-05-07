---
title: Spark map vs mapPartitions
date: 2018-10-16 10:43:54
tags:
  - Spark
  - BigData
---
官方定义:
```scala
  /**
   * Return a new RDD by applying a function to all elements of this RDD.
   */
  def map[U: ClassTag](f: T => U): RDD[U] = withScope {
    val cleanF = sc.clean(f)
    new MapPartitionsRDD[U, T](this, (context, pid, iter) => iter.map(cleanF))
  }

  /**
   * Return a new RDD by applying a function to each partition of this RDD.
   *
   * `preservesPartitioning` indicates whether the input function preserves the partitioner, which
   * should be `false` unless this is a pair RDD and the input function doesn't modify the keys.
   */
  def mapPartitions[U: ClassTag](
      f: Iterator[T] => Iterator[U],
      preservesPartitioning: Boolean = false): RDD[U] = withScope {
    val cleanedF = sc.clean(f)
    new MapPartitionsRDD(
      this,
      (context: TaskContext, index: Int, iter: Iterator[T]) => cleanedF(iter),
      preservesPartitioning)
  }
```

可以看到的是 mapPartitions 需要的函数参数传入的是一个 iter, 返回的也是一个 iter, 而 map 仅仅是一个元素.

假设我们有 10k 个元素, 10个 partitions, 数据均匀分布:
1. map:调用10k 次 map 方法
2. mapPartitions:调用10次 mapPartitions 方法, 每次传入1k个(一个 partition 的数据量)进行计算. 结果先存到 memory 中, 直到可以返回.
3. flatMap在 单个元素(map)上工作，生成结果是多个元素(mapPartitions)

结论:mapPartitions 转换比 map 快，因为它调用你的函数 一次/分区，而不是 一次/元素, 像有一些高开销的 init 的时候, 如数据库连接, 使用 mapPartitions, 每个分区就只需要一次 init.

来看一个使用 mapPartitions 报错的例子, 运行这段代码, 会报 already closed exception, 因为 spark 的计算都是 lazy 的, 下面的 partition.map 到真的触发计算的时候, conn 已经 close 了.解决方法就是 `val newPartition = partition.map(...}).toList` 触发计算.

```scala
val newDF = myDF.mapPartitions(
  partition => {
    val conn = new DbConnection
    val newPartition = partition.map(record => { readMatchingFromDB(record, connection) })
    conn.close()
    newPartition
  }).toDF()
```

map mapPartitions 对外部引用的更新都没法作用出来: 下面的 flag 还是0.
```scala
var flag = 0
val test = rdd.map {
  row => flag += 1
}.collect()
```

