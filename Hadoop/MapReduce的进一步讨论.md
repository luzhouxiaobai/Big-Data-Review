# 第4节 MapReduce的进一步讨论

我们前面提了一嘴MapReduce。说它是一个采用了分治思想的分布式计算框架，本节我们就进一步细致讨论一下MapReduce。

大数据背景下，数据量巨大，这点没有问题。数据巨大带来的问题就是计算耗时、传输耗时。

- 计算耗时无法避免，因为那么大的数据就是需要进行计算的。我们只能想办法提升算力或者优化算法来提升计算的速度。
- 传输耗时却可以避免，或者说优化。MapReduce中采用了计算向数据偏移的策略，尽量维持数据不动，在本地计算，这叫数据本地性。但是很多场合我们无法避免移动数据，但是我们也应该尽量选择靠近的节点。

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
- Combiner会处理Map生成的数据，需要注意的是，此时Map生产的仅仅是中间结果。Combiner是一个可选的组件，用户不设置，他就不存在。
- 之后，数据会到达Partitioner，Partitioner组件会将中间数据按照哈希函数的对应规则，将中间结果分配到对应的Reducer所在节点上。
- Reducer会处理中间数据，得到最终的结果。

这就是，一个完整的MapReduce作业的生老病死的概括，其真实的流程自然远不止此，我们会在后面娓娓道来。

先让我们仔仔细细地了解一下上述过程的每一个组件。

### 一、扯一扯Map

有了上述的内容，我们可以进行下一步了。

按照我们说的，我们应该将这个小短文分成几个部分。也就是图中的数据划分。

#### （1）首先进行数据划分

当我们开启一个MapReduce程序，一般传入的输入都是一个体积巨大的数据。MapReduce接收到数据后，需要对数据进行划分。通俗来讲，就是我们前文说的，我们该如果将一个小短文划分成多行，分配个多个人进行统计。

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

**因为Combiner可以存在，也可以不存在，所有，我们设计Combiner时，要保证Combiner的key-value和Map的key-value一致** 。这也意味着，若你设计的Combiner改变了原先Map的键值对设计，那么你的Combiner设计就是不合法的。

### 三、瞅一瞅Partitioner

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

#### Collector

Map的输出结果是由collector处理的，每个Map任务不断地将键值对输出到在内存中构造的一个环形数据结构Collector中。Collector的大小可以通过io.sort.mb设置，默认大小为100M。该空间不够用就会触发Spill方法。但是一般不会将整个空间占满才触发Spill方法，而是会设置一个一个Spill门限，默认为0.8。当当前占用空间达到Collector空间的80%，就会触发Spill方法。

#### Sort

Spill被触发后，并不会直接将键值对溢出，而是先调用SortAndSpill方法，按照partition值和key两个关键字升序排序。这样的排序的结果是，键值对 **key-value** 按照partition值聚簇在一起，同一个partition值，按照key值有序。

#### Spill

Spill线程为这次Spill过程创建一个磁盘文件：从所有的本地目录中轮训查找能存储这么大空间的目录，找到之后在其中创建一个类似于“spill12.out”的文件。Spill线程根据排过序键值对 **key-value** 挨个partition的把数据吐到这个文件中，一个partition对应的数据吐完之后顺序地吐下个partition，直到把所有的partition遍历完。一个partition在文件中对应的数据也叫段(segment)。在这个过程中如果用户配置了combiner类，那么在写之前会先调用combineAndSpill方法，对结果进行进一步合并后再写出。Combiner会优化MapReduce的中间结果，所以它在整个模型中会多次使用。

#### Merge

Map任务如果输出数据量很大，可能会进行好几次Spill，会产生很多Spill??.out类型的文件，分布在不同的磁盘上。这个时候，我们就需要将所有的文件合并的Merge过程。

Merge会扫描本地文件，找到所有的的Spill文件，这些Spill文件都是局部有序的（同一个Spill文件按照partition值有序，同一个partition值内按照key值有序）。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/merge流程.jpg" style="zoom:80%;" />

然后为merge过程会一个partition一个partition的进行合并输出。对于某个partition来说，从索引列表中查询这个partition对应的所有的键值对 **key-value** ，也就是一个Segment的数据。

然后对这个partition对应的所有的segment进行合并，目标是合并成一个segment。当这个partition对应很多个segment时，会分批地进行合并：先从segment列表中把第一批取出来，以key为关键字放置成最小堆，然后从最小堆中每次取出最小的输出到一个临时文件中，这样就把这一批段合并成一个临时的段，把它加回到segment列表中；再从segment列表中把第二批取出来合并输出到一个临时segment，把其加入到列表中；这样往复执行，直到剩下的段是一批，输出到最终的文件中。

这样Map端的任务就算是完成了。

### 三、Reduce端的Shuffle

在Reduce端，shuffle主要分为复制Map输出、排序合并两个阶段。

#### Copy

