# 第4节 MapReduce的进一步讨论

我们前面提了一嘴MapReduce。说它是一个采用了分治思想的分布式计算框架，本节我们就进一步细致讨论一下MapReduce。

## 第4.1节 MapReduce的并行化计算模型

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/mr并行编程.jpg" style="zoom:80%;" />

看图说话，我们基于一个大数据领域的“HelloWorld“，”wordcount"，结合上图中的MapReduce并行计算模型来讲解MapReduce的计算过程。

首先，我们假设我们拥有一篇初始的文章，我们需要统计文章中的所有的词语出现的个数。以一下这个小短文为例：

```
This distribution includes cryptographic software.  The country in which you currently reside 
may have restrictions on the import, possession, use, and/or re-export to another country, 
of encryption software. BEFORE using any encryption software, please check your country's laws, 
regulations and policies concerning the import, possession, or use, and re-export of encryption 
software, to see if this is permitted. See <http://www.wassenaar.org/> for more information.
```

细心的人已经发现了，这个是Hadoop安装包的README.txt中的一部分内容。

要统计这篇文章的词语方法很简单，读取文章，顺次统计每个词语出现的个数即可。**如果只有一个人，就一个人统计，如果有好几个人，就好几个人每个人分几行，先自己统计，然后将每个人的统计结果合并就可以了。**

**哎哎哎！打住！发现了没有，这就是分治啊！把一个活分给好几个人干。** 这也是MapReduce的想法，让原本需要一台机器做的事，分给好几个人做，这样不就快了嘛！

在MapReduce中，从数据输入到得到输出，整个流程所做的事情称为一个 **作业** 。

## 第4.2节 MapReduce的任务流程

我们按照图中的流程，梳理一下MapReduce的任务流程。

- 初始时，是上述的一个文本。MapReduce接收到作业输入后，会先进行数据拆分。
- 数据拆分完成之后，会有多个 **小文本** 数据，每个小文本都会作为一个Map任务的输入。这样一个大的MapReduce作业，会被分解为多个小的Map任务。
- Combiner会处理Map生成的数据，需要数据的时，此时Map生产的仅仅是中间结果。Combiner是一个可选的组件，用户不设置，他就不存在。
- 之后，数据会到达Partitioner，Partitioner组件会将中间数据按照哈希函数的对应规则，将中间结果分配到对应的Reducer所在节点上。
- Reducer会处理中间数据，得到最终的结果。

这就是，一个完整的MapReduce作业的生老病死的概括，其真实的流程自然远不止此，我们会在后面娓娓道来。

先让我们仔仔细细地了解一下上述过程的每一个组件。

### 一、扯一扯Map

有了上述的内容，我们可以进行下一步了。

按照我们说的，我们应该将这个小短文分成几个部分。也就是图中的数据划分。

#### （1）如果进行数据划分

当我们开启一个MapReduce程序，一般传入的输入都是一个体积巨大的数据。MapReduce接收到数据后，需要对数据进行划分。通俗来将，就是我们前文说的，我们该如果将一个小短文划分成多行，分配个多个人进行统计。

MapReduce中有一个InputFormat类，它会完成如下三个任务：

- 验证作业数据的输入形式和格式
- 将输入数据分割为若干个逻辑意义上的InputSplit，其中每一个InputSplit都将单独作为Map任务的输入。也就是说，InputSplit的个数，代表了Map任务的个数。需要注意，这里并没有做实际切分，仅仅是将数据进行逻辑上的切分。
- 提供一个RecordReader，用于将Map的输入转换为若干个记录。虽然MapReduce作业可以接受很多种格式的数据，但是Map任务接收的任务其实是键值对类型的数据，因此需要将初始的输入数据转化为键值对。RecordReader对象会从数据分片中读取出数据记录，然后转化为 **Key-Value** 键值对，逐个输入到Map中进行处理。

问题在于，这个InputFormat类该如何进行划分呢？在FileInputFormat类中，会有一个getSplits函数，这个函数所做的事情其实就是进行数据切分的过程。我们稍微看一下这个函数：

```java
public List<InputSplit> getSplits(JobContext job) throws IOException {
    StopWatch sw = new StopWatch().start();
    long minSize = Math.max(getFormatMinSplitSize(), getMinSplitSize(job));
    long maxSize = getMaxSplitSize(job);
    //...
    for (FileStatus file: files) {
        if (isSplitable(job, path)) {
          long blockSize = file.getBlockSize();
          long splitSize = computeSplitSize(blockSize, minSize, maxSize);
            //...
        }
        //...
    }
    //...
}

protected long computeSplitSize(long blockSize, long minSize,
                                long maxSize) {
    return Math.max(minSize, Math.min(maxSize, blockSize));
}
```

