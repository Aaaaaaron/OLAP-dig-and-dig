---
title: Parquet 乱弹
date: 2019-05-09 16:30:34
tags: Parquet
hidden: true
---

http://lvheyang.com/wp-content/uploads/2016/02/%E5%88%97%E5%BC%8F%E5%AD%98%E5%82%A8%E4%B8%8EParquet%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E5%88%86%E4%BA%AB.pdf

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190509164025.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190509165611.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190509170805.png)

***
recommended to set the parquet block size to match the MFS chunk size for optimal performance.  The default MFS chunk size is 256 MB. 

A typical Hadoop job is IO bound, not CPU bound, so a light and fast compression codec will actually improve performance. 

For MapReduce, if you need your compressed data to be splittable, BZip2 and LZO formats can be split. Snappy and GZip blocks are not splittable, but files with Snappy blocks inside a container file format such as SequenceFile or Avro can be split. 

Compress Parquet files with Snappy they are indeed splittable, Property parquet.block.size defines Parquet file block size (row group size) and normally would be the same as HDFS block size. Snappy would compress Parquet row groups making Parquet file splittable.

The consequence of storing the metadata in the footer is that reading a Parquet file requires an initial seek to the end of the file (minus 8 bytes) to read the footer metadata length, then a second seek backward by that length to read the footer metadata. Unlike sequence files and Avro datafiles, where the metadata is stored in the header and sync markers are used to separate blocks, Parquet files don’t need sync markers since the block boundaries are stored in the footer metadata. (This is possible because the metadata is written after all the blocks have been written, so the writer can retain the block boundary positions in memory until the file is closed.) Therefore, Parquet files are splittable, since the blocks can be located after reading the footer and can then be processed in parallel (by MapReduce, for example).

***
parquet tool
```
usage: parquet-tools cat [option...] <input>
usage: parquet-tools head [option...] <input>
usage: parquet-tools schema [option...] <input>
usage: parquet-tools meta [option...] <input>
usage: parquet-tools dump [option...] <input>
usage: parquet-tools merge [option...] <input> [<input> ...] <output>
```

***
for spark:
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190509182510.png)


```java

    private void testParquetRead(double[] data, CompressionCodecName compress) throws IOException {
        Configuration conf = new Configuration();
        List<Type> fields = Lists.newArrayList();
        for (int i = 0; i < 3; i++) {
            fields.add(new PrimitiveType(Type.Repetition.REQUIRED, PrimitiveType.PrimitiveTypeName.DOUBLE, "f" + i));
        }
        MessageType schema = new MessageType("ori", fields);

        writeFile(conf, schema, data, compress);
        testRowGroupRead(conf, schema);
        testSimpleRead(conf, data.length);
    }

    private void writeFile(Configuration conf, MessageType schema, double[] data, CompressionCodecName compress)
            throws IOException {
        GroupWriteSupport.setSchema(schema, conf);
        ParquetWriter<Group> writer = new ParquetWriter<>(path, new GroupWriteSupport(), compress, 10 * 1024 * 1024,
                1024 * 1024, 1048576, true, false, PARQUET_2_0, conf);

        SimpleGroupFactory simpleGroupFactory = new SimpleGroupFactory(schema);
        for (double v : data) {
            writer.write(simpleGroupFactory.newGroup().append("f0", v).append("f1", v).append("f2", v));
        }
        writer.close();
    }

    private void testRowGroupRead(Configuration conf, MessageType messageType) throws IOException {
        ParquetMetadata footer = ParquetFileReader.readFooter(conf, path, ParquetMetadataConverter.NO_FILTER);
        ParquetFileReader parquetFileReader = new ParquetFileReader(conf, footer.getFileMetaData(), path,
                footer.getBlocks(), messageType.getColumns());

        long t = System.currentTimeMillis();
        int totalRowCount = 0;
        PageReadStore pageReadStore = parquetFileReader.readNextRowGroup();
        while (pageReadStore != null) {
            totalRowCount += pageReadStore.getRowCount();
            for (ColumnDescriptor columnDescriptor : footer.getFileMetaData().getSchema().getColumns()) {
                pageReadStore.getPageReader(columnDescriptor).readPage();
            }
            pageReadStore = parquetFileReader.readNextRowGroup();
        }
        System.out.println("Row Group read style take:" + (System.currentTimeMillis() - t));
    }

    private void testSimpleRead(Configuration conf, int n) throws IOException {
        ParquetReader<Group> reader = ParquetReader.builder(new GroupReadSupport(), path).withConf(conf).build();

        long t = System.currentTimeMillis();
        for (int i = 0; i < n; i++) {
            Group group = reader.read();
            for (int j = 0; j < 10; j++) {
                group.getDouble("m" + j, 0);
            }
        }
        System.out.println("Simple read take:" + (System.currentTimeMillis() - t));
    }
```


