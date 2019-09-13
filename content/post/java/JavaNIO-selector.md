---
date: 2018-04-22
title: "JavaNIO-selector"
author: "邓子明"
tags:
    - java
    - NIO
categories:
    - Java基础
comment: true
---

## Selector API

选择器(Selector) 管理着一个被注册的通道集合的信息和它们的就绪状态。

可选择通道(SelectableChannel) 提供了实现通道的可选择性所需要的公共方法

选择键(SelectionKey) 封装了特定的通道与特定的选择器的注册关系。
选择键对象被 SelectableChannel.register( ) 返回并提供一个表示这种注册关系的标记。


```java
public abstract class SelectableChannel
        extends AbstractChannel
        implements Channel{
        // This is a partial API listing
        public abstract SelectionKey register (Selector sel, int ops)throws ClosedChannelException;
        public abstract SelectionKey register (Selector sel, int ops,Object att)throws ClosedChannelException;
        public abstract boolean isRegistered( );
 
		public abstract SelectionKey keyFor (Selector sel);
        public abstract int validOps( );
        public abstract void configureBlocking (boolean block)
        throws IOException;
        public abstract boolean isBlocking( );
        public abstract Object blockingLock( );
}
```

```java
public abstract class Selector
{
        public static Selector open( ) throws IOException
        public abstract boolean isOpen( );
        public abstract void close( ) throws IOException;
        public abstract SelectionProvider provider( );
        public abstract int select( ) throws IOException;
        public abstract int select (long timeout) throws IOException;
        public abstract int selectNow( ) throws IOException;
        public abstract void wakeup( );
        public abstract Set keys( );
        public abstract Set selectedKeys( );
}
```

尽管通道实际上是被注册到选择器上的。但是设计者将 register( )放在 SelectableChannel 上而不是 Selector 上，这种做法看起来有点随意。
它将返回一个封装了 两个对象的关系的选择键对象。重要的是要记住选择器对象控制了被注册到它之上的通道的选择过 程。

```java
public abstract class SelectionKey
{
        public static final int OP_READ
        public static final int OP_WRITE
        public static final int OP_CONNECT
        public static final int OP_ACCEPT
        public abstract SelectableChannel channel( );
        public abstract Selector selector( );
        public abstract void cancel( );
        public abstract boolean isValid( );
        public abstract int interestOps( );
        public abstract void interestOps (int ops);
        public abstract int readyOps( );
        public final boolean isReadable( )
        public final boolean isWritable( )
        public final boolean isConnectable( )
        public final boolean isAcceptable( )
        public final Object attach (Object ob)
        public final Object attachment( )
}
```

## 使用选择键

一个键表示了一个特定的通道对象和一个特定的选择器对象之间的注册 关系。
可以看到前两个方法中反映了这种关系。channel( )方法返回与该键相关的 SelectableChannel 对象，而 selector( )则返回相关的 Selector 对象。

键对象表示了一种特定的注册关系。当应该终结这种关系的时候，可以调用 SelectionKey 对象的 cancel( )方法。
可以通过调用 isValid( )方法来检查它是否仍然表示一种有效的关系。

一个 SelectionKey 对象包含两个以整数形式进行编码的比特掩码:
一个用于指示那些通道/ 选择器组合体所关心的操作(instrest 集合)，另一个表示通道准备好要执行的操作(ready 集合)。

interest 集合可以通过调用键对象的 interestOps( )方法来获取。
通道被注册时也会传入，可以通过调用 interestOps( )方法并传 入一个新的比特掩码参数来改变它

可以通过调用键的readyOps( )方法来获取相关的通道的已经就绪的操作。
ready集合是interest 集合的子集，并且表示了interest集合中从上次调用select( )以来已经就绪的那些操作。

```java
if ((key.readyOps( ) & SelectionKey.OP_READ) != 0){
		myBuffer.clear( );
        key.channel( ).read (myBuffer);
        doSomethingWithBuffer (myBuffer.flip( ));
}
```


通过测试比特掩码来检查这些状态， if (key.isWritable( ))
等价于: if ((key.readyOps( ) & SelectionKey.OP_WRITE) != 0)

attach 和 attachment ，这两个方法允许您在键上放置一个“附件”，并在后面获取它。


## 使用选择器

每一个 Selector 对象维护三个键的集合:

已注册的键的集合(Registered key set) 与选择器关联的已经注册的键的集合。并不是所有注册过的键都仍然有效。这个集合通过 keys( )方法返回，并且可能是空的。

已选择的键的集合(Selected key set) 

已注册的键的集合的子集。这个集合的每个成员都是相关的通道被选择器(在前一个选择操作 中)判断为已经准备好的，并且包含于键的interest集合中的操作。
这个集合通过selectedKeys( )方 法返回(并有可能是空的)。
不要将已选择的键的集合与 ready 集合弄混了。这是一个键的集合，每个键都关联一个已经准 备好至少一种操作的通道。
每个键都有一个内嵌的 ready 集合，指示了所关联的通道已经准备好的 操作。

已取消的键的集合(Cancelled key set)
已注册的键的集合的子集，这个集合包含了 cancel( )方法被调用过的键(这个键已经被无效 化)，但它们还没有被注销。
这个集合是选择器对象的私有成员，因而无法直接访问。

基本上来说，选择器是对select( )、poll( )等本地调用(native call)或者类似的操作系统特定的系 统调用的一个包装。

select( ) 方法调用的时候，下面步骤将被执行:

1. 已取消的键的集合将会被检查。如果它是非空的，每个已取消的键的集合中的键将从另外两 个集合中移除，并且相关的通道将被注销。
这个步骤结束后，已取消的键的集合将是空的。

2. 已注册的键的集合中的键的 interest 集合将被检查。
还没准备好的通道将不会执行任何的操作。对于那些操作系统指示至 少已经准备好 interest 集合中的一种操作的通道，将执行以下两种操作中的一种。
a 通道的键还没有处于已选择的键的集合中，那么键的 ready 集合将被清空; b 键的 ready 集合将被表示操作系统发现的当前已经 准备好的操作的比特掩码更新

3. 当步骤 2 结束时，步骤 1 将重新执行，以完成任意一个在选择进行的过程中，键 已经被取消的通道的注销

4. select 操作返回的值是 ready 集合在步骤 2 中被修改的键的数量，而不是已选择的键的集合中 的通道的总数

### 停止选择过程

Selector的API中的最后一个方法，wakeup( )，提供了使线程从被阻塞的select( )方法中优 雅地退出的能力:
```
public abstract class Selector{
        // This is a partial API listing
        public abstract void wakeup( );
}
```
有三种方式可以唤醒在 select( )方法中睡眠的线程:

调用 wakeup( ) 使得选择器上的第一个还没有返回的选择操作立即返 回。
如果当前没有在进行中的选择，那么下一次对 select( )方法的一种形式的调用将立即返回。
后 续的选择操作将正常进行。在选择操作之间多次调用 wakeup( )方法与调用它一次没有什么不同。

调用 close( )
如果选择器的 close( )方法被调用，那么任何一个在选择操作中阻塞的线程都将被唤醒，就像 wakeup( )方法被调用了一样。
与选择器相关的通道将被注销，而键将被取消。

调用 interrupt( )

请注意这些方法中的任意一个都不会关闭任何一个相关的通道。

### 管理选择键

这部分需要通过实践了解

### 并发性

选择器对象是线程安全的，但它们包含的键集合不是。
操作的时候要自己控制好同步。

### 