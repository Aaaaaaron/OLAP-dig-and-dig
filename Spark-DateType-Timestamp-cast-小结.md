---
title: Spark DateType/Timestamp cast 小结
date: 2018-07-19 16:47:39
top: true
tags:
  - Spark
  - BigData
---
# 前言
在平时的 Spark 处理中常常会有把一个如 `2012-12-12` 这样的 date 类型转换成一个 long 的 Unix time 然后进行计算的需求.下面是一段示例代码:

```scala
val schema = StructType(
  Array(
    StructField("id", IntegerType, nullable = true),
    StructField("birth", DateType, nullable = true),
    StructField("time", TimestampType, nullable = true)
  ))

val data = Seq(
  Row(1, Date.valueOf("2012-12-12"), Timestamp.valueOf("2016-09-30 03:03:00")),
  Row(2, Date.valueOf("2016-12-14"), Timestamp.valueOf("2016-12-14 03:03:00")))

val df = spark.createDataFrame(spark.sparkContext.parallelize(data),schema)
```

# 问题 & 解决
首先很直观的是直接把DateType cast 成 LongType, 如下:

`df.select(df.col("birth").cast(LongType))`

但是这样出来都是 null, 这是为什么? 答案就在 `org.apache.spark.sql.catalyst.expressions.Cast` 中, 先看 canCast 方法, 可以看到 DateType 其实是可以转成 NumericType 的, 然后再看下面castToLong的方法, 可以看到`case DateType => buildCast[Int](_, d => null)`居然直接是个 null, 看提交记录其实这边有过反复, 然后为了和 hive 统一, 所以返回最后还是返回 null 了.

虽然 DateType 不能直接 castToLong, 但是TimestampType可以, 所以这里的解决方案就是先把 DateType cast 成 TimestampType. 但是这里又会有一个非常坑爹的问题: **时区问题**.

首先明确一个问题, 就是这个放到了 spark 中的 2012-12-12 到底 UTC 还是我们当前时区? 答案是如果没有经过特殊配置, 这个2012-12-12代表的是 **当前时区的 2012-12-12 00:00:00.**, 对应 UTC 其实是: 2012-12-11 16:00:00, 少了8小时. 这里还顺便说明了Spark 入库 Date 数据的时候是带着时区的.

然后再看DateType cast toTimestampType  的代码, 可以看到`buildCast[Int](_, d => DateTimeUtils.daysToMillis(d, timeZone) * 1000)`, 这里是带着时区的, 但是 Spark SQL 默认会用当前机器的时区. 但是大家一般底层数据比如这个2016-09-30, 都是代表的 UTC 时间, 在用 Spark 处理数据的时候, 这个时间还是 UTC 时间, 只有通过 JDBC 出去的时间才会变成带目标时区的结果. 经过摸索, 这里有两种解决方案:

1. 配置 Spark 的默认时区`config("spark.sql.session.timeZone", "UTC")`, 最直观. 这样直接写 `df.select(df.col("birth").cast(TimestampType).cast(LongType))` 就可以了.
2. 不配置 conf, 正面刚: `df.select(from_utc_timestamp(to_utc_timestamp(df.col("birth"), TimeZone.getTimeZone("UTC").getID), TimeZone.getDefault.getID).cast(LongType))`, 可以看到各种 cast, 这是区别:

- 没有配置 UTC: `from_utc_timestamp(to_utc_timestamp(lit("2012-12-11 16:00:00"), TimeZone.getTimeZone("UTC").getID), TimeZone.getDefault.getID)`
- 配置了 UTC: `from_utc_timestamp(to_utc_timestamp(lit("2012-12-12 00:00:00"), TimeZone.getTimeZone("UTC").getID), TimeZone.getDefault.getID)` 多了8小时

```scala
  /**
   * Returns true iff we can cast `from` type to `to` type.
   */
  def canCast(from: DataType, to: DataType): Boolean = (from, to) match {
    case (fromType, toType) if fromType == toType => true

    case (NullType, _) => true

    case (_, StringType) => true

    case (StringType, BinaryType) => true

    case (StringType, BooleanType) => true
    case (DateType, BooleanType) => true
    case (TimestampType, BooleanType) => true
    case (_: NumericType, BooleanType) => true

    case (StringType, TimestampType) => true
    case (BooleanType, TimestampType) => true
    case (DateType, TimestampType) => true
    case (_: NumericType, TimestampType) => true

    case (StringType, DateType) => true
    case (TimestampType, DateType) => true

    case (StringType, CalendarIntervalType) => true

    case (StringType, _: NumericType) => true
    case (BooleanType, _: NumericType) => true
    case (DateType, _: NumericType) => true
    case (TimestampType, _: NumericType) => true
    case (_: NumericType, _: NumericType) => true
    ...
  }

```

