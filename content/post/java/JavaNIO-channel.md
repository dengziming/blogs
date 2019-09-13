---
date: 2018-04-22
title: "JavaNIO-channel"
author: "邓子明"
tags:
    - java
    - NIO
categories:
    - Java基础
comment: true
---

## 通道基础

```
package java.nio.channels;
public interface Channel
{
    public boolean isOpen( );
    public void close( ) throws IOException;
}
```

不同的操作系统上通道实现(Channel Implementation)会有根本性的差异,通道只能在字节缓冲区上操作

AbstractInterruptibleChannel 和 AbstractSelectableChannel，
它们分别为可中断的(interruptible)和可选择的(selectable)的通道实现提供所需的常用方法，
位于不同的包 java.nio.channels.spi。

### 打开通道

I/O 可以分为广义的两大类别: File I/O 和 Stream I/O。相应的通道是文件(file)通道和 套接字(socket)通道。
FileChannel 类和三个 socket 通 道类:SocketChannel、ServerSocketChannel 和 DatagramChannel。
 
FileChannel 对象却只能通过在一个打开的 RandomAccessFile、FileInputStream 或 FileOutputStream 对象上调用 getChannel( )方法来获取。

socket 类也有新的 getChannel( )方法。这些方法虽然 能返回一个相应的 socket 通道对象，但它们却并非新通道的来源。

### 使用通道

```java

public interface ReadableByteChannel
    extends Channel{
    public int read (ByteBuffer dst) throws IOException;

}
public interface WritableByteChannel
    extends Channel{
    public int write (ByteBuffer src) throws IOException;
}
public interface ByteChannel
    extends ReadableByteChannel, WritableByteChannel
{ }

```

通道可以是单向(unidirectional)或者双向的(bidirectional),
实现 ByteChannel 接口的通道会同时 实现 ReadableByteChannel 和 WritableByteChannel 两个接口，所以此类通道是双向的。

ByteChannel 的 read( ) 和 write( )方法使用 ByteBuffer 对象作为参数。两种方法均返回已传输的 字节数，可能比缓冲区的字节数少甚至可能为零。
缓冲区的位置也会发生与已传输字节相同数量的 前移。如果只进行了部分传输，缓冲区可以被重新提交给通道并从上次中断的地方继续传输。
该过 程重复进行直到缓冲区的 hasRemaining( )方法返回 false 值。

下面展示了两种读写数据的方法，可以参考：

```java
/**
      * Channel copy method 1. This method copies data from the src
      * 复制数据从 src 到 dest 直到 EOF，在 temp buff 中使用了 compact( )  方法处理可能未读完的数据
      * 这会导致数据复制,但是对系统的调用最小。 最终需要一个循环 cleanup 的操作。
      */
    private static void channelCopy1 (ReadableByteChannel src,
        WritableByteChannel dest)
        throws IOException
     {
        // 临时的 buff 
        ByteBuffer buffer = ByteBuffer.allocateDirect (16 * 1024);
        while (src.read (buffer) != -1) {
            // Prepare the buffer to be drained
            buffer.flip( );
            // Write to the channel; may block
            dest.write (buffer);
            // If partial transfer, shift remainder down
            // If buffer is empty, same as doing clear( )

			buffer.compact( );
    }
    // EOF will leave buffer in fill state
    buffer.flip( );
    // Make sure that the buffer is fully drained
    while (buffer.hasRemaining( )) {
        dest.write (buffer);
    }
} 

/**
  * 同样copy 数据，但是保证每次读之前数据都已经是空的，但是需要更多的调用。
  * 最后也不需要 cleanup
  */
private static void channelCopy2 (ReadableByteChannel src,
    WritableByteChannel dest)
    throws IOException{
    ByteBuffer buffer = ByteBuffer.allocateDirect (16 * 1024);
    while (src.read (buffer) != -1) {
        // Prepare the buffer to be drained
        buffer.flip( );
        // Make sure that the buffer was fully drained
        while (buffer.hasRemaining( )) {
        dest.write (buffer);
        }
        // Make the buffer empty, ready for filling
        buffer.clear( );
	} 
}
```

通道可以以阻塞(blocking)或非阻塞(nonblocking)模式运行。
非阻塞模式的通道永远不会 让调用的线程休眠。请求的操作要么立即完成，要么返回一个结果表明未进行任何操作。
只有面向流的(stream-oriented)的通道，如 sockets 和 pipes 才能使用非阻塞模式。

socket 通道类从 SelectableChannel 引申而来。
从 SelectableChannel 引申而 来的类可以和支持有条件的选择(readiness selectio)的选择器(Selectors)一起使用。

### 关闭通道

通道不能重复使用，调用 close 方法时候，当通道关闭时，那个连接会丢失，然后通道将不再连接任何东西。

