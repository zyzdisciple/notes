# SparkStreaming-Kafka集成

<font face="楷体">

>参考链接： [Spark Streaming + Kafka Integration Guide](http://spark.apache.org/docs/latest/streaming-kafka-0-10-integration.html)

<font color="orange">文章基本是官方的翻译</font>， 最多再加入了一小部分自己的思考在内， 如果能看懂官方文档， 也可以自行查看官网。

另外就是提供了自己实现的 zk + kafka + spark 获取offset。 offset的存储在 获取偏移量 与 存储偏移量的 第三小节 有描述。

基于版本：

Kafka broker version 0.10.0 or higher

0.10.0 版本的 SparkStreaming kafka 与 0.8版本的 DirectStream比较接近。

它支持比较简单的并行性，包括 Kafka 分区 和Spark 分区之间 是 1：1对应关系，以及对偏移量和元数据的访问。但是，由于较新的集成使用了新的Kafka使用者API而不是简单的API，因此用法上存在显着差异。

## Maven依赖

    groupId = org.apache.spark
    artifactId = spark-streaming-kafka-0-10_2.12
    version = 2.4.4

注意： 不要自行添加 org.apache.kafka artifacts (例如：kafka-clients)， 在 spark-streaming-kafka-0-10 已经集成了可使用的kafka版本， 如果自行引入其他kakfa版本可能会引发问题。

但同样也需要注意到的是： 这一点是在2.4.4版本才添加的， 在2.4.3版本及以前还是需要自己手动引入 kafka.clients的。

## 创建 Direct Stream

需要注意引入的版本号： 010

    import java.util.*;
    import org.apache.spark.SparkConf;
    import org.apache.spark.TaskContext;
    import org.apache.spark.api.java.*;
    import org.apache.spark.api.java.function.*;
    import org.apache.spark.streaming.api.java.*;
    import org.apache.spark.streaming.kafka010.*;
    import org.apache.kafka.clients.consumer.ConsumerRecord;
    import org.apache.kafka.common.TopicPartition;
    import org.apache.kafka.common.serialization.StringDeserializer;
    import scala.Tuple2;

    Map<String, Object> kafkaParams = new HashMap<>();
    kafkaParams.put("bootstrap.servers", "localhost:9092,anotherhost:9092");
    kafkaParams.put("key.deserializer", StringDeserializer.class);
    kafkaParams.put("value.deserializer", StringDeserializer.class);
    kafkaParams.put("group.id", "use_a_separate_group_id_for_each_stream");
    kafkaParams.put("auto.offset.reset", "earliest");
    kafkaParams.put("enable.auto.commit", false);

    Collection<String> topics = Arrays.asList("topicA", "topicB");

    JavaInputDStream<ConsumerRecord<String, String>> stream =
    KafkaUtils.createDirectStream(
        streamingContext,
        LocationStrategies.PreferConsistent(),
        ConsumerStrategies.<String, String>Subscribe(topics, kafkaParams)
    );

    stream.mapToPair(record -> new Tuple2<>(record.key(), record.value()));

> 参数：auto.offset.reset
> 
> earliest: 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
> 
> latest: 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
> none: topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常
> 默认建议用earliest。设置该参数后 kafka出错后重启，找到未消费的offset可以继续消费。

但是对于 Spark而言，在某些情况下 采取哪一种并没有太大区别， 这个稍后再说。

对于kafka中可配置的参数， 可以在 [KAFKA_CONFIGURATION](http://kafka.apache.org/documentation.html#configuration) 中找到.

如果你的 spark batch interval 时间要大于 Kafka heartbeat session timeout(默认是30s)，
If your Spark batch duration is larger than the default Kafka heartbeat session timeout (30 seconds), 需要自行增加 heartbeat.interval.ms 和 session.timeout.ms. 因为 Spark是 每隔一个 batch interval才去拉取数据， 如果间隔太久， kafka就会认为已经断开连接。 对于  batch interval 大于5分钟的， 还需要配置另一个参数:group.max.session.timeout.ms.

另外就是 注意到 在 例子中设置： enable.auto.commit false，在稍后会描述原因。

## LocationStrategies

在方法参数中， 需要传入 LocationStrategies。

在新版的kafka Consumer API中， 会将 message 预加载到缓存中，因此， 出于性能的原因， Spark集成kafka 会将 consumers 缓存到 executor中（而不是在每个批次都重新创建 consumers），并且更倾向于 在具有 适当的 consumers 的主机上 安排分区。

在大多数情况下， 我们需要使用 LocationStrategies.PreferConsistent 它将会在 可用的 executors上均匀分配分区。

如果 executors 和 kafka brokers 在同一台主机上， 则LocationStrategies.PreferBrokers 是更好的选择。因为它会 将 partition 优先分配到存在  kafka broker 的机器上。

因为kafka的分区会与 spark 分区一一对应， 因此， 可能会因为 kafka的数据倾斜， 导致 spark中同样出现数据倾斜的问题， 因此 LocationStrategies.PreferFixed 允许您指定分区到主机的显式映射（任何未指定的分区将使用一致的位置）。

consumers 的缓存的默认最大大小为64. 如果你希望处理超过（64 * executors）Kafka分区，则可以通过spark.streaming.kafka.consumer.cache.maxCapacity更改此设置。

如果你想禁用 kafka consumer 的缓存， 可以设置 spark.streaming.kafka.consumer.cache.enabled 为 false。

kafka consumer cache 的 缓存 是用 topicpartition 和 group.id 做区分的， 因此对于同时启动 多个receiver， 需要为每个 direct stream 创建不同的 groupId。

## ConsumerStrategies

kafka新的 api中， 提供了大量的不同的方法 去指定 topic，其中一部分 要求 特别大的 post对象实例（原文是: post-object-instantiation 不太理解）配置。  ConsumerStrategies 提供了一种抽象，即使从检查点重新启动后，Spark也可以获得正确配置的消费者。

ConsumerStrategies.Subscribe, 允许你订阅固定的 topic 集合。

ConsumerStrategies.SubscribePattern 允许你使用正则表达式来指定感兴趣的主题。

与0.8版本的集成不同， 通过以上两种方式 在运行流期间使用Subscribe或SubscribePattern应该响应添加分区， 在这里的意思应该是， 即使topic一开始不存在， 即使是动态添加的依然能够在 spark 运行期间 拉取数据。

最后 ConsumerStrategies.Assign 允许你指定特定的 分区。

这三种方式 都支持你 指定 对特定分区的起始offset。

如果你具有上述选项无法满足的需求，可以通过 extend ConsumerStrategy实现自己的方法。

最后需要提醒的是：

即使你指定的topic 和 partition 并不存在， 程序也能够正常运行， 这得益于 kafka中的一个参数：

allow.auto.create.topics

默认为true。

## 创建 RDD

你可以通过指定 topic partition 以及 offset的范围的方式， 来创建RDD

    // Import dependencies and create kafka params as in Create Direct Stream above

    OffsetRange[] offsetRanges = {
        // topic, partition, inclusive starting offset, exclusive ending offset
        OffsetRange.create("test", 0, 0, 100),
        OffsetRange.create("test", 1, 0, 100)
    };

    JavaRDD<ConsumerRecord<String, String>> rdd = KafkaUtils.createRDD(
        sparkContext,
        kafkaParams,
        offsetRanges,
        LocationStrategies.PreferConsistent()
    );

注意：在这里不能够使用 LocationStrategies.PreferBrokers 因为在没有流的情况下， 缺乏驱动侧的 consumer 帮你自动查找获取 broker的元信息。 如果必须要用的话， 使用 PreferFixed 来自己查找元信息。

## 获取偏移量

    stream.foreachRDD(rdd -> {
        OffsetRange[] offsetRanges = ((HasOffsetRanges) rdd.rdd()).offsetRanges();
        rdd.foreachPartition(consumerRecords -> {
            OffsetRange o = offsetRanges[TaskContext.get().partitionId()];
            System.out.println(
            o.topic() + " " + o.partition() + " " + o.fromOffset() + " " + o.untilOffset());
        });
    });

注意: HasOffsetRanges的类型转换只有在createDirectStream 获取到的流， 在流处理的第一个方法调用时才会成功，而不能在其之后的方法链中调用。需要认识到，RDD分区和Kafka分区之间的映射关系，在任何一个repartition或shuffle操作后（如reduceByKey（）或Window())函数后都不再存在。

因此，我往往是通过：

    dstream.transform(rdd -> {
        (HasOffsetRanges) rdd.rdd()).offsetRanges();
        return rdd;
    })

