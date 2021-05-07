---
title: Java Streaming Deep Dive
date: 2018-07-19 18:12:38
tags: Java
---
# Memory map file
## map

写到文件:
```java
File f = File.createTempFile("dict", ".dict");
System.out.println(f.getAbsolutePath());
f.deleteOnExit();

try (DataOutputStream out = new DataOutputStream(new FileOutputStream(f))) {
    for (int i = 0; i < 1000; i++) {
        out.writeInt(i);
    }
}
```

```java
RandomAccessFile raf = new RandomAccessFile(path, "r");
File Channel fc = raf.getChannel();
MappedByteBuffer byteBuffer = fc.map(FileChannel.MapMode.READ_ONLY, 0, fc.size());
```

## unmap

```java
try {
    fc.close();
    raf.close();
    Method m = FileChannelImpl.class.getDeclaredMethod("unmap", MappedByteBuffer.class);
    m.setAccessible(true);
    m.invoke(FileChannelImpl.class, byteBuffer);
} catch (Exception e) {
    throw new RuntimeException("Can not release file mapping memory.", e);
}
```

# DirectByteBuffer
ByteBuffer的所有的get/put都有两套, 一套用绝对地址, 一套用相对地址
```java
/**
* Relative <i>get</i> method for reading an int value.
*
* <p> Reads the next four bytes at this buffer's current position,
* composing them into an int value according to the current byte order,
* and then increments the position by four.  </p>
*
* @return  The int value at the buffer's current position
*
* @throws  BufferUnderflowException
*          If there are fewer than four bytes
*          remaining in this buffer
*/
public abstract int getInt();

/**
 * Absolute <i>get</i> method for reading an int value.
 *
 * <p> Reads four bytes at the given index, composing them into a
 * int value according to the current byte order.  </p>
 *
 * @param  index
 *         The index from which the bytes will be read
 *
 * @return  The int value at the given index
 *
 * @throws  IndexOutOfBoundsException
 *          If <tt>index</tt> is negative
 *          or not smaller than the buffer's limit,
 *          minus three
 */
public abstract int getInt(int index);
```

可以看到有个参数`JNI_COPY_TO_ARRAY_THRESHOLD`, 默认是6, 如果大于这个值, 会走 `unsafe.copyMemory` 的方式来把内存的值放到 array 中. 否则
```java
// These numbers represent the point at which we have empirically determined that the average cost of a JNI call exceeds the expense of an element by element copy.These numbers may change over time.
static final int JNI_COPY_TO_ARRAY_THRESHOLD   = 6;

A limit is imposed to allow for safepoint polling during a large copy
static final long UNSAFE_COPY_THRESHOLD = 1024L * 1024L;

public ByteBuffer get(byte[] dst, int offset, int length) {
    if (((long)length << 0) > Bits.JNI_COPY_TO_ARRAY_THRESHOLD) {
        checkBounds(offset, length, dst.length);
        int pos = position();
        int lim = limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);
        if (length > rem)
            throw new BufferUnderflowException();
            Bits.copyToArray(ix(pos), dst, arrayBaseOffset,
                             (long)offset << 0,
                             (long)length << 0);
        position(pos + length);
    } else {
        super.get(dst, offset, length);
    }
    return this;
}

static void copyToArray(long srcAddr, Object dst, long dstBaseOffset, long dstPos,
                        long length){
    long offset = dstBaseOffset + dstPos;
    while (length > 0) {
        long size = (length > UNSAFE_COPY_THRESHOLD) ? UNSAFE_COPY_THRESHOLD : length;
        unsafe.copyMemory(null, srcAddr, dst, offset, size);
        length -= size;
        srcAddr += size;
        offset += size;
    }
}

public ByteBuffer get(byte[] dst, int offset, int length) {
    checkBounds(offset, length, dst.length);
    if (length > remaining())
        throw new BufferUnderflowException();

    int end = offset + length;
    for (int i = offset; i < end; i++) {
        dst[i] = get();
    }
    return this;
}
```

# unsafe 操作
Spark 有一套Platform可以很方便的操作(占个坑先, 日后研究发).
```
Field address = Buffer.class.getDeclaredField("address");
address.setAccessible(true);
long baseAddress = (long) address.get(buffer);

public void copyMemory(int pos, byte[] dst, int offset, int length) {
    Platform.copyMemory(null, ix(pos), dst,
            BYTE_ARRAY_OFFSET + ((long) offset << 0),
            (long) length << 0);
}

private long ix(int i) {
    return baseAddress + ((long) i << 0);
}
```

# 番外 : Delete on exit

deleteOnExit 不一定会成功 如果文件有流还没关闭.

当通过kill -15 pid,kill -2 pid(中断, ctrl+c) 结束该Java进程时ShutdownHook会被调用。但Kill -9 pid不会触发ShutdownHook调用

FileInputStream 类都是操作一个文件的接口，注意到在创建一个 FileInputStream 对象时，会创建一个 FileDescriptor 对象，其实这个对象就是真正代表一个存在的文件对象的描述，当我们在操作一个文件对象时可以通过 getFD() 方法获取真正操作的与底层操作系统关联的文件描述。例如可以调用 FileDescriptor.sync() 方法将操作系统缓存中的数据强制刷新到物理磁盘中。

![](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/image015.jpg)
