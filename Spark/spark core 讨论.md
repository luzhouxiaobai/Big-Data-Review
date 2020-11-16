# 第2节 Spark Core讨论

## 第2.1节 Spark 架构

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/spark架构.jpg" style="zoom:80%;" />

上图展示了Spark的架构的简单示意。

我们不妨先这样认识Spark，它有几个重要的部分：

- Master Node：它是集群部署时候的概念，是整个集群的控制器，负责集群的正常运行，管理Worker Node。
- Worker Node：它是计算节点，会接收Master Node的命令，并进行状态汇报。
- Executors：每个Worker Node上都有一个Executor，它负责完成任务的执行，是一个线程池。
- Spark集群部署后，需要在主从节点启动Master进程和Worker进程。运行Master进程的节点是Master Node，运行Worker进程的节点是Worker Node。

这就是一个简要版的Spark架构解释，让我们明白了Spark集群中最重要的几个组成部分，有了这些部门，我们就可以深入探讨Spark的其他内容。

还记得我们在MapReduce中，用户每次提交的都是一个作业。但是在Spark中却不是。Spark对于Job另有一番定义，我们在Spark基础中有过描述，在Spark，用户编写提交的代码称为一个应用（Application）。详细来说：

- Application：基于Spark的应用程序，包含一个Driver程序和多个Executor，当然，我们知道，这个Executor位于Worker中。下图给出了Application的内部图。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/application.jpg" style="zoom:80%;" />

这个图让我们十分轻松地理解了Spark中的若干个重要的概念。

- Driver：应用程序的执行起点，负责作业调度，它会执行Application中的main函数。
- Task：Spark中的基本执行单元，它在一个Executor中执行，对应Spark中的分区（partition）。
- Stage：一个作业的世系图按照宽依赖进行划分，分出的就是一个Stage，它也被称为 Task Set，一个Stage内部会包含若干个task。
- Job：由Spark中的行动操作触发。

此时，我们再回味这几个概念，是不是更清楚了？当然，你也可以去前一节看看这几个概念我们是怎么引出来的。我们需要注意的是， **一个Worker上只能有一个Executor，Worker和Executor之间是一一对应的关系** 。

这样看来，我们也不难得出结论，Spark也是 **Master/Slaver架构** 。

## 第2.2节 Spark 执行原理

### 一、先学会WordCount

我们依然把WordCount当作我们的基本用例，虽然前文已经给了WordCount代码，但是我们想试着写一下。还记得之前MapReduce的WordCount代码吗？明明简单的WordCount结果由于僵化的两阶段编程，导致代码又臭又长，反观Spark，言简意赅，极具美感。

第一次写，一定会懵，但是无所谓，我们先思考再动手。

- 我们需要写一个Spark的代码，我们前文说过，Driver进程是Application执行起点，它会执行Application的main函数。所以我们知道（当然，我扯了这么多，就算我不扯你也应该知道，代码是从main函数开始执行的）整个代码应该都是再main函数中书写的。
- 我们写Spark代码，需要写创建好Spark的执行环境，包括告诉计算机：Application的名字；启动几个节点。我们基于scala写代码，创建环境需要调用Spark中的 **SparkConf** 和 **SparkContext** 。（local模式是指代码在本地运行而非远程执行）。scala中，定义变量用 **var** ，定义常量用 **val** 。我们创建好Spark的上下文环境，就不要改变了，所以这里都用了 **val** 。其实，如果你还记得我们前文描述的 RDD，你就应该清楚，RDD是不可变的，如果需要修改，就只能创建新的 RDD。所以，其实我们下文都是用 **val** 定义不可变的常量。

```scala
val conf = new SparkConf() //初始话SparkConf类
			.setMaster("local[*]") //告诉集群，我们启用的是local模式，*表示启用尽可能多的节点，你可以把*改成具体数字
			.setAppName("WordCount") //应用名字为 WordCount
val sc = new SparkContext(conf) //创建Spark的执行环境
```

- 之后就是常规操作了，就是调用算子进行计算。我们这样思考，一开始我们仅仅是文本，Spark中没有Hadoop中的InputFormat将输入数据自动转化为键值对，因此就需要我们自己处理。我们的想法也很粗暴简单：

  - 文本拆分，将

  ```
  Hello, What's your name ? =>(拆分) 
  		(Hello,) + (what's) + (your) + (name) + (?)
  ```

  - 拆分完成后，将每个word组成键值对

  ```
  (Hello,) => (Hello, , 1)
  (what's) => (what's , 1)
  (your)  => (your , 1)
  (name) => (name , 1)
  (?) => (? , 1)
  ```

  - 之后，按照key指相加，就得到最后结果了。

  ```scala
  // 调用textFile算子读取文件数据
  val wordpair = sc.textFile("D:\\代码\\java\\Apache-Spark\\data.txt")
  val results = wordpair.flatMap(_.split(" ")) // 按空格对文本做拆分
  					.map((_,1)) // 组成键值对
  					.reduceByKey(_+_) // 按照key值相加
  ```

这样，一个WordCount就写完了。我们只需要将结果打印出来就可以了。

```scala
import org.apache.spark.{SparkConf, SparkContext}

object WordCount {

  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setMaster("local[*]").setAppName("WordCount") 
    val sc = new SparkContext(conf)
    val wordpair = sc.textFile("D:\\代码\\java\\Apache-Spark\\data.txt") 
    val results = wordpair.flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_)

    results.foreach(println) //打印
  }
}
```

### 二、Spark创建环境

Spark执行环境的创建是Spark代码执行的第一步，Driver会执行Application的main函数，创建SparkContext，即Spark的执行环境。我们先看一下Driver的组成。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/sparkenv.jpg" style="zoom:80%;" />

虽然我们说SparkContext就是Spark执行的环境，但是具体来看，内部又复杂了很多。

- RDD DAG：我们知道，Spark是基于RDD抽象进行描述的，RDD执行的流程会构成世系图，世系图其实就是一个有向无环图（DAG）。SparkContext中保有RDD DAG的信息。

- DAG Scheduler：DAG调度器，显然，输入就是DAG了，它会根据DAG中RDD的依赖关系，得到Stage，将Stage提交给Task Scheduler。

- Task Scheduler：接入Stage，拆分其中的Task提交给Executor执行。

- SparkEnv：线程级别的运行环境，存储运行时的重要组件的引用：

  - MapOutPutTracker：我们知道，Spark中会有Shuffle操作的，该组件就存储Shuffle的元信息。
  - BroadcastManager：控制广播变量，并存储其元信息。
  - BlockManager：存储管理，创建和查找块。
  - MetricsSystem：监控运行时性能指标信息。
  - SparkConf：存储配置信息

  

