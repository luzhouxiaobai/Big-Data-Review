# 第2节 Spark Core讨论

## 第2.1节 Spark 架构

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/spark架构.jpg" style="zoom:80%;" />

上图展示了Spark的架构的简单示意。

我们不妨先这样认识Spark，它有几个重要的部门：

- Master Node：它是集群部署时候的概念，是整个集群的控制器，负责集群的正常运行，管理Worker Node。
- Worker Node：它是计算节点，会接收Master Node的命令，并进行状态汇报。
- Executors：每个Worker Node上都有一个Executor，它负责完成任务的执行。
- Spark集群部署后，需要在主从节点启动Master进程和Worker进程。运行Master进程的节点是Master Node，运行Worker进程的节点是Worker Node。

这就是一个简要版的Spark架构解释，让我们明白了Spark集群中最重要的几个组成部分，有了这些部门，我们就可以深入探讨Spark的其他内容。

还记得我们在MapReduce中，用户每次提交的都是一个作业。但是在Spark中却不是。Spark对于Job另有一番定义，我们在Spark基础中有过描述，在Spark，用户编写提交的代码称为一个应用（Application）。详细来说：

- Application：基于Spark的应用程序，包含一个Driver程序和多个Executor，当然，我们知道，这个Executor位于Worker中。下图给出了Application的内部图。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/spark/application.jpg" style="zoom:80%;" />