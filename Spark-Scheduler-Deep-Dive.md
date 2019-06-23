---
title: Spark Scheduler Deep Dive
date: 2019-01-26 17:26:14
tags:
---
# RDD
scheduler 的调度主体.

```scala
 * Internally, each RDD is characterized by five main properties:
 *
 *  - A list of partitions
 *  - A function for computing each split
 *  - A list of dependencies on other RDDs
 *  - Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
 *  - Optionally, a list of preferred locations to compute each split on (e.g. block locations for
 *    an HDFS file)

  /**
   * :: DeveloperApi ::
   * Implemented by subclasses to compute a given partition.
   */
  @DeveloperApi
  def compute(split: Partition, context: TaskContext): Iterator[T]

  /**
   * Implemented by subclasses to return the set of partitions in this RDD. This method will only
   * be called once, so it is safe to implement a time-consuming computation in it.
   *
   * The partitions in this array must satisfy the following property:
   *   `rdd.partitions.zipWithIndex.forall { case (partition, index) => partition.index == index }`
   */
  protected def getPartitions: Array[Partition]

  /**
   * Implemented by subclasses to return how this RDD depends on parent RDDs. This method will only
   * be called once, so it is safe to implement a time-consuming computation in it.
   */
  protected def getDependencies: Seq[Dependency[_]] = deps

  /**
   * Optionally overridden by subclasses to specify placement preferences.
   */
  protected def getPreferredLocations(split: Partition): Seq[String] = Nil
```

## 依赖
DAG中的各个RDD之间存在着依赖关系。换⾔之，正是RDD之间的依 赖关系构建了由RDD所组成的DAG。Spark使⽤Dependency来表⽰RDD之 间的依赖关系.

```scala
abstract class Dependency[T] extends Serializable {
  def rdd: RDD[T]
}
```

### 窄依赖
RDD与上游RDD的分区是⼀对⼀的关系.

```scala
abstract class NarrowDependency[T](_rdd: RDD[T]) extends Dependency[T] {
  /**
   * Get the parent partitions for a child partition.
   * @param partitionId a partition of the child RDD
   * @return the partitions of the parent RDD that the child partition depends upon
   */
  def getParents(partitionId: Int): Seq[Int]

  override def rdd: RDD[T] = _rdd
}
```

有两个实现类:OneToOneDependency/RangeDependency

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190127121445.png)

```scala
  override def getParents(partitionId: Int): List[Int] = List(partitionId)
```

可以看到, OneToOneDependency 子 RDD 分区与依赖的父 RDD 分区相同.

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190127121500.png)
```scala
  override def getParents(partitionId: Int): List[Int] = {
    if (partitionId >= outStart && partitionId < outStart + length) {
      List(partitionId - outStart + inStart)
    } else {
      Nil
    }
  }
```

### Shuffle 依赖
RDD与上游RDD的分区如果不是一对一的关系，或者RDD的分区依赖于上游RDD的多个分区，那么这种依赖关系就叫做Shuffle依赖（ShuffleDependency）
`  override def rdd: RDD[Product2[K, V]] = _rdd.asInstanceOf[RDD[Product2[K, V]]]` 可以看到这边返回的 RDD 是RDD[Product2[K, V].

## Partitioner
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190127153603.png)

```scala
/**
 * An object that defines how the elements in a key-value pair RDD are partitioned by key.
 * Maps each key to a partition ID, from 0 to `numPartitions - 1`.
 *
 * Note that, partitioner must be deterministic, i.e. it must return the same partition id given
 * the same partition key.
 */
abstract class Partitioner extends Serializable {
  def numPartitions: Int
  def getPartition(key: Any): Int
}
```

参照 HashPartitioner
```scala
  def getPartition(key: Any): Int = key match {
    case null => 0
    case _ => Utils.nonNegativeMod(key.hashCode, numPartitions)
  }
```

一个上下游 RDD 的 partitions 数量都是定的().
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190127154456.png)

