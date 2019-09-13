---
date: 2018-05-22
title: "hdfs-client"
author: "邓子明"
tags:
    - hadoop
    - 源码
categories:
    - hadoop源码
comment: true
---


参考资料：

hadoop 技术内幕丛书

#                         写

# 创建流

简单写一个 demo 进行测试，通过打断点方法，另外发现一个问题， 在 dfsClient.namenode.create 打断点调试会报错，可能是因为动态代理卡主了 ：

注意我们上传的文件 nio-data.txt 内容可以进行控制，例如我们写 600 个 a(97)，转换为 DataOutputStream 后的 byte[] 就是 600个97 ，这样调试就知道是哪个数据，600 a 是因为每个chunk的默认大小是 512
```java
public static void main(String[] args) throws IOException {

    FileSystem fs = FileSystem.get(new Configuration());
    
    Path src = new Path("ideaspace/learn-jvm/src/main/resources/data/nio-data.txt"); //文件里面是一个 java 代码
    Path desc = new Path("/tmp/");
    if (fs.exists(desc)){
            fs.delete(desc,true);
        }

    fs.copyFromLocalFile(src,desc);

}
```

经过一系列的调用后进入的第一个关键方法是 ：
```java
@Override
  public FSDataOutputStream create(final Path f, final FsPermission permission,
    final EnumSet<CreateFlag> cflags, final int bufferSize,
    final short replication, final long blockSize, final Progressable progress,
    final ChecksumOpt checksumOpt) throws IOException {
    statistics.incrementWriteOps(1);
    Path absF = fixRelativePart(f);
    return new FileSystemLinkResolver<FSDataOutputStream>() {
      @Override
      public FSDataOutputStream doCall(final Path p)
          throws IOException, UnresolvedLinkException {
        final DFSOutputStream dfsos = dfs.create(getPathName(p), permission,
                cflags, replication, blockSize, progress, bufferSize,
                checksumOpt);
        return dfs.createWrappedOutputStream(dfsos, statistics);
      }
      @Override
      public FSDataOutputStream next(final FileSystem fs, final Path p)
          throws IOException {
        return fs.create(p, permission, cflags, bufferSize,
            replication, blockSize, progress, checksumOpt);
      }
    }.resolve(this, absF);
  }
```

# 1.create 


首先是 create，然后是 dfs.createWrappedOutputStream(out, statistics);


```java
public DFSOutputStream create(String src, 
                           FsPermission permission,
                           EnumSet<CreateFlag> flag, 
                           boolean createParent,
                           short replication,
                           long blockSize,
                           Progressable progress,
                           int buffersize,
                           ChecksumOpt checksumOpt,
                           InetSocketAddress[] favoredNodes) throws IOException {
  checkOpen();
  if (permission == null) {
    permission = FsPermission.getFileDefault();
  }
  FsPermission masked = permission.applyUMask(dfsClientConf.uMask);
  if(LOG.isDebugEnabled()) {
    LOG.debug(src + ": masked=" + masked);
  }
  String[] favoredNodeStrs = null;
  if (favoredNodes != null) {
    favoredNodeStrs = new String[favoredNodes.length];
    for (int i = 0; i < favoredNodes.length; i++) {
      favoredNodeStrs[i] = 
          favoredNodes[i].getHostName() + ":" 
                       + favoredNodes[i].getPort();
    }
  }
  final DFSOutputStream result = DFSOutputStream.newStreamForCreate(this,
      src, masked, flag, createParent, replication, blockSize, progress,
      buffersize, dfsClientConf.createChecksum(checksumOpt),
      favoredNodeStrs);
  beginFileLease(result.getFileId(), result);
  return result;
}
```

两个方法比较关键：

DFSOutputStream.newStreamForCreate 和 beginFileLease(result.getFileId(), result)


## 1. newStreamForCreate 方法是第一次创建真正的 流，类是 DFSOutputStream

```java
static DFSOutputStream newStreamForCreate(DFSClient dfsClient, String src,
      FsPermission masked, EnumSet<CreateFlag> flag, boolean createParent,
      short replication, long blockSize, Progressable progress, int buffersize,
      DataChecksum checksum, String[] favoredNodes) throws IOException {
    ...
    while (shouldRetry) {
      shouldRetry = false;
      try {
        stat = dfsClient.namenode.create(src, masked, dfsClient.clientName,
            new EnumSetWritable<CreateFlag>(flag), createParent, replication,
            blockSize, SUPPORTED_CRYPTO_VERSIONS);
        break;
      } catch (RemoteException re) {...}
    Preconditions.checkNotNull(stat, "HdfsFileStatus should not be null!");
    final DFSOutputStream out = new DFSOutputStream(dfsClient, src, stat,
        flag, progress, checksum, favoredNodes);
    out.start();
    return out;
  }
```

1. rpc
这部分没法调试，因为在远程。只能自己观看
通过 RPC 调用 NameNodeRpcServer.create -> namesystem.startFile -> startFileInt -> startFileInternal ，先跳过，



2. new DFSOutputStream(dfsClient, src, stat,flag, progress, checksum, favoredNodes);

```java
/** Construct a new output stream for creating a file. */
  private DFSOutputStream(DFSClient dfsClient, String src, HdfsFileStatus stat,
      EnumSet<CreateFlag> flag, Progressable progress,
      DataChecksum checksum, String[] favoredNodes) throws IOException {
    this(dfsClient, src, progress, stat, checksum);
    this.shouldSyncBlock = flag.contains(CreateFlag.SYNC_BLOCK);

    computePacketChunkSize(dfsClient.getConf().writePacketSize, bytesPerChecksum);

    Span traceSpan = null;
    if (Trace.isTracing()) {
      traceSpan = Trace.startSpan(this.getClass().getSimpleName()).detach();
    }
    streamer = new DataStreamer(stat, traceSpan);
    if (favoredNodes != null && favoredNodes.length != 0) {
      streamer.setFavoredNodes(favoredNodes);
    }
  }
```