在这之后再执行更复杂流处理过程。

## 存储偏移量

在失败的情况下， kafka的交付语义 取决于在什么时候 其 offset被存储，存储则意味着 归属于当前 offset之前的所有数据 都已经被正确处理， 因此相当于 之前的数据已经被 丢弃， 不会再度进行处理。

而这也是我们不使用 enable.auto.commit 为 true的原因。

在kafka中的自动提交机制是： 

enable.auto.commit 的默认值是 true；就是默认采用自动提交的机制。

auto.commit.interval.ms 的默认值是 5000，单位是毫秒。

这样，默认5秒钟，一个 Consumer 将会提交它的 Offset 给 Kafka，或者每一次数据从指定的 Topic 取回时，将会提交最后一次的 Offset。

也就是说，当我们从 topic partition中取回数据时，每隔固定时间， 这个offset就会被提交。

在绝大多数情况下， 这并不是我们想要的方式。

spark的 输出语义是 至少一次，所以 如果 你想要获取 与  至少一次等效的语义， 你必须在 幂等的输出操作后存储 或在一次与输出操作并行的原子操作中存储。为了达到上述目的， 你有以下三种方式去处理：

1. checkPoints（检查点）

    如果你打开了Spark的checkpointing选项，偏移量会被保存在checkpoint里面。
    
    这确实是一种很简单的方式， 然而有一些缺点， 首先为了对于同一数据得到的输出是 重复的， 所以你的输出操作必须是幂等的；事务并不是一个好的选择。
    
    此外，如果你的代码有了更改，就不能从checkpoint之中恢复。对于计划升级，可以在旧代码运行的同时部署运行新的代码来缓解这个问题（因为输出是幂等的，所以不会造成冲突）。
    
    但是对于意料之外的故障而需要更改代码的，除非你有其他的方式来获取开始的偏移量，否则就会丢失数据。

