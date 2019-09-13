---
date: 2018-03-22
title: "JavaNIO"
author: "邓子明"
tags:
    - java
    - NIO
categories:
    - Java基础
comment: true
---

// 参考资料
http://tutorials.jenkov.com/java-nio/nio-vs-io.html#main-differences-between-java-nio-and-io

## 一、IO和NIO有什么区别

1. Stream Oriented vs. Buffer Oriented

2. Blocking vs. Non-blocking IO

IO是阻塞的，一旦调用了 read() or write()，线程就堵住了，NIO非阻塞。

## 二、Channels and Buffers

Typically, all IO in NIO starts with a Channel. 
A Channel is a bit like a stream. From the Channel data can be read into a Buffer. 
Data can also be written from a Buffer into a Channel. 

There are several Channel and Buffer types. Here is a list of the primary Channel implementations in Java NIO:
```
FileChannel
DatagramChannel
SocketChannel
ServerSocketChannel
As you can see, these channels cover UDP + TCP network IO, and file IO.
```
Here is a list of the core Buffer implementations in Java NIO:
```
ByteBuffer
CharBuffer
DoubleBuffer
FloatBuffer
IntBuffer
LongBuffer
ShortBuffer
```

### 1. Channel

```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
    FileChannel inChannel = aFile.getChannel();

    ByteBuffer buf = ByteBuffer.allocate(48);

    int bytesRead = inChannel.read(buf);
    while (bytesRead != -1) {

      System.out.println("Read " + bytesRead);
      buf.flip();

      while(buf.hasRemaining()){
          System.out.print((char) buf.get());
      }

      buf.clear();
      bytesRead = inChannel.read(buf);
    }
    aFile.close();
```

Notice the buf.flip() call. First you read into a Buffer. Then you flip it. Then you read out of it.

### 2.Buffer

Java NIO Buffers are used when interacting with NIO Channels. As you know, data is read from channels into buffers, and written from buffers into channels.

A buffer is essentially a block of memory into which you can write data, which you can then later read again. 
This memory block is wrapped in a NIO Buffer object, which provides a set of methods that makes it easier to work with the memory block.

Using a Buffer to read and write data typically follows this little 4-step process:
```
1. Write data into the Buffer
2. Call buffer.flip()
3. Read data out of the Buffer
4. Call buffer.clear() or buffer.compact()
```

When you write data into a buffer, the buffer keeps track of how much data you have written. 
Once you need to read the data, you need to switch the buffer from writing mode into reading mode using the flip() method call. 
In reading mode the buffer lets you read all the data written into the buffer.

Once you have read all the data, you need to clear the buffer, to make it ready for writing again. 
You can do this in two ways: By calling clear() or by calling compact(). The clear() method clears the whole buffer. 
The compact() method only clears the data which you have already read. 
Any unread data is moved to the beginning of the buffer, and data will now be written into the buffer after the unread data.

above had given a simple Buffer usage example, with the write, flip, read and clear operations maked in bold.

### 3. Buffer Capacity, Position and Limit

A Buffer has three properties you need to be familiar with, in order to understand how a Buffer works. These are:
```
capacity
position
limit
```

The meaning of position and limit depends on whether the Buffer is in read or write mode. Capacity always means the same, no matter the buffer mode.

position and limit依赖于模式，而capacity在两种模式下都是一样的。

Being a memory block, a Buffer has a certain fixed size, also called its "capacity".

When you write data into the Buffer, you do so at a certain position. Initially the position is 0. 
When a byte, long etc. has been written into the Buffer the position is advanced to point to the next cell in the buffer to insert data into.
Position can maximally become capacity - 1.

When you read data from a Buffer you also do so from a given position.
 When you flip a Buffer from writing mode to reading mode, the position is reset back to 0. 
As you read data from the Buffer you do so from position, and position is advanced to next position to read.

In write mode the limit of a Buffer is the limit of how much data you can write into the buffer. In write mode the limit = capacity

When flipping the Buffer into read mode, limit means the limit of how much data you can read from the data. 
Therefore, when flipping a Buffer into read mode, limit is set to write position of the write mode.

### 4. Buffer Types
Java NIO comes with the following Buffer types:
```
ByteBuffer
MappedByteBuffer
CharBuffer
DoubleBuffer
FloatBuffer
IntBuffer
LongBuffer
ShortBuffer
```
As you can see, these Buffer types represent different data types. In other words, they let you work with the bytes in the buffer as char, short, int, long, float or double instead.

The MappedByteBuffer is a bit special, and will be covered in its own text.



### 5. Allocating a Buffer

```
ByteBuffer buf = ByteBuffer.allocate(48);

CharBuffer buf = CharBuffer.allocate(1024);
```

### 6. Writing Data to a Buffer
You can write data into a Buffer in two ways:
```
// Write data from a Channel into a Buffer
int bytesRead = inChannel.read(buf); //read into buffer.
//
Write data into the Buffer yourself, via the buffer's put() methods.
buf.put(127);    
```

### 7. flip

The flip() method switches a Buffer from writing mode to reading mode. 
Calling flip() sets the position back to 0, and sets the limit to where position just was.


### 8. Reading Data from a Buffer


There are two ways you can read data from a Buffer.
```
Read data from the buffer into a channel.
//read from buffer into channel.
int bytesWritten = inChannel.write(buf);

Read data from the buffer yourself, using one of the get() methods.
byte aByte = buf.get();    
```

### 9. rewind

The Buffer.rewind() sets the position back to 0, so you can reread all the data in the buffer. 

