---
date: 2018-03-22
title: "On Disk IO"
author: "邓子明"
tags:
    - 数据库
    - 架构
categories:
    - 底层原理
comment: true
---


文章来源：https://medium.com/databasss/on-disk-io-part-1-flavours-of-io-8e1ace1de017

了解 IO 原理有助于工作。Network IO 相关问题经常讨论，然而和磁盘相关的问题很少讨论。一方面实现上，网络 IO 更为复杂的巧妙，Filesystem 相关 tools 很少。
另一方面，大家都使用 databases 作为存储系统，和底层打交道的事情交给了数据库设计人员，但是了解相关资料 依然重要。

目录：
- Syscalls: open, write, read, fsync, sync, close
- Standard IO: fopen, fwrite, fread, fflush, fclose
- Vectored IO: writev, readv
- Memory mapped IO: open, mmap, msync, munmap

## Buffered IO

“buffering”  总是让人困惑，当使用 Standard IO，可以选择  full and line buffering 或者 out from any buffering whatsoever，
这些其实和我们要讨论的 Kernel buffering (Page Cache)  没什么关系。你可以吧上面的称之为 caching。

## Sector/Block/Page

块状设备（Block Device ）是为读取提供  HDDs or SSDs 等硬件设备提供 buffered access 的特殊文件类型。
Block Devices 工作于 Sectors（一组相邻的 bytes） 之上，Most disk devices have a sector size of 512 bytes。
Sector 是 block device 之间数据传输的最小单元，读取数据不可能比 Sector 更小，但是却可以同时读 multiple adjacent segments。
File System  最小读取单元就是 Block，Block就是一组相邻的sectors，block sizes 一般为 512, 1024, 2048 and 4096 bytes。
通常 IO操作会通过 Virtual Memory，将 filesystem blocks 缓存到 memory 中作为buffer提供中间操作，
Virtual Memory 又和 pages 一起工作，pages 会 map 到 blocks，Typical page size is 4096 bytes。

总结来说，Virtual Memory 和 pages 一起工作，map 到 Filesystem blocks，blocks 又 map 到 Block Device sectors。

## 	Standard IO

通过 read() and write()  方法操作文件系统。

reading: 首先访问 Page Cache，如果 Page Cache 中没有需要的数据，将会触发 Page Fault 然后将需要的数据 paged in。所以读取 unmapped 文件会慢一些，尽管对用户是透明的。

writes: buffer contents 首先写到 Page Cache，数据并不会立马写进磁盘，真正的 hardware write 是当 Kernel 决定进行  writeback of the dirty page.

## Page Cache

https://github.com/torvalds/linux/blob/master/include/linux/buffer_head.h 
将最近访问过的 fragments 存储起来，read() and write() 操作并不会触发磁盘操作而是通过Page Cache 。

读的时候，读的时候会查看 Page Cache，如果有的话直接给用户，没有的话先加载再给用户，如果满了，least recently used pages 会被刷到磁盘，并且 evicted from cache。

写操作仅仅是将数据放到 Page Cache，marking the written page as dirty。后续会有 flush or writeback 的进程将数据写到磁盘。

标记为dirty 的page会被flush到磁盘，这个过程被称作 writeback。writeback 有缺点，例如 queuing up IO requests。

Page Cache 的理论是 时间本地性原则，也就是刚刚被访问的内容很有可能被再次访问。当然还有空间本地性原则，也就是被访问的数据附近的数据很有可能被访问，prefetch 操作就是机遇这个原则。

Page Cache ，保存了最近访问的和将要访问的 file chunks，read-write-read 的操作可以直接通过缓存，加快了速度。

## Delaying Errors

既然不会立即写到磁盘，就会有 可能突然出错没有写到磁盘的问题，可以查看： https://lwn.net/Articles/457667/

## Direct IO

当我们不想用 Kernel Page Cache。打开文件的时候使用 O_DIRECT 标志，直接读取绕过 Page Cache，写也是直接写到磁盘。
大部分使用 Direct IO 的application 都会导致性能的 下降，但是高手可以通过 fine-grained 操作提高性能，因为会实现自己的缓存，例如 PostgreSQL and MySQL use Direct IO 。

例如 write-ahead-log，一般不会立即查看。同时使用 O_DIRECT 和 PageCache 打开文件会带来很多问题，不鼓励。

## Block Alignment

使用 Direct IO 的时候，最好让操作都对准 sector 的边界上。也就是说，每个操作的 starting offset 是 512 的倍数，缓存大小也是 512 的倍数，
Crossing segment boundary 会导致加载多个sectors。

Page Cache 会在内存自动对齐，所以不需要。


## Memory Mapping

Memory Mapping (mmap) 让你可以像整个文件都放进内存了一样访问文件。