```scala
  private[this] def castToLong(from: DataType): Any => Any = from match {
    case StringType =>
      val result = new LongWrapper()
      buildCast[UTF8String](_, s => if (s.toLong(result)) result.value else null)
    case BooleanType =>
      buildCast[Boolean](_, b => if (b) 1L else 0L)
    case DateType =>
      buildCast[Int](_, d => null)
    case TimestampType =>
      buildCast[Long](_, t => timestampToLong(t))
    case x: NumericType =>
      b => x.numeric.asInstanceOf[Numeric[Any]].toLong(b)
  }
```

```scala
  // TimestampConverter
  private[this] def castToTimestamp(from: DataType): Any => Any = from match {
    ...
    case DateType =>
      buildCast[Int](_, d => DateTimeUtils.daysToMillis(d, timeZone) * 1000)
    // TimestampWritable.decimalToTimestamp
    ...
  }
```

```scala
  /**
   * Given a timestamp, which corresponds to a certain time of day in the given timezone, returns
   * another timestamp that corresponds to the same time of day in UTC.
   * @group datetime_funcs
   * @since 1.5.0
   */
  def to_utc_timestamp(ts: Column, tz: String): Column = withExpr {
    ToUTCTimestamp(ts.expr, Literal(tz))
  }

  /**
   * Given a timestamp, which corresponds to a certain time of day in UTC, returns another timestamp
   * that corresponds to the same time of day in the given timezone.
   * @group datetime_funcs
   * @since 1.5.0
   */
  def from_utc_timestamp(ts: Column, tz: String): Column = withExpr {
    FromUTCTimestamp(ts.expr, Literal(tz))
  }
```

# Deep dive
配置源码解读:
```scala
  val SESSION_LOCAL_TIMEZONE = buildConf("spark.sql.session.timeZone").stringConf.createWithDefaultFunction(() => TimeZone.getDefault.getID)
```

`def sessionLocalTimeZone: String = getConf(SQLConf.SESSION_LOCAL_TIMEZONE)`

```scala
/**
 * Replace [[TimeZoneAwareExpression]] without timezone id by its copy with session local
 * time zone.
 */
case class ResolveTimeZone(conf: SQLConf) extends Rule[LogicalPlan] {
  private val transformTimeZoneExprs: PartialFunction[Expression, Expression] = {
    case e: TimeZoneAwareExpression if e.timeZoneId.isEmpty =>
      e.withTimeZone(conf.sessionLocalTimeZone)
    // Casts could be added in the subquery plan through the rule TypeCoercion while coercing
    // the types between the value expression and list query expression of IN expression.
    // We need to subject the subquery plan through ResolveTimeZone again to setup timezone
    // information for time zone aware expressions.
    case e: ListQuery => e.withNewPlan(apply(e.plan))
  }

  override def apply(plan: LogicalPlan): LogicalPlan =
    plan.transformAllExpressions(transformTimeZoneExprs)

  def resolveTimeZones(e: Expression): Expression = e.transform(transformTimeZoneExprs)
}

/**
 * Mix-in trait for constructing valid [[Cast]] expressions.
 */
trait CastSupport {
  /**
   * Configuration used to create a valid cast expression.
   */
  def conf: SQLConf

  /**
   * Create a Cast expression with the session local time zone.
   */
  def cast(child: Expression, dataType: DataType): Cast = {
    Cast(child, dataType, Option(conf.sessionLocalTimeZone))
  }
}
```

org.apache.spark.sql.catalyst.analysis.Analyzer#batches 可以看到有`ResolveTimeZone`
```scala
  lazy val batches: Seq[Batch] = Seq(

    Batch("Resolution", fixedPoint,
      ResolveTableValuedFunctions ::
      ResolveRelations ::
      ResolveReferences ::
      ...
      ResolveTimeZone(conf) ::
      ResolvedUuidExpressions ::
      TypeCoercion.typeCoercionRules(conf) ++
      extendedResolutionRules : _*),
    Batch("Post-Hoc Resolution", Once, postHocResolutionRules: _*),
    Batch("View", Once,
      AliasViewChild(conf)),
    Batch("Nondeterministic", Once,
      PullOutNondeterministic),
    Batch("UDF", Once,
      HandleNullInputsForUDF),
    Batch("FixNullability", Once,
      FixNullability),
    Batch("Subquery", Once,
      UpdateOuterReferences),
    Batch("Cleanup", fixedPoint,
      CleanupAliases)
  )
```