Reduce任务通过HTTP向各个Map任务拖取它所需要的数据。Map任务成功完成后，会通知父TaskTracker状态已经更新，TaskTracker进而通知JobTracker（这些通知在心跳机制中进行）。所以，对于指定作业来说，JobTracker能记录Map输出和TaskTracker的映射关系。Reduce会定期向JobTracker获取Map的输出位置，一旦拿到输出位置，Reduce任务就会从此输出对应的TaskTracker上复制输出到本地，而不会等到所有的Map任务结束。

#### Merge Sort

Copy过来的数据会先放入内存缓冲区中，如果内存缓冲区中能放得下这次数据的话就直接把数据写到内存中，即**内存到内存merge**。Reduce要向每个Map去拖取数据，在内存中每个Map对应一块数据，当内存缓存区中存储的Map数据占用空间达到一定程度的时候，开始启动内存中merge，把内存中的数据merge输出到磁盘上一个文件中，即**内存到磁盘merge**。在将buffer中多个map输出合并写入磁盘之前，如果设置了Combiner，则会化简压缩合并的map输出。Reduce的内存缓冲区可通过mapred.job.shuffle.input.buffer.percent配置，默认是JVM的heap size的70%。内存到磁盘merge的启动门限可以通过mapred.job.shuffle.merge.percent配置，默认是66%。

当属于该reducer的map输出全部拷贝完成，则会在reducer上生成多个文件（如果拖取的所有map数据总量都没有内存缓冲区，则数据就只存在于内存中），这时开始执行合并操作，即**磁盘到磁盘merge**，Map的输出数据已经是有序的，Merge进行一次合并排序，所谓Reduce端的sort过程就是这个合并的过程。一般Reduce是一边copy一边sort，即copy和sort两个阶段是重叠而不是完全分开的。最终Reduce shuffle过程会输出一个整体有序的数据块。

##### 以上就是Shuffle的整个流程。我只是整理了一个简明版的Shuffle流程，如果你想细致了解，可以看下上面给出的博文。就应届生面试来说，这些差不多可以应付了

## 第4.4节 MapReduce中作业（Job）、任务（task）的一生概览

**本节内容完全来自**

**深入理解大数据：大数据处理与编程实践  机械工业出版社**

没啥好说的，直接上内容

### 一、作业

1. 首先， 用户程序客户端通过作业客户端接口程序JobClient提交一个用户程序。
2. 然后JobClient向JobTracker提交作业执行请求并获得一个Job ID。
3.  JobClient同时也会将用户程序作业和待处理的数据文件信息准备好并存储在HDFS中。 
4. JobClient正式向JobTracker提交和执行该作业。
5.  JobTracker接受并调度该作业，并进行作业的初始化准备工作， 根据待处理数据的实际的分片情况， 调度和分配一定的Map节点来完成作业。
6. JobTracker查询作业中的数据分片信息， 构建并准备相应的任务。
7.  JobTracker启动TaskTracker节点开始执行具体的任务。
8.  TaskTracker根据所分配的具体任务， 获取相应的作业数据。
9.  TaskTracker节点创建所需要的Java虚拟机， 并启动相应的Map任务（或Reduce任务）的执行。 
10. TaskTracker执行完所分配的任务之后， 若是Map任务， 则把中间结果数据输出到HDFS中；若是 Reduce任务，则输出最终结果
11. TaskTracker向JobTracker报告所分配的任务完成。 若是Map任务完成并且后续还有 Reduce任务 则JobTracker会分配和启动 Reduce节点继续处理中间结果并输出最终结果。

### 二、包含任务的细粒度作业流程

#### 作业执行流程

作业提交后成，总体上可以把作业的运行和生命周期分为三个阶段： 准备阶段(PREP)、 运行阶段(RUNNING)和结束阶段(FINISHED)。

在准备阶段， 作业从初始状态NEW开始， 进入PREP.INITIALIZING状态进行初始化 初始化所做的主要工作是读取输入数据块描述信息， 并创建所有的Map任务和Reduce任务。初始化成功 后，进入PREP.INITIALIZED状态。此后，一个特殊的作业初始化任务(job setup task) 被启动，以创建作业运行环境，此任务完成后，作业准备阶段结束 作业真正进入了运行阶段 。

在运行阶段作业首先处在 RUNNING.RUN_WAIT状态下等待任务被调度。 当第一个任务开始执行时，作业进入RUNNING.RUNNING_TASKS， 以进行 真正的计算。 当所有的Map任务和Reduce任务执行完成后， 作业进入RUNNING.SUC_WAIT状态。此时， 另一个特殊的作业清理任务(job cleanup task)被启动， 清理作业的运行环境， 作业进入结束阶段 。在结束阶段， 作业清理任务完成后， 作业最终到达成功状态SUCCEEDED, 至此， 整个 作业的 生命周期结束 。在整个过程中，各个状态下作业有可能被客户主动杀死， 最终进入KILLED状态；也有可，能在执行中因各种因素而失败， 最终进入FAILED状态。

#### 任务执行流程

