---
title: Spark bucketing 机制
date: 2019-06-09 17:10:27
tags:
---
# 0. 基础
Spark 的 bucket 机制其实和 repartition 非常相似, 但是 Spark 的 repartition 在读取的时候不会做任何的 pruning, 而 bucket by 会 pruning 掉不必要的文件.

bucket by 与 repartition 都是对于数据里每条记录通过一个 hash 函数得到一个值, 任何相同值的记录放到同一个文件中去. 对于点查, 例如 id=5, 那么可以把5当做一条记录的值, 然后算出它的 hash 值得到它的 bucket 是哪个, 那么我们只要扫这个 bucket 就可以了, 因为其他 bucket 里肯定没有5这个元素.

bucket by 的写入比较特殊, 不能直接 write.parquet, 这里是一种写法: `df.write.format("parquet").option("path", "/tmp/bueket").bucketBy(3, "id").saveAsTable("t")`, buket 的列是 id, 分了三个桶, 最后产生出来的文件会小于等于3个(如果 id 只有一个值, 只会有一个文件). 如下, 文件名中`_`后的, 就是每个文件的 buckId(也是里面记录的 hash 值).

```
part-00000-39d128dc-69a1-4b91-8931-68e81c77c4ae_00000.c000.snappy.parquet
part-00000-39d128dc-69a1-4b91-8931-68e81c77c4ae_00001.c000.snappy.parquet
part-00000-39d128dc-69a1-4b91-8931-68e81c77c4ae_00002.c000.snappy.parquet
```

# 1. Strategy 部分

Spark 具体读取底层数据文件的 SparkStrategy 叫做 FileSourceStrategy, 在其 apply 方法中我们可以看到, 他需要一个 `LogicalRelation` 来触发到apply 方法中的各个逻辑, `LogicalRelation` 中最重要的是`HadoopFsRelation`, 我们也可以构造自己的`HadoopFsRelation`: `sparkSession.baseRelationToDataFrame`. 之后可能会详细说下`HadoopFsRelation`, 这边暂时先不看.

在 `FileSourceStrategy#apply` 中会对于可以进行 bucket pruning 的 case(bucketColumnNames.length == 1 && numBuckets > 1), 会返回一个 `bucketSet`, 它是一个bitset, 通过这个 `bucketSet` 我们就能知道有哪些文件被选中了(010就代表第二个文件被选中了, bitset 这个用法还是有点装逼). 

下面来看看这个 `bucketSet` 是如何返回的, 代码如下:
```scala
  private def genBucketSet(
      normalizedFilters: Seq[Expression],
      bucketSpec: BucketSpec): Option[BitSet] = {
    if (normalizedFilters.isEmpty) {
      return None
    }

    val bucketColumnName = bucketSpec.bucketColumnNames.head
    val numBuckets = bucketSpec.numBuckets

    val normalizedFiltersAndExpr = normalizedFilters
      .reduce(expressions.And)
    val matchedBuckets = getExpressionBuckets(normalizedFiltersAndExpr, bucketColumnName,
      numBuckets)

    val numBucketsSelected = matchedBuckets.cardinality()

    logInfo {
      s"Pruned ${numBuckets - numBucketsSelected} out of $numBuckets buckets."
    }

    // None means all the buckets need to be scanned
    if (numBucketsSelected == numBuckets) {
      None
    } else {
      Some(matchedBuckets)
    }
  }

  private def getExpressionBuckets(
      expr: Expression,
      bucketColumnName: String,
      numBuckets: Int): BitSet = {

    def getBucketNumber(attr: Attribute, v: Any): Int = {
      BucketingUtils.getBucketIdFromValue(attr, numBuckets, v)
    }

    def getBucketSetFromIterable(attr: Attribute, iter: Iterable[Any]): BitSet = {
      val matchedBuckets = new BitSet(numBuckets)
      iter
        .map(v => getBucketNumber(attr, v))
        .foreach(bucketNum => matchedBuckets.set(bucketNum))
      matchedBuckets
    }

    def getBucketSetFromValue(attr: Attribute, v: Any): BitSet = {
      val matchedBuckets = new BitSet(numBuckets)
      matchedBuckets.set(getBucketNumber(attr, v))
      matchedBuckets
    }

    expr match {
      case expressions.Equality(a: Attribute, Literal(v, _)) if a.name == bucketColumnName =>
        getBucketSetFromValue(a, v)
      case expressions.In(a: Attribute, list)
        if list.forall(_.isInstanceOf[Literal]) && a.name == bucketColumnName =>
        getBucketSetFromIterable(a, list.map(e => e.eval(EmptyRow)))
      case expressions.InSet(a: Attribute, hset)
        if hset.forall(_.isInstanceOf[Literal]) && a.name == bucketColumnName =>
        getBucketSetFromIterable(a, hset.map(e => expressions.Literal(e).eval(EmptyRow)))
      case expressions.IsNull(a: Attribute) if a.name == bucketColumnName =>
        getBucketSetFromValue(a, null)
      case expressions.And(left, right) =>
        getExpressionBuckets(left, bucketColumnName, numBuckets) &
          getExpressionBuckets(right, bucketColumnName, numBuckets)
      case expressions.Or(left, right) =>
        getExpressionBuckets(left, bucketColumnName, numBuckets) |
        getExpressionBuckets(right, bucketColumnName, numBuckets)
      case _ =>
        val matchedBuckets = new BitSet(numBuckets)
        matchedBuckets.setUntil(numBuckets)
        matchedBuckets
    }
  }

```

