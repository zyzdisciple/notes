# Spark调优

<font face="楷体">

由于大多数 Spark 计算的内存性质，Spark 程序可能由集群中的任何资源: 如 CPU，网络带宽, 内存 导致瓶颈, 通常情况下，如果数据有合适的内存，瓶颈就是网络带宽，但有时您还需要进行一些调整，例如 以序列化形式存储 RDD 来减少内存的使用。

本指南将涵盖两个主要的主题：数据序列化，这对于良好的网络性能至关重要，并且还可以减少内存使用和内存优化。我们选几个较小的主题进行展开。

## 数据序列化

序列化在任何分布式应用程序的性能中起着重要的作用。当你保存一个RDD, 无论是以 内存 还是 硬盘 的方式, 每个节点都会将其计算的所有分区存储在内存中，并在该数据集（或从该数据集派生的数据集）上的其他操作中重用它们。这样可以使以后的操作更快（通常快10倍以上）. 缓存是用于迭代算法和快速交互使用的关键工具。

你可以通过 persist() 或 cache() 方法标记RDD, 这样 它就会在 将来第一次 执行 action操作的时候 被保存在对应的节点. 

You can mark an RDD to be persisted using the persist() or cache() methods on it. The first time it is computed in an action, it will be kept in memory on the nodes. Spark’s cache is fault-tolerant – if any partition of an RDD is lost, it will automatically be recomputed using the transformations that originally created it.

很慢的将对象序列化或消费大量字节的格式将会大大减慢计算速度。通常，这可能是您优化 Spark 应用程序的第一件事。Spark 宗旨在于方便和性能之间取得一个平衡(允许您使用操作中的任何 Java 类型).

When you persist an RDD, each node stores any partitions of it that it computes in memory and reuses them in other actions on that dataset (or datasets derived from it). This allows future actions to be much faster (often by more than 10x). Caching is a key tool for iterative algorithms and fast interactive use.

有两种序列化方式:



</font>