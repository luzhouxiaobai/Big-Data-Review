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

<img src="../file/spark/speed.png" style="zoom:80%;" />

再者是 **广泛性** ，Spark支持SQL，Streaming，以及Graph等。

其次是 **多处运行** ，Spark可以在Hadoop，Mesos，Standalone或者在云上执行。

最后是 **易用性** ，Spark支持Java， Scala， Python和R

### 三、 Spark的基本组件

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/spark组件.png" style="zoom:80%;" />

上图展示了Spark生态中的基本组件。当然，随着Spark社区的发展，现在的Spark组件会更丰富一些。但是，不论是Spark SQL，Spark Streaming，Spark MLlib，还是GraphX组件，都是基于基本的Spark，即Spark Core。后续，我们也是基于Spark Core来介绍Spark。

## 第1.2节 Spark的安装

### 一、Spark安装

如果你已经安装好了Hadoop，那么你现在安装Spark应该是不费力气的。

首先从官网下载你想要的Spark，一般来说，如果你已经安装好了Hadoop，你可以选择安装 *pre-built with user-provider Apache Hadoop* 版本。否则你可以选择其他自己喜欢的版本。一般来说，很少由于版本问题报错。

但是，为了统一，建议使用 *Scala-2.11* 版本。

将下载的好的Spark解压，放到你想要安装的位置。

说出来你可能不信，但是，这样Spark确实就装好了。这样，Spark就算安装完成了。

之后，我们可以跑一个样例。

```shell
cd spark #进入spark的主目录
bin/run-example SparkPi 2>&1 | grep "Pi is"
```

这是一个计算Pi值的示例程序，我们可以看到相应的结果。

<img src="../file/spark/spark-pi.png" style="zoom:80%;" />

我们也可以调用Spark的scala-shell交互式界面。

```shell
bin/spark-shell
```

<img src="../file/spark/spark-shell.png" style="zoom:80%;" />

我们在4040端口，可以看到Spark的WebUI。

<img src="../file/spark/spark-web.png" style="zoom:80%;" />

### 二、Window下IDEA配置Spark的执行环境

前面我们提到过，如果在Windows下配置Hadoop的运行环境，重点在于安装winutils，并配置 *Hadoop_home* 。忘记的人，可以回过头去看看，有了这些之后，其实就可以直接编写Spark代码了。当然，如果你是在linux环境下，那么这一步其实也是可以省略的。

打开IDEA，新建一个Maven项目，名字就叫 *Apache-Spark* 。添加如下Maven依赖。

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <version>2.4.6</version>
    </dependency>
</dependencies>
```

之后，我们main文件夹下创建一个新的文件夹，命名为Scala，再标记为Source Root。

<img src="../file/spark/idea-p1.png" style="zoom:60%;" />

之后，我们就可以在这个Scala文件夹下创建Scala项目了。需要注意的是，如果你的IDEA还没有配置Scala的编码支持，需要自行百度解决一下。

<img src="../file/spark/idea-p2.png" style="zoom:80%;" />

我们可以创建一个名为WordCount的scala文件。将下面的代码填进去。

```scala
import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object WordCount {

  def main(args: Array[String]): Unit = {

    val conf = new SparkConf().setMaster("local[*]").setAppName("WordCount")
    val sc = new SparkContext(conf)

    val wordpair: RDD[String] = sc.textFile("D:\\代码\\java\\Apache-Spark\\data.txt") //换成你自己的路径
    val results = wordpair.flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_)

    results.foreach(println)
  }
}
```

代码执行没问题，就意味着配置成功了。

### 三、Windows下PyCharm配置Spark的执行环境

直接使用pip命令或者在Anconda中下pyspark。

```shell
pip install pyspark
```

之后就可以进行编程了。当然，前提是你已经配置好了Hadoop在Windows下的执行环境。

```python
from pyspark import SparkConf, SparkContext

conf = SparkConf().setMaster('local[*]').setAppName("WordCount")
sc = SparkContext(conf=conf)

wordpair = sc.textFile("D:\\代码\\python\\Apache-Spark\\data.txt") # 换成你的路径
results = wordpair.flatMap(lambda x : x.split(" ")).map(lambda x : (x,1)).reduceByKey(lambda x, y : x + y)