## Scatter/Gather

暂且简单理解为 传入的 buffer 可以改为 buffer []

## 文件通道 

文件通道总是阻塞式的，因此不能被置于非阻塞模式。网络文件系统一般而言延迟会多些，不过却也因该优化而受 益。
对于文件 I/O，最强大之处在于异步 I/O(asynchronous I/O)。

### 访问文件

访问文件内容比较多，本次我们主要介绍的还是 网络操作，暂且将文件的操作放下。

## Socket 通道

socket 通道类可以运行非阻塞模式并且是可选择的， 
DatagramChannel 和 SocketChannel 实现定义读和写功能的接口而 ServerSocketChannel 不实现。
ServerSocketChannel 负责监听传入的连接和创建新的 SocketChannel 对象，它本身从不传 输数据。

在我们具体讨论每一种 socket 通道前，您应该了解 socket 和 socket 通道之间的关系。
之前的章 节中有写道，通道是一个连接 I/O 服务导管并提供与该服务交互的方法。
就某个 socket 而言，它不 会再次实现与之对应的 socket 通道类中的 socket 协议 API，
而 java.net 中已经存在的 socket 通 道都可以被大多数协议操作重复使用。

全部 socket 通道类(DatagramChannel、SocketChannel 和 ServerSocketChannel)在被实例化时 都会创建一个对等 socket 对象。
这些是我们所熟悉的来自 java.net 的类(Socket、ServerSocket 和 DatagramSocket)，它们已经被更新以识别通道。
对等 socket 可以通过调用 socket( )方法从一个 通道上获取。此外，这三个 java.net 类现在都有 getChannel( )方法。

### 非阻塞模式

```java
public abstract class SelectableChannel
    extends AbstractChannel
    implements Channel
{
    // This is a partial API listing
    public abstract void configureBlocking (boolean block)
        throws IOException;
    public abstract boolean isBlocking( );
public abstract Object blockingLock( );
}
```

有条件的选择(readiness selection)是一种可以用来查询通道的机制，该查询可以判断通道是 否准备好执行一个目标操作，如读或写。
非阻塞 I/O 和可选择性是紧密相连的，那也正是管理阻塞 模式的 API 代码要在 SelectableChannel 超级类中定义的原因。

设置或重新设置一个通道的阻塞模式是很简单的，只要调用 configureBlocking( )方法即可，
传 递参数值为 true 则设为阻塞模式，参数值为 false 值设为非阻塞模式
可以通过调用 isBlocking( )方法来判断某个 socket 通道当前处于哪种模式:

```java
SocketChannel sc = SocketChannel.open( );
sc.configureBlocking (false); // nonblocking
...
if ( ! sc.isBlocking( )) {
        doSomething (cs);
}
```

### ServerSocketChannel

```java
public abstract class ServerSocketChannel
    extends AbstractSelectableChannel
{
    public static ServerSocketChannel open( ) throws IOException
    public abstract ServerSocket socket( );
    public abstract SocketChannel accept( ) throws IOException;
    public final int validOps( )
}
```

用静态的 open( )工厂方法创建一个新的 ServerSocketChannel 对象，将会返回同一个未绑定的 java.net.ServerSocket 关联的通道。
该对等 ServerSocket 可以通过在返回的 ServerSocketChannel 上调 用 socket( )方法来获取。

```java
ServerSocketChannel ssc = ServerSocketChannel.open( );
ServerSocket serverSocket = ssc.socket( );
// Listen on port 1234
serverSocket.bind (new InetSocketAddress (1234));
```

使用方法：

```java
public class ChannelAccept
{
    public static final String GREETING = "Hello I must be going.\r\n";
    public static void main (String [] argv)
        throws Exception
    {
        int port = 1234; // default
        if (argv.length > 0) {
            port = Integer.parseInt (argv [0]);
        }
        ByteBuffer buffer = ByteBuffer.wrap (GREETING.getBytes( ));
        ServerSocketChannel ssc = ServerSocketChannel.open( );

ssc.socket( ).bind (new InetSocketAddress (port));
        ssc.configureBlocking (false);
        while (true) {
            System.out.println ("Waiting for connections");
            SocketChannel sc = ssc.accept( );
            if (sc == null) {
                // no connections, snooze a while
                Thread.sleep (2000);
            } else {
                System.out.println ("Incoming connection from: "
                   + sc.socket().getRemoteSocketAddress( ));
                buffer.rewind( );
                sc.write (buffer);
                sc.close( );
            }
        }
    }
}

```

validOps 是有从来和 Selector 合作使用的

### SocketChannel

每个 SocketChannel 对象创建时都是同一个对等的 java.net.Socket 对象串联的。
静态的 open( )方 法可以创建一个新的 SocketChannel 对象，而在新创建的 SocketChannel 上调用 socket( )方法能返回 它对等的 Socket 对象;
在该 Socket 上调用 getChannel( )方法则能返回最初的那个 SocketChannel。