任务(Task)是HadoopMapReduce框架进行并行化计算的基本单位。 需要说明的一点是：任务是一个逻辑上的概念， 在MapReduce并行计算框架的实现中布千JobTracker和TaskTracker两端， 分别对应TasklnProgress和TaskTracker.TasklnProgress两个对象。当一个作业提交到 Hadoop系统时，JobTracker对作业进行初始化， 作业内的任务(TasklnProgress)被全部创建好，等待 TaskTracker来请求任务， 我们对任务 的分析就从这里开始 。

1. JobTracker为作业创建一个新 的TasklnProgress任务 ；此时Task处在 UNASSIGNED状态。 
2. TaskTracker 经过一个心跳周期 后， 向JobTracker发送一次心跳消息(heartbeat), 请求分配任务， JobTracker收到请求 后分配个 TasklnProgress任务 给TaskTracker 。 这是第一次心跳通信， 心跳间隔一般为3秒。
3.  TaskTracker收到任务 后， 创建 一个对应的 TaskTracker. TasklnProgress对象， 并启动独立的Child 进程去执行这个任务 。此时 TaskTracker已将任务状态更新为 RUNNING 。
4. 又经过一个心跳周期， TaskTracker向JobTracker报告任务状态的改变， JobTracker也将任务状态更新为 RUNNIN1G 。 这是第二次心跳通信 。
5. 经过一定时间，任务在Child进程内执行完成，Child进程向TaskTracker进程发出通知， 任务状态变为 COMMIT_PENDING (任务在执行期间 TaskTracker 还会周期性地向JobTracker 发送心跳信息 ）。
6.  TaskTracker 再次向 JobTracker 发送心跳信息报告任务状态的改变， JobTracker 收到消息后也将任务状态更新为 COMMIT—PENDING, 并返回确认消息， 允许提交。
7. TaskTracker 收到确认可以提交的消息后将结果提交， 并把任务状态更新为SUCCEEDED。
8.  一个心跳周期后 TaskTracker 再次发送心跳消息， JobTracker 收到消息后也更新任务的状态为 SUCCEEDED, 一个任务至此结束。

##### 抛个问题，有了上面的知识之后，那么MapReduce是如何确定Map和Reduce的数量呢？

首先，还记得前面介绍Map的时候，谈到了分片（Split），我们说一个分片就对应一个Map任务。所以一般来讲，Map task的数量应该是和Split息息相关的。

**以下内容来自**

[1] **如何确定 Hadoop map和reduce的个数--map和reduce数量之间的关系是什么？** https://blog.csdn.net/u013063153/article/details/73823963

###### Map数量

map的数量通常是由hadoop集群的DFS块大小确定的，也就是输入文件的总块数，正常的map数量的并行规模大致是每一个Node是10~100个，对于CPU消耗较小的作业可以设置Map数量为300个左右，但是由于hadoop的每一个任务在初始化时需要一定的时间，因此比较合理的情况是每个map执行的时间至少超过1分钟。具体的数据分片是这样的，InputFormat在默认情况下会根据hadoop集群的DFS块大小进行分片，每一个分片会由一个map任务来进行处理，当然用户还是可以通过参数mapred.min.split.size参数在作业提交客户端进行自定义设置。还有一个重要参数就是mapred.map.tasks，这个参数设置的map数量仅仅是一个提示，只有当InputFormat 决定了map任务的个数比mapred.map.tasks值小时才起作用。同样，Map任务的个数也能通过使用JobConf 的conf.setNumMapTasks(int num)方法来手动地设置。这个方法能够用来增加map任务的个数，但是不能设定任务的个数小于Hadoop系统通过分割输入数据得到的值。当然为了提高集群的并发效率，可以设置一个默认的map数量，当用户的map数量较小或者比本身自动分割的值还小时可以使用一个相对交大的默认值，从而提高整体hadoop集群的效率。

###### reduce数量

reduce在运行时往往需要从相关map端复制数据到reduce节点来处理，因此相比于map任务。reduce节点资源是相对比较缺少的，同时相对运行较慢，正确的reduce任务的个数应该是0.95或者1.75 *（节点数 ×mapred.tasktracker.tasks.maximum参数值）。如果任务数是节点个数的0.95倍，那么所有的reduce任务能够在 map任务的输出传输结束后同时开始运行。如果任务数是节点个数的1.75倍，那么高速的节点会在完成他们第一批reduce任务计算之后开始计算第二批 reduce任务，这样的情况更有利于负载均衡。同时需要注意增加reduce的数量虽然会增加系统的资源开销，但是可以改善负载匀衡，降低任务失败带来的负面影响。同样，Reduce任务也能够与 map任务一样，通过设定JobConf 的conf.setNumReduceTasks(int num)方法来增加任务个数。有些作业不需要进行归约进行处理，那么就可以设置reduce的数量为0来进行处理，这种情况下用户的作业运行速度相对较高，map的输出会直接写入到 SetOutputPath(path)设置的输出目录，而不是作为中间结果写到本地。同时Hadoop框架在写入文件系统前并不对之进行排序。



