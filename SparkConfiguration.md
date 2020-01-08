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
spark.ui.liveUpdate.period | 100ms | 多久更新一次实体， -1 表示永不更新。
spark.ui.liveUpdate.minFlushPeriod | 1 | 在刷新UI数据之前，最小过期时间。这避免了当task events 更新频率不高的时候，UI 数据也不进行更新。
spark.ui.port | 4040 | application的 SparkUI端口， 显示对应application详情信息的页面。 | <font color="orange">默认从4040开始， 如果已被占用会向后一位。</font>
spark.ui.retainedJobs | 1000 | 在垃圾回收之前 Spark UI 和 状态类API最多记忆多少个jobs，这是最大目标值，在某些情况下 还有一小部分能够被保留下来。
spark.ui.retainedTasks | 100000 | 与上面 类似 不同的是 task。
spark.ui.reverseProxy | false | 如果启用的话， 只能够通过SparkMaster反向代理的方式访问 worker 和 application的UI，这意味着不能够通过 URL直接访问 worker 和 application的UI。 开启需谨慎，一旦开启 对集群中所有 application都有效， 并且需要在所有服务器都设置。 | 默认worker是8081， application是4040， 当启用之后，就不能够通过ip：port直接访问了。
spark.ui.reverseProxyUrl | | 指的是 代理服务器的URL， 必须是完整路径。
spark.ui.showConsoleProgress | false | 在控制台显示进度条，只有执行时间大于500ms的才显示进度条，在shell环境下， 这个参数默认为true
spark.worker.ui.retainedExecutors | 1000 | 在垃圾回收前 最多保存多少个 已经完成的 executor信息。
spark.worker.ui.retainedDrivers | 1000 | 在垃圾回收前 最多保存多少个 已经完成的 driver 信息。
spark.sql.ui.retainedExecutions | 1000 |  在垃圾回收前 最多保存多少个 已经完成的 execution 信息。
spark.streaming.ui.retainedBatches | 1000 | 在垃圾回收前 最多保存多少个 已经完成的 batches 信息。
spark.ui.retainedDeadExecutors | 100 | 在垃圾回收前 最多保存多少个 已经死亡的 executor 信息。
spark.ui.filters | None | SparkUI的过滤器， 多个class之间通过，分割， filter的实现与 javax servlet Filter 标准一致。 可接收参数， 通过 spark.\<class name of filter\>.param.\<param name\>=\<value\> 的方式， 例如 ：spark.ui.filters=com.test.filter1 spark.com.test.filter1.param.name1=foo spark.com.test.filter1.param.name2=bar | <font color="orange">有点意思</font>
spark.ui.requestHeaderSize | 8k | 单位是byte 除非指定， http请求头的最大限制， 对 Spark History Server 同样有效。

#### Compression and Serialization