新建 DFSOutputStream 中有个重要的线程 DataStreamer，功能后续研究。
DFSOutputStream 中的成员变量我们可以好好看看，什么是 checksum，chunk，packet。另外它的父类 FSOutputSummer 也很重要。


## 2. beginFileLease 这个可以暂时忽略，后面专门研究 lease


dfs.createWrappedOutputStream(dfsos, statistics) 对上面创建的流就行一个 包装

返回 return new HdfsDataOutputStream(dfsos, statistics, startPos);

至此创建流的过程就完成了。我们大概回顾一下：

```java
DistributedFileSystem.create(final Path f, final FsPermission permission,
final EnumSet<CreateFlag> cflags, final int bufferSize,
    final short replication, final long blockSize, final Progressable progress,
    final ChecksumOpt checksumOpt)
{
    // 1. 
    DFSOutputStream dfsos = DfsClient.create(getPathName(p), permission,
                cflags, replication, blockSize, progress, bufferSize,
                checksumOpt)
    {
        // 1. 
        DFSOutputStream.newStreamForCreate(this,
        	src, masked, flag, createParent, replication, blockSize, progress,
        	buffersize, dfsClientConf.createChecksum(checksumOpt),
        	favoredNodeStrs);
        {
            // 1.
            stat = dfsClient.namenode.create(src, masked, dfsClient.clientName,
                new EnumSetWritable<CreateFlag>(flag), createParent, replication,
                blockSize, SUPPORTED_CRYPTO_VERSIONS);
            {
                // RPC
            }
            // 2.
            final DFSOutputStream out = new DFSOutputStream(dfsClient, src, stat,
            flag, progress, checksum, favoredNodes);
            {
                // 
                class DataStreamer
                class Patket
                streamer = new DataStreamer(stat, traceSpan);
            }
            
        }
        // 2.
        beginFileLease(result.getFileId(), result);
    }
                
    // 2. 
    return dfs.createWrappedOutputStream(dfsos, statistics);
    {
        return new HdfsDataOutputStream(dfsos, statistics, startPos);
    }
}
```

首先是 DistributedFileSystem 创建，然后调用 DfsClient 的 create ，DfsClient需要创建流和lease，创建流 由 DFSOutputStream 完成，
DFSOutputStream 需要分别和namenode、datanode通信。DFSOutputStream 内部有 Packet 和 DataStreamer，继承自 FSOutputSummer ，FSOutputSummer 完成了write的真正逻辑


# 2. out.write(buf, 0, bytesRead)

创建完成后就是写数据，HdfsDataOutputStream 写数据比较复杂，先写到缓存，然后发送。需要做 checksum 检验，然后做成一个 chuck，然后将多个 chuck 合成一个 Packet，然后发送 Packet。


FSDataOutputStream.out.write(byte[])
调用过程：
out.write(bytes) -> FilterOutputStream.write -> DataOutputStream.write -> out.write(byte[], off, len) -> FSOutputSummer.write(byte b[], int off, int len)


FSOutputSummer.write

```java
public synchronized void write(byte b[], int off, int len)
      throws IOException {
    
    checkClosed();
    
    if (off < 0 || len < 0 || off > b.length - len) {
      throw new ArrayIndexOutOfBoundsException();
    }
    // 循环调用 write ，每次写入 #write1() 长度
    for (int n=0;n<len;n+=write1(b, off+n, len-n)) {
    }
  }
```
这个 byte b[] 就是我们的数据流，通过断点我们可以看到是600个97，也就是600个a。

write1，这里有几个比较核心的内容，如果写入长度比较大，直接写入流，如果写入比较少，先写到 Buffer，到达一定长度再统一进行写到流。这么做是为了减少拷贝

```java
private int write1(byte b[], int off, int len) throws IOException {
  
  // 写入长度大于本地buf的长度时，直接写入本地buf的长度。
  if(count==0 && len>=buf.length) {
    // local buffer is empty and user buffer size >= local buffer size, so
    // simply checksum the user buffer and send it directly to the underlying
    // stream
    final int length = buf.length;
    writeChecksumChunks(b, off, length);
    return length;
  }
  // 当len小于本地buf的长度时，先写入buf，当buf写满之后，flushBuffer
  // copy user data to local buffer
  
  int bytesToCopy = buf.length-count; // 这个 count 代表以及复制的数据长度，第一次是 0
  bytesToCopy = (len<bytesToCopy) ? len : bytesToCopy;  // 这时候就是要复制的数据长度，600
  System.arraycopy(b, off, buf, count, bytesToCopy);
  count += bytesToCopy;
  if (count == buf.length) {
    // local buffer is full
    flushBuffer();
  } 
  return bytesToCopy;
}
```

可以看到，写入数据大的话，直接调用 writeChecksumChunks 将buf长度大小的数据生成 chunksum ，
（chunksum 是检查数据完整性的，相关知识可以查看计算机网络。）并写入 packet中。如果写入数据比较少，直接放进 buffer，等待buffer比较大，再统一flush

数据写完了关闭流的时候会再调用一次 fulshBuffer ,会调用 writeChecksumChunks


