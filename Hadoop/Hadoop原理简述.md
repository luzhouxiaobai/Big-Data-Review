### 第1节 Hadoop原理简述

### 一、原理

分而治之是大数据技术的基本基本思想。MapReduce同样借助了分治思想，将大数据切分为“小”数据，再并行进行处理。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/mapreduce1.jpg" style="zoom:50%;" />

MapReduce能够解决的问题有一个共同特点：任务可以被分解为多个子问题，且这些子问题相对独立，彼此之间不会有牵制，待并行处理完这些子问题后，任务便被解决。在实际应用中，这类问题非常庞大，谷歌在论文中提到了MapReduce的一些典型应用，包括分布式grep、URL访问频率统计、Web连接图反转、倒排索引构建、分布式排序等