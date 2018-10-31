---
title: 'Storage:different strategies about high cardinal string'
date: 2018-09-12 17:03:14
tags:
  - BigData
  - DigAndBuried
---
# Keywords
run length encoding:对数据稀疏的列有较好的压缩率和访问速度

字典

# Clickhouse

ClickHouse doesn’t have the concept of encodings. Strings can contain an arbitrary set of bytes, which are stored and output as-is. If you need to store texts, we recommend using UTF-8 encoding. At the very least, if your terminal uses UTF-8 (as recommended), you can read and write your values without making conversions. Similarly, certain functions for working with strings have separate variations that work under the assumption that the string contains a set of bytes representing a UTF-8 encoded text. For example, the ‘length’ function calculates the string length in bytes, while the ‘lengthUTF8’ function calculates the string length in Unicode code points, assuming that the value is UTF-8 encoded.

# Druid

使用字典:

字典是 mmap 进来的, 见:`org.apache.druid.segment.serde.ColumnPartSerde.Deserializer#read`
```java
        final GenericIndexed<String> rDictionary = GenericIndexed.read(
            buffer,
            GenericIndexed.STRING_STRATEGY,
            builder.getFileMapper()
        );

        DictionaryEncodedColumnSupplier dictionaryEncodedColumnSupplier = new DictionaryEncodedColumnSupplier(
            rDictionary,
            rSingleValuedColumn,
            rMultiValuedColumn,
            columnConfig.columnCacheSizeBytes()
        );        
```

其中`GenericIndexed.read`具体:
```java
  private static <T> GenericIndexed<T> createGenericIndexedVersionTwo(
      ByteBuffer byteBuffer,
      ObjectStrategy<T> strategy,
      SmooshedFileMapper fileMapper
  )
  {
    if (fileMapper == null) {
      throw new IAE("SmooshedFileMapper can not be null for version 2.");
    }
    boolean allowReverseLookup = byteBuffer.get() == REVERSE_LOOKUP_ALLOWED;
    int logBaseTwoOfElementsPerValueFile = byteBuffer.getInt();
    int numElements = byteBuffer.getInt();

    try {
      String columnName = SERIALIZER_UTILS.readString(byteBuffer);
      int elementsPerValueFile = 1 << logBaseTwoOfElementsPerValueFile;
      int numberOfFilesRequired = getNumberOfFilesRequired(elementsPerValueFile, numElements);
      ByteBuffer[] valueBuffersToUse = new ByteBuffer[numberOfFilesRequired];
      for (int i = 0; i < numberOfFilesRequired; i++) {
        // SmooshedFileMapper.mapFile() contract guarantees that the valueBuffer's limit equals to capacity.
        ByteBuffer valueBuffer = fileMapper.mapFile(GenericIndexedWriter.generateValueFileName(columnName, i));
        valueBuffersToUse[i] = valueBuffer.asReadOnlyBuffer();
      }
      ByteBuffer headerBuffer = fileMapper.mapFile(GenericIndexedWriter.generateHeaderFileName(columnName));
      return new GenericIndexed<>(
          valueBuffersToUse,
          headerBuffer,
          strategy,
          allowReverseLookup,
          logBaseTwoOfElementsPerValueFile,
          numElements
      );
    }
    catch (IOException e) {
      throw new RuntimeException("File mapping failed.", e);
    }
  }

```

字典的具体表示是:`CachingIndexed<String> cachedLookups`, get 的时候先 get 从 cache 拿, 然后没有的话再从 mmap 中拿.
```java
public class CachingIndexed<T> implements Indexed<T>, Closeable
{
  public static final int INITIAL_CACHE_CAPACITY = 16384;

  private static final Logger log = new Logger(CachingIndexed.class);

  private final GenericIndexed<T>.BufferIndexed delegate;
  private final SizedLRUMap<Integer, T> cachedValues;

  /**
   * Creates a CachingIndexed wrapping the given GenericIndexed with a value lookup cache
   *
   * CachingIndexed objects are not thread safe and should only be used by a single thread at a time.
   * CachingIndexed objects must be closed to release any underlying cache resources.
   *
   * @param delegate the GenericIndexed to wrap with a lookup cache.
   * @param lookupCacheSize maximum size in bytes of the lookup cache if greater than zero
   */
  public CachingIndexed(GenericIndexed<T> delegate, final int lookupCacheSize)
  {
    this.delegate = delegate.singleThreaded();

    if (lookupCacheSize > 0) {
      log.debug("Allocating column cache of max size[%d]", lookupCacheSize);
      cachedValues = new SizedLRUMap<>(INITIAL_CACHE_CAPACITY, lookupCacheSize);
    } else {
      cachedValues = null;
    }
  }

  @Override
  public Class<? extends T> getClazz()
  {
    return delegate.getClazz();
  }

  @Override
  public int size()
  {
    return delegate.size();
  }

  @Override
  public T get(int index)
  {
    if (cachedValues != null) {
      final T cached = cachedValues.getValue(index);
      if (cached != null) {
        return cached;
      }

      final T value = delegate.get(index);
      cachedValues.put(index, value, delegate.getLastValueSize());
      return value;
    } else {
      return delegate.get(index);
    }
  }

  @Override
  public int indexOf(T value)
  {
    return delegate.indexOf(value);
  }

  @Override
  public Iterator<T> iterator()
  {
    return delegate.iterator();
  }

  @Override
  public void close()
  {
    if (cachedValues != null) {
      log.debug("Closing column cache");
      cachedValues.clear();
    }
  }

  @Override
  public void inspectRuntimeShape(RuntimeShapeInspector inspector)
  {
    inspector.visit("cachedValues", cachedValues != null);
    inspector.visit("delegate", delegate);
  }

  private static class SizedLRUMap<K, V> extends LinkedHashMap<K, Pair<Integer, V>>
  {
    private final int maxBytes;
    private int numBytes = 0;

    public SizedLRUMap(int initialCapacity, int maxBytes)
    {
      super(initialCapacity, 0.75f, true);
      this.maxBytes = maxBytes;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, Pair<Integer, V>> eldest)
    {
      if (numBytes > maxBytes) {
        numBytes -= eldest.getValue().lhs;
        return true;
      }
      return false;
    }

    public void put(K key, V value, int size)
    {
      final int totalSize = size + 48; // add approximate object overhead
      numBytes += totalSize;
      super.put(key, new Pair<>(totalSize, value));
    }

    public V getValue(Object key)
    {
      final Pair<Integer, V> sizeValuePair = super.get(key);
      return sizeValuePair == null ? null : sizeValuePair.rhs;
    }
  }
}
```

# IndexR

# Carbondata