```java
private void writeChecksumChunks(byte b[], int off, int len)
throws IOException {
  // 计算checksum
  sum.calculateChunkedSums(b, off, len, checksum, 0);
  for (int i = 0; i < len; i += sum.getBytesPerChecksum()) {
    int chunkLen = Math.min(sum.getBytesPerChecksum(), len - i);
    int ckOffset = i / sum.getBytesPerChecksum() * getChecksumSize();
    // 一个chunk一个chunk的写入packet
    writeChunk(b, off + i, chunkLen, checksum, ckOffset, getChecksumSize());
  }

```

sum.calculateChunkedSums 计算校验值，计算完以后b还是 600个97，off和len分别是 0和600，checksum是一个36位的数组，
但是只有前八位有值：0 = 111,1 = 50,2 = -90,3 = 31,4 = -99,5 = 97,6 = -69,7 = 102。因为每512位生成4个校验码。现在是600位，需要8个。
然后写出 chunk，chunk的长度为： 512，所以这里会调用两次 writeChunk。

writeChunk 先将 chunk 写入 currentPacket 中，当currentPacket写满之后调用 waitAndQueueCurrentPacket，
将packet放入dataQueue队列，等待DataStreamer线程将packet写入pipeline中，整个block发送完毕之后将发送一个空的packet。

```java
writeChunk{}
currentPacket.writeChecksum(checksum, ckoff, cklen);
currentPacket.writeData(b, offset, len);
currentPacket.numChunks++;
bytesCurBlock += len;
waitAndQueueCurrentPacket()
```

我们把这段代码跑两遍以后，去看看生成的Packet，里面有4个属性：checksumStart = 33,checksumPos = 41,dataStart = 541,dataPos = 1141

可以看出这几个的意思分别是 check 的开始位置和结束为止，data的开始位置和结束为止，之所以留了 33个位置，是 Packet 的 header。然后还有一个 buffer数组，里面就是按照检查数据和数据。

这时候 Packet 的结构我们已经一清二楚了。

相关详细内容可以继续深入查看源码。waitAndQueueCurrentPacket() 可以看具体代码，这时候已经报连接超时异常了。接下来我们调试发送数据到 DataNode。


# 三.DFSOutputStream.DataStreamer 发送 packet

上面已经调试到：waitAndQueueCurrentPacket() ,会将 Packet 发送给 DFSOutputStream.DataStreamer

DFSOutputStream.DataStreamer 是一个线程，里面有个 stage 标识到了哪一步。还有一个 response 用来回复消息。


DataStreamer 会从 dataQueue 中拿出 packet 发送到 pipeline , 相关代码很长。取出部分：
```java
synchronized (dataQueue) {

	 // 发送packet，dataQueue为null，则发送一个心跳
	 if (dataQueue.isEmpty()) {
	   one = createHeartbeatPacket();
	 } else {
	   one = dataQueue.getFirst(); // regular data packet
	 }
	}
	
        // 建立pipeline
        setPipeline(nextBlockOutputStream());
        // 启动ResponseProcessor线程，更新DataStreamer的状态为DATA_STREAMING

// 当前packet是block的最后一个packet，等待接收之前所有packet的ack
      if (one.lastPacketInBlock) {
        // wait for all data packets have been successfully acked
        synchronized (dataQueue) {
          while (!streamerClosed && !hasError && 
              ackQueue.size() != 0 && dfsClient.clientRunning) {
            try {
              // wait for acks to arrive from datanodes
              dataQueue.wait(1000);
            } catch (InterruptedException  e) {
              DFSClient.LOG.warn("Caught exception ", e);
            }
          }
        }
        if (streamerClosed || hasError || !dfsClient.clientRunning) {
          continue;
        }
        stage = BlockConstructionStage.PIPELINE_CLOSE;
      }
       
     
     // 将 packet 从 dataQueue 移到 ackQueue，准备发送packet
      synchronized (dataQueue) {
        // move packet from dataQueue to ackQueue
        if (!one.isHeartbeatPacket()) {
          dataQueue.removeFirst();
          ackQueue.addLast(one);
          dataQueue.notifyAll();
        }
      }
     // 将packet写入pipeline
        one.writeTo(blockStream);
        blockStream.flush();   
    
      // 如果当前packet是最后一个，则继续等待此packet的ack，
      // 然后endBlock
    if (one.lastPacketInBlock) {
        // wait for the close packet has been acked
        synchronized (dataQueue) {
          while (!streamerClosed && !hasError && 
              ackQueue.size() != 0 && dfsClient.clientRunning) {
            dataQueue.wait(1000);// wait for acks to arrive from datanodes
          }
        }
        if (streamerClosed || hasError || !dfsClient.clientRunning) {
          continue;
        }
        endBlock();
      }
```


代码很长也比较杂乱，我们主要看一下 setPipeline(nextBlockOutputStream()); 和 one.writeTo(blockStream);blockStream.flush();  等

第一步是：one = dataQueue.getFirst();  
我们看看取到的 Packet 是什么样子：和我们上面发送的一样，buf=[0,0,0(33个0),11,50..(6个检测码),0,0,(很多0)，97,97,97(600个97)，],lastPacketInBlock=false ...


nextBlockOutputStream 是通过 namenode 得到一个 LocatedBlock ，而 setPipeline 是和 Datanode 的通道。

```java
nextBlockOutputStream(){
lb = locateFollowingBlock(startTime,excluded.length > 0 ? excluded : null); 
-- dfsClient.namenode.addBlock(src, dfsClient.clientName,block, excludedNodes, fileId, favoredNodes);

success = createBlockOutputStream(nodes, storageTypes, 0L, false);

blockStream = out; //给写出流赋值。
}
```