最主要的逻辑在 `FileSourceStrategy#getExpressionBuckets` 中, 可以看到 pruning 只支持 Equality, In, InSet, IsNull (And, Or 也支持, 只不过 and or 的 left/right 也必须是前面的类型), 其他情况是不支持 pruning 的, 直接返回所有 buckets.

```scala
  // Given bucketColumn, numBuckets and value, returns the corresponding bucketId
  def getBucketIdFromValue(bucketColumn: Attribute, numBuckets: Int, value: Any): Int = {
    val mutableInternalRow = new SpecificInternalRow(Seq(bucketColumn.dataType))
    mutableInternalRow.update(0, value)

    val bucketIdGenerator = UnsafeProjection.create(
      HashPartitioning(Seq(bucketColumn), numBuckets).partitionIdExpression :: Nil,
      bucketColumn :: Nil)
    bucketIdGenerator(mutableInternalRow).getInt(0)
  }
```

我们来看 Equality 情况的处理, 调用了 `getBucketIdFromValue`, 里面逻辑是使用 HashPartitioning 求出 filter 里 literal 的 hash 值. 直接把这个 hash 值写入bitset 就是 最后返回的 `bucketSet`

# 2. Exec 部分
Spark 的 Strategy 是用来构造 Exec 的, 在第一部分中, 我们得到了一个 `bucketSet`, 这个会作为一个参数传给 `FileSourceScanExec`. 如果有relation 上带了 bucket 的话, 会走到 `FileSourceScanExec#createBucketedReadRDD`, 在这里面会根据具体的文件名字去得到 bucketID `getBucketId`, 就是一个简单的正则, 那 `_` 后的值, 有了这个值, 只要找到 `bucketSet` 中与其匹配的值, 就选中了这个文件.

在
```scala
  private val bucketedFileName = """.*_(\d+)(?:\..*)?$""".r

  def getBucketId(fileName: String): Option[Int] = fileName match {
    case bucketedFileName(bucketId) => Some(bucketId.toInt)
    case other => None
  }

  /**
   * Create an RDD for bucketed reads.
   * The non-bucketed variant of this function is [[createNonBucketedReadRDD]].
   *
   * The algorithm is pretty simple: each RDD partition being returned should include all the files
   * with the same bucket id from all the given Hive partitions.
   *
   * @param bucketSpec the bucketing spec.
   * @param readFile a function to read each (part of a) file.
   * @param selectedPartitions Hive-style partition that are part of the read.
   * @param fsRelation [[HadoopFsRelation]] associated with the read.
   */
  private def createBucketedReadRDD(
      bucketSpec: BucketSpec,
      readFile: (PartitionedFile) => Iterator[InternalRow],
      selectedPartitions: Seq[PartitionDirectory],
      fsRelation: HadoopFsRelation): RDD[InternalRow] = {
    logInfo(s"Planning with ${bucketSpec.numBuckets} buckets")
    val filesGroupedToBuckets =
      selectedPartitions.flatMap { p =>
        p.files.map { f =>
          val hosts = getBlockHosts(getBlockLocations(f), 0, f.getLen)
          PartitionedFile(p.values, f.getPath.toUri.toString, 0, f.getLen, hosts)
        }
      }.groupBy { f =>
        BucketingUtils
          .getBucketId(new Path(f.filePath).getName)
          .getOrElse(sys.error(s"Invalid bucket file ${f.filePath}"))
      }

    val prunedFilesGroupedToBuckets = if (optionalBucketSet.isDefined) {
      val bucketSet = optionalBucketSet.get
      filesGroupedToBuckets.filter {
        f => bucketSet.get(f._1)
      }
    } else {
      filesGroupedToBuckets
    }

    val filePartitions = Seq.tabulate(bucketSpec.numBuckets) { bucketId =>
      FilePartition(bucketId, prunedFilesGroupedToBuckets.getOrElse(bucketId, Nil))
    }

    new FileScanRDD(fsRelation.sparkSession, readFile, filePartitions)
  }
```