Property Name | Default | Meaning | 个人理解
-|-|-|-|
spark.broadcast.compress | true | 在发送广播变量之前是否压缩， 默认压缩算法是 spark.io.compression.codec.
spark.checkpoint.compress | false | 检查点是否压缩，算法是  spark.io.compression.codec。
spark.io.compression.codec | lz4 | 压缩编码， 在 Rdd的分区，事件日志， 广播变量， shuffle输出 中都会使用。 Spark 提供 lz4， lzf， snappy， zstd， 也可以通过class全名使用。 比如： org.apache.spark.io.LZ4CompressionCodec
spark.io.compression.lz4.blockSize | 32k | 数据分块的大小， 在使用lz4压缩时，降低这个值同时会降低 shuffle memory的使用。
spark.io.compression.snappy.blockSize | 32k | 使用snappy算法时数据块大小。降低这个值同时会降低 shuffle memory的使用。
spark.io.compression.zstd.level | 1 | 提升zstd的级别， 将会使用更多的内存 cpu，同时会提升压缩效率。
spark.io.compression.zstd.bufferSize | 32k | Zstd压缩时使用的缓存大小， 降低这个值同时会降低 shuffle memory的使用。但是 会增加压缩的开销， 因为调度 JNI的频次变高了。
spark.kryo.classesToRegister | （None） | 通过逗号分隔的 class列表， 在使用Kryo作为序列化方法时， 需要注册class。 | 这部分在 Spark调优有提到，Spark 在RDD处理时 默认使用了Kryo，同时 默认注册了很多类进去。
spark.kryo.referenceTracking | true | 当使用 Kryo 序列化数据时，是否跟踪对同一对象的引用，如果对象图具有循环，并且如果它们包含同一对象的多个副本对效率有用，则这是必需的。如果您知道这不是这样，可以禁用此功能来提高性能。| <font color="orange">有点意思</font>
spark.kryo.registrationRequired | false | 是否需要注册 Kryo。如果设置为 'true'，如果未注册的类被序列化，Kryo 将抛出异常。如果设置为 false（默认值），Kryo 将与每个对象一起写入未注册的类名。编写类名可能会导致显著的性能开销，因此启用此选项可以严格强制用户没有从注册中省略类。
spark.kryo.registrator | （None）| 	如果你采用 Kryo 序列化，则给一个逗号分隔的类列表，自定义注册器 注册自己的class。如果你需要以自定义方式注册你的类，则此属性很有用，例如以指定自定义字段序列化程序。否则，使用 spark.kryo.classesToRegisteris 更简单。它应该设置为 KryoRegistrator 的子类。
spark.kryo.unsafe | false | 是否启用Kryo Unsafe， 可以达到更快的性能， 更高的速率。
spark.kryoserializer.buffer.max | 64m | Kryo序列化时 允许使用的最大缓存， 必须大于你尝试序列化的任何对象， 也必须小于 2048m。当遇到  "buffer limit exceeded" exception时， 则说明需要增大这个值了。| <font color="orange">比较重要</font>
spark.kryoserializer.buffer | 64k | Kryo初始化使用的缓存， 最大增长到 spark.kryoserializer.buffer.max，需要注意的是在每个worker的每个core都有独立的缓存。
spark.rdd.compress | false | Rdd是否压缩， 可以占用更小的内存，消耗更多的CPU。
spark.serializer | org.apache.spark.serializer.JavaSerializer | Spark使用的序列化方法， 对性能要求较高的化， 使用Kryo更好。 org.apache.spark.serializer.KryoSerializer。也可以使用自定义的方式。需要实现或 继承 org.apache.spark.Serializer。| <font color="orange">比较重要</font>
spark.serializer.objectStreamReset | 100 | 当正使用 org.apache.spark.serializer.JavaSerializer 序列化时，序列化器缓存对象虽然可以防止写入冗余数据，但是却阻止这些缓存对象的垃圾回收。通过调用 'reset' 你从序列化程序中清除该信息，并允许收集旧的对象。要禁用此周期性重置，请将其设置为 -1。默认情况下，序列化器会每过 100 个对象被重置一次。

#### Memory Management
Property Name | Default | Meaning
-|-|-|-|
spark.memory.fraction | 0.6 | 用于执行和存储的（堆空间 - 300MB）的分数。这个值越低，溢出和缓存数据逐出越频繁。此配置的目的是在稀疏、异常大的记录的情况下为内部元数据，用户数据结构和不精确的大小估计预留内存。推荐使用默认值。
spark.memory.storageFraction | 0.5 | 存储 使用的内存， 不会被逐出内存的总量，表示为 s​park.memory.fraction 留出的区域大小的一小部分。这个越高，工作内存可能越少，执行和任务可能更频繁地溢出到磁盘。推荐使用默认值。
spark.memory.offHeap.enabled | false | 如果为 true，Spark 会尝试对某些操作使用堆外内存。如果启用了堆外内存使用，则 spark.memory.offHeap.size 必须为正值。
spark.memory.offHeap.size | 0 | 堆外内存可使用的绝对内存。需要同步调整JVM内存，以保证两者加起来不会超过机器可用内存。
spark.memory.useLegacyMode | false |  是否启用 Spark 1.5 及以前版本中使用的传统内存管理模式。传统模式将堆空间严格划分为固定大小的区域，如果未调整应用程序，可能导致过多溢出。只有当本参数启用时，以下选项才可用：spark.shuffle.memoryFraction
spark.shuffle.memoryFraction | 0.2 | 已废弃
spark.storage.memoryFraction | 0.6 | 已废弃
spark.storage.unrollFraction | 0.2 | 已废弃
spark.storage.replication.proactive | false | 针对失败的executor，主动去cache 有关的RDD中的数据。默认false
spark.cleaner.periodicGC.interval | 30min | 执行GC的周期间隔。
spark.cleaner.referenceTracking | true | 是否启用contextCleaner
spark.cleaner.referenceTracking.blocking | true | 清理非Shuffle的其它数据是否是阻塞式的。
spark.cleaner.referenceTracking.blocking.shuffle | false | 清理 shuffle 是否是阻塞式的。
spark.cleaner.referenceTracking.cleanCheckpoints | false | 是否清理引用超出范围的checkpoint 文件。 