## Stage
```scala
/**
 * A stage is a set of parallel tasks all computing the same function that need to run as part
 * of a Spark job, where all the tasks have the same shuffle dependencies. Each DAG of tasks run
 * by the scheduler is split up into stages at the boundaries where shuffle occurs, and then the
 * DAGScheduler runs these stages in topological order.
 *
 * Each Stage can either be a shuffle map stage, in which case its tasks' results are input for
 * other stage(s), or a result stage, in which case its tasks directly compute a Spark action
 * (e.g. count(), save(), etc) by running a function on an RDD. For shuffle map stages, we also
 * track the nodes that each output partition is on.
 *
 * Each Stage also has a firstJobId, identifying the job that first submitted the stage.  When FIFO
 * scheduling is used, this allows Stages from earlier jobs to be computed first or recovered
 * faster on failure.
 *
 * Finally, a single stage can be re-executed in multiple attempts due to fault recovery. In that
 * case, the Stage object will track multiple StageInfo objects to pass to listeners or the web UI.
 * The latest one will be accessible through latestInfo.
 *
 * @param id Unique stage ID
 * @param rdd RDD that this stage runs on: for a shuffle map stage, it's the RDD we run map tasks
 *   on, while for a result stage, it's the target RDD that we ran an action on
 * @param numTasks Total number of tasks in stage; result stages in particular may not need to
 *   compute all partitions, e.g. for first(), lookup(), and take().
 * @param parents List of stages that this stage depends on (through shuffle dependencies).
 * @param firstJobId ID of the first job this stage was part of, for FIFO scheduling.
 * @param callSite Location in the user program associated with this stage: either where the target
 *   RDD was created, for a shuffle map stage, or where the action for a result stage was called.
 */
private[scheduler] abstract class Stage(
    val id: Int,
    val rdd: RDD[_],
    val numTasks: Int,
    val parents: List[Stage],
    val firstJobId: Int,
    val callSite: CallSite)
  extends Logging {

  val numPartitions = rdd.partitions.length

  /** Set of jobs that this stage belongs to. */
  val jobIds = new HashSet[Int]

  /** The ID to use for the next new attempt for this stage. */
  private var nextAttemptId: Int = 0

  val name: String = callSite.shortForm
  val details: String = callSite.longForm

  /**
   * Pointer to the [[StageInfo]] object for the most recent attempt. This needs to be initialized
   * here, before any attempts have actually been created, because the DAGScheduler uses this
   * StageInfo to tell SparkListeners when a job starts (which happens before any stage attempts
   * have been created).
   */
  private var _latestInfo: StageInfo = StageInfo.fromStage(this, nextAttemptId)

  /**
   * Set of stage attempt IDs that have failed. We keep track of these failures in order to avoid
   * endless retries if a stage keeps failing.
   * We keep track of each attempt ID that has failed to avoid recording duplicate failures if
   * multiple tasks from the same stage attempt fail (SPARK-5945).
   */
  val failedAttemptIds = new HashSet[Int]

  private[scheduler] def clearFailures() : Unit = {
    failedAttemptIds.clear()
  }

  /** Creates a new attempt for this stage by creating a new StageInfo with a new attempt ID. */
  def makeNewStageAttempt(
      numPartitionsToCompute: Int,
      taskLocalityPreferences: Seq[Seq[TaskLocation]] = Seq.empty): Unit = {
    val metrics = new TaskMetrics
    metrics.register(rdd.sparkContext)
    _latestInfo = StageInfo.fromStage(
      this, nextAttemptId, Some(numPartitionsToCompute), metrics, taskLocalityPreferences)
    nextAttemptId += 1
  }

  /** Returns the StageInfo for the most recent attempt for this stage. */
  def latestInfo: StageInfo = _latestInfo

  override final def hashCode(): Int = id

  override final def equals(other: Any): Boolean = other match {
    case stage: Stage => stage != null && stage.id == id
    case _ => false
  }

  /** Returns the sequence of partition ids that are missing (i.e. needs to be computed). */
  def findMissingPartitions(): Seq[Int]
}
```

makeNewStageAttempt的执行步骤如下。

- 调用StageInfo的fromStage方法（见代码清单7-19）创建新的StageInfo。
- 增加nextAttemptId。

### ResultStage
ResultStage可以使用指定的函数对RDD中的分区进行计算并得出最终结果。ResultStage是最后执行的Stage，此阶段主要进行作业的收尾工作（例如，对各个分区的数据收拢、打印到控制台或写入到HDFS）

可以看到多了 func, 即对 RDD 分区进行计算的函数.

在 DAGScheduler#handleJobSubmitted 中 调用了 `setActiveJob`.

```scala
/**
 * ResultStages apply a function on some partitions of an RDD to compute the result of an action.
 * The ResultStage object captures the function to execute, `func`, which will be applied to each
 * partition, and the set of partition IDs, `partitions`. Some stages may not run on all partitions
 * of the RDD, for actions like first() and lookup().
 */
private[spark] class ResultStage(
    id: Int,
    rdd: RDD[_],
    val func: (TaskContext, Iterator[_]) => _,
    val partitions: Array[Int],
    parents: List[Stage],
    firstJobId: Int,
    callSite: CallSite)
  extends Stage(id, rdd, partitions.length, parents, firstJobId, callSite) {

  /**
   * The active job for this result stage. Will be empty if the job has already finished
   * (e.g., because the job was cancelled).
   */
  private[this] var _activeJob: Option[ActiveJob] = None

  def activeJob: Option[ActiveJob] = _activeJob

  def setActiveJob(job: ActiveJob): Unit = {
    _activeJob = Option(job)
  }

  def removeActiveJob(): Unit = {
    _activeJob = None
  }

  /**
   * Returns the sequence of partition ids that are missing (i.e. needs to be computed).
   *
   * This can only be called when there is an active job.
   */
  override def findMissingPartitions(): Seq[Int] = {
    val job = activeJob.get
    (0 until job.numPartitions).filter(id => !job.finished(id))
  }

  override def toString: String = "ResultStage " + id
}

```

### ShuffleMapStage

生成 shuffle 的数据. 将对 shuffle 的数据映射到下游 stage 的各个分区中.

# DAGScheduler
DAGScheduler实现了面向DAG的高层次调度，即将DAG中的各个RDD划分到不同的Stage。DAGScheduler可以通过计算将DAG中的一系列RDD划分到不同的Stage，然后构建这些Stage之间的父子关系，最后将每个Stage按照Partition切分为多个Task，并以Task集合（即TaskSet）的形式提交给底层的TaskScheduler。

所有的组件都通过向DAGScheduler投递DAGSchedulerEvent来使用DAGScheduler。DAGScheduler内部的DAGSchedulerEventProcessLoop将处理这些DAGScheduler-Event，并调用DAGScheduler的不同方法。JobListener用于对作业中每个Task执行成功或失败进行监听，JobWaiter实现了JobListener并最终确定作业的成功与失败。在正式介绍DAGScheduler之前，我们先来看看DAGScheduler所依赖的组件DAGSchedulerEventProcessLoop、Job Listener及ActiveJob的实现。