createBlockOutputStream 对于我们第一次调试来说，关注三个就够了。
创建连接： /到 datanode 的 socket连接 Socket[addr=/127.0.0.1,port=50010,localport=58006]，
new Sender(out).writeBlock 建立 block

```java
createBlockOutputStream{}
// 到 datanode 的 socket连接 Socket[addr=/127.0.0.1,port=50010,localport=58006]
s = createSocketForPipeline(nodes[0], nodes.length, dfsClient);
long writeTimeout = dfsClient.getDatanodeWriteTimeout(nodes.length);

OutputStream unbufOut = NetUtils.getOutputStream(s, writeTimeout);
InputStream unbufIn = NetUtils.getInputStream(s);
IOStreamPair saslStreams = dfsClient.saslClient.socketSend(s,
  unbufOut, unbufIn, dfsClient, accessToken, nodes[0]);
unbufOut = saslStreams.out;
unbufIn = saslStreams.in;
out = new DataOutputStream(new BufferedOutputStream(unbufOut,
    HdfsConstants.SMALL_BUFFER_SIZE));
    
-- 发送写block请求，
new Sender(out).writeBlock(blockCopy, nodeStorageTypes[0], accessToken,
dfsClient.clientName, nodes, nodeStorageTypes, null, bcs, 
nodes.length, block.getNumBytes(), bytesSent, newGS,
checksum4WriteBlock, cachingStrategy.get(), isLazyPersistFile);

blockStream = out; //给写出流赋值。
```


回到 run 方法，通过 initDataStreaming ，然后调用 one.writeTo(blockStream); 这个 blockStream 上面已经赋值了，就是通过 socket 建立 的连接。这里只是将packet写入pipeline中的第一个dn。

看看 Packet 的 writeTo 方法，Packet 的成员变量上面写了，是检查点位置，检查长度，数据位置，数据长度，等。

主要步骤：
- 计算 pktLen 头长度+检查长度+数据长度
- 新建Header，包括 packet 是否是最后一个packet的信息等。
- 判断 checksumPos != dataStart 不等于需要删掉中间的空缺数据，`System.arraycopy(buf, checksumStart, buf, dataStart - checksumLen , checksumLen)`
这个是因为 packet 容量是好几万，我们假设 10000，能容纳的 chunk 是 20个左右，所以应该有 20 * 4 = 80 的检查数据。所以前80个位置留给了检查位置，数据从81开始写。
但是如果我们没写满一个 Packet，检查数据就不需要80个，数据也是从 81开始写，这就空缺了一些数据。
- 将 header.getBytes() [0 = 0,1 = 0,2 = 2,3 = 100,4 = 0,5 = 25 ....(一共33个值)] 的值放到 Packet 的头部。
- stm.write(buf, headerStart, header.getSerializedSize() + checksumLen + dataLen);

```java
void writeTo(DataOutputStream stm) throws IOException {
  
  final int dataLen = dataPos - dataStart;  1411 - 541
  final int checksumLen = checksumPos - checksumStart; 
  final int pktLen = HdfsConstants.BYTES_IN_INTEGER + dataLen + checksumLen;
  PacketHeader header = new PacketHeader(
    pktLen, offsetInBlock, seqno, lastPacketInBlock, dataLen, syncBlock);
    
  // checksumPos不等于dataStart时，将checksum移动到data前面，
  // 紧挨着data，为header空出足够的空间
  if (checksumPos != dataStart) {
    // Move the checksum to cover the gap. This can happen for the last
    // packet or during an hflush/hsync call.
    System.arraycopy(buf, checksumStart, buf, 
                     dataStart - checksumLen , checksumLen); 
    checksumPos = dataStart;
    checksumStart = checksumPos - checksumLen;
  }
  
  final int headerStart = checksumStart - header.getSerializedSize();
  assert checksumStart + 1 >= header.getSerializedSize();
  assert checksumPos == dataStart;
  assert headerStart >= 0;
  assert headerStart + header.getSerializedSize() == checksumStart;
  
  // Copy the header data into the buffer immediately preceding the checksum
  // data.
  
  // 将header复制到packet的buf中，组成一个完整的packet
  System.arraycopy(header.getBytes(), 0, buf, headerStart,
      header.getSerializedSize());
  
  // corrupt the data for testing.
  if (DFSClientFaultInjector.get().corruptPacket()) {
    buf[headerStart+header.getSerializedSize() + checksumLen + dataLen-1] ^= 0xff;
  }
  // Write the now contiguous full packet to the output stream.
  // 将buf写入输出流中
  stm.write(buf, headerStart, header.getSerializedSize() + checksumLen + dataLen);
  
  // undo corruption.
  if (DFSClientFaultInjector.get().uncorruptPacket()) {
    buf[headerStart+header.getSerializedSize() + checksumLen + dataLen-1] ^= 0xff;
  }
}
```

我们可以看一下 最后的 stm.write， 通过跟踪我们发现最终调用： channel.write(buf);  是通过 NIO实现的。

最后我们可以看到打印了错误信息： Exception in thread "main" java.io.IOException: All datanodes 127.0.0.1:50010 are bad. Aborting...
这是因为我们调试时间长，连接已经断了。这个错误是哪一步打印的我们后续可以研究研究。

其实这里有个疑问，这个 Packet 是最后一个 Packet，但是 它的 lastPacketInBlock=false？，其实这个确实不是最后一个，后面还要发送一个空的 Packet，只有 37个字节的头信息。

# 四、DataXceiver 线程写入 DataNode

注意：