### page
read : int rowGroup, int column, long offset
```java
ParquetMetadata parquetMetadata = ParquetFileReader.readFooter(config, path, ParquetMetadataConverter.NO_FILTER)
BlockMetaData blockMetaData = parquetMetadata.getBlocks().get(rowGroup);
ColumnChunkMetaData columnChunkMetaData = blockMetaData.getColumns().get(column);
ColumnDescriptor columnDescriptor = parquetMetadata.getFileMetaData().getSchema().getColumns().get(column)
inputStream.seek(offset);
PageHeader pageHeader = Util.readPageHeader(inputStream);
DataPageHeader dataPageHeader = pageHeader.getData_page_header();
int numValues = dataPageHeader.getNum_values();
```

```java
    public ColumnChunkPageReader readAllPages() throws IOException {
      List<DataPage> pagesInChunk = new ArrayList<DataPage>();
      DictionaryPage dictionaryPage = null;
      PrimitiveType type = getFileMetaData().getSchema()
          .getType(descriptor.col.getPath()).asPrimitiveType();
      long valuesCountReadSoFar = 0;
      int dataPageCountReadSoFar = 0;
      while (hasMorePages(valuesCountReadSoFar, dataPageCountReadSoFar)) {
        PageHeader pageHeader = readPageHeader();
        int uncompressedPageSize = pageHeader.getUncompressed_page_size();
        int compressedPageSize = pageHeader.getCompressed_page_size();
        switch (pageHeader.type) {
          case DICTIONARY_PAGE:
            // there is only one dictionary page per column chunk
            if (dictionaryPage != null) {
              throw new ParquetDecodingException("more than one dictionary page in column " + descriptor.col);
            }
            DictionaryPageHeader dicHeader = pageHeader.getDictionary_page_header();
            dictionaryPage =
                new DictionaryPage(
                    this.readAsBytesInput(compressedPageSize),
                    uncompressedPageSize,
                    dicHeader.getNum_values(),
                    converter.getEncoding(dicHeader.getEncoding())
                    );
            break;
          case DATA_PAGE:
            DataPageHeader dataHeaderV1 = pageHeader.getData_page_header();
            pagesInChunk.add(
                new DataPageV1(
                    this.readAsBytesInput(compressedPageSize),
                    dataHeaderV1.getNum_values(),
                    uncompressedPageSize,
                    converter.fromParquetStatistics(
                        getFileMetaData().getCreatedBy(),
                        dataHeaderV1.getStatistics(),
                        type),
                    converter.getEncoding(dataHeaderV1.getRepetition_level_encoding()),
                    converter.getEncoding(dataHeaderV1.getDefinition_level_encoding()),
                    converter.getEncoding(dataHeaderV1.getEncoding())
                    ));
            valuesCountReadSoFar += dataHeaderV1.getNum_values();
            ++dataPageCountReadSoFar;
            break;
          case DATA_PAGE_V2:
            DataPageHeaderV2 dataHeaderV2 = pageHeader.getData_page_header_v2();
            int dataSize = compressedPageSize - dataHeaderV2.getRepetition_levels_byte_length() - dataHeaderV2.getDefinition_levels_byte_length();
            pagesInChunk.add(
                new DataPageV2(
                    dataHeaderV2.getNum_rows(),
                    dataHeaderV2.getNum_nulls(),
                    dataHeaderV2.getNum_values(),
                    this.readAsBytesInput(dataHeaderV2.getRepetition_levels_byte_length()),
                    this.readAsBytesInput(dataHeaderV2.getDefinition_levels_byte_length()),
                    converter.getEncoding(dataHeaderV2.getEncoding()),
                    this.readAsBytesInput(dataSize),
                    uncompressedPageSize,
                    converter.fromParquetStatistics(
                        getFileMetaData().getCreatedBy(),
                        dataHeaderV2.getStatistics(),
                        type),
                    dataHeaderV2.isIs_compressed()
                    ));
            valuesCountReadSoFar += dataHeaderV2.getNum_values();
            ++dataPageCountReadSoFar;
            break;
          default:
            LOG.debug("skipping page of type {} of size {}", pageHeader.getType(), compressedPageSize);
            stream.skipFully(compressedPageSize);
            break;
        }
      }
      if (offsetIndex == null && valuesCountReadSoFar != descriptor.metadata.getValueCount()) {
        // Would be nice to have a CorruptParquetFileException or something as a subclass?
        throw new IOException(
            "Expected " + descriptor.metadata.getValueCount() + " values in column chunk at " +
            getPath() + " offset " + descriptor.metadata.getFirstDataPageOffset() +
            " but got " + valuesCountReadSoFar + " values instead over " + pagesInChunk.size()
            + " pages ending at file offset " + (descriptor.fileOffset + stream.position()));
      }
      BytesInputDecompressor decompressor = options.getCodecFactory().getDecompressor(descriptor.metadata.getCodec());
      return new ColumnChunkPageReader(decompressor, pagesInChunk, dictionaryPage, offsetIndex,
          blocks.get(currentBlock).getRowCount());
    }

```

