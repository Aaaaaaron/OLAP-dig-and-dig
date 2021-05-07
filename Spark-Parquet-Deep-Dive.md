---
title: Spark Parquet Deep Dive
date: 2018-08-21 15:11:45
tags: 
  - Spark
  - BigData
hidden: true

---
ParquetFileFormat#buildReaderWithPartitionValues

注意看 enableVectorizedReader. enable 了之后用的是`VectorizedParquetRecordReader`, 否则用的是`ParquetRecordReader[UnsafeRow](new ParquetReadSupport(convertTz))`.

```scala
// Try to push down filters when filter push-down is enabled.
// Notice: This push-down is RowGroups level, not individual records.
if (pushed.isDefined) {
  ParquetInputFormat.setFilterPredicate(hadoopAttemptContext.getConfiguration, pushed.get)
}
val taskContext = Option(TaskContext.get())
if (enableVectorizedReader) {
  val vectorizedReader = new VectorizedParquetRecordReader(
    convertTz.orNull, enableOffHeapColumnVector && taskContext.isDefined, capacity)
  val iter = new RecordReaderIterator(vectorizedReader)
  // SPARK-23457 Register a task completion lister before `initialization`.
  taskContext.foreach(_.addTaskCompletionListener[Unit](_ => iter.close()))
  vectorizedReader.initialize(split, hadoopAttemptContext)
  logDebug(s"Appending $partitionSchema ${file.partitionValues}")
  vectorizedReader.initBatch(partitionSchema, file.partitionValues)
  if (returningBatch) {
    vectorizedReader.enableReturningBatches()
  }

  // UnsafeRowParquetRecordReader appends the columns internally to avoid another copy.
  iter.asInstanceOf[Iterator[InternalRow]]
} else {
  logDebug(s"Falling back to parquet-mr")
  // ParquetRecordReader returns UnsafeRow
  val reader = if (pushed.isDefined && enableRecordFilter) {
    val parquetFilter = FilterCompat.get(pushed.get, null)
    new ParquetRecordReader[UnsafeRow](new ParquetReadSupport(convertTz), parquetFilter)
  } else {
    new ParquetRecordReader[UnsafeRow](new ParquetReadSupport(convertTz))
  }
  val iter = new RecordReaderIterator(reader)
  // SPARK-23457 Register a task completion lister before `initialization`.
  taskContext.foreach(_.addTaskCompletionListener[Unit](_ => iter.close()))
  reader.initialize(split, hadoopAttemptContext)

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
```

来看 `VectorizedParquetRecordReader`: 
```scala
  /**
   * Advances to the next batch of rows. Returns false if there are no more.
   */
  public boolean nextBatch() throws IOException {
    for (WritableColumnVector vector : columnVectors) {
      vector.reset();
    }
    columnarBatch.setNumRows(0);
    if (rowsReturned >= totalRowCount) return false;
    checkEndOfRowGroup();

    int num = (int) Math.min((long) capacity, totalCountLoadedSoFar - rowsReturned);
    for (int i = 0; i < columnReaders.length; ++i) {
      if (columnReaders[i] == null) continue;
      columnReaders[i].readBatch(num, columnVectors[i]);
    }
    rowsReturned += num;
    columnarBatch.setNumRows(num);
    numBatched = num;
    batchIdx = 0;
    return true;
  }

  private void checkEndOfRowGroup() throws IOException {
    if (rowsReturned != totalCountLoadedSoFar) return;
    PageReadStore pages = reader.readNextRowGroup();
    if (pages == null) {
      throw new IOException("expecting more rows but reached last block. Read "
          + rowsReturned + " out of " + totalRowCount);
    }
    List<ColumnDescriptor> columns = requestedSchema.getColumns();
    List<Type> types = requestedSchema.asGroupType().getFields();
    columnReaders = new VectorizedColumnReader[columns.size()];
    for (int i = 0; i < columns.size(); ++i) {
      if (missingColumns[i]) continue;
      columnReaders[i] = new VectorizedColumnReader(columns.get(i), types.get(i).getOriginalType(),
        pages.getPageReader(columns.get(i)), convertTz);
    }
    totalCountLoadedSoFar += pages.getRowCount();
  }

```

