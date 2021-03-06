# 第3节 HDFS深聊一下

HDFS 是一个分布式文件系统，我们在前面也稍微讨论了一下集中式文件系统的一些弊端，我们还是从之前的那张HDFS架构图出发。

<img src="../file/HDFS1.jpg" style="zoom:80%;" />

当我们开始关注线条所代表的数据流向后，其实上图就可以看作是一个HDFS数据读写的流程图了。在探讨具体的读写之前，我们先看看不同节点中存储的数据。

## 第3.1节 再谈不同的节点

- 应用程序所在的Client端先过滤掉，因为这个端只是承载了与用户的交互作用，其实就是一台接口机。用户在这台机器写代码、调用API与HDFS进行交互。
- 我们前面说过，NameNode是集群的主控节点，管理HDFS的目录树和文件元数据信息。这样看来，NameNode中存储的就是一些元数据，包括：1）命名空间，也就是整个文件系统的目录结构，即目录树；2）数据块与文件名的映射表，这也很好理解，因为数据真正存储的地方其实位于DataNode，并且DataNode是以块为基本单位组织文件内容的，所以必须要有数据到具体数据块的映射信息，这些信息保存在了HDFS中；3）每个数据块的副本信息，前面提过，HDFS采用了多副本方式存储数据，因此我们必须要了解一个数据的副本信息。**发现了没有，说白了，不存储具体的数据的NameNode中保存了具体数据在文件系统的位置。也是因此，我们说NameNode可以作为集群的主控节点。因为，既然所有的映射信息都在NameNode，那么所有关于数据的操作自然都必须经过NameNode**。
- DataNode是集群中存有具体数据的节点，那么不管什么操作，不和如何与NameNode交互，那么最后具体的实施必须交由DataNode来完成。**这一点应该不难理解，因为具体数据是在DataNode上的**。因此，DataNode是用来实际存储和管理数据的。

到这，我们会有一个总观，即，NameNode拥有至高的权利，可以下发命令，但是这些命令他不用自己去做（也是因为做不了，因为数据不在他这里），具体的实施都有DataNode这一个小兵来完成。

谈到这里，我们最起码明白了两点：（1）NameNode是HDFS中元数据的唯一拥有者；（2）DataNode是HDFS中具体数据的拥有者。但是，程序在访问文件的时候，Client和HDFS的数据交互并不会通过NameNode，而是让Client直接和DataNode交互数据。

我们不妨先假设，我们将HDFS设计为Client和HDFS的交互数据都必须流经NameNode，那么首先：Client和NameNode交互，获取了数据实际所在的数据块，NameNode到对应的DataNode上获取数据，再将该数据传输给Client。访问量小当然没问题，但是高并发的场合呢？大量的用户同时存取大量数据，NameNode很容易就成为了性能瓶颈，桎梏集群的服务能力。

从这样的分析来看，让Client和DataNode交互来读写数据会有两点好处：

1. 可以允许同一份数据在不同的DataNode上被并发访问，因为数据是多副本存储在不同的DataNode上的，这样会提升数据访问的速度。
2. 可以大大减少NameNode的负担，避免NameNode成为数据访问的瓶颈。

这里可能有人会有疑问，为什么NameNode传输具体数据就会成为性能瓶颈，只为Client提供数据块信息就不会？它和Client传输数据块信息不也是数据的传输吗？

根源在于数据量。我们不能百分之百保证NameNode仅仅传输元数据信息一定不会出现单点瓶颈（虽然确实很少会），但是在大数据传输的场合，用户与HDFS交互的数据可能会到达GB、TB，如果所有的数据都经过NameNode来进行，NameNode这一单一节点显然难以支持成百上千的用户的GB、TB级别的数据。但是，如果仅仅是元数据信息，数据量很小，让NameNode成为瓶颈的可能也很小。

## 第3.2节 HDFS的数据读写过程

