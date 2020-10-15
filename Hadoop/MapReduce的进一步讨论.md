# 第4节 MapReduce的进一步讨论

我们前面提了一嘴MapReduce。说它是一个采用了分治思想的分布式计算框架，本节我们就进一步细致讨论一下MapReduce。

## 第4.1节 MapReduce的并行化计算模型

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/mr并行编程.jpg" style="zoom:80%;" />

看图说话，我们基于一个大数据领域的“HelloWorld“，”wordcount"，结合上图中的MapReduce并行计算模型来讲解MapReduce的计算过程。

首先，我们假设我们拥有一篇初始的文章，我们需要统计文章中的所有的词语出现的个数。以一下这个小短文为例：

```
This distribution includes cryptographic software.  The country in which you currently reside may have restrictions on the import, possession, use, and/or re-export to another country, of encryption software. BEFORE using any encryption software, please check your country's laws, regulations and policies concerning the import, possession, or use, and re-export of encryption software, to see if this is permitted. See <http://www.wassenaar.org/> for more information.
```

细心的人已经发现了，这个是Hadoop安装包的README.txt。

要统计这篇文章的词语方法很简单，读取文章，顺次统计每个词语出现的个数即可。**如果只有一个人，就一个人统计，如果有好几个人，就好几个人每个人分几行，先自己统计，然后将每个人的统计结果合并就可以了。**

**哎哎哎！打住！发现了没有，这就是分治啊！把一个活分给好几个人干。**这也是MapReduce的想法，让原本需要一台机器做的事，分给好几个人做，这样不就快了嘛！

### 一、扯一扯Map