2. kafka自身

    其 auto.commit 不必多说， 自然是不合适的， 因此，你可以在你确保输出操作已经完成后使用commitSync API向Kafka提交偏移量。与checkpoint方式相比，该种方式的好处是Kafka是一个持久化的存储，而不需要考虑代码的更新。 然而， kafka是非事务性的， 因此仍然需要 输出操作 是幂等的。

        stream.foreachRDD(rdd -> {
            OffsetRange[] offsetRanges = ((HasOffsetRanges) rdd.rdd()).offsetRanges();

            // some time later, after outputs have completed
            ((CanCommitOffsets) stream.inputDStream()).commitAsync(offsetRanges);
        });

    这本身就是一种比较好的方式。

3. 自己实现

    对于<font color="orange">支持事务</font>的数据存储，可以在同一个事务中保存偏移量，这样即便在失败的情况下也可以保证两者的同步。

    如果你关心重复的或者跳过的偏移量的范围，回滚事务可以防止重复或丢失消息影响结果。这等价于仅仅一次的语义。也可以使用这种策略来对那些通常很难保证幂等性的聚合输出操作起作用。

        // The details depend on your data store, but the general idea looks like this

        // begin from the the offsets committed to the database
        Map<TopicPartition, Long> fromOffsets = new HashMap<>();
            for (resultSet : selectOffsetsFromYourDatabase)
                fromOffsets.put(new TopicPartition(resultSet.string("topic"), resultSet.int("partition")), resultSet.long("offset"));
        }

        JavaInputDStream<ConsumerRecord<String, String>> stream = KafkaUtils.createDirectStream(
            streamingContext,
            LocationStrategies.PreferConsistent(),
            ConsumerStrategies.<String, String>Assign(fromOffsets.keySet(), kafkaParams, fromOffsets)
        );

        stream.foreachRDD(rdd -> {
            OffsetRange[] offsetRanges = ((HasOffsetRanges) rdd.rdd()).offsetRanges();
            
            Object results = yourCalculation(rdd);

            // begin your transaction

            // update results
            // update offsets where the end of existing offsets matches the beginning of this batch of offsets
            // assert that offsets were updated correctly

            // end your transaction
        });

    这部分，目前比较常见的方式是， 通过 zk存储数据，但不限于 zk， redis， mysql等方式都是可以的。

    因为在这里的数据更新频次实际上并不会太高，一般是每一批次提交一次， 因此即使是存储在mysql中也是可以接受的。

    在这里给出我自己的项目所使用的实现：

        import java.util.ArrayList;
        import java.util.Arrays;
        import java.util.HashMap;
        import java.util.List;
        import java.util.Map;

        import org.I0Itec.zkclient.ZkClient;
        import org.apache.kafka.clients.consumer.ConsumerRecord;
        import org.apache.kafka.clients.consumer.KafkaConsumer;
        import org.apache.kafka.common.PartitionInfo;
        import org.apache.kafka.common.TopicPartition;
        import org.apache.kafka.common.serialization.StringDeserializer;
        import org.apache.spark.streaming.api.java.JavaInputDStream;
        import org.apache.spark.streaming.api.java.JavaStreamingContext;
        import org.apache.spark.streaming.kafka010.ConsumerStrategies;
        import org.apache.spark.streaming.kafka010.KafkaUtils;
        import org.apache.spark.streaming.kafka010.LocationStrategies;
        import org.slf4j.Logger;
        import org.slf4j.LoggerFactory;

        public class SparkDataSource {
            
            private static final Logger logger = LoggerFactory.getLogger(SparkDataSource.class);
            
            private static final String ZK_PATH_PREFIX = "/consumer/spark/project/offset/";

            public static JavaInputDStream<ConsumerRecord<Object, Object>> getInputDStreamByKakfa(JavaStreamingContext jssc, 
                    @SuppressWarnings("rawtypes") Class valueDeserializerClass, 
                    String groupId,
                    String topic
                    ) {
                Map<String, Object> kafkaConfig = new HashMap<>();
                kafkaConfig.put("bootstrap.servers", "localhost:9092,anotherhost:9092");
                kafkaConfig.put("key.deserializer", StringDeserializer.class);
                kafkaConfig.put("value.deserializer", valueDeserializerClass);
                kafkaConfig.put("group.id", "groupId");
                kafkaConfig.put("auto.offset.reset", "lastest");
                kafkaConfig.put("enable.auto.commit", false);
                
                KafkaConsumer<String, Object> consumer = new KafkaConsumer<>(kafkaConfig);

                //可能出现连接超时， topic 不存在等情况，会引起报错，导致启动中断.
                while (true) {
                    try {
                        List<TopicPartition> topicPartitions = topicPartitions(consumer, topic);
                        if (topicPartitions != null && !topicPartitions.isEmpty()) {
                            break;
                        }
                        Thread.sleep(5000);
                    } catch (Exception e) {
                        logger.warn("获取topic partition信息失败", e);
                    }
                }
                
                
                return KafkaUtils.createDirectStream(jssc, 
                            LocationStrategies.PreferConsistent(),
                            ConsumerStrategies.Subscribe(Arrays.asList(new String[] {topic}), kafkaConfig, getOffset(consumer, topic))
                        );
            }
            
            /**
            * 获取偏移量, 如果zk中有，则取zk，否则直接去获取.
            * @param consumer kafkaConsumer
            * @param topic topic
            * @return 最终的topic partition 和 offset
            */
            private static Map<TopicPartition, Long> getOffset(KafkaConsumer<String, Object> consumer, String topic) {
                String zkPath = ZK_PATH_PREFIX + topic;
                //创建zk, 需要传入自身的连接信息。
                ZkClient zkClient = new ZkClient("zkServer");
                //检查当前路径是否存在子节点， 默认是有值的，是我们在保存信息时创建的 zk节点。
                int childNumber = zkClient.countChildren(zkPath);
                Map<TopicPartition, Long> fromOffset = new HashMap<>();
                
                if (childNumber > 0) {
                    //获取对应topic的最大 offset, 因为如果请求的offset超出最大值是会报错的.
                    Map<TopicPartition, Long> endOffsets = getEndOffsetByTopic(consumer, topic);
                    for (int i = 0; i < childNumber; i++) {
                        TopicPartition tap = new TopicPartition(topic, i);
                        //存储kafka对应的各个partition对应 offset 的 路径.
                        String realPath = zkPath + "/" + i;
                        String offset = zkClient.readData(realPath);
                        Long lastOffset = endOffsets.get(tap);
                        //然而这种方式也不见得完全正确， 依然存在一种可能性，topic已经被删除，这是重新创建的数据， 且已经灌入一批数据
                        //所以此时应该选择从头开始读, 或者说从最新处开始读，要看个人选择， 同时最好可以加入相关的信息标识
                        //表明是来自同一批数据.
                        if (lastOffset != null) {
                            if (lastOffset < Long.parseLong(offset)) {
                                //如果记录的offset过大，则可以选择最新的offset.
                                fromOffset.put(tap, lastOffset);
                            } else {
                                fromOffset.put(tap, Long.parseLong(offset));
                            }
                        } else {
                            //如果为null的话， 说明kafka的分区可能已经经过调整, 需要删除zk对应的节点.
                            zkClient.delete(realPath);
                        }
                    }
                } else {
                    fromOffset = getBeginningOffsetByTopic(consumer, topic);
                }
                return fromOffset;
            }
            
            private static Map<TopicPartition, Long> getEndOffsetByTopic(KafkaConsumer<String, Object> consumer, String topic) {
                return consumer.endOffsets(topicPartitions(consumer, topic));
            }
            
            private static Map<TopicPartition, Long> getBeginningOffsetByTopic(KafkaConsumer<String, Object> consumer, String topic) {
                return consumer.beginningOffsets(topicPartitions(consumer, topic));
            }
            
            private static List<TopicPartition> topicPartitions(KafkaConsumer<String, Object> consumer, String topic) {
                List<PartitionInfo> partitions = consumer.partitionsFor(topic);
                List<TopicPartition> topicPartitons = new ArrayList<>(partitions.size());
                partitions.forEach(pInfo -> {
                    topicPartitons.add(new TopicPartition(topic, pInfo.partition()));
                });
                return topicPartitons;
            }
        }

    存储 offset倒是没有什么特别的地方， 主要是在 项目启动 offset的获取上。

