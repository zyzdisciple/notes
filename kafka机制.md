# Kafka

## 持久化

kafka性能中, 最重要的一点, 或说其整体的设计思路源于, ***硬盘的性能要远远高于我们的预期***,  而其中的核心则是在于 ***顺序读写*** 与 随机读写之间的巨大性能差异.

在官方文档中提到, 在同等机器条件下, 顺序写 能够达到 600MB/s的速度, 而与此同时, 随机写入仅仅只有 100kb/s, 之间有6000倍的性能差距. 甚至于对于硬盘的顺序读, 有可能要比内存中的随机读 要来的快.

而kafka的核心设计则也是基于此的.

而另一点核心则是在于 ***page Cache***.

> 参考链接:
> 
> [页面缓存（Page Cache）-内存和文件之间的那点事儿（上）](https://zhuanlan.zhihu.com/p/54762255)
>
> [页面缓存（Page Cache）-内存和文件之间的那点事儿（下）](https://zhuanlan.zhihu.com/p/54764446)
>
> [Linux系统中的Page cache和Buffer cache](https://blog.51cto.com/ultrasql/1627647)
>
> [Linux内核学习笔记（八）Page Cache与Page回写](https://blog.csdn.net/damontive/article/details/80552566)
>
> 当数据被从磁盘加载到用户内存中,  需要先加载到pageCache中, 然后再被加载到用户进程内. 对于pageCache而言, 不仅仅会加载需要的那部分数据, 也会进行预加载, 将整个块, 前后的文件都加载进来.
>
> 其次则是: 只要有足够的空闲物理内存，缓存就应该被尽可能的使用。因此，它不依赖于特定的进程，而是一个系统范围的资源.

因此, 一方面来说, kafka通过将数据放在pageCache中, 而不是 用户进程中, 减少了一倍的内存, 其次则是通过紧凑的数据结构, 而非Java对象的方式 又进一步减少了内存消耗.

因此有效内存大大增加, 可以在一个 32G内存的系统上, 占用28~30G内存, 而不用受到GC惩罚.

其次, 当kafka服务停止之后, 缓存在pageCache中的数据, 并不会因为kafka重启, 就需要被重新加载, 原因如前面所提到的, 只要有足够的空间, 缓存就会被尽可能的使用, 而如果没有其他资源要来占用这部分内存, 那么pageCache中缓存的东西自然就不会被释放. 如果是coldCache的话, 那么需要大概十分钟的时间才能加载10GB数据到缓存中.

也因此, kafka不必将数据存储到内存中, 直到内存快要耗尽才将数据写入磁盘, 而是将数据 写入pageCache, 将flush到 磁盘这件事交给系统去做. 

消息传递系统中使用的持久数据结构往往是每个消费者队列有一个关联的BTree或其他通用的随机访问数据结构用于维护关于消息的元数据。

基于文件的简单追加, 读取这种日志的解决方案, kafka的读写也都是极为高效的, 并且读写之间并不相互阻塞, 通过顺序读这种方式, 就可以利用起来庞大的硬盘空间 用以记录日志, 并且保持较高的读写性能.

也正因为得以利用硬盘本身近乎不受限制的存储空间, kafka就不需要删除掉那些已经消费过的数据, 而是将它们保存在硬盘里, 这样也可以使得Consumer更为灵活.

## 效率

尽管磁盘的顺序读写已经解决了一大问题, 但依然有两点成为性能的瓶颈/制约

太多IO操作（too many small I/O operation）和过多的字节拷贝（excessive byte coping）。

特别是当有太多个客户端向内写数据的时候, 带来的IO操作是相当频繁的.

* 对于IO操作过多, kafka采用了 批量发送数据的办法, 即在Producer端, 将数据组合成 "messageSet"的方式, 完成数据的批量发送, 减少网络IO频次, 将间歇性的 随机数据流 转换成 线性的数据流发送, cache, 写入disk.

* 而对于过多的字节拷贝, 常见的数据从文件系统传输至socket经过的路径：

  1. 操作系统将数据从磁盘读入内核空间的页缓冲区；
  2. 应用程序将数据从内核空间读到用户空间缓冲区；
  3. 应用程序再将数据写入内核空间的socket buffer，套接字缓冲区；
  4. 操作系统从socket buffer拷贝数据至NIC buffer网卡缓冲区，之后发送至网络中：

    这样就经过了4次拷贝, 即使使用了 mmp 也依然不够, 只是省去了从 pageCache到应用程序的拷贝.

    而在linux 2.4版本以上, 使用 sendFile, 即可省去从 pageCache 到 SocketCache的拷贝.

    这样文件在加载到pageCache中之后, 仅仅需要拷贝到 NIC Buffer 即可完成数据的发送.

    因此数据可能在没有完成落盘操作的前提下, 就已经能够被Consumer消费到了.

    除此以外, 在broker, producer, consumer 之间采用了统一的标准的 二进制消息格式.

> 参考链接：
>
> [Zero Copy（零拷贝）](https://blog.csdn.net/zero__007/article/details/89970806)

第三点 在于数据压缩

某些场景下，瓶颈并不是CPU或磁盘，而是网络带宽。当然，用户可以不需要kafka的支持压缩每一条消息之后再发送，但这将导致非常低的压缩率。高效的压缩应该是把多条消息压缩成为一条。kafka支持端到端的压缩，一批消息可以被压缩然后发送给broker，在服务端同样保持压缩状态，最终被消费者解压缩。kafka支持GZIP和Snappy,  LZ4 和 ZStandard 这几种压缩协议.

## Producer

在producer 则主要有, 直接将消息发送给topic某一个分区的leader broker, 为了实现这一点, producer可以在任何时候通过kafka的任意nodes 获取server的metadata, 获知topic 对应的partition的leader究竟在哪一台服务器上运行.

对于消息的发送，可以自己指定分区策略（可以指定一个字段作为key，使用它进行hash分区），也可以使用默认的轮询机制）。

第三点是 异步发送, producer为了尽量减少发送次数, 增大每次发送的数据, 采取异步发送的方式, 默认是 缓存 64kb 或 10ms 完成一次发送.
这个参数是可以进行配置的, 这样可以在发送延迟 和 效率之间做一个权衡.

## Consumer

Consumer在每次发送拉请求的时候, 都需要制定一个offset, 从指定的offset开始一次性拉取一批数据.

对于拉模型和推模型的选择而言:

kafka采取了拉模型.
