# Spark 集群

<font face="楷体">

来源于官方, 可以理解为是官方译文, 外加一点自己的理解. 版本是2.4.4

本篇文章涉及到:

## 名词解释

Term（术语） | Meaning（含义）
-|-|-
Application | 用户构建在 Spark 上的程序。由集群上的一个 driver 程序和多个 executor 组成。
Driver program | 该进程运行应用的 main() 方法并且创建了 SparkContext。
Cluster manager | 一个外部的用于获取集群上资源的服务。（例如，Standlone Manager，Mesos，YARN）
Worker node | 任何在集群中可以运行应用代码的节点。
Executor | 一个为了在 worker 节点上的应用而启动的进程，它运行 task 并且将数据保持在内存中或者硬盘存储。每个应用有它自己的 Executor。
Task | 一个将要被发送到 Executor 中的工作单元。
Job | 一个由多个任务组成的并行计算，并且能从 Spark action 中获取响应（例如 save，collect）; 您将在 driver 的日志中看到这个术语。
Stage | 每个 Job 被拆分成更小的被称作 stage（阶段）的 task（任务）组，stage 彼此之间是相互依赖的（与 MapReduce 中的 map 和 reduce stage 相似）。您将在 driver 的日志中看到这个术语。


## 概述

>参考链接: [Cluster Mode Overview](http://spark.apache.org/docs/latest/cluster-overview.html)
>
>中文链接: [集群模式概述](http://spark.apachecn.org/#/docs/12)

Spark Application 在集群上作为独立的进程组来运行，在 main 程序中通过 SparkContext 来协调（称之为 driver 程序）。

具体来说，为了运行在集群上，SparkContext 可以连接至几种类型的 Cluster Manager（既可以用 Spark 自己的 Standlone Cluster Manager，或者 Mesos，也可以使用 YARN），用以在 applications 之间 分配资源。

一旦连接上，Spark 获得集群中节点上的 Executor，这些进程可以运行计算并且为应用存储数据。

接下来，它将发送 application 的代码（通过 JAR 或者 Python 文件定义传递给 SparkContext）至 Executor。 而这一点大概也是 在 work目录下, 每个application中都有对应的 jar包的原因. 最终，SparkContext 将发送 Task 到 Executor 以运行。

有这么几点要注意的地方:

1. 每个application拥有它自身的 executor 进程. 它们会保持在整个 application 的生命周期中并且在多个线程中运行 task. 这样做的优点是 可以将 application 之间相互隔离, 无论是在 任务调度 层面(即driver, driver 负责任务调度.) 又或者是 executor的层面. 这意味着 如果没有外部存储机制, 各个 application之间是无法进行数据共享的. 

2. Spark并不关心究竟是 基于怎样的 集群模式, 它只关心 能够获取自身的 executor进程, 并且彼此之间可以相互通信即可.

3. Driver 程序必须在自己的生命周期内监听和接受来自它的 Executor 的连接请求。（配置: spark.driver.port) 同样的， 对于 worker node 而言, driver 程序也必须能够从网络中连接到.

4. 因为 driver 负责在 整个集群上 调度任务， 因此能够与 worker node 处于同一局域网下是更优的选择(否则的话, 网络通信可能就成为了 整个Spark最大的时间开销)。如果你不喜欢发送请求到远程的集群，倒不如打开一个 RPC 至 driver 并让它就近提交操作而不是从很远的节点上运行一个 driver。

## Cluster Manager 类型

系统目前支持三种 Cluster Manager:

Standalone – 包含在 Spark 中, 简单易使用。

Apache Mesos – 一个通用的 Cluster Manager，它也可以运行 Hadoop MapReduce 和其它服务应用。

Hadoop YARN – Hadoop 2 中的 resource manager（资源管理器）。

Kubernetes (experimental)

Nomad: 存在第三方的项目(并非受到Spark项目支持的) 可以添加对应的集群支持.

## 提交应用程序

>官方链接: [Submitting Applications](http://spark.apache.org/docs/latest/submitting-applications.html)
>
>中文链接: [Submitting Applications](http://spark.apachecn.org/#/docs/13)

在 Spark的 bin 目录中的spark-submit 脚本 用于 在集群上启动应用程序。它可以通过一个统一的接口使用所有 Spark 支持的 cluster managers，所以您不需要专门的为每个cluster managers配置您的应用程序。

### 打包

打包的时候, 需要将程序自身的jar与 程序的 依赖jar一起进行打包, 这一点可以通过maven 的 shade / assembly 来实现. 在 项目中 将 spark 和 hadoop 的包 范围权限定义为 provided即可.它们不需要被打包，因为在运行时它们已经被 Cluster Manager 提供了.

### 启动

打包完成之后, 就可以通过 bin/spark-submit 进行提交了.

这个脚本负责设置 Spark 和它的依赖的 classpath，并且可以支持 Spark 所支持的不同的 Cluster Manager 以及 deploy mode（部署模式）:

    ./bin/spark-submit \
    --class <main-class> \
    --master <master-url> \
    --deploy-mode <deploy-mode> \
    --conf <key>=<value> \
    ... # other options
    <application-jar> \
    [application-arguments]

常用的参数有:

* --class：您的应用程序的入口点（例如 org.apache.spark.examples.SparkPi)
* --master：集群的 master URL（例如 spark://23.195.26.187:7077）
* --deploy-mode：是在 worker 节点（cluster）上还是在本地作为一个外部的客户端（client）部署您的 driver（默认：client）
* --conf：按照 key=value 格式任意的 Spark 配置属性。对于包含空格的 value（值）使用引号包 “key=value” 起来。
* application-jar：包括您的应用以及所有依赖的一个打包的 Jar 的路径。该 URL 在您的集群上必须是全局可见的，例如，一个 hdfs:// path 或者一个 file:// 在所有节点是可见的。
* application-arguments：传递到您的 main class 的 main 方法的参数，如果有的话。

其中 参数顺序并没有严格要求,  但要求 jar路径 必须在倒数第二 或 最后一个参数位置(如果不通过 application-jar 来指定的话).

有一些特定于所使用的集群管理器的可用选项 。例如，对于具有部署模式的Spark standalone Cluster，您还可以指定--supervise以确保驱动程序在非零退出代码失败的情况下自动重新启动。要枚举所有可用的此类选项，请使用来spark-submit运行它--help.

其中 StandaloneCluster的 可配置参数在稍后会有所说明.

### Master URLS

Master URL | Meaning
-|-|-|
local | 使用一个线程本地运行 Spark（即，没有并行性）。
local[ K ] | 使用 K 个 worker 线程 在本地运行 Spark（理想情况下，设置这个值的数量为你的机器的 core 数量）。
local[K, F] | 使用 K 个 worker 线程本地运行 Spark并允许最多失败 F次（对于任意job失败会进行重试, 重试次数等 F - 1)
local[ * ] | 使用与机器的 逻辑 core数量相等的 worker线程.
local[*, F]	| 使用与机器的 逻辑 core数量相等的 worker线程. 并允许最多失败 F次。
spark://HOST:PORT | 连接至给定的 Spark standalone cluster master. master。该 port（端口）必须有一个作为您的 master 配置来使用，默认是 7077。
spark://HOST1:PORT1,HOST2:PORT2 | 连接至给定的 Spark standalone cluster with standby masters with Zookeeper。该列表必须包含由zookeeper设置的高可用集群中的所有master主机。该 port（端口）必须有一个作为您的 master 配置来使用，默认是 7077。
mesos://HOST:PORT | 连接至给定的 Mesos 集群。该 port（端口）必须有一个作为您的配置来使用，默认是 5050。或者，对于使用了 ZooKeeper 的 Mesos cluster 来说，使用 mesos://zk://...。使用 --deploy-mode cluster，来提交，该 HOST:PORT 应该被配置以连接到 MesosClusterDispatcher。
yarn | 以 client 或 cluster 模式 连接至一个 YARN cluster, 模式取决于 --deploy-mode. 该 cluster 的位置将根据 HADOOP_CONF_DIR 或者 YARN_CONF_DIR 变量来找到。
k8s://HOST:PORT | 以集群模式 连接至 k8s 集群, 在目前版本不支持设定客户端模式(在将来会提供), HOST PORT 指向对应 k8s API服务, 默认使用 TSL连接. 如果不想使用 TSL, 需要强制指定 k8s://http://HOST:PORT.

### 配置

spark-submit 脚本可以从一个 properties 文件加载默认的 Spark configuration values。默认情况下，它将从 Spark 目录下的 conf/spark-defaults.conf 读取配置. 

加载默认的 Spark 配置，可以在提交时省略一部分参数, 例如，如果 spark.master 属性被设置了，您可以在 spark-submit 中安全的省略 --master 配置. 一般情况下，明确设置在 SparkConf 上的配置值的优先级最高，然后是传递给 spark-submit的值，最后才是 default value（默认文件）中的值。

如果你不是很清楚其中的配置设置来自哪里，您可以通过使用 --verbose 选项来运行 spark-submit 打印出细粒度的调试信息.

### 高级依赖管理

并非只有把所有的需要的jar包都打包在一起这一种方式.

通过 --jars 选项包括的应用程序的 jar 和任何其它的 jar 都将被自动的传输到集群.  --jars 后面提供的 URL 必须用逗号分隔。该列表会被包含到 driver 和 executor 的 classpath 中。 --jars 不支持目录的形式。

URL有以下几种方式:

* file: 绝对路径和 file:/ URI 通过 driver 的 HTTP file server 提供服务，并且每个 executor 会从 driver 的 HTTP server 拉取这些文件。

* hdfs:, http:, https:, ftp: 指定下载 文件的 URI.

* local: 一个用 local:/ 开头的 URL,  要求作在每个 worker 节点上都存在。这样意味着没有网络 IO 发生，并且非常适用于那些已经被推送到每个 worker 或通过 NFS，GlusterFS 等共享的大型的 file/JAR。

<font color="orange">

注意: JARS 和 files 被复制到 每个SparkContext 的 executor 节点 的 工作目录. 在长时间的运行中, 所需要的空间会逐渐加大, 因此去清理掉这些文件. 在 YARN 模式下, 可以自动清理文件, 而在 standalone模式下, 需要在配置中加入spark.worker.cleanup.appDataTtl 用以自动清理.

</font>

## Standalone 模式

由于在我们目前的项目中, 采用的就是 standalone模式, 因此只介绍这一种模式.

Spark 提供了一个简单的 standalone 部署模式。你可以手动启动 master 和 worker 来启动 standalone 集群.

安装 Spark Standalone 集群，只需要将编译好的版本部署在集群中的每个节点上。








</font>