`VectorizedColumnReader`:Decoder to return values from a single column.
```java
/**
 * Reads `total` values from this columnReader into column.
 */
void readBatch(int total, WritableColumnVector column) throws IOException {
  int rowId = 0;
  WritableColumnVector dictionaryIds = null;
  if (dictionary != null) {
    // SPARK-16334: We only maintain a single dictionary per row batch, so that it can be used to
    // decode all previous dictionary encoded pages if we ever encounter a non-dictionary encoded
    // page.
    dictionaryIds = column.reserveDictionaryIds(total);
  }
  while (total > 0) {
    // Compute the number of values we want to read in this page.
    int leftInPage = (int) (endOfPageValueCount - valuesRead);
    if (leftInPage == 0) {
      readPage();
      leftInPage = (int) (endOfPageValueCount - valuesRead);
    }
    int num = Math.min(total, leftInPage);
    PrimitiveType.PrimitiveTypeName typeName =
      descriptor.getPrimitiveType().getPrimitiveTypeName();
    if (isCurrentPageDictionaryEncoded) {
      // Read and decode dictionary ids.
      defColumn.readIntegers(
          num, dictionaryIds, column, rowId, maxDefLevel, (VectorizedValuesReader) dataColumn);

      // TIMESTAMP_MILLIS encoded as INT64 can't be lazily decoded as we need to post process
      // the values to add microseconds precision.
      if (column.hasDictionary() || (rowId == 0 &&
          (typeName == PrimitiveType.PrimitiveTypeName.INT32 ||
          (typeName == PrimitiveType.PrimitiveTypeName.INT64 &&
            originalType != OriginalType.TIMESTAMP_MILLIS) ||
          typeName == PrimitiveType.PrimitiveTypeName.FLOAT ||
          typeName == PrimitiveType.PrimitiveTypeName.DOUBLE ||
          typeName == PrimitiveType.PrimitiveTypeName.BINARY))) {
        // Column vector supports lazy decoding of dictionary values so just set the dictionary.
        // We can't do this if rowId != 0 AND the column doesn't have a dictionary (i.e. some
        // non-dictionary encoded values have already been added).
        column.setDictionary(new ParquetDictionary(dictionary));
      } else {
        decodeDictionaryIds(rowId, num, column, dictionaryIds);
      }
    } else {
      if (column.hasDictionary() && rowId != 0) {
        // This batch already has dictionary encoded values but this new page is not. The batch
        // does not support a mix of dictionary and not so we will decode the dictionary.
        decodeDictionaryIds(0, rowId, column, column.getDictionaryIds());
      }
      column.setDictionary(null);
      switch (typeName) {
        case BOOLEAN:
          readBooleanBatch(rowId, num, column);
          break;
        case INT32:
          readIntBatch(rowId, num, column);
          break;
        case INT64:
          readLongBatch(rowId, num, column);
          break;
        case INT96:
          readBinaryBatch(rowId, num, column);
          break;
        case FLOAT:
          readFloatBatch(rowId, num, column);
          break;
        case DOUBLE:
          readDoubleBatch(rowId, num, column);
          break;
        case BINARY:
          readBinaryBatch(rowId, num, column);
          break;
        case FIXED_LEN_BYTE_ARRAY:
          readFixedLenByteArrayBatch(
            rowId, num, column, descriptor.getPrimitiveType().getTypeLength());
          break;
        default:
          throw new IOException("Unsupported type: " + typeName);
      }
    }

    valuesRead += num;
    rowId += num;
    total -= num;
  }
}

private void readIntBatch(int rowId, int num, WritableColumnVector column) throws IOException {
  // This is where we implement support for the valid type conversions.
  // TODO: implement remaining type conversions
  if (column.dataType() == DataTypes.IntegerType || column.dataType() == DataTypes.DateType ||
      DecimalType.is32BitDecimalType(column.dataType())) {
    defColumn.readIntegers(
        num, column, rowId, maxDefLevel, (VectorizedValuesReader) dataColumn);
  } else if (column.dataType() == DataTypes.ByteType) {
    defColumn.readBytes(
        num, column, rowId, maxDefLevel, (VectorizedValuesReader) dataColumn);
  } else if (column.dataType() == DataTypes.ShortType) {
    defColumn.readShorts(
        num, column, rowId, maxDefLevel, (VectorizedValuesReader) dataColumn);
  } else {
    throw constructConvertNotSupportedException(descriptor, column);
  }
}

```

