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

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/spark-pi.png" style="zoom:80%;" />

我们也可以调用Spark的scala-shell交互式界面。

```shell
bin/spark-shell
```

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/spark-shell.png" style="zoom:80%;" />

我们在4040端口，可以看到Spark的WebUI。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/spark-web.png" style="zoom:80%;" />

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

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/idea-p1.png" style="zoom:80%;" />

之后，我们就可以在这个Scala文件夹下创建Scala项目了。需要注意的是，如果你的IDEA还没有配置Scala的编码支持，需要自行百度解决一下。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/idea-p2.png" style="zoom:80%;" />

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

- 创建操作

  他是指