## SSL / TLS

> Tips: [HTTPS、SSL、TLS三者之间的联系和区别](https://blog.csdn.net/enweitech/article/details/81781405) 通俗来说， TLS就是 SSL标准化后的产物。

新的kafkaConsumer支持 SSL，为了支持这一点， 需要在接入kafka之前 加入一部分配置， 注意，这仅仅适用于spark和kafka 服务器之间的交流，你同样需要保证Spark节点内部之间的安全（Spark安全）通信。

    Map<String, Object> kafkaParams = new HashMap<String, Object>();
    // the usual params, make sure to change the port in bootstrap.servers if 9092 is not TLS
    kafkaParams.put("security.protocol", "SSL");
    kafkaParams.put("ssl.truststore.location", "/some-directory/kafka.client.truststore.jks");
    kafkaParams.put("ssl.truststore.password", "test1234");
    kafkaParams.put("ssl.keystore.location", "/some-directory/kafka.client.keystore.jks");
    kafkaParams.put("ssl.keystore.password", "test1234");
    kafkaParams.put("ssl.key.password", "test1234");

## 程序部署

 对于JAVA或Scala应用来说，如果你使用SBT或MAVEN来做项目管理，需要将spark-streaming-kafka-010_2.11包以及它的依赖包添加到你的应用的JAR包中。确保spark-core+2.11包和spark-streaming_2.11包在你的依赖中位provided级别，因为他们在Spark的安装包中已经提供了。接下来使用spark-submit命令来部署你的应用。

</font>