`VectorizedRleValuesReader#readIntegers`

```java
/**
 * Reads `total` ints into `c` filling them in starting at `c[rowId]`. This reader
 * reads the definition levels and then will read from `data` for the non-null values.
 * If the value is null, c will be populated with `nullValue`. Note that `nullValue` is only
 * necessary for readIntegers because we also use it to decode dictionaryIds and want to make
 * sure it always has a value in range.
 *
 * This is a batched version of this logic:
 *  if (this.readInt() == level) {
 *    c[rowId] = data.readInteger();
 *  } else {
 *    c[rowId] = null;
 *  }
 */
public void readIntegers(
    int total,
    WritableColumnVector c,
    int rowId,
    int level,
    VectorizedValuesReader data) throws IOException {
  int left = total;
  while (left > 0) {
    if (this.currentCount == 0) this.readNextGroup();
    int n = Math.min(left, this.currentCount);
    switch (mode) {
      case RLE:
        if (currentValue == level) {
          data.readIntegers(n, c, rowId);
        } else {
          c.putNulls(rowId, n);
        }
        break;
      case PACKED:
        for (int i = 0; i < n; ++i) {
          if (currentBuffer[currentBufferIdx++] == level) {
            c.putInt(rowId + i, data.readInteger());
          } else {
            c.putNull(rowId + i);
          }
        }
        break;
    }
    rowId += n;
    left -= n;
    currentCount -= n;
  }
}
```

`VectorizedPlainValuesReader#readIntegers`:
```java
public class VectorizedPlainValuesReader extends ValuesReader implements VectorizedValuesReader {
  @Override
  public final void readIntegers(int total, WritableColumnVector c, int rowId) {
    int requiredBytes = total * 4;
    ByteBuffer buffer = getBuffer(requiredBytes);
  
    if (buffer.hasArray()) {
      int offset = buffer.arrayOffset() + buffer.position();
      c.putIntsLittleEndian(rowId, total, buffer.array(), offset);
    } else {
      for (int i = 0; i < total; i += 1) {
        c.putInt(rowId + i, buffer.getInt());
      }
    }
  }
}
```

`OffHeapColumnVector#putIntsLittleEndian`:
```java
/**
 * Column data backed using offheap memory.
 */
public final class OffHeapColumnVector extends WritableColumnVector {
  @Override
  public void putIntsLittleEndian(int rowId, int count, byte[] src, int srcIndex) {
    if (!bigEndianPlatform) {
      Platform.copyMemory(src, srcIndex + Platform.BYTE_ARRAY_OFFSET,
          null, data + 4L * rowId, count * 4L);
    } else {
      int srcOffset = srcIndex + Platform.BYTE_ARRAY_OFFSET;
      long offset = data + 4L * rowId;
      for (int i = 0; i < count; ++i, offset += 4, srcOffset += 4) {
        Platform.putInt(null, offset,
            java.lang.Integer.reverseBytes(Platform.getInt(src, srcOffset)));
      }
    }
  }
}
```

***
### ColumnarBatch