> 参考链接：
> 
> [Spark ContextCleaner](https://blog.csdn.net/beliefer/article/details/84998806) 包含部分参数说明， 基本讲解
> 
> [spark源码分析-ContextCleaner缓存清理](https://blog.csdn.net/Shie_3/article/details/81051133) 

#### Execution Behavior
Property Name | Default | Meaning | 个人理解
-|-|-|-|
spark.broadcast.blockSize | 4m | TorrentBroadcastFactory 的一个块的每个分片大小。过大的值会降低广播期间的并行性（更慢了）; 但是，如果它过小，BlockManager 可能会受到性能影响。
spark.broadcast.checksum | true | 是否加入校验和机制， 如果加入， 会在计算和 数据发送上增加一点， 但能够保证在广播过程中的数据准确性，如果自己有其他方式校验数据准确性， 则可以关闭当前选项。 |  <font color="orange">有点意思</font>
spark.executor.cores | 在 YARN 模式下默认为 1，standlone 和 Mesos 粗粒度模型中的 worker 节点的所有可用的 core。 | 在每个 executor（执行器）上使用的 core 数。
spark.default.parallelism | 对于分布式混洗（shuffle）操作，如 reduceByKey 和 join，父 RDD 中分区的最大数量。对于没有父 RDD 的 parallelize 操作，它取决于集群管理器：本地模式：本地机器上的 core 数，Mesos 细粒度模式：8， 其他：所有执行器节点上的 core 总数或者 2，以较大者为准 | 如果用户没有指定参数值，则这个属性是 join，reduceByKey，和 parallelize 等转换返回的 RDD 中的默认分区数。|  <font color="orange">比较重要</font>
spark.executor.heartbeatInterval | 10s | executor 和 driver 之间心跳间隔。 其值要明显小于  spark.network.timeout | <font color="orange">比较重要</font>
spark.files.fetchTimeout | 60s | 获取文件的通讯超时时间，所获取的文件是从驱动程序通过 SparkContext.addFile() 添加的。
spark.files.useFetchCache | true | 如果设置为 true（默认），文件提取将使用由属于同一应用程序的executor共享的本地缓存，这可以提高在同一主机上运行许多执行器时的任务启动性能。如果设置为 false，这些缓存优化将被禁用，所有executor 将获取它们自己的文件副本。如果使用驻留在 NFS 文件系统上的 Spark 本地目录，可以禁用此优化
spark.files.overwrite | false | 当目标文件存在且其内容与源不匹配的情况下，是否覆盖通过 SparkContext.addFile() 添加的文件。
spark.files.maxPartitionBytes | 134217728（128M） | 单partition中最多能容纳的文件大小,单位Bytes。
spark.files.openCostInBytes | 4194304 （4M） | 打开一个文件的开销预估， 用于衡量可在同一时间扫描的字节数。 当在将多个小文件合并到同一个partition时使用， 最好是进行过量估计， 分区中是小文件 会比 大文件要快。
spark.hadoop.cloneConf | false | 如果设置为true，则为每个任务克隆一个新的Hadoop Configuration 对象。应该启用此选项以解决 Configuration 线程安全问题
spark.hadoop.validateOutputSpecs | true | 如果设置为 true，则验证 saveAsHadoopFile 和其他变体中使用的输出规范（例如，检查输出目录是否已存在）。可以禁用此选项以静默由于预先存在的输出目录而导致的异常。我们建议用户不要禁用此功能，除非需要实现与以前版本的 Spark 的兼容性。可以简单地使用 Hadoop 的 FileSystem API 手动删除输出目录。对于通过 Spark Streaming 的StreamingContext 生成的作业会忽略此设置，因为在检查点恢复期间可能需要将数据重写到预先存在的输出目录。
spark.storage.memoryMapThreshold | 2m | 当从磁盘读取块时，Spark 内存映射的块大小。这会阻止 Spark 从内存映射过小的块。通常，存储器映射对于接近或小于操作系统的页大小的块具有高开销。
spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version | 1 | 文件输出 提交的算法版本， 2性能更好， 但是1在某些情况下能够更好地 处理错误。

#### Networking

Property Name | Default | Meaning | 个人理解
-|-|-|-|
spark.rpc.message.maxSize | 128 | 在 “control plane” 通信中允许的最大消息大小（以 MB 为单位）; 一般只适用于在 executors 和 driver 之间发送的映射输出大小信息。如果您正在运行带有数千个 map 和 reduce 任务的作业，并查看有关 RPC 消息大小的消息，请增加此值。
spark.blockManager.port | （random） | 	所有块管理器监听的端口。这些都存在于 driver 和 executors 上。
spark.driver.blockManager.port | （等于spark.blockManager.port） | driver节点的 block manager 监听端口。
spark.driver.bindAddress | (value of spark.driver.host) | 
spark.driver.host | (local hostname) | driver节点的hostName 或 IP地址， 这用于与 executors 和 standalone Master 进行通信。 | 比如 当executor和 driver分处于两台机器上时， 并且无法通过hostName寻址到 driver节点， 就需要指定这个参数。| <font color="orange">比较重要</font>
spark.driver.port | 	(random) | driver节点监听的端口。
spark.rpc.io.backLog | 64 | 接收的 rpc Server 队列的最大长度， 当存在大量的 applications时， 需要增加这个值， 这样当短时间内进入大量连接时， 不至于被中断，或丢弃。
spark.network.timeout | 120s | 所有网络交互的默认超时。如果未配置此项，将使用此配置替换 spark.core.connection.ack.wait.timeout，spark.storage.blockManagerSlaveTimeoutMs，spark.shuffle.io.connectionTimeout，spark.rpc.askTimeout or spark.rpc.lookupTimeout
spark.port.maxRetries | 16 | 在绑定端口放弃之前的最大重试次数。当端口被赋予特定值（非 0）时，每次后续重试将在重试之前将先前尝试中使用的端口增加 1。这本质上允许它尝试从指定的开始端口到端口 + maxRetries 的一系列端口。| 所以即使不小心占用了原有端口也不影响， 因为端口的实际选择范围并不小。
spark.rpc.numRetries | 3 | 在 RPC 任务放弃之前重试的次数.
spark.rpc.retry.wait | 3s | RPC 请求操作在重试之前等待的持续时间。
spark.rpc.askTimeout | spark.network.timeout | RPC 请求操作在超时前等待的持续时间。
spark.rpc.lookupTimeout | 120s | 	RPC 远程端点查找操作在超时之前等待的持续时间。
spark.core.connection.ack.wait.timeout | spark.network.timeout | 连接等待ack响应的最大时长, 为了避免长时间的停顿, 比如GC 导致超时 被丢弃, 可以设置的更大一些.

#### Scheduling

Property Name | Default | Meaning | 个人理解
-|-|-|-|
spark.cores.max | (not set) | 	当以 “coarse-grained（粗粒度）” 共享模式在 standalone deploy cluster 或 Mesos cluster in "coarse-grained" sharing mode 上运行时，从集群（而不是每台计算机）请求应用程序的最大 CPU 内核数量。如果未设置，默认值将是 Spar k的 standalone deploy 管理器上的 spark.deploy.defaultCores，或者 Mesos上的无限（所有可用核心）。
spark.locality.wait | 3s | 全局的数据本地化等待时长 | 查看参考连接
spark.locality.wait.node | spark.locality.wait | node locality 等待时长
spark.locality.wait.process | spark.locality.wait | process locality 等待时长
spark.locality.wait.rack | spark.locality.wait | rack locality 等待时长
spark.scheduler.mode | FIFO | 作业之间的 scheduling mode（调度模式） 提交到同一个 SparkContext。可以设置为 FAIR 使用公平共享，而不是一个接一个排队作业。对多用户服务有用。
spark.scheduler.revive.interval | 1s | worker 恢复 重启的时间间隔，默认1s
spark.scheduler.listenerbus.eventqueue.capacity | 10000 | spark事件监听队列容量，默认10000，必须为正值，增加可能会消耗更多内存
spark.scheduler.blacklist.unschedulableTaskSetTimeout | 120s | 在 因为无法调度导致被彻底列入黑名单 的 taskSet 终止之前，等待获取一个新的 executor 和 分配任务 的时间。| blacklist 参见参考链接。
spark.blacklist.enabled | false | 参见Spark blackList机制，可以通过其他的blacklist相关参数， 辅助，促进 blacklist达到更好的效果。| <font color="orange">比较有趣</font>
spark.blacklist.timeout | 1h | 实验性的，多久之后将对应的 executor从blacklist中移除出来。
spark.blacklist.task.maxTaskAttemptsPerExecutor | 1 | 实验性的，对于给定的任务， 在单个executor最多重试几次之后会被列入blacklist。| 在这里需要注意的是blacklist 并非是对所有资源都有效， 而是不同的 task， stage， application 都有自身的 blacklist。
spark.blacklist.task.maxTaskAttemptsPerNode | 2 | 实验性的， 对于给定的任务， 在单个节点上 最多重试几次会被列入blacklist
spark.blacklist.stage.maxFailedTasksPerExecutor | 2 | 实验性的 对于给定的 stage 在executor被列入 blacklist之前， 必须有多少个不同的task 在同一 executor上失败。
spark.blacklist.stage.maxFailedExecutorsPerNode | 2 | 实验性的， 对于给定的stage， 在节点被列入黑名单前， 要求有多少个 executor被列入黑名单。
spark.blacklist.application.maxFailedTasksPerExecutor | 2 | 与上述类似， 不同的是将 executor 列入 application的黑名单。
spark.blacklist.application.maxFailedExecutorsPerNode | 2 | 相似的， 将 node 列入application的黑名单。
spark.blacklist.killBlacklistedExecutors | false | 是否可以kill application 对应的executor， 如果是 node被加入blacklist， 则kill node上的所有executor。
spark.blacklist.application.fetchFailure.enabled | false | 当发生 fetch failure时， 将executor立即加入 blacklist。如果 external shuffle service 为 enable， 则将整个 node都加入 blacklist。
spark.speculation | false | spark推测机制 | 查看参考链接
spark.speculation.interval | 100ms | 多久去检测一次 task 去执行 推测。
spark.speculation.multiplier | 1.5 |  当一个task的执行时间比 中值 慢多少倍的时候 考虑开启推测机制。
spark.speculation.quantile | 0.75 | 对于特定的stage而言， 要开启推测机制至少需要 task 已经完成的总量比例。
spark.task.cpus | 1 | 对每个task需要申请的cpu数量。
spark.task.maxFailures | 4 | 对于单个task， 而非stage中task总的失败次数。 超过这个值就会放弃当前job。最小值为1， 重试的最大次数为 当前值 - 1. | <font color="orange">比较重要</font>
spark.task.reaper.enabled | false | 表示是否监控被杀死或中断的task， 直到task真正的 完成执行/killed。如果设置为false， 当task被kill 或中断之后， 是没有办法监控其状态的。
spark.task.reaper.pollingInterval | 10s | executor多久下拉一次 被kill的task的状态。



Fraction of tasks which must be complete before speculation is enabled for a particular stage.


	How many times slower a task is than the median to be considered for speculation.



The timeout in seconds to wait to acquire a new executor and schedule a task before aborting a TaskSet which is unschedulable because of being completely blacklisted.


> 参考链接
> 
> [Spark数据本地化-->如何达到性能调优的目的](https://www.cnblogs.com/jxhd1/p/6702224.html?utm_source=itdadao&utm_medium=referral)
>
>> 总而言之， 在任务分配之前先获取各个节点存储的数据， 而后进行任务分配， 分配任务优先： PROCESS_LOCAL， NODE_LOCAL， RACK_LOCAL，NO_PREF， ANY。 Spark数据的本地化 核心在于 移动计算，而不是移动数据

> [Apache Spark 黑名单(Blacklist)机制介绍](https://www.iteblog.com/archives/1907.html)
>
>> 黑名单机制其实是通过维护之前出现问题的执行器（Executors）和节点（Hosts）的记录。当某个任务（Task）出现失败，那么黑名单机制将会追踪这个任务关联的执行器以及主机，并记下这些信息；当在这个节点调度任务出现失败的次数超过一定的数目（默认为2），那么调度器将不会再将任务分发到那台节点。调度器甚至可以杀死那台机器对应的执行器，这些都可以通过相应的配置实现。

> [Spark 推测执行(speculative)](https://www.jianshu.com/p/d9470df93469)
> 
>> 有的task很快就执行完成了，而有的可能执行很长一段时间也没有完成。造成这种情况的原因可能是集群内机器的配置性能不同、网络波动、或者是由于数据倾斜引起的。而推测执行(speculative)就是当出现同一个stage里面有task长时间完成不了任务，spark就会在不同的executor上再启动一个task来跑这个任务，然后看哪个task先完成，就取该task的结果，并kill掉另一个task。其实对于集群内有不同性能的机器开启这个功能是比较有用的。


</font>