## Test Example
#### 对于时区理解
在不同的时区下 sql.Timestamp 对象的表现:

这里是 GMT+8:
```
Timestamp "2014-06-24 07:22:15.0"
    - fastTime = 1403565735000
    - "2014-06-24T07:22:15.000+0700"
```

如果是 GMT+7, 会显示如下,可以看到是同一个毫秒数
```scala
Timestamp "2014-06-24 06:22:15.0"
    - fastTime = 1403565735000
    - "2014-06-24T06:22:15.000+0700"
```

```scala
  test("ColumnBatch") {
    val schema = StructType(
      Array(
        StructField("id", IntegerType, nullable = true),
        StructField("birth", DateType, nullable = true),
        StructField("time", TimestampType, nullable = true)
      ))

    val columnarBatch = ColumnarBatch.allocate(schema, MemoryMode.ON_HEAP, 1024)
    val c0 = columnarBatch.column(0)
    val c1 = columnarBatch.column(1)
    val c2 = columnarBatch.column(2)

    c0.putInt(0, 0)
    // 1355241600, /3600/24 s to days
    c1.putInt(0, 1355241600 / 3600 / 24)
    // microsecond
    c2.putLong(0, 1355285532000000L)

    val internal0 = columnarBatch.getRow(0)

    //a way converting internal row to unsafe row.
    //val convert = UnsafeProjection.create(schema)
    //val internal = convert.apply(internal0)

    val enc = RowEncoder.apply(schema).resolveAndBind()
    val row = enc.fromRow(internal0)
    val df = spark.createDataFrame(Lists.newArrayList(row), schema)

    TimeZone.setDefault(TimeZone.getTimeZone("UTC"))
    val tsStr0 = df.select(col("time")).head().getTimestamp(0).toString
    val ts0 = df.select(col("time").cast(LongType)).head().getLong(0)

    TimeZone.setDefault(TimeZone.getTimeZone("GMT+8"))
    val tsStr1 = df.select(col("time")).head().getTimestamp(0).toString
    val ts1 = df.select(col("time").cast(LongType)).head().getLong(0)

    assert(true, "2012-12-12 04:12:12.0".equals(tsStr0))
    assert(true, "2012-12-12 12:12:12.0".equals(tsStr1))
    // to long 之后毫秒数都是一样的
    assert(true, ts0 == ts1)
  }
```

# 番外 : ImplicitCastInputTypes
我们自己定义了一个Expr, 要求接受两个 input 为 DateType 的参数.
```scala
case class MockExpr(d0: Expression, d1: Expression)
  extends BinaryExpression with ImplicitCastInputTypes {

  override def left: Expression = d0

  override def right: Expression = d1

  override def inputTypes: Seq[AbstractDataType] = Seq(DateType, DateType)

  override def dataType: DataType = IntegerType

  override def nullSafeEval(date0: Any, date1: Any): Any = {
    ...
  }
}
```

假设我们有如下调用, 请问这个调用符合预期吗? 结论是符合的, 因为有`ImplicitCastInputTypes`.
```scala
lit("2012-11-12 12:12:12.0").cast(TimestampType)
lit("2012-12-12 12:12:12.0").cast(TimestampType)
Column(MockExpr(tsc1.expr, tsc2.expr))
```

**org.apache.spark.sql.catalyst.analysis.TypeCoercion.ImplicitTypeCasts**

```scala
case e: ImplicitCastInputTypes if e.inputTypes.nonEmpty =>
val children: Seq[Expression] = e.children.zip(e.inputTypes).map { case (in, expected) =>
  // If we cannot do the implicit cast, just use the original input.
  implicitCast(in, expected).getOrElse(in)
}
e.withNewChildren(children)

def implicitCast(e: Expression, expectedType: AbstractDataType): Option[Expression] = {
  implicitCast(e.dataType, expectedType).map { dt =>
    if (dt == e.dataType) e else Cast(e, dt)
  }
}
```

**org.apache.spark.sql.catalyst.expressions.Cast#castToDate #DateConverter**
```scala
private[this] def castToDate(from: DataType): Any => Any = from match {
  case StringType =>
    buildCast[UTF8String](_, s => DateTimeUtils.stringToDate(s).orNull)
  case TimestampType =>
    // throw valid precision more than seconds, according to Hive.
    // Timestamp.nanos is in 0 to 999,999,999, no more than a second.
    buildCast[Long](_, t => DateTimeUtils.millisToDays(t / 1000L, timeZone))
}
```