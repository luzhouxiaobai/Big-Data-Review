# 第1节 Spark基础

## 第1.1节 为什么需要Spark

### 一、MapReduce计算模式的弊端

- 首先，MapReduce的设计是为了用于高吞吐量的批数据处理作业的，本身的延迟就相对较高
- 其次，数据是存储与HDFS中的，数据共享效率低。我们在前面WordCount的例子中可以看到，Job最后的结果必须要落盘。在MapRduce的Shuffle流程中，我们曾经介绍过，MapReduce的数据在Map端处理完成之后也都是存储在磁盘上的，没有很好的利用内存。当然，这一点其实是时代的限制，因为MapReduce模式在设计时，内存还是十分昂贵的
- 再者，MapReduce对复杂计算的支持并不好，尤其是对于图计算、迭代计算。还记得那个WordCount吗？还记得那个两阶段的编程框架Mapper-Reducer吗？Hadoop的表达是匮乏的，在复杂计算中，无法高效的表达出计算逻辑。并且，MapReduce中的每次操作落盘，这倒是大量的时间浪费在IO上。熟悉操作系统的人都知道，IO操作是相当耗时的，这些问题，导致MapReduce在复杂计算上表现乏力。

### 二、Spark能做到什么

MapReduce的弊端促使科学家探索新的计算模式。在后Hadoop时代，开始出现新的大数据计算模式和系统。其中尤其以内存计算为核心，集诸多计算模式之大成的Spark生态是典型代表。Spark支持如下的计算：

- 大数据查询分析计算
- 批处理计算
- 流式计算
- 迭代计算
- 图计算
- 内存计算

相较于Hadoop MapReduce，Spark的优势也很明显，首先是 **速度** ，官网的对比图简洁明了。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/speed.png" style="zoom:80%;" />

再者是 **广泛性** ，Spark支持SQL，Streaming，以及Graph等。

其次是 **多处运行** ，Spark可以在Hadoop，Mesos，Standalone或者在云上执行。

最后是 **易用性** ，Spark支持Java， Scala， Python和R

### 三、 Spark的基本组件

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/spark组件.png" style="zoom:80%;" />

上图展示了Spark生态中的基本组件。当然，随着Spark社区的发展，现在的Spark组件会更丰富一些。但是，不论是Spark SQL，Spark Streaming，Spark MLlib，还是GraphX组件，都是基于基本的Spark，即Spark Core。后续，我们也是基于Spark Core来介绍Spark。

## 第1.2节 Spark的安装

### 一、在Linux的安装

如果你已经安装好了Hadoop，那么你现在安装Spark应该是不费力气的。

首先从官网下载你想要的Spark，一般来说，如果你已经有了

## 第1.3节 Spark RDD

RDD的全称是Resilient Dirstributed DataSets，叫做弹性分布式数据集。

首先，需要明确的一点，它是一个 **抽象** 数据集，言下之意是，它并不是一个包含数据本体的变量或者集合，而是一个仅保存数据描述的抽象集合，这个集合中存储了该数据的一些信息（数据位置、分区等）。

其次，他是一个 **分布式** 的，也就是，这些数据可能存储在集群上的任何地方，并不是在本地。

再者，他是 **弹性** 的，言下之意是，它既可以保存在内存中，也可能会因为内存空间不足而转而存储到磁盘上的。

RDD解决的第一个问题就是数据共享的问题，MapReduce的数据共享是基于磁盘的，但是Spark的数据共享是基于内存的，RDD的就是基于内存的。其次，由于RDD是分布式的，所以我们在基于RDD进行计算时，执行的并行计算。RDD中会有 **分区** 的概念。存储在不同地区的数据，所在的不同区域称之为分区，不同的分区会共同进行计算，即，并行计算。

最后，RDD是 **不可变** 的。可以理解为RDD是一个常量，一旦RDD计算生成，其中的数据就不可改变的。我们只能基于该RDD再次计算，生成新的RDD，而不能直接修改原RDD。

### 一、Spark中的操作

Spark Core中的操作基本都是基于Spark RDD来进行的。大体上来看，这些操作可以分为四大类。

- 创建操作

  他是指