服务端的调试和客户端调试是反过来的，
需要先 stop-dfs.sh, 然后 `hadoop-daemons.sh start namenode` `hadoop-daemons.sh start secondarynamenode`  启动 namenode，然后我们在 IDEA 中运行 datanode
然后我们通过客户端命令: hadoop fs -copyFromLocal data.txt /tmp/data.txt
另外多线程调试不太方便，每次只能在一个线程打赏断点，否则会跳来跳去很麻烦，我们就在 DataXCerver 的 writeBlock 方法打上断点


以上的流程可以看做是client端，client端将数据发送到dn上，由dn负责将packet写入本地磁盘，并向下一个dn发送。
DataXceiverServer 是在 DataNode 启动的线程，run 方法：

```java
public void run() {
  Peer peer = null;
  while (datanode.shouldRun && !datanode.shutdownForUpgrade) {
    try {
      // 接收client的socket请求
      peer = peerServer.accept();
     
     // 一个 Daemon线程
      new Daemon(datanode.threadGroup,
          DataXceiver.create(peer, datanode, this))
          .start();
    } catch 
}
```
这里先循环接受请求，每次接受到一个就新建一个线程，加入线程组中。每个 DataXceiver 线程的 run 方法就是处理请求的：


```java
dataXceiverServer.addPeer(peer, Thread.currentThread(), this);
peer.setWriteTimeout(datanode.getDnConf().socketWriteTimeout);
InputStream input = socketIn;
try {
  IOStreamPair saslStreams = datanode.saslServer.receive(peer, socketOut,
    socketIn, datanode.getXferAddress().getPort(),
    datanode.getDatanodeId());
  input = new BufferedInputStream(saslStreams.in,
    HdfsConstants.SMALL_BUFFER_SIZE);
  socketOut = saslStreams.out;
} catch 

super.initialize(new DataInputStream(input));

op = readOp();
processOp(op);
```

processOp():

```java
switch(op) {
    case READ_BLOCK:
      opReadBlock();
      break;
    case WRITE_BLOCK:
      opWriteBlock(in);
      break;

    default:
      throw new IOException("Unknown op " + op + " in data stream");
    }
```


```java
private void opWriteBlock(DataInputStream in) throws IOException {
    final OpWriteBlockProto proto = OpWriteBlockProto.parseFrom(vintPrefixed(in));
    final DatanodeInfo[] targets = PBHelper.convert(proto.getTargetsList());
    TraceScope traceScope = continueTraceSpan(proto.getHeader(),
        proto.getClass().getSimpleName());
    try {
      writeBlock(PBHelper.convert(proto.getHeader().getBaseHeader().getBlock()),
          PBHelper.convertStorageType(proto.getStorageType()),
          PBHelper.convert(proto.getHeader().getBaseHeader().getToken()),
          proto.getHeader().getClientName(),
          targets,
          PBHelper.convertStorageTypes(proto.getTargetStorageTypesList(), targets.length),
          PBHelper.convert(proto.getSource()),
          fromProto(proto.getStage()),
          proto.getPipelineSize(),
          proto.getMinBytesRcvd(), proto.getMaxBytesRcvd(),
          proto.getLatestGenerationStamp(),
          fromProto(proto.getRequestedChecksum()),
          (proto.hasCachingStrategy() ?
              getCachingStrategy(proto.getCachingStrategy()) :
            CachingStrategy.newDefaultStrategy()),
            (proto.hasAllowLazyPersist() ? proto.getAllowLazyPersist() : false));
     } finally {
      if (traceScope != null) traceScope.close();
     }
```

这个方法有我们的断点，我们会在里面进行调试 writeBlock ，大致步骤为：

新建 给客户端发确认消息的 replyOut 流，
新建 给其他DataNode 发消息的 mirror 流
新建 BlockReceiver  接收数据，接收到 Packet 后进行判断，如果不是最后一个，写到输出流，如果是最后一个或者长度为0，不做处理。

```java

public void writeBlock(final ExtendedBlock block,
    final StorageType storageType, 
    final Token<BlockTokenIdentifier> blockToken,
    final String clientname,
    final DatanodeInfo[] targets,
    final StorageType[] targetStorageTypes, 
    final DatanodeInfo srcDataNode,
    final BlockConstructionStage stage,
    final int pipelineSize,
    final long minBytesRcvd,
    final long maxBytesRcvd,
    final long latestGenerationStamp,
    DataChecksum requestedChecksum,
    CachingStrategy cachingStrategy,
    final boolean allowLazyPersist) throws IOException {
    
    // 我们简单取一部分代码
    
    // 输出流，现在在 DataXCerver 中，输入流就是客户端写，输出流自然即时发送到客户端的
    final DataOutputStream replyOut = new DataOutputStream(
        new BufferedOutputStream(
            getOutputStream(),
            HdfsConstants.SMALL_BUFFER_SIZE));
    
    }
    
    // 这里一大堆的 mirrorOut 开头的流都是 数据写到 DataNode 后复制副本用的，由于在本地只有一个副本，所以都是 null 
    DataOutputStream mirrorOut = null;  // stream to next target
    DataInputStream mirrorIn = null;    // reply from next target
    Socket mirrorSock = null;           // socket to next target
    String mirrorNode = null;           // the name:port of next target
    String firstBadLink = "";           // first datanode that failed in connection setup
    Status mirrorInStatus = SUCCESS;
    final String storageUuid;
    
    // 新建 BlockReceiver， BlockReceiver 的作用可以好好看注释，另外它的构造方法也有很大的信息量，其中有个 in 输入流，
    // 是在 DataXCerver 的 run 方法中：super.initialize(new DataInputStream(input)); 也就是对输入的 socket 流的二层包装
    blockReceiver = new BlockReceiver(block, storageType, in,
    peer.getRemoteAddressString(),
    peer.getLocalAddressString(),
    stage, latestGenerationStamp, minBytesRcvd, maxBytesRcvd,
    clientname, srcDataNode, datanode, requestedChecksum,
    cachingStrategy, allowLazyPersist)
    
    // 然后是一大堆的副本拷贝，我们本地调试线跳过。
    if (targets.length > 0) {// 跳过}
    
      // 给客户端发一个应答消息
      if (isClient && !isTransfer) {
        if (LOG.isDebugEnabled() || mirrorInStatus != SUCCESS) {
          LOG.info("Datanode " + targets.length +
                   " forwarding connect ack to upstream firstbadlink is " +
                   firstBadLink);
        }
        BlockOpResponseProto.newBuilder()
          .setStatus(mirrorInStatus)
          .setFirstBadLink(firstBadLink)
          .build()
          .writeDelimitedTo(replyOut);
        replyOut.flush();
      }
      
      // 这里开始写数据，
    blockReceiver.receiveBlock(mirrorOut, mirrorIn, replyOut,mirrorAddr, null, targets, false);
    
    // 后续处理
```