columnarBatch.column 返回一个 ColumnVector, 可以看到是一列作为一个 ColumnVector.一次 put 是 put 一行, rowId 会 ++.

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
  // 1355241600 <=> 2012-12-12, /3600/24 to days
  c1.putInt(0, 1355241600 / 3600 / 24)
  // microsecond
  c2.putLong(0, 1355285532000000L)

  val internal0 = columnarBatch.getRow(0)

  val convert = UnsafeProjection.create(schema)
  val internal = convert.apply(internal0)

  val enc = RowEncoder.apply(schema).resolveAndBind()
  val row = enc.fromRow(internal0)

  val df = spark.createDataFrame(Lists.newArrayList(row), schema)
  print(df.take(1))
}
```

### ColumnVector

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-8-21/71064933.jpg)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-8-21/4432890.jpg)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-8-21/77850304.jpg)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/18-8-21/54044866.jpg)

```java
 * Capacity: The data stored is dense but the arrays are not fixed capacity. It is the
 * responsibility of the caller to call reserve() to ensure there is enough room before adding
 * elements. This means that the put() APIs do not check as in common cases (i.e. flat schemas),
 * the lengths are known up front.

 * Most of the APIs take the rowId as a parameter. This is the batch local 0-based row id for values
 * in the current RowBatch.
 *
 * A ColumnVector should be considered immutable once originally created. In other words, it is not
 * valid to call put APIs after reads until reset() is called.
 *
 * ColumnVectors are intended to be reused.

/**
 * A column backed by an in memory JVM array. This stores the NULLs as a byte per value
 * and a java array for the values.
 */
public final class OnHeapColumnVector extends ColumnVector {

  private static final boolean bigEndianPlatform =
    ByteOrder.nativeOrder().equals(ByteOrder.BIG_ENDIAN);

  // The data stored in these arrays need to maintain binary compatible. We can
  // directly pass this buffer to external components.

  // This is faster than a boolean array and we optimize this over memory footprint.
  private byte[] nulls;

  // Array for each type. Only 1 is populated for any type.
  private byte[] byteData;
  private short[] shortData;
  private int[] intData;
  private long[] longData;
  private float[] floatData;
  private double[] doubleData;

  // Only set if type is Array.
  private int[] arrayLengths;
  private int[] arrayOffsets;

  protected OnHeapColumnVector(int capacity, DataType type) {
    super(capacity, type, MemoryMode.ON_HEAP);
    reserveInternal(capacity);
    reset();
  }

  @Override
  public void close() {
    super.close();
    nulls = null;
    byteData = null;
    shortData = null;
    intData = null;
    longData = null;
    floatData = null;
    doubleData = null;
    arrayLengths = null;
    arrayOffsets = null;
  }

  @Override
  public void putInts(int rowId, int count, int[] src, int srcIndex) {
    System.arraycopy(src, srcIndex, intData, rowId, count);
  }

  @Override
  public int getInt(int rowId) {
    if (dictionary == null) {
      return intData[rowId];
    } else {
      return dictionary.decodeToInt(dictionaryIds.getDictId(rowId));
    }
  }
}
```

```java
/**
 * Column data backed using offheap memory.
 */
public final class OffHeapColumnVector extends ColumnVector {

  private static final boolean bigEndianPlatform =
    ByteOrder.nativeOrder().equals(ByteOrder.BIG_ENDIAN);

  // The data stored in these two allocations need to maintain binary compatible. We can
  // directly pass this buffer to external components.
  private long nulls;
  private long data;

  // Set iff the type is array.
  private long lengthData;
  private long offsetData;

  protected OffHeapColumnVector(int capacity, DataType type) {
    super(capacity, type, MemoryMode.OFF_HEAP);

    nulls = 0;
    data = 0;
    lengthData = 0;
    offsetData = 0;

    reserveInternal(capacity);
    reset();
  }

  @Override
  public long valuesNativeAddress() {
    return data;
  }

  @Override
  public long nullsNativeAddress() {
    return nulls;
  }

  @Override
  public void close() {
    super.close();
    Platform.freeMemory(nulls);
    Platform.freeMemory(data);
    Platform.freeMemory(lengthData);
    Platform.freeMemory(offsetData);
    nulls = 0;
    data = 0;
    lengthData = 0;
    offsetData = 0;
  }

  @Override
  public void putInts(int rowId, int count, int[] src, int srcIndex) {
    Platform.copyMemory(src, Platform.INT_ARRAY_OFFSET + srcIndex * 4,
        null, data + 4 * rowId, count * 4);
  }