*虽然每个 SocketChannel 对象都会创建一个对等的 Socket 对象，反过来却不成 立。
直接创建的 Socket 对象不会关联 SocketChannel 对象，它们的 getChannel( )方法只返回 null。*

新创建的 SocketChannel 虽已打开却是未连接的。在一个未连接的 SocketChannel 对象上尝试一 个 I/O 操作会导致 NotYetConnectedException 异常。
我们可以通过在通道上直接调用 connect( )方法 或在通道关联的 Socket 对象上调用 connect( )来将该 socket 通道连接。
一旦一个 socket 通道被连 接，它将保持连接状态直到被关闭。
可以通过调用布尔型的 isConnected( )方法来测试某个 SocketChannel 当前是否已连接。

在 SocketChannel 上并没有一种 connect( )方法可以让您指定超时(timeout)值，
当 connect( )方 法在非阻塞模式下被调用时 SocketChannel 提供并发连接:它发起对请求地址的连接并且立即返回 值。
如果返回值是 true，说明连接立即建立了(这可能是本地环回连接);
如果连接不能立即建 立，connect( )方法会返回false且并发地继续连接建立过程。

调用 finishConnect( )方法来完成连接过程，该方法任何时候都可以安全地进行调用。
假如在一 个非阻塞模式的 SocketChannel 对象上调用 finishConnect( )方法，将可能出现下列情形之一:
1. connect( )方法尚未被调用。那么将产生 NoConnectionPendingException 异常。 
2. 连接建立过程正在进行，尚未完成。那么什么都不会发生，finishConnect( )方法会立即返回false 值。
3. 在非阻塞模式下调用connect( )方法之后，SocketChannel又被切换回了阻塞模式。
那么如果有必要的话，调用线程会阻塞直到连接建立完成，finishConnect( )方法接着就会返回true值。
4. 在初次调用 connect( )或最后一次调用 finishConnect( )之后，连接建立过程已经完成。那么
SocketChannel 对象的内部状态将被更新到已连接状态，finishConnect( )方法会返回 true值，然后 SocketChannel 对象就可以被用来传输数据了。
5. 连接已经建立。那么什么都不会发生，finishConnect( )方法会返回 true 值。

当通道处于中间的连接等待(connection-pending)状态时，您只可以调用 finishConnect( )、 isConnectPending( )或 isConnected( )方法。一旦连接建立过程成功完成，isConnected( )将返回 true 值。

```java
InetSocketAddress addr = new InetSocketAddress (host, port);
SocketChannel sc = SocketChannel.open( );
sc.configureBlocking (false);
sc.connect (addr);
while ( ! sc.finishConnect( )) {
    doSomethingElse( );
}
doSomethingWithChannel (sc);
sc.close( );
```

异步的代码：

```java
public class ConnectAsync
{
public static void main (String [] argv) throws Exception
{
String host = "localhost";
int port = 80;
if (argv.length == 2) {
host = argv [0];
port = Integer.parseInt (argv [1]);
}
InetSocketAddress addr = new InetSocketAddress (host, port);
SocketChannel sc = SocketChannel.open( );
sc.configureBlocking (false);
System.out.println ("initiating connection");
sc.connect (addr);
while ( ! sc.finishConnect( )) {
doSomethingUseful( );
}
System.out.println ("connection established");
// Do something with the connected socket
// The SocketChannel is still nonblocking
sc.close( );
}
private static void doSomethingUseful( )
{
System.out.println ("doing something useless");
}
}
```


sockets 是面向流的而非包导向的。它们可 以保证发送的字节会按照顺序到达但无法承诺维持字节分组。
某个发送器可能给一个 socket 写入了 20 个字节而接收器调用 read( )方法时却只收到了其中的 3 个字节。
剩下的 17 个字节还是传输中。 由于这个原因，让多个不配合的线程共享某个流 socket 的同一侧绝非一个好的设计选择。

### DatagramChannel

最后一个 socket 通道是 DatagramChannel。正如 SocketChannel 对应 Socket， ServerSocketChannel 对应 ServerSocket，
每一个 DatagramChannel 对象也有一个关联的 DatagramSocket 对象。
不过原命名模式在此并未适用:“DatagramSocketChannel”显得有点笨拙， 因此采用了简洁的“DatagramChannel”名称。
正如 SocketChannel 模拟连接导向的流协议(如 TCP/IP)，DatagramChannel 则模拟包导向的 无连接协议(如 UDP/IP)

相关使用暂且放下，毕竟用得少。

## 管道

Pipe 是一个工具类

## 通道工具类

工具类可以方便快速的使用，例如 Readers 和 Writers 运行在字符的基础上。


