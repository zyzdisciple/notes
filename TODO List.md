SparkStreaming（1） ~ 编程指南

待完善1

* 测试n个Stream是否需要n + 1个线程来启动？
* 测试n个stream并行执行, 是否每个处理程序都需要一个线程？

待完善2

* 添加kakfa说明连接

待完善3

* kafka 可靠receiver的实现

* 是否需要ZKOffset(判断是仍然需要的, 因为这仅指数据拉取, 接收, 但并不知道何时数据被处理完全.)

* 数据的分块算法， 及速率控制算法。 (需要参考Spark Receiver源码.)

待完善4

* 验证transform相关

待完善5

* Spark cache的真正时机

    如果有:

        RDD.filter(1).cache();
        RDD.filter(2).cahce();
        RDD.foreach();

    其中filter(1) 会被执行几次？ 真正的cache发生在什么时候？

待完善6

* 完善 standAlone 集群模式， 及 错误自启动。

待完善7

* 对于 kafka 如果是相同的 client 不同的 程序， 是否能够延续上次的 offset继续执行？

* 补全第二点中的说明， SparkStreaming kafka 究竟如何保存数据 offset。