我们看一下 blockReceiver.receiveBlock


new PacketResponder
receivePacket()

```java
// 如果是来自客户端而且传输，新建回复的线程
if (isClient && !isTransfer) {
        responder = new Daemon(datanode.threadGroup, 
            new PacketResponder(replyOut, mirrIn, downstreams));
        responder.start(); // start thread to processes responses
      }

// 循环写
while (receivePacket() >= 0) { /* R} 这里就是写所在逻辑了

```

receivePacket 方法：

packetReceiver.receiveNextPacket(in); 接受一个 Packet


```java
receivePacket{
packetReceiver.receiveNextPacket(in);

PacketHeader header = packetReceiver.getHeader();

// 这里把数据写到 副本上面
//First write the packet to the mirror:
    if (mirrorOut != null && !mirrorError) {
      try {
        long begin = Time.monotonicNow();
        packetReceiver.mirrorPacketTo(mirrorOut);
        mirrorOut.flush();
        long duration = Time.monotonicNow() - begin;
        if (duration > datanodeSlowLogThresholdMs) {
          LOG.warn("Slow BlockReceiver write packet to mirror took " + duration
              + "ms (threshold=" + datanodeSlowLogThresholdMs + "ms)");
        }
      } catch (IOException e) {
        handleMirrorOutError(e);
      }
    }
    
    // 得到 数据和 检查数据
    ByteBuffer dataBuf = packetReceiver.getDataSlice();
    ByteBuffer checksumBuf = packetReceiver.getChecksumSlice();
    
    // 如果是最后一个 packet
    if (lastPacketInBlock || len == 0) {
      if(LOG.isDebugEnabled()) {
        LOG.debug("Receiving an empty packet or the end of the block " + block);
      }
      // sync block if requested
      if (syncBlock) {
        flushOrSync(true);
      }
    } else {
        。。。。。
        out.write(dataBuf.array(), startByteToDisk, numBytesToDisk);
    }
    // 这里就行 check 很复杂，部分数据的 check 需要重新计算。
    checksumOut.write(buf);
}
```

这里的代码太复杂，而且调试的时候稍微时间长一点就出现 连接异常。可以通过配置参数解决。
然后得到了 dataBuf 和 checksumBuf，如果不是最后一个，我们需要写到 DataNode 的具体数据中。
到这里整个写的操作就完成。中间的 和客户端心跳应答、失败处理。都没有涉及。简单梳理一下：

# 五、PipeLineAck 和 PacketResponder 信息处理

上面的 one.writeTo(blockStream); 代码是将 Packet 写入到输出流，写出之前有一段代码：

```java
 // send the packet
 synchronized (dataQueue) {
   // move packet from dataQueue to ackQueue
   if (!one.isHeartbeatPacket()) {
     dataQueue.removeFirst();
     ackQueue.addLast(one);
     dataQueue.notifyAll();
   }
 }
```
就是将 Packet 放进 ackQueue 中，很明显这又是一个 消费者生产者。

新建 DataStreamer 的run中，得到输出流，
建立 pipeLine 之后，会有 initDataStreaming 操作，还要启一个新的线程 ResponseProcessor 接收packet的ack，这个线程在initDataStreaming中启动，并更新DataStreamer线程的状态为DATA_STREAMING
ResponseProcessor，在 ResponseProcessor 的run方法：
```java
synchronized (dataQueue) {
  one = ackQueue.getFirst();
}
```
注意每次锁住的都是 dataQueue 对象。
与他对应的是 BlockReceiver 中的 PacketResponder 。 PacketResponder 的 run 方法比较长，主要是按照 packet 的写入顺序发送 ack。

由于调试器的问题，这里一直跳不进去，能看到大概逻辑： 当数据节点顺利处理完数据，而且当前节点处在数据节点中间（收到下游DataNode的消息）
如果 ackQueue 中有数据，获取一个记录，接下来如果当前节点位于数据管道的中间，就在mirror流读取下游的确认，我们在本地调试是没有这一步的。

如果当前节点位于数据管道的最后， 调用 pkt = waitForAckHead(seqno) 从 ackQueue 取出对应的 pkt ，然后调用 lastPacketInBlock = pkt.lastPacketInBlock;
然后是  if (lastPacketInBlock) {finalizeBlock(startTime);} ，然后是 sendAckUpstream ，这个就是给客户端发送 ack 

