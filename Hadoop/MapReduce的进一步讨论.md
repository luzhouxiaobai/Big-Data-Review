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

#### （2）这样，我们可以理一理划分逻辑

- 遍历输入目录中的每个文件，拿到该文件
- 计算文件长度，A:如果文件长度为0，如果`mapred.split.zero.file.skip=true`，则不划分split ; 如果`mapred.split.zero.file.skip`为false，生成一个length=0的split .B:如果长度不为0，跳到步骤3
- 判断该文件是否支持split :如果支持，跳到步骤4;如果不支持，该文件不切分，生成1个split，split的length等于文件长度。
- 根据当前文件，计算`splitSize`，本文中为100M
-  判断`剩余待切分文件大小/splitsize`是否大于`SPLIT_SLOP`(该值为1.1，代码中写死了) 如果true，切分成一个split，待切分文件大小更新为当前值-splitsize ，再次切分。生成的split的length等于splitsize； 如果false 将剩余的切到一个split里，生成的split length等于剩余待切分的文件大小。之所以需要判断`剩余待切分文件大小/splitsize`,主要是为了避免过多的小的split。比如文件中有100个109M大小的文件，如果`splitSize`=100M，如果不判断`剩余待切分文件大小/splitsize`，将会生成200个split，其中100个split的size为100M，而其中100个只有9M，存在100个过小的split。MapReduce首选的是处理大文件，过多的小split会影响性能。

