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