---
title: Spark Parquet Deep Dive
date: 2018-11-01 11:05:52
tags:
  - Spark
  - Parquet
  - BigData
top: true

---
## 读取流程
### WholeStageCodegenExec
负责数据文件扫描的执行算子是 FileSourceScanExec, 借一张图![Spark SQL 内核剖析](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/00014.jpeg)

先到 WholeStageCodegenExec 的doExecute, 这里要下面会提到的有两个地方, 一个是 `inputRDDs`, 另一个是 `buffer.hasNext` :
```scala
  override def doExecute(): RDD[InternalRow] = {
    ...
    val (ctx, cleanedSource) = doCodeGen()
    ...
    val rdds = child.asInstanceOf[CodegenSupport].inputRDDs()
    assert(rdds.size <= 2, "Up to two input RDDs can be supported")
    if (rdds.length == 1) {
      rdds.head.mapPartitionsWithIndex { (index, iter) =>
        val (clazz, _) = CodeGenerator.compile(cleanedSource)
        val buffer = clazz.generate(references).asInstanceOf[BufferedRowIterator]
        buffer.init(index, Array(iter))
        new Iterator[InternalRow] {
          override def hasNext: Boolean = {
            val v = buffer.hasNext
            if (!v) durationMs += buffer.durationMs()
            v
          }
          override def next: InternalRow = buffer.next()
        }
      }
    } 
  }
```

这里的其中`child.asInstanceOf[CodegenSupport].inputRDDs()` 处于所有的 exec 的最头部, 会沿着 exec 树一层层调用下去最后会走到:`FileSourceScanExec#inputRDD`, 如图:

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20181118165922.png)

### FileSourceScanExec
inputRDD 首先会调用具体 `FileFormat` 实现类的 `buildReaderWithPartitionValues` ,对于这里是 `ParquetFileFormat`, build 出来一个 `readFunction : ((PartitionedFile) => Iterator[InternalRow])`, 顾名思义后面会用这个 func 来读取文件.

下一步的 `createBucketedReadRDD` 我们之前的博客分析过( [Spark-Parquet-file-split](https://aaaaaaron.github.io/2018/10/22/Spark-Parquet-file-split) ), 主要是用来切分 partitions, 并且返回一个 FileScanRDD.

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

### FileScanRDD
比较重要的是 `currentIterator:Iterator[Object]` 这个东西, `compute` 方法吐出去的就是这个 iter. 可以看到这个 iter 通过之前 FileFormat 里生成的 readFunction 来生成的.

```scala
private def readCurrentFile(): Iterator[InternalRow] = {
  try {
    readFunction(currentFile)
  } catch {
    case e: FileNotFoundException =>
      throw new FileNotFoundException(
        e.getMessage + "\n" +
          "It is possible the underlying files have been updated. " +
          "You can explicitly invalidate the cache in Spark by " +
          "running 'REFRESH TABLE tableName' command in SQL or " +
          "by recreating the Dataset/DataFrame involved.")
  }
}
```

### todo:
在 driver

  def collect(): Array[T] = withScope {
    val results = sc.runJob(this, (iter: Iterator[T]) => iter.toArray)
    Array.concat(results: _*)
  }

org.apache.spark.SparkContext#runJob#    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)

org.apache.spark.scheduler.DAGScheduler#runJob
    
pending 在 ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)

```scala
  def runJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): Unit = {
    val start = System.nanoTime
    val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
    ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
    waiter.completionFuture.value.get match {
      case scala.util.Success(_) =>
        logInfo("Job %d finished: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
      case scala.util.Failure(exception) =>
        logInfo("Job %d failed: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
        // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
        val callerStackTrace = Thread.currentThread().getStackTrace.tail
        exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
        throw exception
    }
  }
```
===
在 executor

org.apache.spark.executor.Executor.TaskRunner#run
```scala
        val value = Utils.tryWithSafeFinally {
          val res = task.run(
            taskAttemptId = taskId,
            attemptNumber = taskDescription.attemptNumber,
            metricsSystem = env.metricsSystem)
          threwException = false
          res
        }
```

org.apache.spark.scheduler.Task#run
```scala
      runTask(context)
```



在 `ShuffleMapTask#runTask` 里, 
```
var writer: ShuffleWriter[Any, Any] = null
try {
  val manager = SparkEnv.get.shuffleManager
  writer = manager.getWriter[Any, Any](dep.shuffleHandle, partitionId, context)
  writer.write(rdd.iterator(partition, context).asInstanceOf[Iterator[_ <: Product2[Any, Any]]])
  writer.stop(success = true).get
 } 
```

注意这个 rdd.iterator 
codegen

scan_mutableStateArray_0[0].hasNext() 调用到了 rdd 的 iter

```scala
    private void scan_nextBatch_0() throws java.io.IOException {
        long getBatchStart = System.nanoTime();
        if (scan_mutableStateArray_0[0].hasNext()) {
            scan_mutableStateArray_1[0] = (org.apache.spark.sql.vectorized.ColumnarBatch)scan_mutableStateArray_0[0].next();
            ((org.apache.spark.sql.execution.metric.SQLMetric) references[0] /* numOutputRows */).add(scan_mutableStateArray_1[0].numRows());
            scan_batchIdx_0 = 0;
            scan_mutableStateArray_2[0] = (org.apache.spark.sql.execution.vectorized.OnHeapColumnVector) scan_mutableStateArray_1[0].column(0);

        }
        scan_scanTime_0 += System.nanoTime() - getBatchStart;
    }
```

#### RecordReaderIterator
上面的 currentIterator 其实是一个 RecordReaderIterator, 里面包装了 RecordReader. 对于 Parquet 的实现是 `VectorizedParquetRecordReader`.

##### initialize
      blocks = filterRowGroups(filter, footer.getBlocks(), fileSchema);
      this.reader = new ParquetFileReader(
        configuration, footer.getFileMetaData(), file, blocks, requestedSchema.getColumns())
##### nextKeyValue

##### getCurrentValue

##### checkEndOfRowGroup