  @Override
  public int getInt(int rowId) {
    if (dictionary == null) {
      return Platform.getInt(null, data + 4 * rowId);
    } else {
      return dictionary.decodeToInt(dictionaryIds.getDictId(rowId));
    }
  }
}
```

### Something interesting about decimal in Spark.
```java
  /**
   * Returns the decimal for rowId.
   */
  public final Decimal getDecimal(int rowId, int precision, int scale) {
    if (precision <= Decimal.MAX_INT_DIGITS()) {
      return Decimal.createUnsafe(getInt(rowId), precision, scale);
    } else if (precision <= Decimal.MAX_LONG_DIGITS()) {
      return Decimal.createUnsafe(getLong(rowId), precision, scale);
    } else {
      // TODO: best perf?
      byte[] bytes = getBinary(rowId);
      BigInteger bigInteger = new BigInteger(bytes);
      BigDecimal javaDecimal = new BigDecimal(bigInteger, scale);
      return Decimal.apply(javaDecimal, precision, scale);
    }
  }


  public final void putDecimal(int rowId, Decimal value, int precision) {
    if (precision <= Decimal.MAX_INT_DIGITS()) {
      putInt(rowId, (int) value.toUnscaledLong());
    } else if (precision <= Decimal.MAX_LONG_DIGITS()) {
      putLong(rowId, value.toUnscaledLong());
    } else {
      BigInteger bigInteger = value.toJavaBigDecimal().unscaledValue();
      putByteArray(rowId, bigInteger.toByteArray());
    }
  }
```

***
### data flow
最底下的流在 `VectorizedPlainValuesReader`
```java
public class VectorizedPlainValuesReader extends ValuesReader implements VectorizedValuesReader {
  private ByteBufferInputStream in = null;

  @Override
  public void initFromPage(int valueCount, ByteBufferInputStream in) throws IOException {
    this.in = in;
  }

  private ByteBuffer getBuffer(int length) {
    try {
      return in.slice(length).order(ByteOrder.LITTLE_ENDIAN);
    } catch (IOException e) {
      throw new ParquetDecodingException("Failed to read " + length + " bytes", e);
    }
  }

  @Override
  public final void readIntegers(int total, WritableColumnVector c, int rowId) {
    int requiredBytes = total * 4;
    ByteBuffer buffer = getBuffer(requiredBytes);

    if (buffer.hasArray()) {
      int offset = buffer.arrayOffset() + buffer.position();
      c.putIntsLittleEndian(rowId, total, buffer.array(), offset);
    } else {
      for (int i = 0; i < total; i += 1) {
        c.putInt(rowId + i, buffer.getInt());
      }
    }
  }
}
```

可以看到是 `initFromPage` 的时候传入的, 是在 `VectorizedColumnReader#readPage` 时, 读出的 page:
```java
/**
 * Decoder to return values from a single column.
 */
public class VectorizedColumnReader {
  private ValuesReader dataColumn;
  private final PageReader pageReader;

  private void readPage() {
    DataPage page = pageReader.readPage();
    // TODO: Why is this a visitor?
    page.accept(new DataPage.Visitor<Void>() {
      @Override
      public Void visit(DataPageV1 dataPageV1) {
        try {
          readPageV1(dataPageV1);
          return null;
        } catch (IOException e) {
          throw new RuntimeException(e);
        }
      }

      @Override
      public Void visit(DataPageV2 dataPageV2) {
        try {
          readPageV2(dataPageV2);
          return null;
        } catch (IOException e) {
          throw new RuntimeException(e);
        }
      }
    });
  }

  private void initDataReader(Encoding dataEncoding, ByteBufferInputStream in) throws IOException {
    ...
    if (dataEncoding != Encoding.PLAIN) {
      throw new UnsupportedOperationException("Unsupported encoding: " + dataEncoding);
    }
    this.dataColumn = new VectorizedPlainValuesReader();
    this.isCurrentPageDictionaryEncoded = false;
    }

    try {
      dataColumn.initFromPage(pageValueCount, in);
    } catch (IOException e) {
      throw new IOException("could not read page in col " + descriptor, e);
    }
  }

  private void readPageV1(DataPageV1 page) throws IOException {
    this.pageValueCount = page.getValueCount();
    ValuesReader rlReader = page.getRlEncoding().getValuesReader(descriptor, REPETITION_LEVEL);
    ValuesReader dlReader;

    // Initialize the decoders.
    if (page.getDlEncoding() != Encoding.RLE && descriptor.getMaxDefinitionLevel() != 0) {
      throw new UnsupportedOperationException("Unsupported encoding: " + page.getDlEncoding());
    }
    int bitWidth = BytesUtils.getWidthFromMaxInt(descriptor.getMaxDefinitionLevel());
    this.defColumn = new VectorizedRleValuesReader(bitWidth);
    dlReader = this.defColumn;
    this.repetitionLevelColumn = new ValuesReaderIntIterator(rlReader);
    this.definitionLevelColumn = new ValuesReaderIntIterator(dlReader);
    try {
      BytesInput bytes = page.getBytes();
      ByteBufferInputStream in = bytes.toInputStream();
      rlReader.initFromPage(pageValueCount, in);
      dlReader.initFromPage(pageValueCount, in);
      initDataReader(page.getValueEncoding(), in);
    } catch (IOException e) {
      throw new IOException("could not read page " + page + " in col " + descriptor, e);
    }
  }

  private void readPageV2(DataPageV2 page) throws IOException {
    this.pageValueCount = page.getValueCount();
    this.repetitionLevelColumn = createRLEIterator(descriptor.getMaxRepetitionLevel(),
        page.getRepetitionLevels(), descriptor);

    int bitWidth = BytesUtils.getWidthFromMaxInt(descriptor.getMaxDefinitionLevel());
    // do not read the length from the stream. v2 pages handle dividing the page bytes.
    this.defColumn = new VectorizedRleValuesReader(bitWidth, false);
    this.definitionLevelColumn = new ValuesReaderIntIterator(this.defColumn);
    this.defColumn.initFromPage(
        this.pageValueCount, page.getDefinitionLevels().toInputStream());
    try {
      initDataReader(page.getDataEncoding(), page.getData().toInputStream());
    } catch (IOException e) {
      throw new IOException("could not read page " + page + " in col " + descriptor, e);
    }
  }
}
```

