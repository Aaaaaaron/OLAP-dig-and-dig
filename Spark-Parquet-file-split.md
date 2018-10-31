---
title: Spark Parquet file split
date: 2018-10-22 20:14:43
top: true
tags:
  - Spark
  - BigData

---
### 问题
在实际使用 Spark + Parquet 的时候, 遇到了两个不解的地方:
1. 我们只有一个 Parquet 文件(小于 HDFS block size), 但是 Spark 在某个 stage 生成了4个 tasks 来处理.
2. 4个 tasks 中只有一个 task 处理了所有数据, 其他几个都没有在处理数据.

这两个问题牵涉到对于 Parquet, Spark 是如何来进行切分 partitions, 以及每个 partition 要处理哪部分数据的.

**先说结论**, Spark 中, Parquet 是 splitable 的, 代码见`ParquetFileFormat#isSplitable`. 那会不会把数据切碎? 答案是不会, 因为是以 Spark row group 为最小单位切分 Parquet 的, 这也会导致一些 partitions 会没有数据, 极端情况下, 如果只有一个 row group 的话, partitions 再多, 也只会一个 partition 有数据.

接下来开始我们的源码之旅:

### 处理流程
#### 1. 根据 Parquet 按文件大小切块生成 partitions:
 
在 `FileSourceScanExec#createNonBucketedReadRDD` 中, 如果文件是 splitable 的, 会按照 maxSplitBytes 把文件切分, 最后生成的数量, 就是 RDD partition 的数量, 这个解释了**不解1**, 代码如下:

```scala
val maxSplitBytes = Math.min(defaultMaxSplitBytes, Math.max(openCostInBytes, bytesPerCore))
logInfo(s"Planning scan with bin packing, max size: $maxSplitBytes bytes, " +
      s"open cost is considered as scanning $openCostInBytes bytes.")

val splitFiles = selectedPartitions.flatMap { partition =>
  partition.files.flatMap { file =>
    val blockLocations = getBlockLocations(file)
    if (fsRelation.fileFormat.isSplitable(
        fsRelation.sparkSession, fsRelation.options, file.getPath)) {
      (0L until file.getLen by maxSplitBytes).map { offset =>
        val remaining = file.getLen - offset
        val size = if (remaining > maxSplitBytes) maxSplitBytes else remaining
        val hosts = getBlockHosts(blockLocations, offset, size)
        PartitionedFile(
          partition.values, file.getPath.toUri.toString, offset, size, hosts)
      }
    } else {
      val hosts = getBlockHosts(blockLocations, 0, file.getLen)
      Seq(PartitionedFile(
        partition.values, file.getPath.toUri.toString, 0, file.getLen, hosts))
    }
  }
}.toArray.sortBy(_.length)(implicitly[Ordering[Long]].reverse)

val partitions = new ArrayBuffer[FilePartition]
val currentFiles = new ArrayBuffer[PartitionedFile]
var currentSize = 0L

/** Close the current partition and move to the next. */
def closePartition(): Unit = {
  if (currentFiles.nonEmpty) {
    val newPartition =
      FilePartition(
        partitions.size,
        currentFiles.toArray.toSeq) // Copy to a new Array.
    partitions += newPartition
  }
  currentFiles.clear()
  currentSize = 0
}

// Assign files to partitions using "First Fit Decreasing" (FFD)
splitFiles.foreach { file =>
  if (currentSize + file.length > maxSplitBytes) {
    closePartition()
  }
  // Add the given file to the current partition.
  currentSize += file.length + openCostInBytes
  currentFiles += file
}
closePartition()

new FileScanRDD(fsRelation.sparkSession, readFile, partitions)
```

如果是一个文件被分成多个 splits, 那么一个 file split 对应一个 partition. 如果是很多小文件这种的多个 file splits, 可能一个 partition 会有多个 file splits.

**一个 partition 对应一个 task.**

题外话, MR 引擎对于小文件的处理也有优化, 可以一个 map 处理多个 文件..

#### 2. 使用 ParquetInputSplit 构造 reader:

