# Spark 问题集

用Spark也有一段时间了， 目前遇到的问题， 主要是在环境搭建， 集群部署 等 方面遇到各种各样的问题， 在此记录。


参考资料链接：
> [Spark中master、worker、executor和driver的关系](https://blog.csdn.net/hongmofang10/article/details/84587262)

## 部署相关

先描述一下我们的环境， 测试环境。

2台机器， 都是 8 核 16G， linux， centos系统。

机器hostname 分别为 test1， test2， 且不允许更改。

Ip地址假定为 192.168.0.1 192.168.0.2.

且通过 ping test1， ping test2 无法访问， 也即无法通过主机名加端口的方式相互访问， 因此需要用 ip + 端口才能够进行正常访问。



### Spark UI 相关

通过在 $SPARK_HOME/conf/spark-env.sh 下进行配置。

加入：

    export SPARK_MASTER_WEBUI_PORT=8081

即可设定MasterUI的访问路径为 192.168.0.1:8081 ， 原始值为8080.

即使端口已经被占用也不用担心， 在Spark的各种端口中， 如果当前端口已经被占用， 会自动向后顺延一位， 如此重复16次。 直到找到可用端口位置。

Spark Worker的UI位置默认在 8081， application 的默认位置在 4040.

需要注意的一个地方是：

当我在 master1， IP地址： 192.168.0.1 配置了 从节点 192.168.0.2

此时在 master1 节点上， 执行了 $SPARK_HOME/sbin/start-all.sh 命令， 此时会在 192.168.0.2 服务器上启动 一个 worker。

由于 192.168.0.2 上没有master，  因此worker 自动占用了 8081 端口。 当我们在 192.168.0.2 上再度通过 $SPARK_HOME/sbin/start-master.sh 命令 启动一个 master，此时即使配置的是 8081， 也会由于8081已被占用， 访问地址变为了 192.168.0.2:8082. 

