# 第1节 Hadoop原理简述

## 第1.1节 Hadoop架构

Hadoop系统由两部分组成，分别是分布式文件系统HDFS (Hadoop Distributed File System) 和分布式计算框架MapReduce。其中，分布式文件系统主要用于大规模数据的分布式存储，而MapReduce则构建在分布式文件系统之上，对存储在分布式文件系统中的数据进行分布式计算。下图简单展示了Hadoop系统的架构。

<img src='D:\远程仓库\Big-Data-Review\file\hadoop1.png' style='zoom:80%'/>

从图中可以清晰的看出Hadoop系统由MapReduce和HDFS两个部分组成。从软件架构角度来看，Hadoop基于每个节点上的本地文件系统，构建了逻辑上整体化的分布式文件系统（HDFS），以此提供大规模可扩展的分布式数据存储功能；同时，Hadoop利用每个节点的计算资源，协调管理集群中的各个节点完成计算任务。图中涉及到几个概念：

- 主控节点：JobTracker为MapReduce的主控节点，NameNode 为HDFS的主控节点，他们负责控制和管理整个集群的正常运行。
- 从节点：TaskTracker为MapReduce的从节点，负责具体的任务执行；DataNode为HDFS的从节点，负责存储具体的数据。

可以看出，Hadoop服从Master/Slaver（主从模式）。在集群部署的时候，一般JobTracker和NameNode部署在同一节点上，TaskTracker和DataNode部署在同一节点。

## 第1.2节 HDFS原理简述

### 

## 第1.3节 MapReduce原理简述

### 一、MapReduce思想

分而治之是大数据技术的基本基本思想。MapReduce同样借助了分治思想，将大数据切分为“小”数据，再并行进行处理。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/mapreduce1.jpg" style="zoom:50%" />

MapReduce能够解决的问题有一个共同特点：任务可以被分解为多个子问题，且这些子问题相对独立，彼此之间不会有牵制，待并行处理完这些子问题后，任务便被解决。在实际应用中，这类问题非常庞大，谷歌在论文中提到了MapReduce的一些典型应用，包括分布式grep、URL访问频率统计、Web连接图反转、倒排索引构建、分布式排序等