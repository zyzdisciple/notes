SparkStreaming（1） ~ 编程指南

待完善6

* 完善 standAlone 集群模式， 及 错误自启动。

待完善7

* 对于 kafka 如果是相同的 client 不同的 程序， 是否能够延续上次的 offset继续执行？

* 补全第二点中的说明， SparkStreaming kafka 究竟如何保存数据 offset。

待完善8

* 性能调优 介绍链接。

待完善9

* 加入调优代码

待完善10

* 验证猜想： 对于batch中的数据量无论多大都能够存储下， 因为存储级别默认为 MEMORY_DISK_SER2, 内存中是否能够容纳下足够数据， 取决于最终的实现 与 操作。

待完善11

* G1 CMS对比
* OffHeapMemory

待完善12

* 测试kafka会创建多少rdd。 是否是  batchInterval / blockInterval * partitions
* spark.locality.wait 参数含义