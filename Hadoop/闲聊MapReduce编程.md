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