这几乎成为了一道应届生大数据岗的必问题。其实过程本身并不复杂，即便面试前多背背也能成，但是会忘记，而且忘得很快。我希望写出一个让你再也忘不掉的读写过程解释，当然，前提是你能看得清楚3.1节讲的东西。

- 我们说过，NameNode保有元数据，是HDFS的管理者，这意味我们对HDFS的操作必须要先经过NameNode的许可，并获得要操作的数据的信息。

- 同样，我们也说过，DataNode保有具体的数据，是HDFS指令的具体践行者。这意味着，读写指令的实际执行者是DataNode。

- 我们还提到，Client与HDFS的具体数据的交互过程并不经过NameNode。

有了这三条，我们应该可以勾勒出一个大概的交互的过程：

- 首先，Client应当先和NameNode交互获得许可还有要读写的DataNode上的数据块信息。
- 其次，拿到许可和数据块信息的Client就会去寻找具体的DataNode。
- Client和DataNode交互，进行数据传输。

有了这些讨论，我们来看具体的读写流程。

### 一、HDFS的读流程

所谓具体的读流程，应该就是在上述的三步交互过程的基础上添加一些属于读过程的具体细节。

我们先看本地的操作系统中，一个简单的读流程应该是怎么样的。首先，我们需要知道文件的全路径名，通过全路径访问到这个文件，读取文件中的数据。在HDFS中这个过程应该是类似的。示意图如下：

<img src="../file/hdfs读.jpg" style="zoom:80%;" />

------

- 客户端首先需要获取到FileSystem的一个实例，这里就是HDFS对应的实例。
- 获取到该实例后，客户端需要调用该实例的Open方法，获取到这个文件对应的输入流，在HDFS中就是DFSInputStream。
  - 构建输入流的过程中，客户端会通过RPC远程调用方法，调用NameNode，获取要读取的文件对应的数据块的保存位置，以及副本的保存位置（其实也就是DataNode的地址）。在输入流中，所有的DataNode会按照网络拓扑结构，根据与客户端的距离进行简单排序。
- 获取到输入流之后，客户端会调用read方法读取数据。输入流会按照排序结果，选择最近的DataNode建立连接，读取数据。
- 若当前数据块读取完毕，HDFS会关闭与这个DataNode的连接，查找下一个数据块。查找过程还会通过NameNode进行，重复上述过程。
- 读取结束，调用close方法，关闭输入流DFSInputStream。

------

### 二、HDFS的写流程

写过程与上述过程类似。示意图如下：

<img src="../file/hdfs写.jpg" style="zoom:80%;" />

------

- 客户端首先需要获取到FileSystem的一个实例，这里就是HDFS对应的实例。
- 客户端首先会调用create方法创建文件。当该方法被调用后，NameNode会进行一些检查，检查要写入文件是否存在（同名文件）；客户端是否拥有创建权限等。检查完成之后，需要在NameNode上创建文件信息，但是此时并没有进行实际的写入，因此，此时HDFS中并没有实际的数据，同理，NameNode上也没有数据块信息。create完成之后，会返回一个输出流DFSOutputStream给客户端。
- 客户端会调用输出流的write方法向HDFS中写入数据。注意，此时数据并没有真正写入，数据首先会被分包，分包完成之后，这些数据会被写入输出流DFSOutputStream的一个内部Data队列中，数据分包接收完成之后，DFSOutputStream会向NameNode申请保存文件和副本数据块的若干个DataNode，这若干个DataNode会形成一个数据传输管道。
- DFSOutputStream会依据网络拓扑结构排序，将数据传输给最近的DataNode，这个DataNode接收到数据包之后，将数据包传输给下一个DataNode。数据在各个DataNode上通过管道流动，而不是全部由输出流分发，这样可以减少传输开销。
- 并且，因为DataNode位于不同的机器上，数据需要通过网络（socket）发送。为了保证数据的准确性，接收到数据的DataNode需要向发送者发送确认包（ACK）。对于某个数据块，只有当DFSOutputStream接收到所有DataNode的正确ACK，才能确认传输结束。DFSOutputStream内专门维护了一个等待ACK队列，这个队列保存已经进入管道传输数据、但是并未完全确认的数据包。
- DFSOutputStream会不断传输数据包，直到所有的数据包传输都完成。
- 客户端调用close方法，关闭传输通道。

