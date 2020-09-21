# 第1节 Hadoop原理简述

## 第1.1节 Hadoop架构

Hadoop系统由两部分组成，分别是分布式文件系统HDFS (Hadoop Distributed File System) 和分布式计算框架MapReduce。其中，分布式文件系统主要用于大规模数据的分布式存储，而MapReduce则构建在分布式文件系统之上，对存储在分布式文件系统中的数据进行分布式计算。下图简单展示了Hadoop系统的架构。

<img src='https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/hadoop1.png' style='zoom:80%'/>

从图中可以清晰的看出Hadoop系统由MapReduce和HDFS两个部分组成。从软件架构角度来看，Hadoop基于每个节点上的本地文件系统，构建了逻辑上整体化的分布式文件系统（HDFS），以此提供大规模可扩展的分布式数据存储功能；同时，Hadoop利用每个节点的计算资源，协调管理集群中的各个节点完成计算任务。图中涉及到几个概念：

- 主控节点：JobTracker为MapReduce的主控节点，NameNode 为HDFS的主控节点，他们负责控制和管理整个集群的正常运行。
- 从节点：TaskTracker为MapReduce的从节点，负责具体的任务执行；DataNode为HDFS的从节点，负责存储具体的数据。

可以看出，Hadoop服从Master/Slaver（主从架构）。在集群部署的时候，一般JobTracker和NameNode部署在同一节点上，TaskTracker和DataNode部署在同一节点。

## 第1.2节 HDFS原理简述

### 一、HDFS的特征

- 大规模数据的分布存储能力

  HDFS以分布式方式和良好的可扩展提供了大规模数据的存储能力。

- 高并发访问能力

  HDFS以多节点并发访问的方式提供很高的数据访问带宽（高数据吞吐率）。

- 容错能力

  在分布式环境中，失效应当被认为是常态。因此，HDFS必须具备正确检测硬件故障，并且能快速从故障中恢复过来，确保数据不丢失。为了保证数据的不丢不出错，HDFS采用了多副本的方式（默认副本数目为3）。

- 顺序文件访问

  大数据批处理在大多数情况下都是大量简单记录的顺序处理。针对这个特性，为了提高大规模数据访问的效率，HDFS对顺序读进行了优化，但是对于随机访问负载较高。

- 简单的一致性模型

  支持大量数据的一次写入，多次读取。

- 数据块存储模式

  HDFS采用基于大粒度数据块的方式进行文件存储。默认的块大小是64MB，这样的好处是可以减少元数据的数量。

### 二、HDFS的架构

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/tree/master/file/HDFS1.jpg" style="zoom:80%;" />



## 第1.3节 MapReduce原理简述

### 一、MapReduce思想

分而治之是大数据技术的基本基本思想。MapReduce同样借助了分治思想，将大数据切分为“小”数据，再并行进行处理。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/mapreduce1.jpg" style="zoom:50%" />

MapReduce能够解决的问题有一个共同特点：任务可以被分解为多个子问题，且这些子问题相对独立，彼此之间不会有牵制，待并行处理完这些子问题后，任务便被解决。在实际应用中，这类问题非常庞大，谷歌在论文中提到了MapReduce的一些典型应用，包括分布式grep、URL访问频率统计、Web连接图反转、倒排索引构建、分布式排序等
