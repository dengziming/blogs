---
date: 2018-04-22
title: "JavaNIO"
author: "邓子明"
tags:
    - java
    - NIO
categories:
    - Java基础
comment: true
---

## 缓冲区基础

### 缓冲区属性

- 容量(Capacity) 缓冲区能够容纳的数据元素的最大数量。这一容量在缓冲区创建时被设定，并且永远不能被改变。
- 上界(Limit) 缓冲区的第一个不能被读或写的元素。或者说，缓冲区中现存元素的计数。
- 位置(Position) 下一个要被读或写的元素的索引。位置会自动由相应的 get( )和 put( )函数更新。 
- 标记(Mark) 一个备忘位置。调用mark( )来设定mark=postion。调用reset( )设定position= mark。标记在设定前是未定义的(undefined)。

这四个属性之间总是遵循以下关系:
0 <= mark <= position <= limit <= capacity

新创建的 Buffer 没有数据，所以是用来写的。这时候 capacity 和 limit 一样都等于 buffer 长度，position 和 mark 为0.
容量是固定不变的，其他的是可以增长的。

### 缓冲区 API

```java
public abstract class Buffer {
	public final int capacity( )
	public final int position( )
	public final Buffer position (int newPositio 
	public final int limit( )
	public final Buffer limit (int newLimit) 
	public final Buffer mark( )
	public final Buffer reset( )
	public final Buffer clear( )
	public final Buffer flip( )
	public final Buffer rewind( )
	public final int remaining( )
	public final boolean hasRemaining( ) public abstract boolean isReadOnly( );
}
```

很多行数都是 返回 Buffer 的 this 属性，这是为了写代码方便，注意不要滥用。

### put get

一个 buffer 的数据每次可能只被读写了一部分，我们后续还需要继续读写，这时候 position 方便我们进行管理。
调用 put 和 get 的时候会通过 position 找到对应的操作的位置。上面的 Buffer 没有 put 和 get 方法是因为
每一种 Buffer 类型的参数和返回值都不一样，所以不能在 Buffer 类中被抽象地声明。

以 ByteBuffer 为例：
```java
public abstract class ByteBuffer
       extends Buffer implements Comparable
{
       // This is a partial API listing
public abstract byte get( );
public abstract byte get (int index);
public abstract ByteBuffer put (byte b);
public abstract ByteBuffer put (int index, byte b);
}
```

set 和 get 方法 可以传位置，也可以不传位置。调用相对函数的时候 position 位置会 +1，调用绝对函数不会返回。
绝对存取不会影响缓冲区的 position 属性。

### put

我们可以通过 buffer.put((byte)'H').put((byte)'e').put((byte)'l').put((byte)'l').put((byte)'o') 
将 hello 字符串放进 buffer 中，然后调用 buffer.put(0,(byte)'M').put((byte)'w') 将 hello 改为 Mellow。


### flip

写数据操作完毕后，需要传给别的 channel 读，如果直接调用 get 就会从 position 之后读数据，
所以我们需要将 position 改为0，然后将 limit 改为当前的 position 保证不会读越界。通过 flip 函数可以完成这个操作。

### Rewind

Rewind 函数和 flip 类似，但是 Rewind 只是将 position 设置为0，一般 Rewind 是让你重新从0开始读数据。

### hasRemaining 和 clear

hasRemaining 可以告诉你是否已经读完了，实际就是判断 position < limit，
clear 可以清除数据，实际就是 position = 0;limit = capacity;mark = -1

### compact

如果字符串 hello 已经读了 he，你想释放掉 he，继续写，这时候调用 compact，会将 llo 移动到数组前三个位置，
position 标记为 4。

### mark

mark 是 buffer 为了方便我们操作提供的一个参数，一开始是 undefined。
调用 mark() 的时候 mark 指向 position，调用 reset 的时候 position 指向 mark。
rewind( )，clear( )，以及 flip( )总是抛弃标记

### equals 和 compareTo

对比的是剩余的元素。

### 批量移动

Buffer 提供了批量移动数据元素的 方法：