然后 ResponseProcessor 中收到了 ack，进行处理。逻辑比较简单，ack.readFields(blockReplyStream) 读取 ask，从输入流中读取的ack的seqno与ackQueue中取得的seqno不一样则抛出异常

// 接收到ack后，从ackQueue中移除packet
      synchronized (dataQueue) {
        lastAckedSeqno = seqno;
        ackQueue.removeFirst();
        dataQueue.notifyAll();
        one.releaseBuffer(byteArrayManager);
      }
。


# 6.NameNode 处理

上面我们说了 NameNode 的处理我们没法调试所以我们跳过了NameNode，原因是客户端通过远程的RPC调用执行，这次我们在IDEA启动NameNode，然后查看NameNode是如何处理写文件的。

通过 RPC 调用 NameNodeRpcServer.create -> namesystem.startFile -> startFileInt -> startFileInternal

中间有上锁的步骤，还有 BlockManager 的验证，namesystem 中有一个 dir:FSDirectory 的属性，a pure in-memory data structure ，
里面又有一个 rootDir：INodeDirectory 代表根节点，rootDir 中有一个 List<INode> children 代表所以的文件 ，还有一个 INodeMap 对象，保存。
注意还有一个 INodesInPath 类，从他的方法我们看出，它代表一个文件递归得到所有父目录的 InpPath 文件。