***
***
***

# 读

## 原生 parquet
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528104542.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528104910.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528105205.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528105231.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528105342.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528105441.png)

***

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528103634.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528103701.png)

ColumnChunkPageReadStore(RowGroupImpl)
```scala
    @Override
    public DataPage readPage() {
      if (compressedPages.isEmpty()) {
        return null;
      }
      DataPage compressedPage = compressedPages.remove(0);
      final int currentPageIndex = pageIndex++;
      return compressedPage.accept(new DataPage.Visitor<DataPage>() {
        @Override
        public DataPage visit(DataPageV1 dataPageV1) {
          try {
            BytesInput decompressed = decompressor.decompress(dataPageV1.getBytes(), dataPageV1.getUncompressedSize());
            if (offsetIndex == null) {
              return new DataPageV1(
                  decompressed,
                  dataPageV1.getValueCount(),
                  dataPageV1.getUncompressedSize(),
                  dataPageV1.getStatistics(),
                  dataPageV1.getRlEncoding(),
                  dataPageV1.getDlEncoding(),
                  dataPageV1.getValueEncoding());
            } else {
              long firstRowIndex = offsetIndex.getFirstRowIndex(currentPageIndex);
              return new DataPageV1(
                  decompressed,
                  dataPageV1.getValueCount(),
                  dataPageV1.getUncompressedSize(),
                  firstRowIndex,
                  Math.toIntExact(offsetIndex.getLastRowIndex(currentPageIndex, rowCount) - firstRowIndex + 1),
                  dataPageV1.getStatistics(),
                  dataPageV1.getRlEncoding(),
                  dataPageV1.getDlEncoding(),
                  dataPageV1.getValueEncoding());
            }
          } catch (IOException e) {
            throw new ParquetDecodingException("could not decompress page", e);
          }
        }

        @Override
        public DataPage visit(DataPageV2 dataPageV2) {
          if (!dataPageV2.isCompressed()) {
            if (offsetIndex == null) {
              return dataPageV2;
            } else {
              return DataPageV2.uncompressed(
                  dataPageV2.getRowCount(),
                  dataPageV2.getNullCount(),
                  dataPageV2.getValueCount(),
                  offsetIndex.getFirstRowIndex(currentPageIndex),
                  dataPageV2.getRepetitionLevels(),
                  dataPageV2.getDefinitionLevels(),
                  dataPageV2.getDataEncoding(),
                  dataPageV2.getData(),
                  dataPageV2.getStatistics());
            }
          }
          try {
            int uncompressedSize = Math.toIntExact(
                dataPageV2.getUncompressedSize()
                    - dataPageV2.getDefinitionLevels().size()
                    - dataPageV2.getRepetitionLevels().size());
            BytesInput decompressed = decompressor.decompress(dataPageV2.getData(), uncompressedSize);
            if (offsetIndex == null) {
              return DataPageV2.uncompressed(
                  dataPageV2.getRowCount(),
                  dataPageV2.getNullCount(),
                  dataPageV2.getValueCount(),
                  dataPageV2.getRepetitionLevels(),
                  dataPageV2.getDefinitionLevels(),
                  dataPageV2.getDataEncoding(),
                  decompressed,
                  dataPageV2.getStatistics());
            } else {
              return DataPageV2.uncompressed(
                  dataPageV2.getRowCount(),
                  dataPageV2.getNullCount(),
                  dataPageV2.getValueCount(),
                  offsetIndex.getFirstRowIndex(currentPageIndex),
                  dataPageV2.getRepetitionLevels(),
                  dataPageV2.getDefinitionLevels(),
                  dataPageV2.getDataEncoding(),
                  decompressed,
                  dataPageV2.getStatistics());
            }
          } catch (IOException e) {
            throw new ParquetDecodingException("could not decompress page", e);
          }
        }
      });
    }

```

## spark

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528110659.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528110909.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528111155.png)

**spark 读的调用栈**
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528103907.png)

**Vectorized read**
![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528103951.png)

![](https://aron-blog-1257818292.cos.ap-shanghai.myqcloud.com/20190528104202.png)

## code

```scala
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

```