在 `ParquetFileFormat#buildReaderWithPartitionValues` 实现中, 会使用 split 来初始化 reader, 并且根据配置可以把 reader 分为否是 vectorized 的, 关于 vectorized, 我会在日后再开一篇博文来介绍:

- `vectorizedReader.initialize(split, hadoopAttemptContext)`
- `reader.initialize(split, hadoopAttemptContext)`

关于 步骤2 在画外中还有更详细的代码, 但与本文的主流程关系不大, 这里先不表.

#### 3. 划分 Parquet 的 row groups 到不同的Spark partitions 中去

在 步骤1 中根据文件大小均分了一些 partitions, 但不是所有这些 partitions 最后都会有数据. 

接回 步骤2 中的 init, 在 `SpecificParquetRecordReaderBase#initialize` 中, 会在 `readFooter` 的时候传入一个 `RangeMetadataFilter`, 这个 filter 的range 是根据你的 split 的边界来的, 最后会用这个 range 来划定 row group 的归属:

```java
public void initialize(InputSplit inputSplit, TaskAttemptContext taskAttemptContext)
    throws IOException, InterruptedException {
    ...
    footer = readFooter(configuration, file, range(inputSplit.getStart(), inputSplit.getEnd()));
    ...
}
```

Parquet 的`ParquetFileReader#readFooter`方法会用到`ParquetMetadataConverter#converter.readParquetMetadata(f, filter);`, 这个`readParquetMetadata` 使用了一个访问者模式, 而其中对于`RangeMetadataFilter`的处理是:

```java
@Override
public FileMetaData visit(RangeMetadataFilter filter) throws IOException {
  return filterFileMetaDataByMidpoint(readFileMetaData(from), filter);
}
```

终于到了最关键的切分的地方, 最关键的就是这一段, 谁拥有这个 row group的中点, 谁就可以处理这个 row group. 

现在假设我们有一个40m 的文件, 只有一个 row group, 10m 一分, 那么将会有4个 partitions, 但是只有一个 partition 会占有这个 row group 的中点, 所以也只有这一个 partition 会有数据.

```java
long midPoint = startIndex + totalSize / 2;
if (filter.contains(midPoint)) {
  newRowGroups.add(rowGroup);
}
```

完整代码如下:

```java
static FileMetaData filterFileMetaDataByMidpoint(FileMetaData metaData, RangeMetadataFilter filter) {
  List<RowGroup> rowGroups = metaData.getRow_groups();
  List<RowGroup> newRowGroups = new ArrayList<RowGroup>();
  for (RowGroup rowGroup : rowGroups) {
    long totalSize = 0;
    long startIndex = getOffset(rowGroup.getColumns().get(0));
    for (ColumnChunk col : rowGroup.getColumns()) {
      totalSize += col.getMeta_data().getTotal_compressed_size();
    }
    long midPoint = startIndex + totalSize / 2;
    if (filter.contains(midPoint)) {
      newRowGroups.add(rowGroup);
    }
  }
  metaData.setRow_groups(newRowGroups);
  return metaData;
}
```

