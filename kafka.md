# 配置文件

当前文档所对应 版本号是2.7.0

## 名词概念

可以跳过这一节直接看后面, 如果需要补充相关概念了解的话, 再返回来看.

### Rebalance

Consumer Group相关的 rebalance 以及 partition leader相关的rebalance。

Rebalance 是kafka Consumer 会触发的一个机制. 即如果在同一个 Consumer Group中存在多个 Consumer Client， kafka能够比较均衡的将不同的数据发送给各个client。

当：
1. 组成员个数发生变化。例如有新的 consumer 实例加入该消费组或者离开组。
2. 订阅的 Topic 个数发生变化。
3. 订阅 Topic 的分区数发生变化。

时， 则会触发rebalance，使得各个Consumer依然能够比较均衡的消费数据。

> 参考链接：[Kafka Consumer 的 Rebalance 机制](https://zhuanlan.zhihu.com/p/98770059?from_voters_page=true)
> 
> [partitionLeader 的 rebalance](https://cloud.tencent.com/developer/article/1630416)

### kafka Controller

在kafka集群中, 有一个用于为集群中的所有主题分区选取领导者副本, 以及, 它还承载着集群的全部元数据信息，并负责讲这些元数据信息同步到其他broker上的 节点.

在kafka中, 集群中的broker并不直接与 zookeeper交互, 而是与 controller进行交互, 获取当前集群中的各种状态信息.

> 参考链接: [kafka的controller模块](https://www.cnblogs.com/boanxin/p/13618431.html)

### kafka副本机制

> 参考链接: 
>
> [Kafka中的HW、LEO、LSO等分别代表什么？](https://www.cnblogs.com/yoke/p/11486196.html)
> 
> [Kafka详解（四）：Kafka副本剖析、可靠性分析](https://blog.csdn.net/qq_40378034/article/details/90599082)
>
> [Kafka科普系列 | 什么是LW和logStartOffset?](https://blog.csdn.net/u013256816/article/details/88939070)
> 
> [Kafka参数图鉴——unclean.leader.election.enable](https://blog.csdn.net/u013256816/article/details/80790185)

在一个分区中，leader副本所在的节点会记录所有副本的LEO，而follower副本所在的节点只会记录自身的LEO，而不会记录其他副本的LEO。leader副本收到follower副本的FetchRequest请求之后，它会首先从自己的日志文件中读取数据，然后再返回给follower副本数据前先更新follower副本的LEO。对HW而言，各个副本所在的节点都只记录它自身的HW

Kafka的根目录下有cleaner-offset-checkpoint、log-start-offset-checkpoint、recovery-point-offset-checkpoint、replication-offset-checkpoint四个检查点文件。

recovery-point-offset-checkpoint和replication-offset-checkpoint这两个文件分别对应了LEO和HW。Kafka会有一个定时任务负责将所有分区的LEO刷写到恢复点文件recovery-point-offset-checkpoint中，定时周期由broker参数log.flush.offset. checkpoint.interval.ms来配置，默认值为60000。还有一个定时任务负责将所有分区的HW刷写到复制点文件replication-offset-checkpoint中，定时周期由broker端参数replica.high.watermark.checkpoint.interval.ms来配置，默认值为5000

log-start-offset-checkpoint文件对应logStartOffset，用来标识日志的起始偏移量。各个副本在变动LEO和HW的过程中，logStartOffset也可能随之而动。Kafka也有一定定时任务负责将所有分区的logStartOffset书写到起始点文件log-start-offset-checkpoint中，定时周期由broker端参数log.flush.start.offset.checkpoint.interval.ms来配置，默认值为60000

### kafka ack机制
 
> 参考链接: [Kafka生产者ack机制剖析](https://www.cnblogs.com/jmx-bigdata/p/13458135.html)

文章已经很详细了, ack有 0, 1, all 三种配置, 对应于 无确认, leader确认, 所有ISR确认三种方式.

Kafka的Broker端提供了一个参数min.insync.replicas,该参数控制的是消息至少被写入到多少个副本才算是"真正写入",该值默认值为1，生产环境设定为一个大于1的值可以提升消息的持久性. 因为如果同步副本的数量低于该配置值(比如仅有一个leader, 没有 fallower的时候)，则生产者会收到错误响应，从而确保消息不丢失.

### kafka日志存储

 > 参考链接:
 >
 > [Kafka日志存储原理](https://www.cnblogs.com/kaleidoscope/p/9877854.html)

Kafka的Message存储采用了分区(partition)，分段(LogSegment)和稀疏索引这几个手段来达到了高效性. 至于日志如何进行分段, 查找等等 可以参考链接中的文档.

 > 补充参考:
 >
 > [Kafka日志段读写解析](https://blog.csdn.net/yessimida/article/details/106980683)
 > 
 > 在博客中提到另一个比较有趣的东西.
 >
 > redis缓存的过期时间？过期时间加个随机数，防止同一时刻大量缓存过期导致缓存击穿数据库. 特地了解了下.
 >
 > [Redis 对过期数据的处理](https://www.cnblogs.com/xuehao/p/13837659.html) 发现redis中并不需要用这种方式, 而是采用惰性删除 + 定期删除的方式解决, 其次在删除时如果占用了过多的cpu时间, 会退出删除过程.

 ### kafka事务

 > 参考链接:
 >
 > [Kafka事务特性详解](https://www.jianshu.com/p/64c93065473e)
 >
 > 其核心点在于 Kafka事务特性是指一系列的生产者生产消息和消费者提交偏移量的操作在一个事务中，或者说是一个原子操作，生产消息和提交偏移量同时成功或者失败。 在kafka内部 也存在一个 transaction 的topic.
 >
 > 生产者发送多条消息可以封装在一个事务中，形成一个原子操作。
 >
 > read-process-write模式：将消息消费和生产封装在一个事务中，形成一个原子操作。
 >
 > 其实现的核心则在于, 后续补充

 ### Delegation tokens

> 参考链接: [Hadoop Delegation Tokens详解【译文】](https://www.jianshu.com/p/617fa722e057)
> 
> [Kafka Delegation Tokens](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/kafka_delegation_tokens.html)
>
> 简要言之则是: kafka 引入了委托 token 作为一种轻量级的身份验证方法，以补充现有的SASL身份验证。
> 可以减小 Kerberos身份验证 带来的开销. 在创建了 Delegation tokens 可以交由其他人使用.
> 
> 以下步骤描述了如何使用委托令牌的基础知识：
> 用户最初通过SASL向Kafka集群进行身份验证，然后使用以下任一方法获取授权令牌：AdminClient API或  kafka-delegation-token 工具。创建委托令牌的委托人是其所有者。
>
> 委托令牌详细信息已安全地传递给Kafka客户端。这可以通过在SSL / TLS加密的连接上发送令牌数据或将其写入安全的共享存储中来实现。
>
> Kafka客户端不使用Kerberos，而是使用委托令牌对代理进行后续身份验证
>
> 该令牌在特定时间段内有效，但可以是：
>
> Renewed
> 委派令牌可以多次更新，直到它的最大寿命到期为止。令牌可以由所有者或所有者在创建时设置为“更新者”的任何其他委托人进行更新。
>
> Revoked
> 授权令牌可以在其到期时间之前被 Revoked。

## Broker Config

在下面有这样几种读写标识：

read-only: 被标记为 read-only 的参数和原来的参数行为一样，只有重启 Broker，才能令修改生效.

per-broker: 被标记为 per-broker 的参数属于动态参数，修改它之后，只会在对应的Broker 上生效.

cluster-wide: 被标记为 cluster-wide 的参数也属于动态参数，修改它之后，会在整个集群范围内生效，也就是说，对所有 Broker 都生效。你也可以为具体的 Broker 修改cluster-wide 参数

对于参数的动态调整: [kafka参数UpdateMode 和 动态调整](https://blog.csdn.net/java_supermanno1/article/details/107167301)

必要参数有：

* broker.id

 > broker.id, 当前服务的ID, 如果没有设置的话, 会自动生成一个唯一ID, 为了避免用户设置的ID, 和 zookeeper生成的ID发生冲突, 在自动生成的时候会从 reserved.broker.max.id + 1 开始, 这也提醒我们,在自行设置ID的时候, 不要大于 reserved.broker.max.id
 >
 > read-only

* zookeeper.connect

 > zookeeper.connect. 可以通过如下几种方式指定zk, 1. host:port, 例如: 192.168.10.25:2181. 也可以指定多个, 如 hostname1:port1,hostname2:port2,hostname3:port3 
 > 
 > 不仅如此,还可以指定将kafka的相关存储在 zk的某个特定路径下. 比如: hostname1:port1,hostname2:port2,hostname3:port3/chroot/path
 >
 > 仅需要在末尾加上 /chroot/path, 使得kafka的相关信息存储在对应路径下.
 > 
 > read-only
 
* log.dirs

 > log.dirs, kafka的数据存储目录, 默认是 null, 可以配置多个目录, 如果没有设置的话, 则使用log.dir
 >
 > read-only
 
重要参数：

* **listeners**

 > listeners, 是用 , 隔开的多个URI, 如果listener name 不是安全协议(SSL, PLAINTEXT)的一种, 则需要在 listener.security.protocol.map 中指明映射关系. 可以设置为 0.0.0.0 表示监听所有. 默认为null
 >
 > per-broker
 
* **log.flush.interval.messages**
 
 > log.flush.interval.messages, kafka的数据一开始就是存储在PageCache上的, 定期flush到磁盘上的, 也就是说, 不是每个消息都被存储在磁盘了, 如果出现断电或者机器故障等, PageCache上的数据就丢失了. 因此这个flush 就有通过数据量 或 时间控制的两种方式, 尽可能避免数据丢失.
 >
 > 默认值: 9223372036854775807
 >
 > cluster-wide
 >
 > 参考链接: [Kafka丢失数据问题优化总结](https://www.cnblogs.com/qiaoyihang/p/9229854.html)

* **log.flush.interval.ms**

 > log.flush.interval.ms, 与 log.flush.interval.messages 类似, 不过一个是从时间, 一个是从数据量的层面来进行衡量的, 因此当这个参数配置为null, 事实上就是将写数据的任务完全交给了磁盘. 至于多久系统会进行一次flush呢? 
 > 参考链接: [Page Cache的落地问题](https://www.jianshu.com/p/ed5900d31f1f)
 >
 > 默认值: null
 >
 > cluster-wide

* **log.retention.hours**

 > log.retention.hours, kafka的log日志保存多久. 与之相似的参数有, log.retention.minutes 和 log.retention.ms 不过不常用, 一般精确到小时已经足够了.  **但是就优先级而言, 是ms最高, 其次是minutes, 最后是hours**. 如果设置为 -1表示没有限制.
 >
 > 默认值: 168 (一周)
 >
 > read-only

 * **log.roll.hours**

 > log.roll.hours, 日志多久滚动一次. 优先级低于 log.roll.ms. 需要注意ms的 UpdateMode 为cluster-wide
 >
 > 默认值: 168(7天)
 >
 > read-only

 * **log.segment.bytes**

 > log.segment.bytes, 单个日志文件大小, 切割策略.
 >
 > 默认值: 1073741824 (1 GB)
 >
 > cluster-wide

 * **message.max.bytes**

 > message.max.bytes, 单 **批次** 消息的最大字节数, 如果开启了数据压缩, 则是压缩后的数据最大限制,  如果你更改了这个值, 但是使用的是 0.10.2版本以前的consumer, 则consumer的 fetch size也需要更改. 在最新的版本中, 数据总是 按照批次进行分组, 为了效率更高. 但在更老的版本里面, 未压缩的数据, 并不会参与分组, 而是以单条数据的形式发送出去的. 
 >
 > 这个参数可以为每个topic单独配置, 使用 max.message.bytes 即可.
 >
 > 默认值: 1048588
 >
 > cluster-wide

 * **offsets.retention.minutes**

 > offsets.retention.minutes, 当consumer group中的 所有consumer都挂掉之后, 他的offset在 彻底废弃之前的保存周期. 如果通过 standalone consumers (通过 consumer.assign) 的方式, offset 只会保存到最后一次提交 + 当前参数 之后 过期.
 >
 > 默认值: 10080 (7天)
 >
 > read-only

 * **default.replication.factor**

 > default.replication.factor, 对于自动创建的topic的默认副本数, 需要注意的是 对于 offset和transaction topic其副本数都为3
 >
 > 默认值: 1
 >
 > read-only

有趣的参数:

* **advertised.listeners**

 > advertised.listeners. 发布在zookeeper中的 供客户端使用的 listeners, 在一般情况下仅使用 listeners 即可, [kafka listeners 和 advertised.listeners 的应用](https://segmentfault.com/a/1190000020715650), 即外界访问kafka服务的地址, 如果不设置这个值的话, 默认使用listeners. 与listeners不同的是, 不允许是用0.0.0.0. 
 >
 > per-broker

* **log.roll.jitter.hours**

 > log.roll.jitter.hours, 优先级低于 log.roll.jitter.ms, 需要注意ms的 UpdateMode是 cluster-wide
 > 
 > 应该是版本变更导致的问题, 在参考连接中的log.segment.ms 参数已经不存在了, 但作用是相同的. 不同之处在于当前版本使用log.roll.hours 或 log.roll.ms 来控制文件自动切割的.
 >
 >参考链接原文 : 再说下rollJitterMs,这其实是个扰动值，对应的参数是log.roll.jitter.ms,这其实就要说到日志段的切分了，log.segment.bytes,这个参数控制着日志段文件的大小，默认是1G，即当文件存储超过1G之后就新起一个文件写入。这是以大小为维度的，还有一个参数是log.segment.ms,以时间为维度切分。
 >
 > 那配置了这个参数之后如果有很多很多分区，然后因为这个参数是全局的，因此同一时刻需要做很多文件的切分，这磁盘IO就顶不住了啊，因此需要设置个rollJitterMs，来岔开它们。
 >
 > 参考链接: [Kafka日志段读写解析](https://blog.csdn.net/yessimida/article/details/106980683)
 >
 > 默认值: 0
 >
 > read-only

 * **min.insync.replicas**

 > min.insync.replicas, 这个参数需要联合其他参数使用, 当 producer 将 ack 设置成 "all" 或者 (-1), 这个参数就指定了 至少要收到 几个副本的 写确认 才考虑本次写入成功的. 如果没有达到这个数量, 则会告诉生产者, 本次写入是失败的.  正是通过这种方式, 保证写入的数据不会丢失. 详情可以参考 kafka ack机制.
 >
 > 需要注意的是, 这个参数默认值为1, 为了在仅有一个leader没有副本的情况下也能够正常使用. 但推荐配置是: 设置同步副本为3, 当前参数,也就是确认数为2, producer 的 ack设置为 all.
 >
 > 默认值: 1
 > 
 > cluster-wide

* **unclean.leader.election.enable**

 > unclean.leader.election.enable, 是否允许ISR列表之外的节点 参与选取, 可以增强可用性, 但是数据的可靠性得不到保证.
 >
 > 默认值: false
 >
 > cluster-wide
 >
 > 参考链接: [Kafka参数图鉴——unclean.leader.election.enable](https://blog.csdn.net/u013256816/article/details/80790185)

* **delegation.token.expiry.time.ms**

 > delegation.token.expiry.time.ms, 在这里有趣的不是这个参数, 而是  delegation.token 代理token. 见名词解释中.
 > 
 > 当前参数表示, 在 token被renewd之前的有效期.
 >
 > 默认值: 86400000 (1 day)
 >
 > read-only

 * **delete.records.purgatory.purge.interval.requests**

 > delete.records.purgatory.purge.interval.requests, 这里比较有趣的概念是: purgatory, 详情见名词解释.
 
其他参数:

* auto.create.topics.enable

 > auto.create.topics.enable, 是否允许自动创建topic, 在使用的时候如果没有对应的topic, 会自动创建.
 >
 > 默认值: true
 >
 > read-only

* delete.topic.enable

 > delete.topic.enable, 当关掉时, 通过admin api无法删除topic
 >
 > read-only

* auto.leader.rebalance.enable

 > auto.leader.rebalance.enable, 是否允许leader节点自动 进行partition rebalance操作。
 >
 > 有两个相关参数: leader.imbalance.check.interval.seconds 和 leader.imbalance.per.broker.percentage
 >
 > read-only
 
* leader.imbalance.check.interval.seconds

 > leader.imbalance.check.interval.seconds, 顾名思义, controller多久检测一次 imbalance情况, 单位秒, 默认为 300.
 >
 > read-only
 
* leader.imbalance.per.broker.percentage

 > leader.imbalance.per.broker.percentage, 当不平衡的比例达到多大的时候, 才进行rebalance.
 >
 > 其过程大致如下：
 > 
 > 对 Partition 的 AR 列表根据 Preferred Replica 进行分组操作。
 >
 > 遍历 Broker，对其上的 Partition 进行处理。
 >
 > 统计 Broker 上的 Leader Replica 和 Preferred Replica 不相等的 Partition 个数。
 >
 > 统计 Broker 上的 Partition 个数。
 >
 > Partition 个数 / 不相等的 Partition 个数，如果大于 10%，则触发重平衡操作；反之，则不做任何处理。

* group.initial.rebalance.delay.ms

 > group.initial.rebalance.delay.ms,  在进行第一次rebalance之前，group 协调器会等待多久， 等待新的consumer加入当前group， 可以用来减少rebalance频次， 但是会延长启动时间。
 >
 > 默认值: 3000 (3 seconds)
 >
 > read-only

* *group.max.session.timeout.ms*

 > group.max.session.timeout.ms, 对于已经注册的 consumer, 其session的最大超时时间,  如果给定了更长的超时时间, 则允许consumer 处理消息的时间更长. 但代价是可能需要更长的时间, 才能够检测出来问题.
 >
 > 默认值: 1800000 (30 minutes)
 >
 > read-only

* *group.min.session.timeout.ms*

 > group.min.session.timeout.ms, 对于已经注册的 consumer, 其session的最小超时时间, 如果配置的更小, 则是 会更频繁地发送心跳, 加大broker资源的开销.
 >
 > *说说个人对这两个参数的理解, 但是没有代码作为依据, min这个参数, 决定了 broker与consumer之间心跳互动的最短间隔, 也即consumer多久发送一次心跳, 而max则表示了, 在多久没有收到consumer的心跳时会判断当前consumer已经超时.*
 >
 > 默认值: 6000 (6 seconds)
 >
 > read-only

* background.threads

 > background.threads, 为各种各样后台任务所开启的最大线程数, 默认为10.
 >
 > cluster-wide
 
* compression.type
 
 > compression.type, 指定topic的数据压缩类型, 包含, ['gzip', 'snappy', 'lz4', 'zstd'], 除了这几种压缩算法以外, 还接收 'uncompressed' 表示不对数据进行压缩, 以及  'producer' 表示保持和 远端的 producer的压缩算法一致.
 >
 > 默认为 producer
 >
 > cluster-wide
  
* control.plane.listener.name

 > control.plane.listener.name, 指定controller所在listener的name. 需要与 listener.security.protocol.map 联合使用, 通常来说会在: listeners 中指定多个listener, 在listener.security.protocol.map 中指定每个listener所使用的网络协议. 而这个参数则是指定了 controller所在的listener 对应的名称.
 >
 > 参考链接: [关于Kafka区分请求处理优先级的讨论](https://www.bbsmax.com/A/1O5Ev4Zyd7/) 比较有趣的一篇文章, 能了解的更多一点.
 >
 > read-only
  
* log.dir

 > log.dir, log.dirs的替代参数, 默认值为 /tmp/kafka-logs
 >
 > read-only
 
* log.flush.offset.checkpoint.interval.ms

 > log.flush.offset.checkpoint.interval.ms, Kafka标记上一次写入磁盘结束点为一个检查点用于日志恢复的间隔. 参考 kafka副本机制名次解释 及其 参考链接.
 >
 > 默认值: 60000, 一分钟, kafka不推荐修改当前值.
 >
 > read-only
 
* log.flush.scheduler.interval.ms

 > log.flush.scheduler.interval.ms, log flusher 检测是否需要将log flush 入disk的频率.
 >
 > 默认值: 9223372036854775807
 >
 > read-only
 
* log.flush.start.offset.checkpoint.interval.ms

 > log.flush.start.offset.checkpoint.interval.ms,  这个参数是定时将 log.start.offset 定时写入文件中的 频率， 在 名词描述中的kafka offset章节有所描述。
 >
 > 默认值： 60000（1min）
 > 
 > read-only

* log.retention.bytes  

 > log.retention.bytes, 当超过当前限制时, 则删除log文件.
 >
 > 默认值: -1 (无限制)
 >
 > cluster-wide

* log.retention.minutes

 > log.retention.minutes, 优先级为2. 如果设置为-1表示无限制
 >
 > 默认值: null
 >
 > read-only

* log.retention.ms

 > log.retention.ms, 优先级为1, 如果设置为-1表示无限制. 需要注意的是, 当前数据地 UpdateMode是 cluster-wide
 >
 > 默认值: null
 >
 > cluster-wide

* log.roll.ms
  
 > log.roll.ms,  功能与log.roll.hours一致, 单位不同, 同时UpdateMode是cluster-wide
 >
 > 默认值: null
 >
 > cluster-wide

* log.roll.jitter.ms

 > log.roll.jitter.ms, 与hours类似, 需要注意UpdateMode 是 cluster-wide
 >
 > 默认值: null
 >
 > cluster-wide

* log.segment.delete.delay.ms

 > log.segment.delete.delay.ms, 在kafka删除log文件时, 一般先采用标记删除, 而后等待 当前参数 时间之后, 再真正删除文件. 目前日志切割有两种方式, 时间 和  大小, 时间通过 log.roll.ms来控制. 大小则通过 log.segment.bytes来控制.
 >
 > 默认值: 60000 (1minute)
 >
 > cluster-wide

* num.io.threads

 > num.io.threads,  进行IO操作的最大线程数, 配置线程数量为cpu核数2倍，最大不超过3倍.
 >
 > 默认值: 8
 >
 > cluster-wide

 * num.network.threads

 > num.network.threads, 接收网络请求, 以及通过网络发送消息, 所启动的最大线程数. 一般延迟较低, 可配置为 :配置线程数量为cpu核数加1
 > 
 > 默认值: 3
 >
 > cluster-wide

 * queued.max.requests

 > queued.max.requests, 最大接收的请求数, 如果超出 则阻塞网络线程.
 >
 > 默认值: 500
 >
 > read-only

* num.recovery.threads.per.data.dir

 > num.recovery.threads.per.data.dir, 在 log.dir 或 log.dirs 中定义的log目录, 对**每个目录**可使用的线程数, 用于在 启动时 recovery使用, 以及shutdown时 flush使用.
 >
 > 默认值: 1
 >
 > cluster-wide

 * num.replica.alter.log.dirs.threads

 > num.replica.alter.log.dirs.threads, 在多个log directory之间移动副本时, 可以开启的最大线程数. (这个操作可能包括了 磁盘IO)
 >
 > 默认值: null
 >
 > read-only

* num.replica.fetchers

 > num.replica.fetchers, 增加这个值能够 增加 follower 从 leader节点 拉取消息的 并行度.
 >
 > 默认值: 1
 >
 > cluster-wide

 * replica.lag.time.max.ms

 > replica.lag.time.max.ms, 用于判断当前副本是否失效, 当超过这么长时间没有向 leader 发送 fetch时, 或者在这么长的时间内没有 追赶上 leader时, 就会将当前 follower移除 ISR.
 >
 > 默认值: 30000(30s)
 >
 > read-only

 * replica.fetch.min.bytes

 > replica.fetch.min.bytes, follower 在每次fetch时的最小字节数, 如果没有达到, 则会等待 replica.fetch.wait.max.ms
 >
 > 默认值: 1
 >
 > read-only

* replica.fetch.wait.max.ms

 > replica.fetch.wait.max.ms, 对每一个 follower 副本产生的 fetch请求 的最大等待时间, 要求当前值必须小于 replica.lag.time.max.ms 防止频繁的收缩或扩充ISR.
 >
 > 默认值: 500
 > 
 > read-only

 * replica.high.watermark.checkpoint.interval.ms

 > replica.high.watermark.checkpoint.interval.ms, 将 high watermark存储到硬盘中的频率.
 >
 > 默认值: 5000 (5s)
 >
 > read-only

 * replica.socket.receive.buffer.bytes

 > replica.socket.receive.buffer.bytes, leader在复制的时候, socket所能接收的最大缓存.
 >
 > 默认值: 65536 (64kb)
 >
 > read-only

 * replica.socket.timeout.ms

 > replica.socket.timeout.ms, 复制的时候, socket链接的最大超时时间, 这个值最小应该等于 replica.fetch.wait.max.ms
 >
 > 默认值: 30000 (30 seconds)
 >
 > read-only

 > 以上可以参考kafka 副本机制. 就会明白 ISR, HW这些概念.

* fetch.max.bytes

 > fetch.max.bytes, 对于单次的fetch请求, 最大会返回的数据大小, 最小要求是 1024
 >
 > 默认值: 57671680 (55 mb)
 >
 > read-only

* offset.metadata.max.bytes

 > offset.metadata.max.bytes,  单个offset信息所占到的最大空间.
 >
 > 默认值: 4096 (4k)
 >
 > read-only

* offsets.commit.required.acks

 > offsets.commit.required.acks, 当commit时, 需要 这么多的 ack 才能接收本次提交. 这个值不建议调整.
 >
 > 默认值: -1
 >
 > read-only

* offsets.commit.timeout.ms

 > offsets.commit.timeout.ms,  在提交offset的时候, 会有延迟, 直到 所有的 offset topic的副本 都收到了提交信息, 或者说 到达这个timeout 时间.
 >
 > 默认值: 5000 (5s)
 >
 > read-only

 * offsets.load.buffer.size
 
 > offsets.load.buffer.size, 当从 offset的 segments中 读取 offset到缓存中, 控制批次的大小. (这是个软限制, 如果记录数过大的话, 会自动调整.)
 >
 > 默认值: 5242880
 >
 > read-only

* offsets.retention.check.interval.ms

 > offsets.retention.check.interval.ms, 多久检查一次过期的 offset.
 >
 > 默认值: 600000 (10 minutes)
 >
 > read-only

* offsets.retention.minutes

 > offsets.retention.minutes, 

* offsets.topic.compression.codec

 > offsets.topic.compression.codec, offset topic的压缩编码方式, 默认是 atomic. 
 >
 > 默认值: 0
 >
 > read-only

* offsets.topic.num.partitions

 > offsets.topic.num.partitions, offset topic所拥有的分区数. 一旦部署之后就不能修改了.
 >
 > 默认值: 50
 >
 > read-only

* offsets.topic.replication.factor

 > offsets.topic.replication.factor, offset topic的复制因子, 当集群数量达不到节点要求时, 内部的 topic会创建失败.
 >
 > 默认值: 3
 >
 > read-only

* offsets.topic.segment.bytes

 > offsets.topic.segment.bytes, offset topic 日志切分大小, 应该相对小一点, 以便于更容易压缩, 和 加载到缓存里.
 >
 > 默认值: 104857600(100MB) 
 >
 > read-only

* request.timeout.ms

 > request.timeout.ms, 用于配置 client的最大超时时间, 用于请求 或者 响应时. 如果达到最大时间, 客户端会重新发送请求, 如果达到最大重试次数, 则会失败.
 >
 > 默认值: 30000 (30 seconds)
 >
 > read-only

* socket.receive.buffer.bytes

 > socket.receive.buffer.bytes, SO_RCVBUF 的值, 如果设置成 -1, 则是选择系统自带的.
 >
 > 默认值: 102400 (100 kb)
 >
 > read-only

* socket.send.buffer.bytes

 > socket.send.buffer.bytes, SO_SNDBUF, 如果设置成 -1, 则是选择系统自带的.
 >
 > 默认值: 102400 (100 kb)
 >
 > read-only

 > 参考链接: [server.properties 文件详解](http://www.kafka.cc/archives/235.html) 以及 [TCP选项之SO_RCVBUF和SO_SNDBUF](https://www.cnblogs.com/kex1n/p/7801343.html)
 >
 > 每个TCP socket在内核中都有一个发送缓冲区和一个接收缓冲区，TCP的全双工的工作模式以及TCP的滑动窗口便是依赖于这两个独立的buffer以及此buffer的填充状态。接收缓冲区把数据缓存入内核，应用进程一直没有调用read进行读取的话，此数据会一直缓存在相应socket的接收缓冲区内。再啰嗦一点，不管进程是否读取socket，对端发来的数据都会经由内核接收并且缓存到socket的内核接收缓冲区之中。read所做的工作，就是把内核缓冲区中的数据拷贝到应用层用户的buffer里面，仅此而已。进程调用send发送的数据的时候，最简单情况（也是一般情况），将数据拷贝进入socket的内核发送缓冲区之中，然后send便会在上层返回。换句话说，send返回之时，数据不一定会发送到对端去（和write写文件有点类似），send仅仅是把应用层buffer的数据拷贝进socket的内核发送buffer中。

* socket.request.max.bytes

 > socket.request.max.bytes, 在一次socket请求中, 最大接收的字节.
 > 
 > 默认值: 104857600 (100 mb)
 >
 > read-only

* transaction.max.timeout.ms

 > transaction.max.timeout.ms, 在事务执行过程中, 所能允许的最大超时时间.
 >
 > 默认值: 900000 (15 minutes)
 >
 > read-only

* transaction.state.log.load.buffer.size

 > transaction.state.log.load.buffer.size, 当从 transaction 日志分片文件中 加载 producer ids 和 事务 到缓存时候的 最大 Batch size 限制. (软限制, 如果records太大的话, 会被覆盖.)
 >
 > 默认值: 5242880
 >
 > read-only

* transaction.state.log.min.isr 

 > transaction.state.log.min.isr, 参数的功能与 min.insync.replicas 一致, 不过是针对于 transaction topic而言的.
 >
 > 默认值: 2
 >
 > read-only

* transaction.state.log.num.partitions

 > transaction.state.log.num.partitions, 对于transaction topic而言, 其partition数量.
 >
 > 默认值: 50
 >
 > read-only

* transaction.state.log.replication.factor

 > transaction.state.log.replication.factor, transaction topic的复制因子. 如果cluster中的节点数量少于当前值, 则创建失败.
 >
 > 默认值: 3
 >
 > read-only

* transaction.state.log.segment.bytes

 > transaction.state.log.segment.bytes, transaction topic的 日志分片大小. 同样应该小一点, 以便能够快速加载到缓存里.
 >
 > 默认值: 104857600 (100 mb)
 >
 > read-only

* transactional.id.expiration.ms

 > transactional.id.expiration.ms,  transaction coordinator在一段时间内接收不到任何 事务状态的更新消息的时候 就会将对应的 transaction id 过期掉. 这个参数同样也对 producer id 的过期有影响, 即当给定的producer id 在最后一次写入操作之后, 多久没有新消息, 就会过期掉 producer id. 需要注意的是, producer id 也有可能很快被删除, 因为当最后写入的数据 因为 topic的保留策略被删除的时候 也会导致 producer id 被删除掉.
 >
 > 默认值: 604800000 (7 days)
 >
 > read-only

* zookeeper.session.timeout.ms

 > zookeeper.session.timeout.ms,  zk的session timeout
 >
 > 默认值: 18000 (18s)
 >
 > read-only

* zookeeper.connection.timeout.ms

 > zookeeper.connection.timeout.ms,  zk的链接超时时间, 如果没有设置则使用 zookeeper.session.timeout.ms
 >
 > 默认值: null
 >
 > read-only

* zookeeper.max.in.flight.requests

 > zookeeper.max.in.flight.requests, client向 zk发送消息 却没有收到确认的请求 的最大数量. 超过当前数量, 则阻塞.
 >
 > 默认值: 10
 >
 > read-only

* zookeeper.set.acl

 > zookeeper.set.acl, 启用ACL.
 >
 > 默认值: false
 >
 > read-only
 >
 > 参考链接: [保护Kafka环境的最佳实践](https://zhuanlan.zhihu.com/p/152520613?from_voters_page=true) , [实战Kafka ACL机制](https://www.cnblogs.com/smartloli/p/9191929.html)
 >
 > Apache Kafka带有现成的授权机制来建立ACL配置，以保护特定的Kafka资源。Kafka ACL存储在ZooKeeper中，并通过kafka-acls命令进行管理。

不够重要的参数

 * broker.id.generation.enable

 > broker.id.generation.enable,  是否自动生成 broker id, 为了避免用户设置的ID, 和 zookeeper生成的ID发生冲突, 在自动生成的时候会从 reserved.broker.max.id + 1 开始, 这也提醒我们,在自行设置ID的时候, 不要大于 reserved.broker.max.id
 >
 > 默认值: true
 >
 > read-only

* broker.rack

 > broker.rack, 指定broker机架信息。若设置了机架信息，kafka在分配副本时会考虑把某个分区的多个副本分配在多个机架上，这样即使某个机架上的broker全部崩溃，也能保证其他机架上的副本可以正常工作.
 >
 > 默认值： null
 >
 > read-only

* connections.max.idle.ms

 > connections.max.idle.ms, 当当前connection空闲多久之后,  server socket 线程会关掉这个连接
 >
 > 默认值: 600000 (10 minutes)
 >
 > read-only

* connections.max.reauth.ms
 
 > connections.max.reauth.ms, 当被设置成一个正值, (默认值 0不是正值.) 一个会话其生命周期不会超过当前配置的值, 当其进行认证的时候, 会被通报给 2.2.0 及 以后版本的 客户端. broker会断开 在生命周期内 没有进行再度认证的连接, 并且在随后的过程中, 连接不能够被用来做 再度认证以外的任何事情. 配置的名称的前缀可以是 listener的前缀 和 sasl机制名称的小写. 例如:  listener.name.sasl_ssl.oauthbearer.connections.max.reauth.ms=3600000
 >
 > 默认值: 0
 > 
 > read-only

* controlled.shutdown.enable

 > controlled.shutdown.enable, 是否允许控制器关闭broker ,若是设置为true,会关闭所有在这个broker上的leader，并转移到其他broker.
 >
 > 默认值: false
 >
 > read-only

* controlled.shutdown.max.retries

 > controlled.shutdown.max.retries, controller的shutdown 会因为各种原因失败, 这个是 失败的最大重试次数.
 >
 > 默认值: 3
 >
 > read-only

* controlled.shutdown.retry.backoff.ms

 > controlled.shutdown.retry.backoff.ms, 在每次重试之前, 系统需要一点时间来从导致失败的状态中恢复过来, 当前配置决定了在重试之前等待多久.
 > 
 > 默认值: 5000 (5s)
 > 
 > read-only

* controller.socket.timeout.ms

 > controller.socket.timeout.ms. controller 对broker的socket超时时间.
 >
 > 默认值: 30000 (30 seconds)
 > 
 > read-only

* delegation.token.master.key

 > delegation.token.master.key,  master的秘钥 用于生成和校验 delegation token. 在所有brokers之间 必须配置相同的 秘钥. 其次如果这个秘钥被 设置为空字符串, 或是不设置, brokers则会禁用 delegation.token 支持.
 >
 > 默认值: null
 >
 > read-only

* delegation.token.max.lifetime.ms

 > delegation.token.max.lifetime.ms, delegation token的最大存活时间. 超出这个时间就不能renew而需要获取新的.
 >
 > 默认值: 604800000 (7 days)
 >
 > read-only

* fetch.purgatory.purge.interval.requests
 
 > fetch.purgatory.purge.interval.requests, 

* group.max.size

 > group.max.size,  一个consumer group中最大能容纳多少 consumers.
 >
 > 默认值:	2147483647
 >
 > read-only

* inter.broker.listener.name

 > inter.broker.listener.name, 在两个broker之间交流所使用的 listener的名称, 如果没有设置, 则会由 security.inter.broker.protocol决定, 需要注意, 这两个参数不能同时设定.
 >
 > 默认值: null
 >
 > read-only

* log.cleaner.backoff.ms

 > log.cleaner.backoff.ms,  当没有日志要清理的时候, 线程休眠时间.
 >
 > 默认值: 15000 (15 seconds)
 >
 > cluster-wide

* log.cleaner.dedupe.buffer.size

 > log.cleaner.dedupe.buffer.size, 对于所有用来检查重复日志的线程, 所能够消耗的总内存.
 >
 > 默认值: 134217728 (128mb)
 >
 > cluster-wide

DEPRECATED 参数:
* advertised.host.name

 > 参数被 advertised.listeners 替代

* advertised.port

 > 参数被 advertised.listeners 替代
 
* host.name
 
 > 参数被 listeners 替代

* port

 > 参数被listeners替代

 * quota.consumer.default

> 无替代参数

* quota.producer.default

 > 无替代参数
 

 未完成

 kafka consumer offset相关理解, 包括 offsets.load.buffer.size

 kafka事务原理补充

 kafka purgatory 补充. 

 > 相关参数: delete.records.purgatory.purge.interval.requests, fetch.purgatory.purge.interval.requests

 时间轮的概念及实现.