另外，在数据传输的过程中，如果发现某个DataNode失效了（未联通，ACK超时），那么HDFS会进行如下操作：

1. 关闭数据传输管道
2. 将等待ACK队列中的数据放到Data队列的头部
3. 更新DataNode中所有数据块的版本，当失效的DataNode重启后，之前的数据块会因为版本不对而被清楚
4. 在传输管道中删除失效的DataNode，重新建立管道并发送数据包

------

HDFS的写过程复杂了很多。首先获取文件系统的实例这个无可厚非，拿到了实例才能进行交互。过程的难点：

（1）需要维护一个Data队列，Data队列做了两件事：1）接受分包，并传输到DataNode；2）传输出错后，需要将ACK队列中的数据放入Data队列中。

（2）每个接收到分包需要传输给发送方ACK。

（3）数据在DataNode之间是通过管道水平传输的。

### 三、HDFS的读写背后

`Q 什么是数据分包？`

我们前面提过，HDFS是面向块（block）存储的，一个block的大小是128M，分包就是将数据切分为大小为128M的数据分片。这种块为单位的存储方式有两个好处：

- 减少NameNode中的元数据数量。所以我们可以联想到，HDFS不适合存储大量小文件，因为小文件是造成NameNode中的元数据爆炸增长。
- 提高寻址速度。HDFS以块为单位进行寻址，在一个拥有海量数据的分布式系统中可以显著提升寻找速度，但是这样的设计会带来的问题就是HDFS不支持随机访问，只支持顺序访问数据。**另外，当HDFS中存储小于一个块大小的文件时不会占据整个块的空间，也就是说，1MB的文件存储时只占用1MB的空间而不是128MB。**

`Q HDFS中的副本放置有什么要求？（副本放置策略）`

- 第一个副本放置在与Client交互的DataNode上；
- 第二个副本会放置在与第一个副本不同的机架上，这样，就算前一个机架损坏，HDFS中依然存有数据；
- 第三个副本保存在第二个副本的机架节点上；
- 其他更多的副本则随机选择节点放置。

`Q HDFS中的DataNode数据分布不均，在你读取的时候发现某些节点的负载高怎么办？`

调整读取策略。

我们知道，读取数据的时候，HDFS中的DFSInputStream会获得一个排好序的DataNode节点序列，选择排序靠前的节点进行数据传输。我们可以将节点的负载情况也作为排序时候的权重，将负载高的节点放在后面。写入时遇到类似的问题也可以采取相同的办法。

## 第3.3节 HDFS中的可靠性

### 一、容错、高可用（High Availability, HA）、灾备、可靠性等

谈HDFS容错之前，我们可以先看看上述几个概念。

容错（fault tolerance）指的是， **发生故障时，系统还能继续运行。**

 容错的目的是，发生故障时，系统的运行水平可能有所下降，但是依然可用，不会完全失败。

可用性：描述的是系统可提供服务的时间长短。用公式来说就是A=MTBF/(MTBF+MTTR)，即正常工作时间/(正常工作时间+故障时间)。

高可用（high availability）指的是， **系统能够比正常时间更久地保持一定的运行水平。**

-  注意，高可用不是指系统不中断（那是容错能力），而是指一旦中断能够快速恢复，即中断必须是短暂的。如果需要很长时间才能恢复可用性，就不叫高可用了。上面例子中，更换备胎就必须停车，但只要装上去，就能回到行驶状态。

灾备（又称灾难恢复，disaster recovery）指的是， **发生灾难时恢复业务的能力。**