results.foreach(print)
```

###### Notice !!!  已经但凡涉及到Windows下的执行，基础都是之前安装好了Hadoop的Windows环境。如果你还没安装，建议先安装。

## 第1.3节 Spark RDD

RDD的全称是Resilient Dirstributed DataSets，叫做弹性分布式数据集。

首先，需要明确的一点，它是一个 **抽象** 数据集，言下之意是，它并不是一个包含数据本体的变量或者集合，而是一个仅保存数据描述的抽象集合，这个集合中存储了该数据的一些信息（数据位置、分区等）。

其次，他是一个 **分布式** 的，也就是，这些数据可能存储在集群上的任何地方，并不是在本地。

再者，他是 **弹性** 的，言下之意是，它既可以保存在内存中，也可能会因为内存空间不足而转而存储到磁盘上的。

RDD解决的第一个问题就是数据共享的问题，MapReduce的数据共享是基于磁盘的，但是Spark的数据共享是基于内存的，RDD的就是基于内存的。其次，由于RDD是分布式的，所以我们在基于RDD进行计算时，执行的并行计算。RDD中会有 **分区** 的概念。存储在不同地区的数据，所在的不同区域称之为分区，不同的分区会共同进行计算，即，并行计算。

最后，RDD是 **不可变** 的。可以理解为RDD是一个常量，一旦RDD计算生成，其中的数据就不可改变的。我们只能基于该RDD再次计算，生成新的RDD，而不能直接修改原RDD。

### 一、Spark中的操作

Spark Core中的操作基本都是基于Spark RDD来进行的。大体上来看，这些操作可以分为四大类。

- **创建操作**

  他是用于RDD创建的操作，RDD的创建只有两种方式：一种是来自于内部集合和外部存储系统，另一种是通过转换操作生成的RDD。这里的 **创建操作** 指的是第一种。

  *常用创建算子举例* 注意，这里给出的所有API都是Scala API。 ：

  ```scala
  parallelize[T](seq: Seq[T], numSlices: Int = defaultParallelism): RDD[T]
  // seq表示一个集合，numSlices表示分区的数目，你可以理解为它存储在多少机器上
  // 当Spark基于RDD进行计算的时候，分区数目其实就决定了他的并行度
  
  textFile(path: String, minPartitions: Int = defaultMinPartitions): RDD[String]
  // spark会从该路径path中读取数据，默认按照系统设置的最小分区defaultMinPartition进行分区
  // 得到的RDD的类型是String字符串类型
  ```

- **转换操作**

  他是指将RDD通过一定的操作转换为新的RDD的操作。

  *常用RDD算子举例*：

  ```scala
  map[U](f:(T) => U): RDD[U]
  // 对RDD中数据集合应用函数f
  
  distinct(): RDD[(T)]
  // 去重
  
  flatMap[U](f:(T) => TraversableOnce[U]): RDD[U]
  // 对RDD中的数据集合应用函数f
  ```

  **Notice!!! map和flatMap的区别**

  <img src="../file/spark/spark-rdd1.png" style="zoom:80%;" />

  可以这么认为：

  **map保有了集合原有的结构，所以我们看到结果是一个Array内部含有多个Array** 。

  **flatMap则会破坏原有的集合结构，相当于把集合先拍扁，再应用函数f** 。

- **控制操作**

  进行RDD持久化的的操作，可以让RDD按照不同的存储策略保存在磁盘中或者内存中。

  *常用算子举例* ：

  ```scala
  cache(): RDD[T]
  // 将数据缓存
  
  persist(): RDD[T]
  // 数据持久化
  ```

- **行动操作**

  能够触发RDD运行的操作。

  *常用算子举例* ：

  ```scala
  fist()
  // 返回RDD中的第一个元素
  
  collect()
  // 将RDD转化为数组
  ```

### 二、Spark计算

Spark中引入了 **惰性计算** 的概念，它是指，程序在执行过程，并不是真正的进行了计算，而是使用某种方式记录了计算的线路，等到合适的时候再触发真正的计算。

**Spark中，行动操作的算子是触发计算的标志** 。每次行动操作触发的计算称为一个 **Job** 。

Spark采用 **世系图（lineage）** 的方式记录计算的线路。也就是说，在遇到行动操作前，Spark仅仅是记录当前所做的操作，并不会进行真正的计算，遇到行动操作后，才会触发计算，得到结果。

既然Spark的计算可以用世系图来记录，那我们当然可以将他们画出来。我们以上文中的WordCount为例。首先，我们先明确几个需要注意的点：

- 我们前面提过，Spark中的RDD是不可变，只能有创建操作生成，或者从前几个RDD通过转化操作生成。这就意味着Spark的RDD具有依赖关系，若A RDD通过转换操作变成了B RDD，那么A称为B的父RDD，子RDD同理。
- 我们前面也说过，RDD中有分区的概念。分区决定了计算时的并行度，也就是说，一个RDD中，有多个分区，每个分区中的数据在计算时是单独计算的，所以，其实 **一个分区就是对应Spark中的一个 Task** 。这样，我们就介绍了Spark中两个重要的概念 Job 和 Task了。但是我们介绍Task只是个引子，目的是为了引入Shuffle。还记得MapReduce中的Shuffle，数据从Map端到Reduce端，会经历Partitioner函数的划分，将一个Map节点上的数据划分到多个Reduce节点上。Spark中也有Shuffle，一个分区中的数据，自然也有可能会被发送到子RDD的多个分区中。 **这里就引入了一个概念，如果父RDD到子RDD之间的依赖不是Shuflle，也就是说不存在父RDD的一个分区被子RDD的多个分区依赖，就称为窄依赖，否则就是宽依赖** 。但是需要注意到是，并不是所有的转换算子都是Shuffle算子，只有类似于 `join` 这种会将一个分区中的数据发送到子RDD的多个分区中的算子才是Shuffle算子。

这样，我们可以画一个世系图了。我们先看WordCount的代码：

```scala
import org.apache.spark.{SparkConf, SparkContext}

