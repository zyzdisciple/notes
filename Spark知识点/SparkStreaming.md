# SparkStreaming(1) ~ SparkStreaming编程指南

<font face="楷体">

之所以写这部分内容的原因是, 无论是网络上可以直接找到的资料, 还是出版的书籍种种, 版本大都在1.6~2.0不等, 且资源零零散散, 需要到处百度, 搜罗资源.

但根据个人开发了一段时间的感觉来看, 会遇到的绝大多数问题, 都可以在官方文档中找到答案.

因此也可以理解为这是官方文档的部分翻译.

个人英文水平有限, 如有错漏欢迎指正.

就目前来看, 主要分为这样几个板块.

1. [Spark Streaming Programming Guide](http://spark.apache.org/docs/latest/streaming-programming-guide.html) 也即SparkStreaming编程指南.

2. [Submitting Applications](http://spark.apache.org/docs/latest/submitting-applications.html) Spark部署发布相关

3. [Tuning Spark](http://spark.apache.org/docs/latest/tuning.html) Spark调优

4. [Spark Configuration](http://spark.apache.org/docs/latest/configuration.html) Spark可用配置, 可选参数.

目前已经有了Spark Streaming的中文翻译. 参考:

> [Spark Streaming编程指南](http://spark.apachecn.org/#/docs/6)

## Spark编程指南

内容本身会比较多, 因此会拆开来, 分多篇介绍.

在这里就不从word count的简单示例开始了, 而是直接从基础概念开始.

## Maven依赖

    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <version>2.4.3</version>
    </dependency>

    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-streaming_2.12</artifactId>
        <version>2.4.3</version>
        <scope>provided</scope>
    </dependency>

而同样当前版本对应的中间件:

    
    Source	Artifact
    Kafka	spark-streaming-kafka-0-10_2.12
    Flume	spark-streaming-flume_2.12
    Kinesis spark-streaming-kinesis-asl_2.12 [Amazon Software License]

而更完整的, 更新的中间件 Maven 仓库路径为:

[Maven repository](https://search.maven.org/search?q=g:org.apache.spark%20AND%20v:2.4.3)

如果觉得欠缺什么, 不妨找找试试.

## 初始化Streaming Context

为了初始化一个 Spark Streaming 程序,一个 StreamingContext 对象必须要被创建出来,它是所有的 Spark Streaming 功能的主入口点.

有两种创建方式:

    import org.apache.spark.*;
    import org.apache.spark.streaming.api.java.*;

    SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
    JavaStreamingContext ssc = new JavaStreamingContext(conf, new Duration(1000));

其中:

appName 是在Spark UI上展示所使用的名称.

master 是一个 [Spark, Mesos or YARN cluster URL](http://spark.apache.org/docs/latest/submitting-applications.html#master-urls), 不了解也没关系, 这部分会在Spark-submit时介绍到.

master 这个指的是Spark项目运行所在的集群. 如果想在本地启动SparkStreaming项目: 可以使用一个特殊的 “local[*]” , 启动Spark的本地模式, *表示会自动检测系统的内核数量. 

然而在集群环境下, 一般不采用硬编码的方式使用spark, 即 setMaster. 我们有更好的方式 通过 spark-submit 在提交时指定master参数即可.

需要注意到的是, 这句代码会在内部创建一个 SparkContext, 可以通过 ssc.sparkContext 访问使用.

batch interval 也即 new Duration(1000) 在这里指的是毫秒值, 还可以采用Durations来创建.

    Durations.seconds(5)

这个时间, 必须根据您的应用程序和可用的集群资源的等待时间要求进行设置.

另一种创建 SparkStreamingContext的方式为:

    import org.apache.spark.streaming.api.java.*;

    JavaSparkContext sc = ...   //existing JavaSparkContext
    JavaStreamingContext ssc = new JavaStreamingContext(sc, Durations.seconds(1));

在已经有了Context之后, 我们需要做的是:

1. 创建Input DStreams, 如kafka就有相应的方法可以创建DStream

2. 对输入流做 转换 处理, 也即我们的功能部分.

3. 开始接收输入并且使用 streamingContext.start() 来处理数据.

4. 使用 streamingContext.awaitTermination() 等待处理被终止(手动或者由于任何错误).

5. 使用 streamingContext.stop() 来手动的停止处理.

同时, 有几点需要注意的地方:

* 一旦一个 context 已经启动,将不会有新的数据流的计算可以被创建或者添加到它.

* 一旦一个 context 已经停止,它不会被重新启动.

* 同一时间内在 JVM 中只有一个 StreamingContext 可以被激活. 也即假设在使用SparkStreaming的同时, 需要依赖 SparkContext 或 SparkSQL等做一些操作, 此时不能重新创建 SparkContext 或是 SparkSQL(因为SparkSQL依然会创建Context.) 需要直接使用ssc.sparkContext. 

* 调用 ssc.stop() 会在停止 SparkStreamingContext的同时 停止 SparkContext. 如果需要仅停止 StreamingContext,需要使用 ssc.stop(false);

* 一个 SparkContext 就可以被重用以创建多个 StreamingContexts,只要前一个 StreamingContext 在下一个StreamingContext 被创建之前停止(不停止 SparkContext, 即使用 ssc.stop(false)).

在这里额外添加一条说明, ssc.stop 可以接收第二个参数, 是指定是否执行完当前批次的剩余数据.

## Discretized Streams

Discretized Stream or DStream 是 Spark Streaming 提供的基本抽象.

有且仅有两种方式创建一个DStream, 第一种是通过 Spark的API去创建流, 第二种是从一个流转换成另一个流.

在内部,一个 DStream 被表示为一系列连续的 RDDs,它是 Spark 中一个不可改变的抽象,distributed dataset.在一个 DStream 中的每个 RDD 包含来自一定的时间间隔的数据,如下图所示.

![](http://spark.apache.org/docs/latest/img/streaming-dstream.png)

应用于 DStream 的任何操作转化为对于底层的 RDDs 的操作.例如,在 先前的示例,转换一个行(lines)流成为单词(words)中,flatMap 操作被应用于在行离散流(lines DStream)中的每个 RDD 来生成单词离散流(words DStream)的 RDDs.如下所示.

![](http://spark.apache.org/docs/latest/img/streaming-dstream-ops.png)

因此对于RDD支持的操作, DStream也基本都支持.

## Input DStreams 和 Receivers(接收器)

输入 DStreams 是代表输入数据是从流的源数据(streaming sources)接收到的流的 DStream.每一个 input DStream(除了 file stream 之外)与一个 Receiver 对象关联,它从 source(数据源)中获取数据,并且存储它到 Spark 的内存中用于处理.

receiver的java代码如下:

    class MyReceiver extends Receiver<String> {

        public MyReceiver(StorageLevel storageLevel) {
            //StorageLevel表示存储级别
            super(storageLevel);
        }

        public void onStart() {
            //1. 启动线程, 打开Socket连接, 准备开始接收数据
            //2. 启动一个非阻塞线程去接收数据.
            //3. 调用Store方法将数据存储到 Spark的内存中, store方法有多种实现,支持将多种多样的数据进行存储.
            //4. 在发生错误或异常时根据自身的处理策略调用stop, restart, reportError 方法.
        }

        public void onStop() {
            //清理各种线程,未关闭的链接等等
        }
    }

Spark Streaming 提供了两种内置的 streaming source(流的数据源).

* Basic sources(基础的数据源):在 StreamingContext API 中直接可以使用的数据源.例如:file systems 和 socket connections.

* Advanced sources(高级的数据源):像 Kafka,Flume,Kinesis,等等这样的数据源.可以通过对应的maven repository 找到依赖.

需要注意到的是, 如果你想要在你的流处理程序中并行的接收多个数据流,你可以创建多个 input DStreams.这将创建同时接收多个数据流的多个 receivers(接收器),然而,一个 Spark 的 worker/executor 是一个长期运行的任务(task),因此它将占用分配给 Spark Streaming 的应用程序的所有核中的一个核(core).

因此,需要记住,一个 Spark Streaming 应用需要分配足够的核(core)(或线程(threads),如果本地运行的话)来处理所接收的数据,以及来运行接收器(receiver(s)).

因此相应的就需要在创建master的时候 不要使用local[1] 或 local 仅分配一个线程, 这将会使得receiver得到一个线程 而对应的程序则没有线程可以处理.

在集群模式下, 则需要分配适当的核心数.

<font color="orange">

而在使用中, 我的数据源是来自于kafka, 使用的是 kafkaUtils.createDirectStream. 而使用的 core数只有1, 或采用 local[1] 也能够正常运行, 这是不是说明上面的说法是错误的呢?

并不是, 由KafkaUtils.createDirectStream 创建的是DStream, 而并非单纯的使用 receiver的方式实现.

如果采用了自定义的 receiver, 那么此时通过 javaSparkContext.receiveData() 的方式创建流, 就至少需要两个线程, 或两个核心才能够正常运行.

</font>

## Basic Sources

* ssc.socketTextStream() 通过Socket来读取数据.

* 通过文件读取数据,  需要注意的是,文件系统必须是与 HDFS API 兼容的文件系统中(即,HDFS,S3,NFS 等),一个 DStream 可以像下面这样创建:

   声明: 对于下面所描述的这种方式,我个人并没有经过验证, 由于个人使用的数据源主要是kafka, mysql, es. 在尝试过程中fileStream并不能直接使用, 因此有以下猜想.

        streamingContext.textFileStream(dataDirectory);
    
   而windows的文件系统是 NTFS,这就要求我们有对应的文件环境才行.

   >参考:[Hadoop入门系列(一)Window环境下搭建hadoop和hdfs的基本操作](https://blog.csdn.net/qq_32938169/article/details/80209083)

   Spark Streaming 将监控dataDirectory 目录 以及 该目录中任何新建的文件 (写在嵌套目录中的文件是不支持的)

   需要注意的几个地方有:

   * 可以直接监控路径, 也可以通过目录的通配符来监控,hdfs://namenode:8040/logs/ 或 hdfs://namenode:8040/logs/2017/* 都是可以的.

   * 文件必须具有相同的数据格式.

   * 在读取文件的时候,参考的时间是 文件的更新时间,而非创建时间.

   * 一旦被加载,即使文件被更新了 在当前窗口内 也不会被重新读取, 因此即使文件不停被追加新的内容, 但是并不会读入.

   * 文件越多,检索文件变更情况所需要的时间也就越多,即使大多数文件都没有变更.

   * 如果使用的是通配符的方式 去识别文件目录,如: hdfs://namenode:8040/logs/2016-*, 在这种情况下, 通过重命名文件夹 也可以将对应文件夹下的文件 加入被监控的文件列表中, 当然需要修改时间在对应的window内.

   * 通过调用 FileSystem.setTimes() (hadoop api) 可以更改文件的更新时间.

   以下部分是对文件流的一个说明, 如何将文件转换成DStream的概念？

   >参考: org.apache.spark.streaming.dstream.FileInputDStream scala api

       *                      |<----- remember window ----->|
       * ignore threshold --->|                             |<--- current batch time
       *                      |____.____.____.____.____.____|
       *                      |    |    |    |    |    |    |
       * ---------------------|----|----|----|----|----|----|-----------------------> Time
       *                      |____|____|____|____|____|____|
       *                             remembered batches

   依然是按照时间批次来将数据转换成RDDs,整合成 DStream.

   在每个时间批次中,检测 被监控的文件列表 如果修改时间在 current batch范围内的, 在纳入列表, 转换成DStream, 在excutor的执行期间 新加入的文件,放入下一批次进行处理.

   而文件的修改时间 在 ignore threshold 之后的,则会被忽略掉.

   要求:

   * 运行Spark的系统时间 要与对应的 文件系统时间保持一致.

   * 文件必须在相同的文件系统下通过 atomically(原子的)moving(移动) 或 renaming(重命名) 到数据目录.

   而duration的定义则是通过:

   spark.streaming.fileStream.minRememberDuration

   默认是一分钟, 即读取修改时间在一分钟以内的文件.

   更细节的可以自行解读代码实现.

* Queue of RDDs as a Stream(RDDs 队列作为一个流)
  
  为了使用测试数据测试 Spark Streaming 应用程序,还可以使用 streamingContext.queueStream(queueOfRDDs) 创建一个基于 RDDs 队列的 DStream,每个进入队列的 RDD 都将被视为 DStream 中的一个批次数据,并且就像一个流进行处理.

## Advanced Sources(高级数据源)

Kafka: Spark Streaming 2.4.3 要求 Kafka versions 0.8.2.1 以上版本.

官方参考链接: [Kafka Integration Guide](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)

> 个人参考链接: [SparkStreaming-Kafka集成](https://www.cnblogs.com/zyzdisciple/p/11578462.html)

## Custom Sources(自定义数据源)

DStreams 可以使用通过自定义的 receiver(接收器)接收到的数据来创建.

[Spark Streaming Custom Receivers](http://spark.apache.org/docs/latest/streaming-custom-receivers.html)

receiver的大致创建过程在上面已经提到过了.

案例代码实现:

[JavaCustomReceiver.java](https://github.com/apache/spark/blob/v2.4.3/examples/src/main/java/org/apache/spark/examples/streaming/JavaCustomReceiver.java).

通过如下方式使用 自定义的 Receiver
    
    // Assuming ssc is the JavaStreamingContext
    JavaDStream<String> customReceiverStream = ssc.receiverStream(new JavaCustomReceiver(host, port));
    JavaDStream<String> words = customReceiverStream.flatMap(s -> ...);

## Receiver Reliability(接收器的可靠性)

有两种receiver, 可靠性, 不可靠性, 区别就在于对于数据的失败处理上, 可靠receiver并不会丢失数据,而不可靠receiver则不对 数据安全性 提供任何保障.

从数据源上来说, 本身就存在两种数据源, 如 kafka flume所提供的可靠性数据源.能够对 下发处理数据的 响应  做出相应的处理. 而不可靠数据源, 只负责下发数据.

如果想要实现一个 可靠的 receiver, 需要注意的是, 即使采用的是可靠数据源, 也不一定就是可靠的receiver.

如果你想实现一个可靠的数据接收器,必须用store方法,这是一个阻塞的方法,在它存入spark内部时才会返回值.如果接受器用的Storage Level 是复制(也可以使用默认),那么在复制完后才会得到返回值.因此,在确定完数据可靠存储之后,再适当的发送确认信息.这就避免了如果在存储中途失败,也不会导致数据丢失.因为缓冲区的数据没有被确认,那么数据源将会重新发送.

如果是不可靠接收器,那么无须以上逻辑,他只是简单地接收数据并存储到Spark内存中而已,但并非说不可靠的 接收器就毫无优点:

   * 系统会自动把数据分割为 大小合适的 块
   * 如果限制速率已经被指定, 那么系统会自动控制接收速率
   * 由于上面提到的优点, 因此实现起来更为简单.

而与之相对的, 可靠的接收器就需要自己实现数据分块, 以及速率控制, 而实现方式主要取决于数据源.

## DStreams 上的 Transformations(转换)

DStreams 支持很多在 RDD 中可用的 transformation 算子, 至于transformation 和 action 算子的区别, 可以自行百度了解.

> 参考链接: [spark算子](https://www.jianshu.com/p/9556f04b9fd9)

而更详细的需要查看官方API

> 参考链接: [http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaRDD.html](http://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaRDD.html)

其中返回值为各种RDD的一般都是 transformation算子, 否则为 action算子.

在茫茫多的 transformation中 选择几个比较特别的来详细说明下:

### UpdateStateByKey

updateStateByKey 操作允许你维护任意状态,同时不断更新新信息.你需要通过两步来使用它:

1.  定义 state - state 可以是任何的数据类型.
2.  定义 state update function(状态更新函数)- 使用函数指定如何使用先前状态来更新状态,并从输入流中指定新值.

在每个 batch 中,Spark 会使用状态更新函数为所有已有的 key 更新状态,不管在 batch 中是否含有新的数据.如果这个更新函数返回一个 none,这个 key-value pair 也会被消除.

    Function2<List<Integer>, Optional<Integer>, Optional<Integer>> updateFunction =
    (values, state) -> {
        Integer newSum = ...  // add the new values with the previous running count to get the new count
        return Optional.of(newSum);
    };

    JavaPairDStream<String, Integer> runningCounts = pairs.updateStateByKey(updateFunction);

请注意,使用 updateStateByKey 需要配置的 checkpoint(检查点)的目录.

<font color="orange">

但是, updateStateByKey 有不可避免的缺点.

>参考: [[spark streaming] 状态管理 updateStateByKey&mapWithState](https://www.jianshu.com/p/3e4b9c5c6e02)

总结来说:

updateStateByKey底层是将preSateRDD和parentRDD进行co-group,然后对所有数据都将经过自定义的mapFun函数进行一次计算,即使当前batch只有一条数据也会进行这么复杂的计算,大大的降低了性能,并且计算时间会随着维护的状态的增加而增加.

mapWithstate底层是创建了一个MapWithStateRDD,存的数据是MapWithStateRDDRecord对象,一个Partition对应一个MapWithStateRDDRecord对象,该对象记录了对应Partition所有的状态,每次只会对当前batch有的数据进行跟新,而不会像updateStateByKey一样对所有数据计算.
</font>

因此才会有 mapWithstate 的性能远远高于 updateStateByKey

### Transform 算子

这是Spark中 自由度最高的一个算子, Spark官方API提供的算子毕竟是有限的, 可能确实不能够满足你的要求, 因此才会有了这个 transform算子.

其核心作用是:

允许在 DStream 运行任何 RDD-to-RDD 函数.它能够被用来应用任何没在 DStream API 中提供的 RDD 操作.例如,连接数据流中的每个批(batch)和另外一个数据集的功能并没有在 DStream API 中提供,然而你可以简单的利用 transform 方法做到.

    import org.apache.spark.streaming.api.java.*;
    // RDD containing spam information
    JavaPairRDD<String, Double> spamInfoRDD = jssc.sparkContext().newAPIHadoopRDD(...);

    JavaPairDStream<String, Integer> cleanedDStream = wordCounts.transform(rdd -> {
        rdd.join(spamInfoRDD).filter(...); // join data stream with spam information to do data cleaning
        ...
    });

transform方法在每个批次都会进行调用, 因此可以根据不同时间进行相应的处理.

但需要注意的一点是:

虽然transform是  transformation算子, 但是 并不意味着其中的方法必然是在job分配完, 真实提交之后才执行.

原因也正是在于这个算子的灵活性相当高. 可以在其中嵌入任何RDD操作.

> 参考链接: [Spark Streaming 误用.transform(func)函数导致的问题解析](https://www.jianshu.com/p/69c64a920da0)

导致其问题的根本原因就在于 在 transform中执行的 action操作, 是会在 生成job的时候执行的.

经过测试，还会有这样一点问题：

首先transform 确确实实是会在 job生成的时候执行相关代码，如果有action的话， 并且使用的线程也是 job generator线程。 其次， 在transform之前的算子 执行次序是会在transform之前的， 如在transform 之前有过filter， 那么filter一定是在transform之前执行的。

在这一点上， 我更倾向于 transform中的 action 提前引发了 transform算子之前的 算子 执行运算。而并没有等到 后续的 真正的 dstream的action触发时再执行。

然而这并不意味着我们可以省掉后续的action算子。 如果没有后续的 dstream的action算子， 生成job的举动也不会有， 因此更不会触发transform中的action。

### Window

Spark Streaming 也支持 windowed computations(窗口计算),它允许你在数据的一个滑动窗口上应用 transformation(转换).下图说明了这个滑动窗口.

![](http://spark.apache.org/docs/latest/img/streaming-dstream-window.png)

如上图显示,窗口在源 DStream 上 slides(滑动),合并和操作落入窗内的源 RDDs,产生窗口化的 DStream 的 RDDs.在这个具体的例子中,程序在三个时间单元的数据上进行窗口操作,并且每两个时间单元滑动一次.这说明,任何一个窗口操作都需要指定两个参数.

window length(窗口长度) - 窗口的持续时间.

sliding interval(滑动间隔) - 执行窗口操作的间隔.

这两个参数必须是 source DStream 的 batch interval(批间隔)的倍数.

    // Reduce last 30 seconds of data, every 10 seconds
    JavaPairDStream<String, Integer> windowedWordCounts = pairs.reduceByKeyAndWindow((i1, i2) -> i1 + i2, Durations.seconds(30), Durations.seconds(10));

常用的 window操作如下:

Transformation(转换)| Meaning(含义)
-|-|-
window(windowLength, slideInterval)	| 返回一个新的 DStream,它是基于 source DStream 的窗口 batch 进行计算的.
countByWindow(windowLength, slideInterval) | 返回 stream(流)中滑动窗口元素的数
reduceByWindow(func, windowLength, slideInterval) | 返回一个新的单元素 stream(流),它通过在一个滑动间隔的 stream 中使用 func 来聚合以创建.该函数应该是 associative(关联的)且 commutative(可交换的),以便它可以并行计算
reduceByKeyAndWindow(func, windowLength, slideInterval, [numTasks]) | 在一个 (K, V) pairs 的 DStream 上调用时,返回一个新的 (K, V) pairs 的 Stream,其中的每个 key 的 values 是在滑动窗口上的 batch 使用给定的函数 func 来聚合产生的.Note(注意): 默认情况下,该操作使用 Spark 的默认并行任务数量(local model 是 2,在 cluster mode 中的数量通过 spark.default.parallelism 来确定)来做 grouping.您可以通过一个可选的 numTasks 参数来设置一个不同的 tasks(任务)数量.
reduceByKeyAndWindow(func, invFunc, windowLength, slideInterval, [numTasks]) | 上述 reduceByKeyAndWindow() 的更有效的一个版本,其中使用前一窗口的 reduce 值逐渐计算每个窗口的 reduce值.这是通过减少进入滑动窗口的新数据,以及 “inverse reducing(逆减)” 离开窗口的旧数据来完成的.一个例子是当窗口滑动时”添加” 和 “减” keys 的数量.然而,它仅适用于 “invertible reduce functions(可逆减少函数)”,即具有相应 “inverse reduce(反向减少)” 函数的 reduce 函数(作为参数 invFunc </ i>).像在 reduceByKeyAndWindow 中的那样,reduce 任务的数量可以通过可选参数进行配置.请注意,针对该操作的使用必须启用 checkpointing.
countByValueAndWindow(windowLength, slideInterval, [numTasks]) | 在一个 (K, V) pairs 的 DStream 上调用时,返回一个新的 (K, Long) pairs 的 DStream,其中每个 key 的 value 是它在一个滑动窗口之内的频次.像 code>reduceByKeyAndWindow</code> 中的那样,reduce 任务的数量可以通过可选参数进行配置.

如果可以, 并且一般都可以调用 reduceByKeyAndWindow 的第二个版本, 描述了那么多,其实旨在说明, 当采用滑动窗口的时候, 有两种对数据的处理方式, 其一是  每次都去统计最新的 窗口中的 所有数据. 其二则是, 在原有数据基础上做出一定更新,这就要求 对已经离开窗口的数据做 ‘减量’ 操作, 对新进入窗口的数据做 ‘增量’ 操作.也即需要提供的 参数:

func 对数据做reduce操作, 完成统计更新

invFunc 对数据做‘减量’ 操作 在原有基础上进行更新.

### Join

join 与 Sql中的 Join极为类似, 在 SQL中 join操作,要以 join on “on”后面的参数为准, 而在 Spark中, 则要求 进行join 的两个 RDD 或 DStream 都必须是 PairRDD 或是 PairDStream. 

这样在进行join操作的时候, SQL中的 “on” 在Spark中就变成了, Tuple2中的第一个参数.

需要注意的是 join操作最终合并成一个流, 因此也会将多个分区的数据进行合并, 是一次窄依赖变换, 因此最终会形成新的分区.同时一般可以自行制定分区数, 如果不指定, 则使用Spark默认的分区数.

>至于Spark默认的分区数: 参考链接:[Spark RDD的默认分区数:(spark 2.1.0)](https://www.jianshu.com/p/4b7d07e754fa)

其核心就是:

sc.defaultParallelism = spark.default.parallelism
sc.defaultMinPartitions = min(spark.default.parallelism,2)

而RDD的分区数,则是 如果在重新分区时指定了分区数, 则采用分区数, 否则,就使用默认值.

另外, 分区的默认方式/规则 是 HashPartition

1. Join

    Join:Join类似于SQL的inner join操作,返回结果是前面和后面集合中配对成功的,过滤掉关联不上的.

2. leftOuterJoin
 
    leftOuterJoin类似于SQL中的左外关联left outer join,返回结果以前面的RDD为主,关联不上的记录为空.从返回类型上就可略知一二.

        JavaPairDStream[K, (V, Optional[W])]

    其第二个参数是 Optional, 可以接收空值.

3. rightOuterJoin
   
    rightOuterJoin类似于SQL中的有外关联right outer join,返回结果以参数也就是右边的RDD为主,关联不上的记录为空.

        JavaPairDStream[K, (Optional[V], W)]

4. fullOuterJoin

    fullOuterJoin相比前几种来说并不常见,是 左外 右外连接的 结合. 最终的结果 是两个 流的并集. 返回的数据类型是:

        JavaPairDStream[K, (Optional[V], Optional[W])]

对于Join还有一种不错的用法:

    JavaPairRDD<String, String> dataset = ...
    JavaPairDStream<String, String> windowedStream = stream.window(Durations.seconds(20));
    JavaPairDStream<String, String> joinedStream = windowedStream.transform(rdd -> rdd.join(dataset));

可以将RDD 与 流数据进行join操作, 进而完成流数据与 固定数据集的合并.

实际上,您也可以动态更改要加入的 dataset.提供给 transform 的函数是每个 batch interval(批次间隔)进行评估,因此可以将 dataset 引用指向当前的 dataset.

偶然间看到的案例:

>[Spark Streaming 流计算优化记录(2)-不同时间片数据流的Join](https://blog.csdn.net/butterluo/article/details/47083891)

主要解决的就是 来自 HDFS的数据 与 流数据合并.

但在这个案例中是不是有一种更合理的处理方式? 即每隔固定时间去 更新dataset, 在transform中 将流的RDD 与 HDFS的数据合并.

如果是不同的 batch interval 之间的数据可以合并吗?

目前来看并没有看到这种可能性, 对于 SparkContext, 同时只能够启动一个 SparkStreamingContext, 而在SparkStreamingContext启动时就需要指定 batch interval, 因此好像不太可能出现 多个 batchInterval.

倒不如说是 JavaPairRDD 与 JavaPairDStream 之间的合并.

### DStreams 上的输出操作

输出操作 | 解释
-|-|-
print() | 在运行流应用程序的 driver 节点上的DStream中打印每批数据的前十个元素.这对于开发和调试很有用.
saveAsTextFiles(prefix, [suffix]) | 将此 DStream 的内容另存为文本文件.每个批处理间隔的文件名是根据 前缀 和 后缀:"prefix-TIMEIN_MS[.suffix]"_ 生成的.
saveAsObjectFiles(prefix, [suffix]) | 将此 DStream 的内容另存为序列化 Java 对象的 SequenceFiles.每个批处理间隔的文件名是根据 前缀 和 后缀:"prefix-TIMEIN_MS[.suffix]"_ 生成的.
saveAsHadoopFiles(prefix, [suffix]) | 将此 DStream 的内容另存为 Hadoop 文件.每个批处理间隔的文件名是根据 前缀 和 后缀:"prefix-TIMEIN_MS[.suffix]"_ 生成的.
foreachRDD(func) | 对从流中生成的每个 RDD 应用函数 func 的最通用的输出运算符.此功能应将每个 RDD 中的数据推送到外部系统,例如将 RDD 保存到文件,或将其通过网络写入数据库.请注意,函数 func 在运行流应用程序的 driver 进程中执行,通常会在其中具有 RDD 动作,这将强制流式传输 RDD 的计算.

<font color="orange"> 需要注意的是: foreachRDD 本身是运行在 Driver节点的, 而通常要对 RDD做相应的 action 操作. 而这部分操作则是在 各自的 work 上执行的. </font>

### foreachRDD 设计模式的使用

错误操作:

    dstream.foreachRDD(rdd -> {
        Connection connection = createNewConnection(); // 在driver节点执行
        rdd.foreach(record -> {
            connection.send(record); // 在 worker节点执行.
        });
    });

而Connection往往又无法被序列化, 因此在 worker节点上依然拿不到连接.

    dstream.foreachRDD(rdd -> {
        rdd.foreach(record -> {
            Connection connection = createNewConnection();
            connection.send(record);
            connection.close();
        });
    });

因此往往需要采用如下这种方式:

    dstream.foreachRDD(rdd -> {
        rdd.foreachPartition(partitionOfRecords -> {
            // ConnectionPool is a static, lazily initialized pool of connections
            Connection connection = ConnectionPool.getConnection();
            while (partitionOfRecords.hasNext()) {
            connection.send(partitionOfRecords.next());
            }
            ConnectionPool.returnConnection(connection); // return to the pool for future reuse
        });
    });

如果仅仅是想单纯的处理数据, 但并不在这一就需要进行foreachRDD 用以完成流的执行计算呢?

伪代码如下:

    dstreamTemp = dstream.foreachRDD(rdd -> {
        rdd.foreachPartition(partitionOfRecords -> {
            Connection connection = ConnectionPool.getConnection();
            while (partitionOfRecords.hasNext()) {
                //进行查询数据, 填充 补全, 过滤 当前stream中的数据, 用以下一步继续处理.
            }
            ConnectionPool.returnConnection(connection); // return to the pool for future reuse
        });
    });

    dstreamTemp.nextAction;

但foreachRDD 是不会返回流的, 因此可以采用

    dstreamTemp = dstream.mapPairtition();
    dstreamTemp = dstream.mapToPairPartition();

另外关于 foreachRDD 需要注意的是:

DStreams 通过输出操作进行延迟执行,就像 RDD 由 RDD 操作懒惰地执行.具体来说,DStream 输出操作中的 RDD 动作强制处理接收到的数据.因此,如果您的应用程序没有任何输出操作,或者具有 dstream.foreachRDD() 等输出操作,而在其中没有任何 RDD 操作,则不会执行任何操作.系统将简单地接收数据并将其丢弃.

默认情况下,输出操作是 one-at-a-time 执行的, 且按照它们在应用程序中定义的顺序执行.

## DataFrame and SQL

Spark SQL在以后的篇章在详细介绍,在流数据上使用 DataFrame 和 SQL也是一件很简单的事情.

首先需要通过 SparkStreaming 获取对应的 SparkContext, 而后通过SparkContext创建 SparkSession. 并且必须创建, 这样就可以在 driver出现故障的时候, 重新启动. 我们可以将 Spark RDD 转换成 DataFrame,注册为临时表,而后进行SQL查询. 案例代码如下:

    /** Java Bean class for converting RDD to DataFrame */
    public class JavaRow implements java.io.Serializable {
        private String word;

        public String getWord() {
            return word;
        }

        public void setWord(String word) {
            this.word = word;
        }
    }

    ...

    /** DataFrame operations inside your streaming program */

    JavaDStream<String> words = ... 

    words.foreachRDD((rdd, time) -> {
        // Get the singleton instance of SparkSession
        SparkSession spark = SparkSession.builder().config(rdd.sparkContext().getConf()).getOrCreate();

        // Convert RDD[String] to RDD[case class] to DataFrame
        JavaRDD<JavaRow> rowRDD = rdd.map(word -> {
            JavaRow record = new JavaRow();
            record.setWord(word);
            return record;
        });
        DataFrame wordsDataFrame = spark.createDataFrame(rowRDD, JavaRow.class);

        //可以理解为 定义表名
        wordsDataFrame.createOrReplaceTempView("words");

        // Do word count on table using SQL and print it
        DataFrame wordCountsDataFrame =
            spark.sql("select word, count(*) as total from words group by word");
        wordCountsDataFrame.show();
    });

代码全连接: [source code](https://github.com/apache/spark/blob/v2.4.4/examples/src/main/java/org/apache/spark/examples/streaming/JavaSqlNetworkWordCount.java)

并不仅仅是这些, SparkSQL的另一大优点是, 能够跨线程 访问 其他Streaming Data定义的 DataFrame.

需要主动将 SparkStreaming 设置为记住足量数据. 因为 对于 当前线程的 StreamingData 并不知道此时还有来自其他线程的SQL查询, 会自动删除清理 已经不需要的数据.

例如,如果要查询最后一个批次,但是您的查询可能需要5分钟才能运行,则可以调用 streamingContext.remember(Minutes(5))

## 缓存 / 持久性

与 RDD 类似,DStreams 还允许开发人员将流的数据保留在内存中.也就是说,在 DStream 上使用 persist() 方法会自动将该 DStream 的每个 RDD 保留在内存中.如果 DStream 中的数据将被多次计算(例如,相同数据上的多个操作),这将非常有用.对于基于窗口的操作,如 reduceByWindow 和 reduceByKeyAndWindow 以及基于状态的操作,如 updateStateByKey,这是隐含的.因此,基于窗口的操作生成的 DStream 会自动保存在内存中,而不需要开发人员调用 persist().

对于通过网络接收数据(例如:Kafka,Flume,sockets 等)的输入流,默认持久性级别被设置为将数据复制到两个节点进行容错.

请注意,与 RDD 不同,DStreams 的默认持久性级别将数据序列化在内存中.

spark的计算是lazy的,只有在执行action时才真正去计算每个RDD的数据.要使RDD缓存,必须在执行某个action之前定义RDD.persist().

而在使用完毕之后, 最好也能够主动调用 unpersist() 释放内存, 当然, 并不意味着, 如果不主动调用, 就不会释放内存, 它会遵循 LRU原则, 进行内存的释放, 无效cache的删除.

>参考:[Spark cache的用法及其误区分析](https://www.imooc.com/article/35448)

在参考文档中,有一点我有不同的意见:

> cache之后一定不能立即有其它算子,不能直接去接算子.因为在实际工作的时候,cache后有算子的话,它每次都会重新触发这个计算过程.

测试代码如下:

    JavaSparkContext jsc = createJavaSparkContext();
    List<Integer> list = new ArrayList<>();
    for (int i = 0; i < 16; i++) {
        list.add(i);
    }

    JavaRDD<Integer> rdd = jsc.parallelize(list);
    JavaRDD<Integer> rdd1 = rdd.filter(item -> {
        System.out.println("rdd1:" + item);
        return item > 3;
    }).cache();
    JavaRDD<Integer> rdd2 = rdd1.filter(item -> {
        System.out.println("rdd2:" + item);
        return item > 6;
    }).cache();
    rdd2.count();

输出结果并没有将 rdd1:1 重复输出两遍,也即意味着 cache之后有算子, 只会将cache之后的算子进行计算, 而已经计算过的并不会导致重复计算. 因此我们可以放心使用.

## CheckPoint机制

鉴于 SparkStreaming 必须永久运行, 因此对于 <font color="orange">程序无关的错误, 如系统错误, JVM崩溃等问题.</font>具有可恢复的功能.因此 Spark必须保存部分信息, 到容错存储系统. check point有两种类型:

* Metadata checkpointing - 将定义 streaming 计算的信息保存到容错存储(如 HDFS)中,元数据包括:

    * Configuration - 用于创建流应用程序的配置.
    * DStream operations - 定义 streaming 应用程序的 DStream 操作集.
    * Incomplete batches - 已经进入队列,但是未完成的 Job的 批次.

* Data checkpointing - 将生成的 RDD 保存到可靠的存储.这在 基于 批次之间数据具有关联性 上的数据处理是有必要的, 如 reduceStateByKey,在这种转换中,生成的 RDD 依赖于先前批次的 RDD,这导致依赖链的长度随时间而增加.为了避免恢复时间的这种无限增加(与依赖关系链成比例),有状态转换的中间 RDD 会定期 checkpoint 到可靠的存储(例如 HDFS)以切断依赖关系链.

    同俗来说, 就是为了防止 在恢复数据时, 需要从第一个RDD开始重新计算状态,导致计算时间过长, 且保存的依赖链太长, 会在中间进行截断, 保存中间点 RDD的状态, 这样在恢复时就无需从最开始进行恢复处理.

### 使用CheckPoint的时机

checkPoint 并非总是必要的,当我们依赖的是可靠数据源,(又或者是丢失部分数据 也无所谓) 并且有自己的方式能够 查找到上次执行的 offset, 则完全无需checkpoint, 此时只需要自行再度拉取数据, 处理数据即可.

1. 如果在应用程序中使用 updateStateByKey或 reduceByKeyAndWindow(具有反向功能),则必须提供 checkpoint 目录以允许定期的 RDD checkpoint.

2. 从运行应用程序的 driver 的故障中恢复 - 元数据 checkpoint 用于使用进度信息进行恢复.

### 如何使用checkPoint

首先你需要有一个 容错的 可靠的 文件系统, 比如 HDFS,S3 去保存你的checkpoint信息.

而后调用 streamingContext.checkpoint(checkpointDirectory), 通过这种方式 就可以使用上面提到的 updateStateByKey 或 reduceByKeyAndWindow(具有反向功能) 这类的状态计算.

另外,如果要使应用程序从 driver 故障中恢复,您应该重写 streaming:

* 当程序第一次启动时,它将创建一个新的 StreamingContext,设置所有流,然后调用 start().
* 当程序在失败后重新启动时,它将从 checkpoint 目录中的 checkpoint 数据重新创建一个 StreamingContext.

        // Create a factory object that can create and setup a new JavaStreamingContext
        JavaStreamingContextFactory contextFactory = new JavaStreamingContextFactory() {
            @Override public JavaStreamingContext create() {
                JavaStreamingContext jssc = new JavaStreamingContext(...);  // new context
                JavaDStream<String> lines = jssc.socketTextStream(...);     // create DStreams
                ...
                jssc.checkpoint(checkpointDirectory);                       // set checkpoint directory
                return jssc;
            }
        };

        //  这行代码即能够满足上述两种行为.
        JavaStreamingContext context = JavaStreamingContext.getOrCreate(checkpointDirectory, contextFactory);

        // Do additional setup on context that needs to be done,
        // irrespective of whether it is being started or restarted
        context. ...

        // Start the context
        context.start();
        context.awaitTermination();

在上面的方法中, 如果是 第一次启动, 会调用 contextFactory 创建 Streaming 环境, 同时需要在 factory中创建对应的流, 以及定义流的处理过程. 如果是第二次启动, 则会通过 checkpoint 恢复程序当时的状态.

这是故障恢复的一部分, 另外就是我们需要当 driver 挂掉之后, 让其能够自动重启, 这部分会在接下来的部署中讲到.

### Accumulators,Broadcast 变量,和 Checkpoint

需要注意的是 Accumulators,Broadcast 无法通过 checkpoint进行恢复, 其唯一处理方式是, 在程序执行时 创建延迟 实例化 的 对象.

    class JavaWordBlacklist {

        private static volatile Broadcast<List<String>> instance = null;

        public static Broadcast<List<String>> getInstance(JavaSparkContext jsc) {
            if (instance == null) {
            synchronized (JavaWordBlacklist.class) {
                if (instance == null) {
                List<String> wordBlacklist = Arrays.asList("a", "b", "c");
                instance = jsc.broadcast(wordBlacklist);
                }
            }
            }
            return instance;
        }
    }

    class JavaDroppedWordsCounter {

        private static volatile LongAccumulator instance = null;

        public static LongAccumulator getInstance(JavaSparkContext jsc) {
            if (instance == null) {
            synchronized (JavaDroppedWordsCounter.class) {
                if (instance == null) {
                instance = jsc.sc().longAccumulator("WordsInBlacklistCounter");
                }
            }
            }
            return instance;
        }
    }

    wordCounts.foreachRDD((rdd, time) -> {
        // Get or register the blacklist Broadcast
        Broadcast<List<String>> blacklist = JavaWordBlacklist.getInstance(new JavaSparkContext(rdd.context()));
        // Get or register the droppedWordsCounter Accumulator
        LongAccumulator droppedWordsCounter = JavaDroppedWordsCounter.getInstance(new JavaSparkContext(rdd.context()));
        // Use blacklist to drop words and use droppedWordsCounter to count them
        String counts = rdd.filter(wordCount -> {
            if (blacklist.value().contains(wordCount._1())) {
            droppedWordsCounter.add(wordCount._2());
            return false;
            } else {
            return true;
            }
        }).collect().toString();
        String output = "Counts at time " + time + " " + counts;
    }


## Spark 部署

个人写的Spark集群部署相关, 详细描述了 standalone模式

>[Spark集群-Standalone 模式](https://www.cnblogs.com/zyzdisciple/p/11599596.html)

要运行Spark 应用程序, 需要以下功能:

1. 集群管理器, 有这样几种

    type | mean
    -|-|-
    Standalone |  一种简单的 Spark 内置 集群管理器
    Apache Mesos | 常用的集群管理器之一, 可以运行 Hadoop MapReduce 和 service applications.
    Hadoop YARN | Hadoop 2 的资源管理器.
    Kubernetes | 简称K8S Kubernetes是Google开源的一个容器编排引擎,它支持自动化部署、大规模可伸缩、应用容器化管理

2. 打包后的应用程序Jar, 要求是一个能够在 Spark环境下直接运行的Jar包,因此需要将引用的各种各样的其他jar包, 如kafkaUtils, ZK, Redis 等都打包到一个jar包中, 同时需要在打包时排除 不必要的 依赖jar, 如 spark core, scala lib 等,Maven的插件 Shade就可以很好地实现这一点.

3. 为 executor 配置足够的内存 - 由于接收到的数据必须存储在内存中,所以 executor 必须配置足够的内存来保存接收到的数据.请注意,如果您正在进行10分钟的窗口操作,系统必须至少保留最近10分钟的内存中的数据.因此,应用程序的内存要求取决于其中使用的操作.

4. 配置 checkpoint - 如果 streaming 应用程序需要它,则 Hadoop API 容错存储(例如:HDFS,S3等)中的目录必须配置为 checkpoint 目录,并且流程应用程序以 checkpoint 信息的方式编写Streaming代码.

5. 配置应用程序 driver 的自动重新启动

    * standalone 

        可以提交 Spark 应用程序 driver 以在Spark Standalone集群中运行即应用程序 driver 本身在其中一个工作节点上运行.此外,可以指示独立的群集管理器来监督 driver,如果由于 非正常退出(non-zero exit code)而导致 driver 发生故障,或由于运行 driver 的节点发生故障,则可以重新启动它.

        可以查看: [Spark Standalone Mode](http://spark.apache.org/docs/latest/spark-standalone.html)

    * YARN
  
        Yarn支持类似的机制实现自启动,可以查看yarn相关文档

    * Mesos [Marathon](https://github.com/mesosphere/marathon) 在 Mesos上实现了这一点.

6. 配置预写日志 - 自 Spark 1.2 以来,我们引入了写入日志来实现强大的容错保证.
   
   如果启用,则从 receiver 接收的所有数据都将写入配置 checkpoint 目录中的写入日志.这可以防止 driver 恢复时的数据丢失,从而确保零数据丢失.
   
   可以通过将 配置参数 spark.streaming.receiver.writeAheadLog.enable 设置为 true来启用此功能.
   
   然而,这些更强的语义可能以单个 receiver 的接收吞吐量为代价.通过 并行运行更多的 receiver (会在稍后提到)可以纠正这一点,以增加总吞吐量.
   
   另外,建议在启用写入日志时,在日志已经存储在复制的存储系统中时,禁用在 Spark 中接收到的数据的复制.这可以通过将输入流的存储级别设置为 StorageLevel.MEMORY_AND_DISK_SER 来完成.
   
   使用 S3(或任何不支持刷新的文件系统)写入日志时,请记住启用 spark.streaming.driver.writeAheadLog.closeFileAfterWrite 和spark.streaming.receiver.writeAheadLog.closeFileAfterWrite.
   
   有关详细信息,请参阅官方的 [Spark Streaming](http://spark.apache.org/docs/latest/configuration.html)配置.
   
   请注意,启用 I/O 加密时,Spark 不会将写入写入日志的数据加密.如果需要对提前记录数据进行加密,则应将其存储在本地支持加密的文件系统中.

7. 设置最大接收速率 - 如果集群资源不够大,需要限制最大接收速率 可以通过:
   
   receiver 的 spark.streaming.receiver.maxRate 和用于 Direct Kafka 方法的 spark.streaming.kafka.maxRatePerPartition 的 配置参数.
   
   而在Spark 1.5中,我们引入了一个称为 backpressure (反压)的功能,无需设置此速率限制,因为Spark Streaming会自动计算速率限制,并在处理条件发生变化时动态调整速率限制.
   
   可以通过将 配置参数 spark.streaming.backpressure.enabled 设置为 true 来启用此 backpressure.

## 升级应用程序代码

有两种处理策略:

1. 新旧代码并行运行, 直到你认为可以的时候, 停掉原有代码.

2. 正常停止,在 JavaStreamingContext.stop() 方法中接收两个参数, 第一个是 是否执行完当前数据, 第二个是是否停止 SparkContext. 在当前任务已经执行完之后, 再停止程序, 新的程序启动之后, 会从上次的checkpoint去启动, 而这也正是 checkpoint的缺点, 即 两次的 代码已经变更, 两次的序列化结果, 反序列化并不一致, 因此必须删除 checkpoint 目录才能够正常启动, 这也就意味着在上次停止是 保存的 application 信息 已经消失.


## 性能调优

性能调优的核心在于:

1. 减少数据处理的时间
2. 设定合理的批处理间隔

当两者基本保持一致, 则就能够快速, 有效地处理数据. 至于如何减少数据处理的时间, 则是仁者见仁智者见智.

基本需要自己的大数据处理经验, 更优良的算法, 再者就是所使用的 大数据处理框架相结合.

在这里, 仅仅介绍框架相关的部分, 更多的 则会在 性能调优方面 详细介绍.

连接如下: [Spark调优](https://www.cnblogs.com/zyzdisciple/p/11621864.html)

### Level of Parallelism in Data Receiving(数据接收中的并行级别)

通过网络接收数据(如Kafka,Flume,socket 等)需要 deserialized(反序列化)数据并存储在 Spark 中.如果数据接收成为系统的瓶颈,那么考虑一下 parallelizing the data receiving(并行化数据接收).

注意每个 input DStream 创建接收 single stream of data(单个数据流)的 single receiver(单个接收器)(在 worker 上运行).因此,可以通过创建多个 input DStreams 来实现 Receiving multiple data streams(接收多个数据流)并配置它们以从 source(s) 接收 data stream(数据流)的 different partitions(不同分区).

例如,接收 two topics of data(两个数据主题)的单个Kafka input DStream 可以分为两个 Kafka input streams(输入流),每个只接收一个 topic(主题).这将运行两个 receivers(接收器),允许 in parallel(并行)接收数据,从而提高 overall throughput(总体吞吐量).这些 multiple DStreams 可以 unioned(联合起来)创建一个 single DStream.然后 transformations(转化)为应用于 single input DStream 可以应用于 unified stream.如下这样做:

    int numStreams = 5;
    List<JavaPairDStream<String, String>> kafkaStreams = new ArrayList<>(numStreams);
    for (int i = 0; i < numStreams; i++) {
        kafkaStreams.add(KafkaUtils.createStream(...));
    }
    JavaPairDStream<String, String> unifiedStream = streamingContext.union(kafkaStreams.get(0), kafkaStreams.subList(1, kafkaStreams.size()));
    unifiedStream.print();

与此同时, 需要关注的另一个参数是: spark.streaming.blockInterval

spark.streaming.blockInterval:

Spark Receiver 接收到的数据 在存入 Spark之前 被进行 分块操作, 分块的间隔. 最低推荐是50ms.

每个 receiver 每批次的任务数量将是大约(batch interval(批间隔)/ block interval(块间隔)).例如,200 ms的 block interval(块间隔)每 2 秒 batches(批次)创建 10 个 tasks(任务).如果 tasks(任务)数量太少(即少于每个机器的内核数量),那么它将无效,因为所有可用的内核都不会被使用处理数据.要增加 given batch interval(给定批间隔)的 tasks(任务)数量,请减少 block interval(块间​​隔). 但不应该低于50ms.

使用 多个输入流/ receivers 接收数据的替代方法是明确 repartition(重新分配)input data stream(输入数据流)(使用 inputStream.repartition(&lt;number of partitions&gt;)).这会在进一步处理之前将收到的批次数据分发到集群中指定数量的计算机.

### Level of Parallelism in Data Processing(数据处理中的并行度)

如果任意 stage 的并行度设置的不够, 则会导致 集群资源 得不到充分利用.

至于并行度的设置 

> 参考:[Spark性能调优:合理设置并行度](https://blog.csdn.net/leen0304/article/details/78674073)

网上类似的文章数不胜数, 就随便找了一篇.

核心点在于, 并行度设置过低, 即使分配的CPU再多, 也用不到那么多资源.

task数量,设置成spark Application 总cpu core数量的2~3倍 ,比如150个cpu core ,基本设置 task数量为 300~ 500, 与理性情况不同的,有些task 会运行快一点,比如50s 就完了,有些task 可能会慢一点,要一分半才运行完,所以如果你的task数量,刚好设置的跟cpu core 数量相同,可能会导致资源的浪费.

### Data Serialization(数据序列化)

可以通过调优 serialization formats(序列化格式)来减少数据 serialization(序列化)的开销.在 streaming 的情况下,有两种类型的数据被 serialized(序列化).

* Input data(输入数据):默认情况下,通过 Receivers 接收的 input data(输入数据)通过 StorageLevel.MEMORY_AND_DISK_SER_2 存储在 executors 的内存中, 数据首先保留在内存中,并且只有在内存不足以容纳流计算所需的所有输入数据时才会 spilled over(溢出)到磁盘.这个序列化显然具有开销 - receiver(接收器)必须使接收的数据 deserialize(反序列化),并使用 Spark 的 serialization format(序列化格式)重新序列化它.

* Persisted RDDs generated by Streaming Operations(流式操作生成的持久 RDDs):通过 streaming computations(流式计算)生成的 RDD 可能会持久存储在内存中.例如,window operations(窗口操作)会将数据保留在内存中,因为它们将被处理多次.但是,与 StorageLevel.MEMORY_ONLY 的 Spark Core 默认情况不同,通过流式计算生成的持久化 RDD 将以 StorageLevel.MEMORY_ONLY_SER(即序列化),以最小化 GC 开销.

在上述两种情况下, 使用kryo都可以有效减少CPU开销 和 内存开销.

然而, 序列化本身也是一种开销, 在需要保存的数据量不大, 内存足够的情况下:

可以将数据作为 deserialized objects(反序列化对象)持久化,而不会导致过多的 GC 开销.例如,如果你使用几秒钟的 batch intervals(批次间隔)并且没有 window operations(窗口操作),那么可以通过明确地相应地设置 storage level(存储级别)来尝试禁用 serialization in persisted data(持久化数据中的序列化).这将减少由于序列化造成的 CPU 开销,潜在地提高性能,而不需要太多的 GC 开销.

### Task Launching Overheads(任务启动开销)

如果每秒启动的任务数量很高(比如每秒 50 个或更多),那么这个开销向 slaves 发送任务可能是重要的,并且将难以实现 sub-second latencies(次要的延迟).可以通过以下更改减少开销:

* Execution mode(执行模式):以 Standalone 模式 或 coarse-grained Mesos 模式运行 Spark 比 fine-grained Mesos 模式更好的任务启动时间.

这些更改可能会将 批处理时间 缩短 100 毫秒,从而允许 sub-second batch size(次秒批次大小)是可行的.

### Setting the Right Batch Interval(设置正确的批次间隔)

毫无疑问,如果想要系统 稳定可持续, 我们必须保证数据的流入流出速率保持均衡, 也即每批次接收的数据 与数据的 处理速度维持平衡.

而调试速率的办法就是 增加batchInterval  和 减少 数据的流入速率, 通过 SparkUI Streaming页签下中可以观测到 处理延时, 也即 total delay列.

保持 total delay 小于等于 batch interval即可.

即使真实环境中, 数据有突发性也无需在乎, 只需要保证数据在整个运行期间的速率基本保持均衡即可.

然而需要注意到的一点是, 增大 batchInterval 也意味着 有可能增加了内存开销.

### Memory Tuning(内存调优)

Spark Streaming application 所需的集群内存量在很大程度上取决于所使用的 transformations 类型.例如,如果要在最近 10 分钟的数据中使用 window operation(窗口操作),那么您的集群应该有足够的内存来容纳内存中 10 分钟的数据.或者如果要使用大量 keys 的 updateStateByKey,那么必要的内存将会很高.相反,如果你想做一个简单的 map-filter-store 操作,那么所需的内存就会很低.

一般来说, receiver中接收到的数据, 在内存溢出的时候 会序列化 存储到硬盘中, 这可能会降低 streaming application 的性能,因此建议提供足够的内存以供使用.最好仔细查看内存使用量并相应地进行估算.

内存调优的另一个方面是 垃圾收集.对于需要低延迟的 streaming application,由 JVM 垃圾回收引起的大量暂停是不希望的.

就以上几点来说:

* DStreams 的持久性级别:如前面在 Data Serialization 部分中所述,input data 和 RDD 默认保持为 serialized bytes(序列化字节).与 deserialized persistence(反序列化持久性)相比,这减少了内存使用量和 GC 开销.

    可以通过启用 Kryo serialization 进一步减少了序列化大小和内存使用.
    
    以及可以通过 compression(压缩)来实现内存使用的进一步减少(参见Spark配置 spark.rdd.compress),代价是 CPU 时间.

* Clearing old data(清除旧数据):默认情况下,DStream 转换生成的所有 input data 和 persisted RDDs 将自动清除.Spark Streaming 决定何时根据所使用的 transformations 来清除数据.例如,如果您使用 10 分钟的 window operation(窗口操作),则 Spark Streaming 将保留最近 10 分钟的数据,并主动丢弃旧数据.数据可以通过设置 streamingContext.remember 保持更长的持续时间(例如交互式查询旧数据, 之前提到的跨线程 访问StreamingData).

* CMS Garbage Collector(CMS垃圾收集器):强烈建议使用 concurrent mark-and-sweep GC,以保持 GC 相关的暂停始终如一.CMS的优点就是, 尽量减少暂停式GC,通过与任务并行执行的方式, 执行GC.即使 concurrent GC 已知可以减少 系统的整体处理吞吐量,但仍然建议实现更多一致的 batch processing times(批处理时间).确保在 driver( 在 spark-submit 中使用 --driver-java-options)和 executors(使用 Spark configuration spark.executor.extraJavaOptions)中设置 CMS GC.

* Other tips(其他提示):为了进一步降低 GC 开销,以下是一些更多的提示.

    使用 OFF_HEAP 存储级别的保持 RDDs,使用更小的 heap sizes 的 executors.这将降低每个 JVM heap 内的 GC 压力.

    至于什么是 OFF_HEAP?

    与ON_HEAP对立, 表示 存储在 Java堆内存之外的数据.

    也即 将数据存储在机器内存中.

    > 参考链接: [Spark 内存管理之—OFF_HEAP](https://www.cnblogs.com/yy3b2007com/p/10816160.html)

### 小结

总结来说:

* 每个DStream 都与 single receiver相关联.为了获得读取并行性,需要创建多个 receivers,即 multiple DStreams.receiver 在一个 executor 中运行.它占据一个 core(内核).确保在 receiver slots are booked 后有足够的内核进行处理,即 spark.cores.max 应该考虑 receiver slots.receivers 以循环方式分配给 executors.

* 当从 stream source 接收到数据时,receiver 创建数据 blocks(块).每个 blockInterval(默认200ms) 毫秒生成一个新的数据块.在 N = batchInterval/blockInterval 的 batchInterval 期间创建 N 个数据块.这些块由当前 executor 的 BlockManager 分发给其他执行程序的 block managers.之后,在驱动程序上运行的 Network Input Tracker(网络输入跟踪器)通知有关进一步处理的块位置

* 在驱动程序中为在 batchInterval 期间创建的块创建一个 RDD.在 batchInterval 期间生成的块是 RDD 的 partitions.每个分区都是一个 spark 中的 task.blockInterval == batchinterval 意味着创建 single partition(单个分区),并且可能在本地进行处理.

* 除非使用non-local(非本地调度)的方式, 否则这些块上的map任务都运行在执行器单元上(一个在接收数据块的位置,另一个在数据块被备份到的位置),而不会考虑block interval,而更大的 block interval 意味着更大的块.
  
    增大spark.locality.wait 即增加了处理 local node(本地节点)上的块的可能性.
    
    因此,需要在这两个参数之间找到平衡,以确保在本地处理较大的块.

* 除了调整block interval 和 batch interval之外, 您可以通过调用 inputDstream.repartition(n) 来定义分区数.
  
  这样可以<font color="orange">随机重新组合</font> RDD 中的数据,创建 n 个分区以求更大的并行性.虽然是 shuffle 的代价.
  
  但是我们需要注意的是,如果数据本身每个partition中的数据有较强的关联性, 使用这种方法需要谨慎.

  另外,需要考虑到的问题是,虽然我们有了更大的并行度, 但自身的集群资源是否支持这样高的并行性？即分配的核心数, executor数量是否足够？

  RDD 的处理由 driver’s jobscheduler 作为一项工作安排.在给定的时间点,只有一个 job 是 active 的.因此,如果一个作业正在执行,则其他作业将排队.

* 如果您有两个 dstream,将会有两个 RDDs 被创建,并且会创建两个任务,然后被一个接一个的调度.为了避免这种情况,你可以对这两个DStream执行union操作.这保证了两个DStream RDD会产生一个unionRDD,这个unionRDD会当做一个单独的job.但 RDD 的 partitioning(分区)不受影响.

    而Spark Job: 每个Action算子本质上是执行了sc的runJob方法,这是一个重载方法.核心是交给DAGScheduler中的submitJob执行, 进而创建了不同的job.

* 如果 批处理时间)超过 batchinterval(批次间隔),那么显然 receiver 的内存将会开始填满,最终会抛出 exceptions(最可能是 BlockNotFoundException).目前没有办法暂停 receiver.使用 SparkConf 配置 spark.streaming.receiver.maxRate,receiver 的 rate 可以受到限制.

</font>