```java
public abstract class CharBuffer
          extends Buffer implements CharSequence, Comparable
   {
          // This is a partial API listing
          public CharBuffer get (char [] dst)
          public CharBuffer get (char [] dst, int offset, int length)
          public final CharBuffer put (char[] src)
          public CharBuffer put (char [] src, int offset, int length)
          public CharBuffer put (CharBuffer src)
          public final CharBuffer put (String src)
          public CharBuffer put (String src, int start, int end)
   }
```

如果 如果缓冲区中的数据不够完全填满数组 ，会报异常，正确操作是这样：

```java
char [] bigArray = new char [1000];
// 得到 buffer 数据量
int length = buffer.remaining( );
 // 我们已经知道 length < 1000
buffer.get (bigArrray, 0, length);
```

如果缓冲区的长度很小装不住那么多数据，需要这样：

```java
char [] smallArray = new char [10];
while (buffer.hasRemaining( )) {
int length = Math.min (buffer.remaining( ), smallArray.length);
        buffer.get (smallArray, 0, length);
        processData (smallArray, length);
 }
```

Put() 从数组移动到缓冲区，原理类似。

另外两个 buffer 也可以相互传送：buffer.put(srcBuffer)， 等价于：
```java
while (srcBuffer.hasRemaining( )) {
	dstBuffer.put (srcBuffer.get( )) 
}
```

## 创建缓冲区

以 CharBuffer 为例，主要方法：

```java
public abstract class CharBuffer
          extends Buffer implements CharSequence, Comparable
   {
          // This is a partial API listing
          public static CharBuffer allocate (int capacity)
          public static CharBuffer wrap (char [] array)
          public static CharBuffer wrap (char [] array, int offset,
     int length)
public final boolean hasArray( ) 
public final char [] array( ) 
public final int arrayOffset( )
}
```

allocate 创建一个缓冲区对象 HeapCharBuffer 并分配空间，
wrap 创建 HeapCharBuffer 但是不分配空间，直接使用数组存储。
这时候调用 put() 会直接影响这个数组，对这个数组的改动 也会对缓冲区可见。

allocate()或者 wrap()函数创建的间接缓冲区，直接缓冲区后续会介绍，
hasArray() 返回一个bool，指示是否有 一个可存取的备份数组。如果返回的是 false，不能调用 array() 和 arrayOffset()

arrayOffset 返回缓冲区数据在数组中存储的开始位置的偏移量.


## 复制缓冲区

duplicate 方法复制了缓冲区，但实际上没有复制数据。
两个缓冲区共享数据元素，拥有同样的容量，但每个缓冲区拥有各自的位置，上界和标记属性。
对一个缓冲区内的数据元素所做的改变会反映在另外一个缓冲区上。

asReadOnlyBuffer 返回一个只读的 缓冲区视图，和 duplicate 类似，但是不能调用 put 方法。

slice 也类似，但是创建一个从原始缓冲区的当前位置开始的新缓冲 区，并且其容量是原始缓冲区的剩余元素数量(limit-position)

## 字节缓冲区

### 直接缓冲区

在 Java 中， 数组是对象，而数据存储在对象中的方式在不同的 JVM 实现中都各有不同。直接字节缓冲区通常是 I/O 操作最好的选择。
ByteBuffer.allocateDirect() 方法即可，直接缓冲区使用的内存是通过调用本地操作系统方面的代码分配的，绕过了标准 JVM 堆栈


### 视图缓冲区

但是 ByteBuffer 类允许创建视 图来将 byte 型缓冲区字节数据映射为其它的原始数据类型。
例如，asLongBuffer()函数 创建一个将八个字节型数据当成一个 long 型数据来存取的视图缓冲区。

### 数据元素视图

int value = buffer.getInt( ); 可以返回一个 Int。
际的返回值取决于缓冲区的当前的比特排序(byte-order)设置，
更准确的写法是 int value = buffer.order (ByteOrder.BIG_ENDIAN).getInt( );这将会返回值 0x3BC5315E

### 存取无符号数据
Java 编程语言对无符号数值并没有提供直接的支持，但是有时候又需要。
getUnsignedByte 等，

### 内存映射缓冲区

映射缓冲区 通常是直接存取内存的，只能通过 FileChannel 类创建。后续在讨论。