object WordCount {

  def main(args: Array[String]): Unit = {
	//创建Spark的配置信息，setMaster是设置CPU个数，local[*]意味采用本地运行模式，调用本地尽可能多的CPU
    //setAppName为设置Application的名字
    val conf = new SparkConf().setMaster("local[*]").setAppName("WordCount") 
    //创建Spark的执行环境
    val sc = new SparkContext(conf)
	//读取对应路径的数据，生成RDD，是创建操作
    val wordpair = sc.textFile("D:\\代码\\java\\Apache-Spark\\data.txt") 
    val results = wordpair.flatMap(_.split(" ")) //转换操作，对数据做拆分
      .map((_,1)) //转换操作，将单个字符串映射为键值对
      .reduceByKey(_+_) //转换操作，按照Key对键值对进行加和

    results.foreach(println) //打印
  }
}
```

有了注释，我们应该可以看懂代码了。然后我们启动spark-shell，在shell中运行代码。在shell运行代码就不必创建Spark的执行环境了。直接贴操作部分的代码就可以。

<img src="../file/spark/wordcount1.png" style="zoom:80%;" />

这个时候，我们可以在4040端口看到世系图。

<img src="../file/spark/wordcount2.png" style="zoom:60%;" />

世系图可以形象地看到算子的变换过程。但是，其实世系图中还有一个概念，`stage` ，也叫`task set` ，他通过宽依赖进行划分，若我们按照宽依赖将世系图切分开，剩下的一个个就是Stage。

由图中可以看出，wordcount的执行分为两个 `stage` ，`stage 0` 内部的算子都是窄依赖的，`stage 1` 内只有一个算子，reduceByKey是一个Shuffle算子，因为他会统计一个RDD中所有分区的内容，假设一个该RDD有3个分区，每个分区中都有一个key为A的数据，那么reduceByKey在计算时，就会统计这三个分区中的key为A的数据，并存储到一个新的RDD的一个分区中。这个过程其实就是Shuffle过程。

综上，我们就聊完了Spark中最基础的三个概念，**Job, task, stage** 。简单来说，Job和Stage都是针对RDD的执行流程而言的，Task则是具体到RDD的每个分区。

**那么，我们可以提一个问题？Spark中的分区和MapReduce中的分区（partition）有什么异同吗** ？

- MapReduce中分区是指Partitioner组件依据哈希函数，将键值对发送到相应的reduce节点上。
- Spark的分区涉及的是任务的并行度，对应的是task这个概念。

