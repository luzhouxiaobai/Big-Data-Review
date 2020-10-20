# 第5节 闲聊MapReduce编程

## 第5.1节 IDEA配置MapReduce编程环境

现在Hadoop可以通过配置在Windows下编程。

### 一、解压安装包、配置环境变量（Linux可跳过此步）

- 从官网下载Hadoop二进制包，解压到你想安装的目录即可。
- 由于Hadoop原生不支持Windows，所以需要一个win工具包，你可以从[此处](https://github.com/steveloughran/winutils)下载。将对应版本或者相近版本的 **hadoop/bin** 目录下的文件全部复制到刚刚解压的Hadoop文件的bin目录下即可。
- 配置Window下的Hadoop路径。右键点击 `我的电脑` ，打开 `属性` , 打开 `高级系统设置` , 打开 `环境变量` , 在 `系统变量` 栏，创建 `HADOOP_HOME` 环境变量，值为你的Hadoop路径。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/hadoop_home.png" style="zoom:80%;" />

之后，在 `系统变量` 栏的 `Path` 条目下，添加的你的Hadoop路径的bin路径。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/hadoop_path.png" style="zoom:80%;" />

以上步骤完成之后，就完成了在Windows下开发MapReduce代码的必备步骤。

### 二、IDEA配置

我们一般使用IDEA基于Maven进行开发。

打开IDEA专业版，选中Maven，一直下一步，为项目起一个好听的名字即可。

点开pom文件，为要编写的代码添加Maven依赖，Maven会自动导入需要的Jar包。

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.hadoop</groupId>
        <artifactId>hadoop-client</artifactId>
        <version>2.7.7</version>  <!--可以自由选择合适的版本-->
    </dependency>
</dependencies>
```

有了这个依赖，我们就可以编程Hadoop代码了。我们还是从WordCount开始。

### 三、WordCount

我们的数据还是使用上一节给的数据。

```
This distribution includes cryptographic software.  The country in which you currently reside 
may have restrictions on the import, possession, use, and/or re-export to another country, 
of encryption software. BEFORE using any encryption software, please check your country's laws, 
regulations and policies concerning the import, possession, or use, and re-export of encryption 
software, to see if this is permitted. See <http://www.wassenaar.org/> for more information.
```

#### 如何设计Map 和 Reduce

虽然这是官方给的例子，但是我们还是例行分析一下，希望从这样简单的例子中找到MapReduce编程的一些方法。

**我们可以有一个这样的定视，虽然有时候并不准确，但是一般可以认为，Map用来做拆分，Reduce用来做合并。**

**基于这样的定视，我们可以设计方法，然后为方法设计额外的功能。**

那么，基于定视，产生思路，想法也很直白。

- 我们也可在Map程序中对文本做拆分，产生 (This, 1) (distribution, 1) 这样的键值对。
- 在Reduce程序中做合并。

这样，WordCount就算是完成了。

#### Map

```java
public static class TokenizerMapper extends Mapper<Object,  //输入数据的Key
            Text,   //输入数据的value
            Text,   //输出数据的key
            IntWritable  //输出数据的value
            > {

        private IntWritable one = new IntWritable(1);
        private Text word = new Text();

        @Override
        public void map(Object key, //输入数据的Key
                        Text value, //输入数据的value
                        Context context  // 用于记录Map程序执行的上下文，
        ) throws IOException, InterruptedException {
            StringTokenizer itr = new StringTokenizer(value.toString());
            while (itr.hasMoreTokens()) {
                word.set(itr.nextToken());
                context.write(word, one);
            }
        }
    }
```

我们需要自己继承MapReduce类，重写map函数，自定义处理逻辑。这里的处理很简单，就是接收输入，进行分割，组合成键值对的形式。

还记得上一节将MapReduce时提过，InputFormat会将输入数据处理成坚持对的形式。他会按照用户给出的类型，对数据进行处理。初始文本对于Java来说就是一堆字符串。我们以行号（Object）为Key，该行对应的值为Value进行处理。

Context的理解有些复杂，我们尽量网简单处理解，秉承够用就行的思想，我们先看一下Map的源码。

```java
public class Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {

  /**
   * The <code>Context</code> passed on to the {@link Mapper} implementations.
   */
  public abstract class Context
    implements MapContext<KEYIN,VALUEIN,KEYOUT,VALUEOUT> {
  }
  
}
```

发现，Map中的Context继承自MapContext。我们一路走下去，会发现继承的最上层父类。

```java
public interface MRJobConfig {

  // Put all of the attribute names in here so that Job and JobContext are
  // consistent.
  public static final String INPUT_FORMAT_CLASS_ATTR = "mapreduce.job.inputformat.class";

  public static final String MAP_CLASS_ATTR = "mapreduce.job.map.class";

  public static final String MAP_OUTPUT_COLLECTOR_CLASS_ATTR
                                  = "mapreduce.job.map.output.collector.class";

  public static final String COMBINE_CLASS_ATTR = "mapreduce.job.combine.class";

  public static final String REDUCE_CLASS_ATTR = "mapreduce.job.reduce.class";

  public static final String OUTPUT_FORMAT_CLASS_ATTR = "mapreduce.job.outputformat.class";

  public static final String PARTITIONER_CLASS_ATTR = "mapreduce.job.partitioner.class";
  
  // .....
}
```

MRJobConfig包含了该Job的一系列属性。我们得知，Context是用来感知该作业的上下文的。如果还是不理解，你就认为Context中包含了该Job的属性信息，我们可以通过调用Context的方法得到或者写入一系列的数据。

#### Reduce

```java
public static class IntSumReducer extends Reducer<Text, //输入的key
            IntWritable,  //输入的value
            Text,  //输出的key
            IntWritable  //输出的value
            > {

        private IntWritable result = new IntWritable();

        @Override
        public void reduce(Text key, Iterable<IntWritable> value, Context context)
            throws IOException, InterruptedException {
            int count = 0;
            for (IntWritable val: value) {
                count += val.get();
            }
            result.set(count);
            context.write(key, result);
        }
    }
```

Reduce代码同上类似，需要继承MapReduce中的Reducer类。尖括号中给出的类型，输入的 **Key-Value** 类型和Mapper中的输出对应。

还记得前一节，我们曾指出，到达Reduce的数据是 **Key-list(Value)** 格式。所以你看，reduce函数中的 **value** 是 迭代器类型 **Iterable<IntWritable>** 。

```java
public class Reducer<KEYIN,VALUEIN,KEYOUT,VALUEOUT> {

  public abstract class Context 
    implements ReduceContext<KEYIN,VALUEIN,KEYOUT,VALUEOUT> {
  }
}
```

Reducer的Context继承自ReduceContext，如果你点下去，会发现它也是继承自MRJocConfig。但是在一层层的继承中，添加了不同于MapContext的内容。

#### WordCount

有了上述代码，再写好驱动程序即可。

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;
import java.util.StringTokenizer;

public class WordCount {

    public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
        private IntWritable one = new IntWritable(1);
        private Text word = new Text();

        @Override
        public void map(Object key, Text value, Context context) throws IOException, InterruptedException {
            StringTokenizer itr = new StringTokenizer(value.toString());
            while (itr.hasMoreTokens()) {
                word.set(itr.nextToken());
                context.write(word, one);
            }
        }
    }

    public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
        private IntWritable result = new IntWritable();

        @Override
        public void reduce(Text key, Iterable<IntWritable> value, Context context)
            throws IOException, InterruptedException {
            int count = 0;
            for (IntWritable val: value) {
                count += val.get();
            }
            result.set(count);
            context.write(key, result);
        }
    }

    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");
        job.setJarByClass(WordCount.class);
        job.setMapperClass(TokenizerMapper.class);
        job.setCombinerClass(IntSumReducer.class);
        job.setReducerClass(IntSumReducer.class);
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        FileInputFormat.addInputPath(job, new Path(args[0])); //输入文件所在路径
        FileOutputFormat.setOutputPath(job, new Path(args[1])); //输出文件所在路径
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

上文展示的就是完整的MapReduce代码，也就是一个Job。

在 `main` 函数中，我们看到输入文件路径是第一个参数，输出文件路径是第二个参数。在IDEA的配置方式如下图：

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/idea1.png" style="zoom:80%;" />

如果点开后发现不一致，可以先运行代码（会报错，但是之后就正常了。）

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/idea2.png" style="zoom:80%;" />

输入的参数就是对应的 `args[0]` 和 `args[1]` 两个参数用空格隔开。需要注意的是，输出的路径必须不存在，不然会报错提示该路径已经存在。

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/output.png" style="zoom:80%;" />

上图展示的部分结果。如果你足够仔细，你会发现结果是按照Key值排序的。这里是不是和前面的MapReduce的讲解对应上了。

## 第5.2 节 Jar运行

### 一、打Jar包的方法

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/jar1.png" style="zoom:80%;" />

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/jar2.png" style="zoom:80%;" />

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/jar3.png" style="zoom:80%;" />

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/jar4.png" style="zoom:80%;" />

<img src="https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/jar5.png" style="zoom:80%;" />