startFileInternal 方法主要步骤如下：
- 根据 src 得到 INodesInPath，再得到 INodeFile 进行检验。
- newNode = dir.addFile(src, permissions, replication, blockSize,holder, clientMachine);
- getEditLog().logOpenFile(src, newNode, overwrite, logRetryEntry); 记录日志
- leaseManager.addLease(newNode.getFileUnderConstructionFeature()

```java
 private BlocksMapUpdateInfo startFileInternal(FSPermissionChecker pc, 
      String src, PermissionStatus permissions, String holder, 
      String clientMachine, boolean create, boolean overwrite, 
      boolean createParent, short replication, long blockSize, 
      boolean isLazyPersist, CipherSuite suite, CryptoProtocolVersion version,
      EncryptedKeyVersion edek, boolean logRetryEntry)

//得到文件
final INodeFile myFile = INodeFile.valueOf(inode, src, true);

try {
      BlocksMapUpdateInfo toRemoveBlocks = null;
      if (myFile == null) {
      // 判断是否创建
        if (!create) {
          throw new FileNotFoundException("Can't overwrite non-existent " +
              src + " for client " + clientMachine);
        }
      } else {
      // 是否覆盖
        if (overwrite) {
          toRemoveBlocks = new BlocksMapUpdateInfo();
          List<INode> toRemoveINodes = new ChunkedArrayList<INode>();
          long ret = dir.delete(src, toRemoveBlocks, toRemoveINodes, now());
          if (ret >= 0) {
            incrDeletedFileCount(ret);
            removePathAndBlocks(src, null, toRemoveINodes, true);
          }
        } else {
        // 存在而且不覆盖的情况？操作 lease
          // If lease soft limit time is expired, recover the lease
          recoverLeaseInternal(myFile, src, holder, clientMachine, false);
          throw new FileAlreadyExistsException(src + " for client " +
              clientMachine + " already exists");
        }
      }

// Always do an implicit mkdirs for parent directory tree.
// 父目录
      Path parent = new Path(src).getParent();
      if (parent != null && mkdirsRecursively(parent.toString(),
              permissions, true, now())) {
              // 添加到 namespace
        newNode = dir.addFile(src, permissions, replication, blockSize,
                              holder, clientMachine);
      }
      // 操作 lease
      leaseManager.addLease(newNode.getFileUnderConstructionFeature()
          .getClientName(), src);

```

INodesInPath：

newNode = dir.addFile(src, permissions, replication, blockSize,holder, clientMachine);

这个 dir 是 FSDirectory 类型的变量，最终调用了 INodeDirectory.addChild(INode) 方法，时间上就是给他的 children add 一个 INode

和 addlease :

```java
synchronized Lease addLease(String holder, String src) {
    Lease lease = getLease(holder);
    if (lease == null) {
      lease = new Lease(holder);
      leases.put(holder, lease);
      sortedLeases.add(lease);
    } else {
      renewLease(lease);
    }
    sortedLeasesByPath.put(src, lease);
    lease.paths.add(src);
    return lease;
  }
```

```java
  /** Get a lease and start automatic renewal */
  private void beginFileLease(final long inodeId, final DFSOutputStream out)
      throws IOException {
    getLeaseRenewer().put(inodeId, out, this);
  }
  
```

调用完 startFileInternal 后，会调用
stat = dir.getFileInfo(src, false, FSDirectory.isReservedRawName(srcArg), true);
实际上就是得到一个 HdfsFileStatus 的对象，返回给客户端。NameNode 穿件文件元数据信息的过程大致如此。

# 七、DataNode 创建文件

上面讲了在 DataNode 的 DataXCeiver 的写数据过程，但是我们忽略了一些和流式接口无关的部分，包括创建数据块等。

实际上客户端 DFSClient 发送创建元数据给 NameNode 以后，就要根据 HdfsFileStatus 创建到 DataNode 的输出流。
在 DataStreamer 的 createBlockOutputStream 方法中创建了 blockStream = out，实际上创建之前有个步骤：
```java
// send the request
new Sender(out).writeBlock(blockCopy, nodeStorageTypes[0], accessToken,
    dfsClient.clientName, nodes, nodeStorageTypes, null, bcs, 
    nodes.length, block.getNumBytes(), bytesSent, newGS,
    checksum4WriteBlock, cachingStrategy.get(), isLazyPersistFile);
```
这个 Sender 和 DataXCerver 一样，继承自 DataTransferProtocol ，我们可以理解为 DataTransferProtocol 是客户端和 DataNode 通信协议，通信实现包含了创建 数据块和发送数据，
而 Sender 和 DataXCerver 就分别是创建数据块和发送数据的实现，分别是 TCP 和 RPC 通信方式实现。

new Sender(out).writeBlock 只是简单的将信息以 ProtoBuf 的格式发送出去。实际上就是发送了一个 op ，这个 op 服务端会收到并解析，然后根据 op 的值调用不同方法，最后调用了：

前面分析过： BlockReceiver.receiveBlock 。里面就有构建输出流写出，但是我们没有相关这个输出流是怎么来的，实际上就是一个文件流。我们看一下 BlockReceiver 的构造方法。
```java
this.block = block;

switch (stage) {
        case PIPELINE_SETUP_CREATE:
          replicaInfo = datanode.data.createRbw(storageType, block, allowLazyPersist);
          datanode.notifyNamenodeReceivingBlock(
              block, replicaInfo.getStorageUuid());
          break;


streams = replicaInfo.createStreams(isCreate, requestedChecksum);
this.out = streams.getDataOut();
this.checksumOut = new DataOutputStream(new BufferedOutputStream(
          streams.getChecksumOut(), HdfsConstants.SMALL_BUFFER_SIZE));
```

这里有个 switch (stage)，实际上写文件都是一个 PIPELINE ，包括输入某个 client - DataNode - DataNode - DataNode ，由于我是本地模式调试，所以感觉不到这种 PIPELINE 的模式而已。

datanode.data 是一个 FSDataSetImpl 类型的变量，控制着整个 DataNode 的文件信息。类型结构大致如下：
```
FSDataSetImpl 
    DataStorage
        bpStorageMap<bp,BlockPoolSliceStorage>
    FSVolumnList
        List<FSVolumnImpl>
             bpSlices<bp, BlockPoolSlice>
    volumeMap:ReplicaMap<bp, Map<blockid, ReplicaInfo>>
```

FSDataSetImpl 代表整个 DataNode 的文件存储，FSVolumnList 是因为我们配置的 data 存储路径，可以用逗号隔开，一般情况下配置1个路径，FSVolumnList 里面就一个 FsVolumeImpl。
FsVolumeImpl 里面有分为 多个 blockpool 存储，一个 blockpool 对应一个文件夹而已，这在一开始版本是没有的。

```
Storage

  List<StorageDirectory> storageDirs
  public int   layoutVersion;   // layout version of the storage data
  public int   namespaceID;     // id of the file system
  public String clusterID;      // id of the cluster
  public long  cTime;   
  static class StorageDirectory
      File root

DataStorage
   private String datanodeUuid = null;
   Map<String, BlockPoolSliceStorage> bpStorageMap
   
```

FsVolumeImpl 


replicaInfo 保存了文件的位置信息，所以可以用来创建输出流。



# 八、租约处理

```java
synchronized void put(final long inodeId, final DFSOutputStream out,
      final DFSClient dfsc) {
    if (dfsc.isClientRunning()) {
      if (!isRunning() || isRenewerExpired()) {
        //start a new deamon with a new id.
        final int id = ++currentId;
        daemon = new Daemon(new Runnable() {
          @Override
          public void run() {
            try {
              if (LOG.isDebugEnabled()) {
                LOG.debug("Lease renewer daemon for " + clientsString()
                    + " with renew id " + id + " started");
              }
              LeaseRenewer.this.run(id);
            } catch(InterruptedException e) {
              if (LOG.isDebugEnabled()) {
                LOG.debug(LeaseRenewer.this.getClass().getSimpleName()
                    + " is interrupted.", e);
              }
            } finally {
              synchronized(LeaseRenewer.this) {
                Factory.INSTANCE.remove(LeaseRenewer.this);
              }
              if (LOG.isDebugEnabled()) {
                LOG.debug("Lease renewer daemon for " + clientsString()
                    + " with renew id " + id + " exited");
              }
            }
          }
          
          @Override
          public String toString() {
            return String.valueOf(LeaseRenewer.this);
          }
        });
        daemon.start();
      }
      dfsc.putFileBeingWritten(inodeId, out);
      emptyTime = Long.MAX_VALUE;
    }
  }
```

最重要的就是  LeaseRenewer.this.run(id)， 在run中调用renew对租约续约。

```java
for(long lastRenewed = Time.now(); !Thread.interrupted();
        Thread.sleep(getSleepPeriod())) {
      final long elapsed = Time.now() - lastRenewed;
      if (elapsed >= getRenewalTime()) {
        try {
          renew();
          if (LOG.isDebugEnabled()) {
            LOG.debug("Lease renewer daemon for " + clientsString()
                + " with renew id " + id + " executed");
          }
          lastRenewed = Time.now();
        } catch (SocketTimeoutException ie) {
          LOG.warn("Failed to renew lease for " + clientsString() + " for "
              + (elapsed/1000) + " seconds.  Aborting ...", ie);
          synchronized (this) {
            while (!dfsclients.isEmpty()) {
              dfsclients.get(0).abort();
            }
          }
          break;
        } catch (IOException ie) {
          LOG.warn("Failed to renew lease for " + clientsString() + " for "
              + (elapsed/1000) + " seconds.  Will retry shortly ...", ie);
        }
      }

```