#### 画外:
步骤2 中的代码其实是 spark 正儿八经如何读文件的代码, 最后返回一个`FileScanRDD`, 也很值得顺路看一下, 完整代码如下:
```scala
    (file: PartitionedFile) => {
      assert(file.partitionValues.numFields == partitionSchema.size)

      val fileSplit =
        new FileSplit(new Path(new URI(file.filePath)), file.start, file.length, Array.empty)

      val split =
        new org.apache.parquet.hadoop.ParquetInputSplit(
          fileSplit.getPath,
          fileSplit.getStart,
          fileSplit.getStart + fileSplit.getLength,
          fileSplit.getLength,
          fileSplit.getLocations,
          null)

      val attemptId = new TaskAttemptID(new TaskID(new JobID(), TaskType.MAP, 0), 0)
      val hadoopAttemptContext =
        new TaskAttemptContextImpl(broadcastedHadoopConf.value.value, attemptId)

      // Try to push down filters when filter push-down is enabled.
      // Notice: This push-down is RowGroups level, not individual records.
      if (pushed.isDefined) {
        ParquetInputFormat.setFilterPredicate(hadoopAttemptContext.getConfiguration, pushed.get)
      }
      val parquetReader = if (enableVectorizedReader) {
        val vectorizedReader = new VectorizedParquetRecordReader()
        vectorizedReader.initialize(split, hadoopAttemptContext)
        logDebug(s"Appending $partitionSchema ${file.partitionValues}")
        vectorizedReader.initBatch(partitionSchema, file.partitionValues)
        if (returningBatch) {
          vectorizedReader.enableReturningBatches()
        }
        vectorizedReader
      } else {
        logDebug(s"Falling back to parquet-mr")
        // ParquetRecordReader returns UnsafeRow
        val reader = pushed match {
          case Some(filter) =>
            new ParquetRecordReader[UnsafeRow](
              new ParquetReadSupport,
              FilterCompat.get(filter, null))
          case _ =>
            new ParquetRecordReader[UnsafeRow](new ParquetReadSupport)
        }
        reader.initialize(split, hadoopAttemptContext)
        reader
      }

      val iter = new RecordReaderIterator(parquetReader)
      Option(TaskContext.get()).foreach(_.addTaskCompletionListener(_ => iter.close()))

      // UnsafeRowParquetRecordReader appends the columns internally to avoid another copy.
      if (parquetReader.isInstanceOf[VectorizedParquetRecordReader] &&
          enableVectorizedReader) {
        iter.asInstanceOf[Iterator[InternalRow]]
      } else {
        val fullSchema = requiredSchema.toAttributes ++ partitionSchema.toAttributes
        val joinedRow = new JoinedRow()
        val appendPartitionColumns = GenerateUnsafeProjection.generate(fullSchema, fullSchema)

        // This is a horrible erasure hack...  if we type the iterator above, then it actually check
        // the type in next() and we get a class cast exception.  If we make that function return
        // Object, then we can defer the cast until later!
        if (partitionSchema.length == 0) {
          // There is no partition columns
          iter.asInstanceOf[Iterator[InternalRow]]
        } else {
          iter.asInstanceOf[Iterator[InternalRow]]
            .map(d => appendPartitionColumns(joinedRow(d, file.partitionValues)))
        }
      }
    }

```

这个返回的`(PartitionedFile) => Iterator[InternalRow] `, 是在`FileSourceScanExec#inputRDD`用的
```scala
private lazy val inputRDD: RDD[InternalRow] = {
  val readFile: (PartitionedFile) => Iterator[InternalRow] =
    relation.fileFormat.buildReaderWithPartitionValues(
      sparkSession = relation.sparkSession,
      dataSchema = relation.dataSchema,
      partitionSchema = relation.partitionSchema,
      requiredSchema = requiredSchema,
      filters = pushedDownFilters,
      options = relation.options,
      hadoopConf = relation.sparkSession.sessionState.newHadoopConfWithOptions(relation.options))

  relation.bucketSpec match {
    case Some(bucketing) if relation.sparkSession.sessionState.conf.bucketingEnabled =>
      createBucketedReadRDD(bucketing, readFile, selectedPartitions, relation)
    case _ =>
      createNonBucketedReadRDD(readFile, selectedPartitions, relation)
  }
}
```

FileScanRDD
```scala
class FileScanRDD(
    @transient private val sparkSession: SparkSession,
    readFunction: (PartitionedFile) => Iterator[InternalRow],
    @transient val filePartitions: Seq[FilePartition])
  extends RDD[InternalRow](sparkSession.sparkContext, Nil) {

  override def compute(split: RDDPartition, context: TaskContext): Iterator[InternalRow] = {

    private[this] val files = split.asInstanceOf[FilePartition].files.toIterator
    private[this] var currentFile: PartitionedFile = null // 根据 currentFile = files.next() 来的, 具体实现我就不贴了 有兴趣的可以自己看下.
    ...
    readFunction(currentFile)
    ...
  }
}
```

### 结论
提升一个 Parquet 中的 row group 中的行数阈值, 籍此提示 Spark 并行度.