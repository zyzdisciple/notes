SparkStreaming（1） ~ 编程指南

待完善10

* 验证猜想： 对于batch中的数据量无论多大都能够存储下， 因为存储级别默认为 MEMORY_DISK_SER2, 内存中是否能够容纳下足够数据， 取决于最终的实现 与 操作。

待完善11

* G1 CMS对比
* OffHeapMemory

待完善12

* 测试kafka会创建多少rdd。 是否是  batchInterval / blockInterval * partitions
* spark.locality.wait 参数含义