-  灾备的目的就是，保存系统的核心部分。一个好的灾备方案，就是从失败的基础设施中获取企业最宝贵的数据，然后在新的基础设施上恢复它们。注意，灾备不是为了挽救基础设置，而是为了挽救业务。

容灾：是指系统冗余部署，当一处由于意外停止工作，整个系统应用还可以正常工作。

可靠性：描述的是系统指定时间单位内无故障的次数。比如：一年365天，以天为单位来衡量。有天发生了故障，哪怕只有1秒，这天算不可靠。其他没有故障的是可靠的。

### 二、HDFS的可靠性设计

- 多副本设计
  - 可以让客户端从多个DataNode上读取数据，加快传输速度
  - 多个副本可以判断是否出错
  - 保证数据不丢失
- 安全模式
  - HDFS刚刚启动时，NameNode会进入安全模式（safe mode）。此时NameNode不支持任何文件操作，内部的副本创建也不允许。NameNode会依次和DataNode通信，获取DataNode上保存的数据块信息，通过NameNode检查的DataNode才会被认为时安全的，当安全的DataNode的比例超过某个阈值时，NameNode才会退出安全模式。
- SecondaryNameNode（SNN）
  - SNN主要是用来备份NameNode中的元数据，包括FsImage和EditLog。FsImage相当于HDFS的检查点，NameNode启动时会读取FsImage到内存，并将其与EditLog日志中的所有修改信息合并生成新的FsImage；在HDFS的运行过程中，所有的对HDFS的修改操作都会被记录到EditLog中。但是SSN并不会处理任何请求。
- 心跳包（HeartBeats）和副本重建（re-replication）
  - NameNode会周期性向DataNode发送心跳包来检测DataNode是否崩溃或掉线。收到心跳包的DataNode需要回复，因为心跳包总是定时发送，因此NameNode会把要执行的命令通过心跳包发送给DataNode，DataNode接收到心跳包之后，一方面回复NameNode，另一方面就开始了与用户的数据传输。
  - 当NameNode没有及时得到DataNode的回复就会判定该DataNode失效，会重新创建存储在该DataNode上的副本。
- 数据一致性
  - HDFS采用校验和（CheckSum）的方式保证数据一致性。数据会和校验和一起传输，应用接收到数据之后可以使用校验和进行验证。如果判定当前数据失效就需要从其他节点上获取数据。
- 租约
  - 为了放置多个进程向同一个文件写入数据，采用了租约（Lease）机制。
  - 写入文件之前，客户端需要先获得NameNode发放的租约，对于同一个文件，NameNode只会发放一个租约。
  - 如果NameNode发送租约后崩溃，则可以通过SSN恢复数据。
  - 如果客户端获得租约后崩溃也无妨。因为租约限制，时间到了之后，即便客户端不释放租约，也会被强制释放。

### 三、HDFS的HA设计

NameNode存在单点失效的问题。如果NameNode失效了，那么所有的客户端——包括MapReduce作业均无法读、写文件，因为NameNode是唯一存储元数据与文件到数据块映射的地方。

HDFS高可用是配置了一对活动-备用（active-standby）NameNode。当活动NameNode（active NameNode）失效，备用NameNode（standby NameNode）就会接管它的任务并开始服务于来自客户端的请求，不会有任何明显的中断。

在这样的情况下，要想从一个失效的NameNode中恢复，只需要“激活” standby NameNode，并配置DataNode和客户端以便使用这个新的NameNode。新的NameNode直到满足以下情形后才能响应服务：

- 将命名空间（目录结构）的映像导入内存中。·
- 重做编辑日志。·
- 接收到足够多的来自DataNode的数据块报告并退出安全模式。对于一个大型并拥有大量文件和数据块的集群，NameNode的冷启动需要30分钟甚至更长时间。·
- 系统恢复时间太长，也会影响到日常维护。

在启用HA的场合，Secondary NameNode会被standby NameNode所替代。

## 第3.4节 Secondary NameNode如果进行Checkpoint