The limit remains untouched, thus still marking how many elements (bytes, chars etc.) that can be read from the Buffer.

### 10. clear() and compact()
Once you are done reading data out of the Buffer you have to make the Buffer ready for writing again. You can do so either by calling clear() or by calling compact().

If you call clear() the position is set back to 0 and the limit to capacity. In other words, the Buffer is cleared. 
The data in the Buffer is not cleared. Only the markers telling where you can write data into the Buffer are.

compact() copies all unread data to the beginning of the Buffer. Then it sets position to right after the last unread element. 
The limit property is still set to capacity, just like clear() does. Now the Buffer is ready for writing, but you will not overwrite the unread data.

### 11. mark() and reset()

You can mark a given position in a Buffer by calling the Buffer.mark() method. You can then later reset the position back to the marked position by calling the Buffer.reset() method. Here is an example:
```
buffer.mark();

//call buffer.get() a couple of times, e.g. during parsing.

buffer.reset();  //set position back to mark.    
```

## 五、Java NIO Scatter / Gather

javaNIO自带了Scatter/gather的支持，用于从Channel读数据和写数据。

scattering read from a channel：从channel读数据到不止一个Buffer，所以会 "scatters" the data from the channel into multiple buffers.

gathering write to a channel: 从多个 buffer 写到一个Channel，多疑 "gathers" the data from multiple buffers into one channel。

### 1. Scattering Reads

```java
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);

ByteBuffer[] bufferArray = { header, body };

channel.read(bufferArray);
```

```java

public static void scatter() throws Exception {

        RandomAccessFile aFile = new RandomAccessFile("src/main/resources/data/nio-data.txt", "rw");

        FileChannel inChannel = aFile.getChannel();


        ByteBuffer header = ByteBuffer.allocate(128);
        ByteBuffer body   = ByteBuffer.allocate(1028);

        ByteBuffer[] bufferArray = { header, body };

        long bytesRead = inChannel.read(bufferArray);

        System.out.println("Read " + bytesRead);
        header.flip();
        body.flip();

        while(header.hasRemaining()){
            System.out.print((char) header.get());
        }
        while(body.hasRemaining()){
            System.out.print((char) body.get());
        }

        header.clear();
        body.clear();


        aFile.close();

    }
```
The fact that scattering reads fill up one buffer before moving on to the next, means that it is not suited for dynamically sized message parts.

### 2. Gathering Writes

Here is a code example that shows how to perform a gathering write:
```
ByteBuffer header = ByteBuffer.allocate(128);
ByteBuffer body   = ByteBuffer.allocate(1024);

//write data into buffers

ByteBuffer[] bufferArray = { header, body };

channel.write(bufferArray);
```

运行下面的代码，会得到一个文件，里面是 header+body
```java
public static void gather() throws Exception {

        RandomAccessFile aFile = new RandomAccessFile("src/main/resources/data/nio-in.txt", "rw");

        FileChannel inChannel = aFile.getChannel();

        ByteBuffer header = ByteBuffer.allocate(12);

        header.put("header".getBytes());
        header.put("body".getBytes());
        ByteBuffer body   = ByteBuffer.allocate(120);

        ByteBuffer[] bufferArray = { header, body };

        header.flip();
        body.flip();
        inChannel.write(bufferArray);

        inChannel.force(true);
        inChannel.close();
        header.clear();
        body.clear();

        aFile.close();

    }
```

## 四、Java NIO Channel to Channel Transfers
channel的传输工具，类似IOUtils

### 1.
The FileChannel.transferFrom() method transfers data from a source channel into the FileChannel. Here is a simple example: transferFrom()

```java

RandomAccessFile fromFile = new RandomAccessFile("fromFile.txt", "rw");
FileChannel      fromChannel = fromFile.getChannel();

RandomAccessFile toFile = new RandomAccessFile("toFile.txt", "rw");
FileChannel      toChannel = toFile.getChannel();

long position = 0;
long count    = fromChannel.size();

toChannel.transferFrom(fromChannel, position, count);
```
The parameters position and count, tell where in the destination file to start writing (position), and how many bytes to transfer maximally (count). If the source channel has fewer than count bytes, less is transfered.

Additionally, some SocketChannel implementations may transfer only the data the SocketChannel has ready in its internal buffer here and now - even if the SocketChannel may later have more data available. Thus, it may not transfer the entire data requested (count) from the SocketChannel into FileChannel.



### 2. 
The transferTo() method transfer from a FileChannel into some other channel. Here is a simple example:
```java
RandomAccessFile fromFile = new RandomAccessFile("fromFile.txt", "rw");
FileChannel      fromChannel = fromFile.getChannel();

RandomAccessFile toFile = new RandomAccessFile("toFile.txt", "rw");
FileChannel      toChannel = toFile.getChannel();

long position = 0;
long count    = fromChannel.size();

fromChannel.transferTo(position, count, toChannel);
```
Notice how similar the example is to the previous. The only real difference is the which FileChannel object the method is called on. The rest is the same.

The issue with SocketChannel is also present with the transferTo() method. The SocketChannel implementation may only transfer bytes from the FileChannel until the send buffer is full, and then stop.

http://tutorials.jenkov.com/java-nio/selectors.html


## 四、Selectors

NIO's Selectors 允许一个 a single thread 去 monitor multiple channels of input。
你可以注册 multiple channels with a selector，然后用一个线程去 "select" the channels that have input available for processing,
或者 or select the channels that are ready for writing。
This selector mechanism makes it easy for a single thread to manage multiple channels.

