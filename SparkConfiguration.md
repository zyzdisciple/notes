<style>


</style>

# SparkConfiguration

<font face="微软雅黑">

这一章节来看看 Spark的相关配置. 并非仅仅能够应用于 SparkStreaming, 而是对于 Spark的各种类型都有支持. 各个不同.

> 官方链接: [Spark Configuration](http://spark.apache.org/docs/latest/configuration.html)
> 中文参考链接: [Spark 配置](http://spark.apachecn.org/#/docs/20)

Spark 提供了三个地方来设置配置参数:

1. Spark properties

    控制着绝大多数的参数设置, 一般可以通过 SparkConf 对象来进行设置, 又或者是通过 Java 系统参数.

    在提交命令中加入 --conf 一般也可以进行设置.

2. 环境变量

    可以为每台机器配置单独的环境变量,  通过在 每个节点上 conf/spark-env.sh 进行配置

3. 日志

    通过 log4j.properties来进行配置.


## Spark Properties

Spark 属性控制大多数应用程序设置，并为每个应用程序单独配置。这些属性可以直接在 SparkConf 上设置并传递给您的 SparkContext。SparkConf 可以让你配置一些常见的属性（例如 master URL 和应用程序名称），以及通过 set() 方法来配置任意 key-value pairs（键值对）。

    val conf = new SparkConf()
             .setMaster("local[2]")
             .setAppName("CountingSheep")
    val sc = new SparkContext(conf)

一些时间类型的配置, 其接收的时间格式如下:

    25ms (milliseconds)
    5s (seconds)
    10m or 10min (minutes)
    3h (hours)
    5d (days)
    1y (years)

一些存储类型的配置, 其接收的格式如下:

    1b (bytes)
    1k or 1kb (kibibytes = 1024 bytes)
    1m or 1mb (mebibytes = 1024 kibibytes)
    1g or 1gb (gibibytes = 1024 mebibytes)
    1t or 1tb (tebibytes = 1024 gibibytes)
    1p or 1pb (pebibytes = 1024 tebibytes)

通常情况下来说 默认的单位都是 bytes, 但也有一小部分的 默认单位是 kb 或 mb, 因此在查看文档时, 需要注意其单位.

然而, 在某些情况下, 我们并不想要通过 硬编码的方式 将配置参数 写在 Spark代码里. 很明显的, 不够灵活.

因此有另一种方式传入参数, 当然, 这种传参的优先级是要低于 将配置写在代码里的.

    ./bin/spark-submit --name "My app" --master local[4] 
    --conf spark.eventLog.enabled=false
    --conf "spark.executor.extraJavaOptions=-XX:+PrintGCDetails -XX:+PrintGCTimeStamps" 
    myApp.jar

对于上面这种传参方式, 除了一些比较特别的参数, 必须参数, 如 master, name这种参数, 其他的都可以通过 --conf 传入. 可以通过  ./bin/spark-submit --help 展示这些可用参数. 

当然, 即使是动态传参, 我们也并不仅仅支持这一种方式.

通过  conf/spark-defaults.conf 也可以进行传参. 不过优先级更低于上一种方式. 将参数名称与值 通过空格隔开即可.

    spark.master            spark://5.6.7.8:7077
    spark.executor.memory   4g
    spark.eventLog.enabled  true
    spark.serializer        org.apache.spark.serializer.KryoSerializer

在设置参数的过程中:

spark-defaults.conf的优先级最低, 其次是 --conf 最高的是 SparkConf.set(), 另外一点需要注意的是, 在Spark的版本迭代中, 旧有参数 可能会被 新的参数名称代替, 这并不意味着, 在新版本中不能够使用旧的名称. 而是新的名称优先级会更高点.

Spark Properties 主要可以分为两类:

一种是与部署有关的.  如  “spark.driver.memory”, “spark.executor.instances”, 这一类的参数不能够通过SparkConf指定, 而是通过 cluster manage 和 cluster mode  来决定的, 因此建议通过 --conf的方式 进行配置.

第二种是Spark运行时配置, 如 “spark.task.maxFailures”, 这种属性就可以通过任意方式进行配置.

配置的参数可以通过 spark ui 的 4040端口进行查看, 需要注意的是, 仅仅包含, 通过以上三种方式设定的属性, 其他类型的配置都只会展示默认值.

### 可用参数

其中比较重要,有趣的 会在个人理解这一栏 标色.

#### Application Properties

Property Name | Default | Meaning | 个人理解
-|-|-|-|
spark.app.name | (none) | application的名称, 在SparkUI 和 log中显示 | 一般通过SparkConf直接指定
spark.master | (none) | Spark cluster master |
spark.driver.cores | 1 | SparkDriver 所使用的核心数, 仅在集群模式有效 | 
spark.driver.maxResultSize | 1g | 限制Spark 的 action算子 在 所有partition中返回的最大内存, 比如 collect操作, (collect 即是将所有分区数据集中到 driver节点中.) 最小为1m, 如果设置为0 表示不进行任何限制. 如果太高的话 很有可能会导致OOM. (取决于你设置的  spark.driver.memory) 所以需要给定合适的值. | <font color="orange">比如 spark.driver.memory 参数设置为了 512M, 而忘记设置这个参数的话, 导致OOM的可能性 就会比较高.</font>
spark.driver.memory | 1g | 当SparkContext初始化的时候, 设置driver进程的内存占用, 需要注意的是, 在client模式下, 一定不要通过 SparkConf直接设定值, 因为此时JVM已经启动了. 需要通过, --driver-memory的方式来指定参数.值| <font color="orange">比较重要</font>
spark.driver.memoryOverhead | driverMemory * 0.10, 最小值为384M | 每个driver中的 OFF-HEAP memory, 默认情况下, 单位是MIB, 目前仅在 YARN 和 K8s下支持. |
spark.executor.memory | 1g | 每个executor所使用的最大内存. | <font color="orange">比较重要</font>
spark.executor.pyspark.memory | Not Set | PySpark的 executor内存.
spark.executor.memoryOverhead | executorMemory * 0.10, with minimum of 384 | 与 spark.driver.memoryOverhead 类似, 且同样只支持 在 Yarn 和 K8s下支持. |
spark.extraListeners | (none) | 通过 , 分隔的 SparkListener的实现类, 初始化 SparkContext 时，这些类的实例会被创建出来，并且注册到 Spark 的监听器上。如果这些类有一个接受 SparkConf 作为唯一参数的构造函数，那么这个构造函数会被调用；否则，就调用无参构造函数。如果没有合适的构造函数，SparkContext 创建的时候会抛异常。 |
spark.local.dir | /tmp | Spark 的"草稿"目录，包括 map 输出的临时文件以及 RDD 存在磁盘上的数据。这个目录最好在本地文件系统中。这个配置可以接受一个以逗号分隔的多个挂载到不同磁盘上的目录列表。注意：Spark-1.0 及以后版本中，这个属性会被 cluster manager 设置的环境变量覆盖：SPARK_LOCAL_DIRS（Standalone，Mesos）或者 LOCAL_DIRS(YARN) | <font color="orange"> 还算不错</font>
spark.logConf | false | SparkContext 启动时是否把生效的 SparkConf 属性以 INFO 日志打印到日志里。| 
spark.submit.deployMode | (none) | Spark driver 程序的部署模式，可以是 "client" 或 "cluster"，意味着部署 dirver 程序本地（"client"）或者远程（"cluster"）在 Spark 集群的其中一个节点上。 | 当然也并非绝对的没有值, 比如在 Spark提交的时候, 根据 提交的 Master的URL不同, 就指定了不同的模式.
spark.log.callerContext | (none) | 当运行在 YARN/HDFS中时, 写入审计日志中的 application信息, 长度由 hadoop.caller.context.max.size 决定, 但通常来说不超过50字符. |
spark.driver.supervise | false | 如果设置为true, 当driver 节点 非0值退出时 会自动重启 driver节点. 仅在 Standalone 和 Mesos 的集群模式下可用. | <font color="orange"> 这个功能比较有用, Spark中 所有的 executor都是注册在 driver中的. 如果driver节点退出, Spark自然就停止运行了. </font>

#### Runtime Environment

Property Name | Default | Meaning | 个人理解
-|-|-|-|
spark.driver.extraClassPath | (none) | driver的额外的 classpath, 同样的 在 client 模式下, 不能够通过SparkConf来进行设置. 而是要通过 --driver-class-path 命令行参数来进行设置. | 对于Spark, 目前已经了解到的 classpath 有 JavaHome, $SPARK_HOME/jars 这两者.
spark.driver.extraJavaOptions | (none) | driver节点的 JavaOptions. 不允许设置 Xmx这样的参数. 一般通过  --driver-java-options 命令行 或 sparkdefaultconfig进行设置 . | <font color="orange">比较重要</font>
spark.driver.extraLibraryPath | (none) | driver 的额外的 lib加载路径, 一般通过 --driver-library-path 或 sparkdefaultconfig进行设置. |
spark.driver.userClassPathFirst | false | 如果设置为true, 当启动driver时, 会优先加载 用户自身的 jars. 用于解决用的 class 和 spark的class冲突的问题. (这是个实验性的参数.)  仅能够在集群模式下使用. |  <font color="orange">比较有趣</font>
spark.executor.extraClassPath | (none) | 与 spark.driver.extraClassPath 类似. 不过是针对 executor的. |
spark.executor.extraJavaOptions | (none) | Spark的 executor的额外参数, 与 spark.driver.extraJavaOptions 类似. |
spark.executor.extraLibraryPath | (none) | 与 spark.driver.extraLibraryPath 类似. |
spark.executor.userClassPathFirst | (false) | 参考 spark.driver.userClassPathFirst 同样是实验性参数.
spark.executor.logs.rolling.maxRetainedFiles | (none) | 滚动日志, 保存的日志的数量, 超出数量的 较早 的日志会被删除. 默认禁用| <font color="orange">比较重要</font>
spark.executor.logs.rolling.enableCompression | false | 是否启用 executor 日志压缩. 默认禁用 | <font color="orange">比较重要</font>
spark.executor.logs.rolling.maxSize | (none) | 日志的最大 bytes. 超过即滚动到下一个文件 | <font color="orange">比较重要</font>
spark.executor.logs.rolling.strategy | (none) | 日志滚动策略, 默认是禁用的, 可以设置为 "time" 或 "size", 如果是 time 需要设置 spark.executor.logs.rolling.time.interval  参数, 如果是size, 需要设置 spark.executor.logs.rolling.maxSize 参数. | <font color="orange">比较重要</font>
spark.executor.logs.rolling.time.interval | daily | 当日志滚动策略设置为 "time" 时的滚动周期. 默认是按天. 还支持其他的方式 如, daily, hourly, minutely, 或者是数值, 表示在几秒内. | <font color="orange">比较重要</font>
spark.executorEnv.[EnvironmentVariableName] | (none) | 通过 指定 EnvironmentVariableName  的方式, 为executor加入环境变量. 用户可以指定多条 EnvironmentVariableName. 如 spark.executorEnv.1 spark.executorEnv.2|
spark.redaction.regex | 
spark.files | | 以,分割的, 文件列表. 被放置在每个executor的工作目录. 可以设定为全局参数. |
spark.jars | | 以, 分隔的 jars列表, 被包含在每个executor 和 driver的classpath中. 可以设定为全局参数.|

#### Shuffle Behavior

Property Name | Default | Meaning | 个人理解
-|-|-|-|
spark.reducer.maxSizeInFlight | 48m | 从每个 Reduce 任务中并行的 fetch 数据的最大大小。因为每个输出都要求我们创建一个缓冲区，这代表要为每一个 Reduce 任务分配一个固定大小的内存。除非内存足够大否则尽量设置小一点。 |
spark.reducer.maxReqsInFlight | Int.MaxValue | 在集群节点上，这个配置限制了远程 fetch 数据块的连接数目。当集群中的主机数量的增加时候，这可能导致大量的到一个或多个节点的主动连接，导致负载过多而失败。通过限制获取请求的数量，可以缓解这种情况。 |
spark.reducer.maxBlocksInFlightPerAddress | Int.MaxValue | 与上面类似, 不过上面限制的是 集群节点, 而这个参数限制的是 host port.
spark.maxRemoteBlockSizeFetchToMem | Int.MaxValue - 512 |  当远程拉取的block 大小超出了阈值之后, 就会被存储在硬盘中, 这是为了防止 block太大, 需要消耗太多的内存. 默认情况下, 这仅仅会在 blocks大于2GB的时候启用.  同样的, 可以将这个值调小,  可以占用更少的内存. 需要注意的是, 无论是 shuffle 和 从远程拉取 block, 这个参数都是有效的. | <font color="orange">而block究竟是什么?  可以参考表格后的描述及链接. </font>
spark.shuffle.file.buffer | 32k | 每个 shuffle 文件输出流的内存大小。这些缓冲区的数量减少了磁盘寻道和系统调用创建的 shuffle 文件。
spark.shuffle.io.maxRetries | 3 | （仅Netty）如果设置了非 0 值，与 IO 异常相关失败的 fetch 将自动重试。在遇到长时间的 GC 问题或者瞬态网络连接问题时候，这种重试有助于大量 shuffle 的稳定性。 | <font color="orange">有点意思</font>
spark.shuffle.io.numConnectionsPerPeer | 1 |（仅Netty）重新使用主机之间的连接，以减少大型集群的连接建立。 对于具有许多硬盘和少量主机的群集，这可能导致并发性不足以使所有磁盘饱和，因此用户可考虑增加此值。
spark.shuffle.compress | true | 是否要对 map 输出的文件进行压缩。默认为 true，
spark.shuffle.io.preferDirectBufs | true | （仅Netty）堆缓冲区用于减少在 shuffle 和缓存块传输中的垃圾回收。对于严格限制的堆内存环境中，用户可能希望把这个设置关闭，以强制Netty的所有分配都在堆上。
spark.shuffle.io.retryWait | 5s | 仅适用于 Netty）fetch 重试的等待时长。默认 15s。计算公式是 maxRetries * retryWait。
spark.shuffle.io.backLog | 64 |  当Application的数量比较大时, 就可能需要调大这个值, 以使 在当前服务 在短时间内有大量链接的时候 保持新进来的链接不被丢弃.
spark.shuffle.service.enabled | false | 目的是在移除 executor 的时候，能够保留 executor 输出的 shuffle 文件. 当 spark.dynamicAllocation.enabled 设置为true的时候, 当前参数也必须设置为 true.
spark.shuffle.service.port | 7337 | shuffle service 的 的 端口.
spark.shuffle.service.index.cache.size | 100m | 在指定的内存中，缓存项所能够占用到的字节数. |
spark.shuffle.maxChunksBeingTransferred | Long.MAX_VALUE | 在 shuffle Service过程中, 最大允许同时 转移的  块的数量. 当达到最大数量时 新进的链接会被关闭, 而后会进行重试, 参数是: spark.shuffle.io.maxRetries and spark.shuffle.io.retryWait. 如果达到相应的最大值, 任务就会失败. |
spark.shuffle.sort.bypassMergeThreshold | 200 | 当ShuffleManager为SortShuffleManager时，如果shuffle read task的数量小于这个阈值（默认是200），则shuffle write过程中不会进行排序操作，而是直接按照未经优化的HashShuffleManager的方式去写数据，但是最后会将每个task产生的所有临时磁盘文件都合并成一个文件，并会创建单独的索引文件。调优建议：使用SortShuffleManager时，且不需要排序操作，将这个参数调大，大于shuffle read task的数量。 | <font color="orange">有点意思,  shuffle可以参考下面的链接. </font>
spark.shuffle.spill.compress | true | 在溢写的时候是否进行压缩, 压缩算法 spark.io.compression.codec. |
spark.shuffle.accurateBlockThreshold | 100 * 1024 * 1024 | 以字节为单位的阈值，在该阈值之上，可以准确记录HighlyCompressedMapStatus中的shuffle块的大小。这有助于通过避免在获取shuffle块时低估shuffle块大小来防止OOM
spark.shuffle.registration.timeout | 5000 | 注册到外部shuffle服务的超时(以毫秒为单位)。
spark.shuffle.registration.maxAttempts | 3 | 当我们注册到外部shuffle服务失败时，我们将重试最大尝试次数。

> block: BlockManager是spark自己的存储系统，RDD-Cache、 Shuffle-output、broadcast 等的实现都是基于BlockManager来实现的，BlockManager也是分布式结构，在driver和所有executor上都会有blockmanager节点，每个节点上存储的block信息都会汇报给driver端的blockManagerMaster作统一管理，BlockManager对外提供get和set数据接口，可将数据存储在memory, disk, off-heap。
> 
> 参考链接:
> 
>  [[spark] BlockManager 解析](https://www.jianshu.com/p/95127b908944)
>  
> [spark block读写流程分析](https://www.cnblogs.com/superhedantou/p/7868053.html)


> shuffle
> 参考链接:
>
> [Spark Shuffle原理、Shuffle操作问题解决和参数调优](https://www.cnblogs.com/arachis/p/Spark_Shuffle.html)
>
>> 至于为什么要进行排序, 我想主要是便于处理, 将相同key的可以直接进行 reduce等相关操作. 否则就需要以Map<Key, List> 形式 存入内存中. 不太合适, 毕竟 Spill已经是在内存不足的情况下发生的.

#### Spark UI

Property Name | Default | Meaning | 个人理解
-|-|-|-|
spark.eventLog.logBlockUpdates.enabled | false | 当更新block 的时候 是否记录相应的事件, 如果打开的话, 日志增长会相当迅速.
spark.eventLog.longForm.enabled | false | 是否使用长 格式的日志记录.
spark.eventLog.compress | false | 是否启用日志压缩 算法使用 spark.io.compression.codec.
spark.eventLog.dir | file:///tmp/spark-events | 如果启用了eventLog 则使用当前文件夹作为 日志的顶级文件夹, 对于不同的application 会创建不同的文件夹.
spark.eventLog.enabled | false | 是否记录Spark的事件, 可用于在 application完成之后重现webui的相关信息.
spark.eventLog.overwrite | false | 是否直接覆盖 Spark的相关文件.
spark.eventLog.buffer.kb | 100k | 日志输出流的缓存.
spark.ui.dagGraph.retainedRootRDDs | Int.MaxValue |  在垃圾回收之前, Spark UI 和 status APIs 记录多少个 DAG图的节点
spark.ui.enabled | true | 是否启用SparkUI
spark.ui.killEnabled | true | 可以通过SparkUI kill application
spark.ui.liveUpdate.period | 

</font>