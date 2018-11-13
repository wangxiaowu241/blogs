## 简介

![image-20181109100858595](/Users/wangwangxiaoteng/Library/Application Support/typora-user-images/image-20181109100858595.png)

源码中是这么描述的，ByteBuf是一个可支持随机及顺序访问零或多字节的序列，这个接口提供了一个或多个原始字节数组的抽象视图。简单来说，ByteBuf就是netty提供用来操作字节码的工具。

## 创建一个ByteBuf

源码中推荐创建一个ByteBuf实例的方式是使用Unpooled类中的静态方法进行构建，而不是直接通过其子类的构造方法去构建。

例：
 
1. Unpooled.buffer(int initialCapacity)方法构造了一个大的字节组，存储在java堆内存中，初始容量为128，可随着需求无限扩大，初始readerIndex和writerIndex均为0。

   ```java
   /**
    * Creates a new big-endian Java heap buffer with the specified {@code capacity}, which
    * expands its capacity boundlessly on demand.  The new buffer's {@code readerIndex} and
    * {@code writerIndex} are {@code 0}.
    */
   public static ByteBuf buffer(int initialCapacity) {
       return ALLOC.heapBuffer(initialCapacity);
   }
   ```

2. Unpooled.directBuffer(int initialCapacity)方法构造了一个大的字节组，存储在直接内存中，初始容量为256，可随着需求无限扩大，初始readerIndex和writerIndex均为0。

   ```java
   /**
    * Creates a new big-endian direct buffer with the specified {@code capacity}, which
    * expands its capacity boundlessly on demand.  The new buffer's {@code readerIndex} and
    * {@code writerIndex} are {@code 0}.
    */
   public static ByteBuf directBuffer(int initialCapacity) {
       return ALLOC.directBuffer(initialCapacity);
   }
   ```

3. Unpooled.wrappedBuffer(byte[] array)方法构造了一个大的字节缓冲区，包裹了指定的array字节码数组，当这个array字节码数组发生变化时，对构造的ByteBuf实例是可见的，初始readerIndex为0，writerIndex为array.length。

   ```java
   /**
    * Creates a new big-endian buffer which wraps the specified {@code array}.
    * A modification on the specified array's content will be visible to the
    * returned buffer.
    */
   public static ByteBuf wrappedBuffer(byte[] array) {
       if (array.length == 0) {
           return EMPTY_BUFFER;
       }
       return new UnpooledHeapByteBuf(ALLOC, array, array.length);
   }
   ```

4. Unpooled.copiedBuffer(byte[] array)方法构造了一个大的字节缓冲区，深拷贝自array字节码数组，当这个array字节码数组发生变化时，对构造的ByteBuf实例是不可见的。就是说构建的ByteBuf实例与原array字节码数组之间不共享任何数据，初始readerIndex为0，writerIndex为array.length。

   ```java
   /**
    * Creates a new big-endian buffer whose content is a copy of the
    * specified {@code array}.  The new buffer's {@code readerIndex} and
    * {@code writerIndex} are {@code 0} and {@code array.length} respectively.
    */
   public static ByteBuf copiedBuffer(byte[] array) {
       if (array.length == 0) {
           return EMPTY_BUFFER;
       }
       return wrappedBuffer(array.clone());
   }
   ```

## 随机访问索引

和普通的以前的字节码数组相同，ByteBuf是以0为基础支持随机访问的。意思是，ByteBuf的第一个字节的索引是0，最后一个字节的索引是capacity - 1。

遍历ByteBuf字节码方式：

```java
for (int i = 0; i < buffer.capacity(); i ++) {
    byte b = buffer.getByte(i);
    System.out.println((char) b);
}
```

## 顺序访问索引

ByteBuf提供了两个指针变量去支持顺序读和写操作，分别是readerIndex和writerIndex。

他们和容量capacity的关系为：readerIndex<=writerIndex<=capacity-1

类似下图：

```java
      +-------------------+------------------+------------------+
      | discardable bytes |  readable bytes  |  writable bytes  |
      |                   |     (CONTENT)    |                  |
      +-------------------+------------------+------------------+
      |                   |                  |                  |
      0      <=      readerIndex   <=   writerIndex    <=    capacity
```

1. 可读的字节码数组就是实际存储数据的部分，以read或skip命名的操作会获得或者跳过当前readerIndex的数据，并且会使readerIndex自增长，如果没有可读的数据或可读的数据不足调用方法时指定的length，则会抛出IndexOutOfBoundsException。

   ```java
   @Override
   public byte readByte() {
       checkReadableBytes0(1);
       int i = readerIndex;
       byte b = _getByte(i);
       readerIndex = i + 1;
       return b;
   }
   ```

   遍历可读字节码的方式：

   ```java
   while (buffer.isReadable()) {
        System.out.println(buffer.readByte());
   }
   ```

2. 可写的字节码数组是一段未定义、需要被填充的空间，任何以write命名的操作会将数据写入当前writerIndex的位置上，而且writerIndex会自增，如果ByteBuf字节码数组已满，无可被填充的空间，调用write方法会抛出IndexOutOfBoundsException。

   ```java
   @Override
   public ByteBuf writeByte(int value) {
       ensureWritable0(1);
       _setByte(writerIndex++, value);
       return this;
   }
   ```

3. 废弃的字节码数组是一段已经被读取过的字节。初始时，这部分的长度为0，但是随着读取的操作进行，这部分数据大小越来越大。discardReadBytes()方法可以回收这部分数据。

```java
  BEFORE discardReadBytes()

      +-------------------+------------------+------------------+
      | discardable bytes |  readable bytes  |  writable bytes  |
      +-------------------+------------------+------------------+
      |                   |                  |                  |
      0      <=      readerIndex   <=   writerIndex    <=    capacity


  AFTER discardReadBytes()

      +------------------+--------------------------------------+
      |  readable bytes  |    writable bytes (got more space)   |
      +------------------+--------------------------------------+
      |                  |                                      |
 readerIndex (0) <= writerIndex (decreased)        <=        capacity
```

## 清理索引

清理缓冲区索引我们使用clear方法使readerIndex和writerIndex均置为0，但是注意，这个操作并不会清除数据，即原先存储的数据还依然存在。

```java
 BEFORE clear()

      +-------------------+------------------+------------------+
      | discardable bytes |  readable bytes  |  writable bytes  |
      +-------------------+------------------+------------------+
      |                   |                  |                  |
      0      <=      readerIndex   <=   writerIndex    <=    capacity

  AFTER clear()
    
      +---------------------------------------------------------+
      |             writable bytes (got more space)             |
      +---------------------------------------------------------+
      |                                                         |
      0 = readerIndex = writerIndex            <=            capacity
```

clear方法与discardReadBytes区别在于，clear方法时彻底的重置了readerIndex和writeIndex，而discardReadBytes只是回收了一部分已读取的区域，相当于readIndex和writeIndex同时向前移动了discard区域的长度。