其实至此，HDFS的原理基本上就讲了大半，当然，HDFS的精深也绝不是一篇博客性质的文章可以讲解清楚的，我仅仅提了一些比较重要（面试容易问；方便后面MapReduce的学习）的内容，本节仅仅是一些补充。

在说具体的Checkpoint之前，我们先聊一下什么是HDFS的Checkpoint。

### 一、HDFS的Checkpoint

我们前面说过，HDFS的NameNode中包含fsimage和editlog。这里，fsimage称为元数据镜像，里面包含HDFS中数据到数据块的对应信息、副本信息等数据；editLog称为编辑日志，里面包含HDFS中进行的一些操作。

每次NameNode启动时，都会加载保存的fsimage数据信息，之后再读取editlog，重建editlog中的操作。

我们可以这样理解：

- fsimage存有的是NameNode在某个时刻的状态
- editlog保存的是fsimage保存到状态到NameNode当前处于的状态所需进行的操作。

为什么要这样保存呢？

- 假设只有fsimage，那么当NameNode进行操作时，每次必须将操作执行后，生成结果，才能将结果存入fsimage。倘若NameNode在操作执行完成之后没有来的将数据写入fsimage，那么数据就丢失了。
- 假设只有editlog，那么每次操作执行前就将操作写入editlog，时间长了以后editlog会变得特别臃肿。每次恢复NameNode时，重现editlog中的日志就需要花费巨大的时间，这严重影响体验。
- 因而，我们可以二者协同起来，fsimage做checkpoint，保存已经成功的操作，editlog仅保存还没有写入fsimage的操作。

这样说，checkpoint的好处和来历我们也算讲清楚了。

### 二、checkpoint过程

 前面我们也提过（在第一节Hadoop原理简述中第1.2节HDFS原理简述的第三段），fsimage和edit合并的工作并不是由NameNode自己完成的，而是有Secondary NameNode来完成的。那么Secondary NameNode节点的工作流程：

------

- 定期的通过远程方法获取NameNode节点上编辑日志的大小；

- 如果NameNode节点上编辑日很小，就不需要合并NameNode上的fsimage文件和编辑日志； 
- 通过远程接口启动一次检查点过程，这是名字节点需要创建一个新的编辑日志文件edits.new，后续对文件系统的任何修改都记录到这个新编辑日志里
- 第二名字节点将Namenode上的fsimage文件和原编辑日志下载到本地，并在内存中合并，合并的结果输出为fsimage.ckpt；
- 再次发起请求通知NameNode节点数据(fsimage.ckpt)已准备好，然后NameNode节点会下载fsimage.ckpt；
- NameNode下载结束后，Secondary NameNode会通过远程调用（NameNodeProtocol.rollFsImage()）完成这次检查点，NameNode在响应该远程调用时，会用fsimage.ckpt覆盖原来的fsimage文件，形成新的命名空间镜像，同时将新的编辑日志edits.new改名为edits。

------

## 第3.5节 HDFS中几个简单的基本shell命令

HDFS支持shell命令和Java API。这里仅介绍一点常用的shell命令。先看下Hadoop的软件分布

<img src="../file/hadoop目录.png" style="zoom:80%;" />

------

（1）启动分布式文件系统HDFS

```shell
sbin/start-dfs.sh  #该命令位于sbin目录
```

（2）查看HDFS文件

```shell
bin/hdfs dfs -ls / #该命令位于bin目录
```

（3）上传文件到HDFS

```shell
bin/hdfs dfs -put [your_path] [hdfs_path]
```

将上述命令的[your_path]换成本地目录，hdfs_path换成hdfs中位置

（4）从HDFS中下载文件

```shell
bin/hdfs dfs -get [hdfs_path] [your_path]
```

同上。

（5）显示文件内容

```shell
bin/hdfs dfs -cat [hdfs_file]
```

至此，我们可以发现一个规律，这些命令其实都是

```shell
hdfs dfs -[linux_op] /
```

的格式，其中，[linux_op]表示linux中的操作。