- minSize :每个split的最小值，默认为1.getFormatMinSplitSize()为代码中写死，固定返回1，除非修改了hadoop的源代码.getMinSplitSize(job)取决于参数mapreduce.input.fileinputformat.split.minsize，如果没有设置该参数，返回1.故minSize默认为1.
- maxSize：每个split的最大值，如果设置了mapreduce.input.fileinputformat.split.maxsize，则为该值，否则为Long的最大值。
- blockSize ：默认为HDFS设置的文件存储BLOCK大小。注意：该值并不一定是唯一固定不变的。HDFS上不同的文件该值可能不同。故将文件划分成split的时候，对于每个不同的文件，需要获取该文件的blocksize。
- splitSize ：根据公式，默认为blockSize 。

从上述代码中可以看到，这个InputSize在 [minSize, maxSize] 之间。

#### （2）这样，我们可以理一理划分逻辑

- 1）遍历输入目录中的每个文件，拿到该文件
- 2）计算文件长度，A:如果文件长度为0，如果`mapred.split.zero.file.skip=true`，则不划分split ; 如果`mapred.split.zero.file.skip`为false，生成一个length=0的split .B:如果长度不为0，跳到步骤3
- 3）判断该文件是否支持split :如果支持，跳到步骤4;如果不支持，该文件不切分，生成1个split，split的length等于文件长度。
- 4）根据当前文件，计算`splitSize`。
- 5）判断`剩余待切分文件大小/splitsize`是否大于`SPLIT_SLOP`(该值为1.1，代码中写死了) 如果true，切分成一个split，待切分文件大小更新为当前值-splitsize ，再次切分。生成的split的length等于splitsize； 如果false 将剩余的切到一个split里，生成的split length等于剩余待切分的文件大小。之所以需要判断`剩余待切分文件大小/splitsize`,主要是为了避免过多的小的split。比如文件中有100个109M大小的文件，如果`splitSize`=100M，如果不判断`剩余待切分文件大小/splitsize`，将会生成200个split，其中100个split的size为100M，而其中100个只有9M，存在100个过小的split。MapReduce首选的是处理大文件，过多的小split会影响性能。

划分好Split之后，这些数据进入Map任务，按照用户设计处理逻辑进行处理。Map可以由用户定义设计处理逻辑。

### 二、聊一聊Combiner

Combiner组件并不是一个必须部分，用户可以按照实际的需求灵活的添加。Combiner组件的主要作用是 **减少网络传输负载，优化网络数据传输优化** 。

当我们Map任务处理完成之后，上述的文本会变成一个一个的 **Key-Value** 对。

```
(This, 1)
(distribution, 1)
...
```

在没有Combiner组件前提下，这些键值对会直接传输到Reducer端，进行最后的统计工作。但是这一步是可以优化的，因为Map端仅仅是将每行的词拆分了，但是其实可以再做一步统计的。

例如，我们假设在Map任务A这里出现了两次 （This, 1），我们可以做一次统计，将这个Map任务上的This做一次统计，生成（This, 2）。在大数据场合，千万个这样的相同词的合并会显著降低网络负载。

**但是并不是所有的场合都适用Combiner，这个组件是可有可无的，用户需要按照自己的需求灵活决定** 。

**因为Combiner可以存在，也可以不存在，所有，我们设计Combiner时，要保证Combiner的key-value和Map的key-value一致** 。

### 三、 瞅一瞅Partitioner

为了保证所有主键相同的键值对会传输到同一个Reducer节点，以便Reducer节点可以在不访问其他Reducer节点的情况下就可以计算粗最终的结果，我们需要对来自Map（如果有Combiner，就是Combiner之后的结果）中间键值对进行分区处理，Partitioner主要就是进行分区处理的。

#### Partitioner 默认的分发规则

**根据 `key` 的 `hashcode%reduce task` 数来分发**，所以：**如果要按照我们自己的需求进行分组，则需要改写数据分发（分区）组件 Partitioner**

