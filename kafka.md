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

在一个分区中，leader副本所在的节点会记录所有副本的LEO，而follower副本所在的节点只会记录自身的LEO，而不会记录其他副本的LEO。leader副本收到follower副本的FetchRequest请求之后，它会首先从自己的日志文件中读取数据，然后再返回给follower副本数据前先更新follower副本的LEO。对HW而言，各个副本所在的节点都只记录它自身的HW

Kafka的根目录下有cleaner-offset-checkpoint、log-start-offset-checkpoint、recovery-point-offset-checkpoint、replication-offset-checkpoint四个检查点文件。

recovery-point-offset-checkpoint和replication-offset-checkpoint这两个文件分别对应了LEO和HW。Kafka会有一个定时任务负责将所有分区的LEO刷写到恢复点文件recovery-point-offset-checkpoint中，定时周期由broker参数log.flush.offset. checkpoint.interval.ms来配置，默认值为60000。还有一个定时任务负责将所有分区的HW刷写到复制点文件replication-offset-checkpoint中，定时周期由broker端参数replica.high.watermark.checkpoint.interval.ms来配置，默认值为5000

log-start-offset-checkpoint文件对应logStartOffset，用来标识日志的起始偏移量。各个副本在变动LEO和HW的过程中，logStartOffset也可能随之而动。Kafka也有一定定时任务负责将所有分区的logStartOffset书写到起始点文件log-start-offset-checkpoint中，定时周期由broker端参数log.flush.start.offset.checkpoint.interval.ms来配置，默认值为60000

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

 > log.flush.interval.ms, 与 log.flush.interval.messages 类似, 不过一个是从时间, 一个是从数据量的层面来进行衡量的.
 >
 > 默认值: null
 >
 > cluster-wide

其他参数:

* auto.create.topics.enable

 > auto.create.topics.enable, 是否允许自动创建topic, 在使用的时候如果没有对应的topic, 会自动创建.
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

 > log.flush.start.offset.checkpoint.interval.ms, 
  

DEPRECATED 参数:

其中加粗的表示是后来版本的替代参数.

* advertised.host.name

* advertised.port

 > 参数被 advertised.listeners 替代

* **advertised.listeners**

 > advertised.listeners. 发布在zookeeper中的 供客户端使用的 listeners, 在一般情况下仅使用 listeners 即可, [kafka listeners 和 advertised.listeners 的应用](https://segmentfault.com/a/1190000020715650), 即外界访问kafka服务的地址, 如果不设置这个值的话, 默认使用listeners. 与listeners不同的是, 不允许是用0.0.0.0. 
 >
 > per-broker
 
* host.name
 
 > 参数被 listeners 替代
 