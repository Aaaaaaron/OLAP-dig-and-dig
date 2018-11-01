---
title: Spark Parquet Deep Dive
date: 2018-11-01 11:05:52
tags:
  - Spark
  - Parquet
  - BigData
---
# How Spark reads Parquet
先来快速过一下, 把重点留到 ParquetFileFormat. 

负责数据文件扫描的执行算子是FileSourceScanExec, 借一张图![Spark SQL 内核剖析](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/00014.jpeg)

FileSourceScanExec#inputRDD
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