- **Partition 的 key value, 就是Mapper输出的key value**

  ```java
  public interface Partitioner<K2, V2> extends JobConfigurable {
    
    /** 
     * Get the paritition number for a given key (hence record) given the total 
     * number of partitions i.e. number of reduce-tasks for the job.
     *   
     * <p>Typically a hash function on a all or a subset of the key.</p>
     *
     * @param key 用来partition的key值。
     * @param value 键值对的值。
     * @param numPartitions 分区数目。
     * @return the partition number for the <code>key</code>.
     */
    int getPartition(K2 key, V2 value, int numPartitions);
  }
  ```

  **输入是Map的结果对<key, value>和Reducer的数目，输出则是分配的Reducer（整数编号）**。**就是指定Mappr输出的键值对到哪一个reducer上去**。系统缺省的Partitioner是HashPartitioner，它以key的Hash值对Reducer的数目取模，得到对应的Reducer。**这样保证如果有相同的key值，肯定被分配到同一个reducre上。如果有N个reducer，编号就为0,1,2,3……(N-1)**。

- MapReduce 中会将 map 输出的 kv 对，按照相同 key 分组，然后分发给不同的 reducetask 默认的分发规则为:根据 key 的 hashcode%reduce task 数来分发，所以:如果要按照我们自 己的需求进行分组，则需要改写数据分发(分组)组件 Partitioner, 自定义一个 CustomPartitioner 继承抽象类:Partitioner

- **因此， Partitioner 的执行时机， 是在Map输出 key-value 对之后**

### 四、MapReduce中的Sort

MapReduce中的很多流程都涉及到了排序，我们会在后面详细说明。

从整个MapReduce的程序执行来看，整个过程涉及到了 **快排、归并排序、堆排** 三种排序方法。

### 五、遛一遛Reduce

Reduce会处理上游（Map，也可能有Combiner）的中间结果。

需要注意的是，Map到Reduce整个过程中，键值的变化是不一样的

1. 初始是文本内容，会被RecordReader处理为键值对 `<key-value>`
2. 经过Map（也可能有Combiner）后，仍然是键值对形式 `<key-value>`
3. 经过Partition，到达Reduce的结果是 `key - list(value)` 形式。所以在Reduce处理的value其实一个整体。

Reduce会把所有的结果处理完成，输出到对应的输出路径。

**弊端**

MapReduce的Reduce处理结果最后都是需要落盘的，当粘结多个MapReduce代码时，无法有效利用内存。

## 第4.3节 MapReduce中的shuffle

shuffle翻译为中文就叫 ”洗牌“，把一组有一定规则的数据尽量转换成一组无规则的数据，越随机越好。MapReduce中的Shuffle更像是洗牌的逆过程，把一组无规则的数据尽量转换成一组具有一定规则的数据。有些抽象，当前不理解也没关系，先了解一下shuffle的流程。

**本节内容参考了如下内容**

[1] **MapReduce:详解Shuffle(copy sort merge)过程** https://blog.csdn.net/xiaolang85/article/details/8528892

[2] **MapReduce shuffle过程详解** https://blog.csdn.net/u014374284/article/details/49205885

[3] **从零开始学Hadoop大数据分析** 6.2.5 Shuffle过程

### 一、Shuffle概览

Shuffle不是一个单独的任务，它是MapReduce执行中的步骤，Shuffle过程会涉及到上面谈到的Combiner、Sort、Partition三个过程。它横跨MapReduce中的Map端和Reduce端，因此，我们也会将Shuffle分为 **Map Shuffle** 和 **Reduce Shuffle** 。

​    在Hadoop这样的集群环境中，大部分map task与reduce task的执行是在不同的节点上。当然很多情况下Reduce执行时需要跨节点去拉取其它节点上的map task结果。如果集群正在运行的job有很多，那么task的正常执行对集群内部的网络资源消耗会很严重。这种网络消耗是正常的，我们不能限制，能做的就是最大化地减少不必要的消耗。还有在节点内，相比于内存，磁盘IO对job完成时间的影响也是可观的。从最基本的要求来说，我们对Shuffle过程的期望可以有： 

- 完整地从map task端拉取数据到reduce 端。
- 在跨节点拉取数据时，尽可能地减少对带宽的不必要消耗。
- 减少磁盘IO对task执行的影响。

### 二、Map端的Shuffle

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/shuffle-spill.jpg" style="zoom:80%;" />

如上图所示，在Map端的shuffle过程是对Map的结果进行分区、排序、分割，然后将属于同一划分（分区）的输出合并在一起并写在磁盘上，最终得到一个**分区有序**的文件，分区有序的含义是Map输出的键值对按分区进行排列，具有相同partition值的键值对存储在一起，每个分区里面的键值对又按key值进行升序排列（默认），这里的partition值是指利用Partitioner.getPartition得到的结果，他也是Partitioner分区的依据。上述流程还可以用如下图进行表示：
<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/shuffle-map.jpg" style="zoom:80%;" />