在 `VectorizedColumnReader` 会 readPage. 见 VectorizedColumnReader#readBatch. readBatch 又被 VectorizedParquetRecordReader#nextBatch 调用. nextBatch 又被 `.VectorizedParquetRecordReader#nextKeyValue` 调用

`nextKeyValue` 的调用见:
```java
/**
 * An adaptor from a Hadoop [[RecordReader]] to an [[Iterator]] over the values returned.
 *
 * Note that this returns [[Object]]s instead of [[InternalRow]] because we rely on erasure to pass
 * column batches by pretending they are rows.
 */
class RecordReaderIterator[T](private[this] var rowReader: RecordReader[_, T]) extends Iterator[T] with Closeable {
  private[this] var havePair = false
  private[this] var finished = false

  override def hasNext: Boolean = {
    if (!finished && !havePair) {
      finished = !rowReader.nextKeyValue
      if (finished) {
        // Close and release the reader here; close() will also be called when the task
        // completes, but for tasks that read from many files, it helps to release the
        // resources early.
        close()
      }
      havePair = !finished
    }
    !finished
  }

  override def next(): T = {
    if (!hasNext) {
      throw new java.util.NoSuchElementException("End of stream")
    }
    havePair = false
    rowReader.getCurrentValue
  }
}
```

最后在 ParquetFileFormat 中

```java
if (enableVectorizedReader) {
  val vectorizedReader = new VectorizedParquetRecordReader(convertTz.orNull, enableOffHeapColumnVector && taskContext.isDefined, capacity)
  val iter = new RecordReaderIterator(vectorizedReader)
  // SPARK-23457 Register a task completion lister before `initialization`.
  taskContext.foreach(_.addTaskCompletionListener[Unit](_ => iter.close()))
  vectorizedReader.initialize(split, hadoopAttemptContext)
  logDebug(s"Appending $partitionSchema ${file.partitionValues}")
  vectorizedReader.initBatch(partitionSchema, file.partitionValues)
  if (returningBatch) {
    vectorizedReader.enableReturningBatches()
  }

  // UnsafeRowParquetRecordReader appends the columns internally to avoid another copy.
  iter.asInstanceOf[Iterator[InternalRow]]
} else { ... }
```