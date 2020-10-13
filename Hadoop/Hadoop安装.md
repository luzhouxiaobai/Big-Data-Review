# 第2节 Hadoop的安装

为了方便后面使用Hadoop的shell命令，我先介绍Hadoop的安装。

Hadoop有多种安装模式，这里介绍伪分布式的安装。

我测试过Ubutun、Centos和WSL，都可以正常安装Hadoop的所有版本。所有一般不会出现版本对应的问题。

Hadoop是基于Java语言进行编写的，在Hadoop程序执行过程中会调用起系统环境的java虚拟机（JVM），所以我们的系统中需要安装JDK。直接搜索JDK进入官网下载即可。考虑到目前的Hadoop基本上都是基于JDK1.8的，建议下载JDK1.8，高版本的Java虽然也可以支持Hadoop的正常执行，但是会报Warning，强迫症看着应该会很难受。

## 一、JDK安装

如果你的系统是Centos可以需要卸载Centos系统自带的OpenJDK。

```shell
java -version
```

使用这个命令会看到当前系统的Java版本，如果系统存在Java，那么可以直接看到Java版本信息。如果没有安装Java，那么应该什么也看不到。如果是OpenJDK，就需要先卸载。

### OpenJDK卸载过程

```shell
rpm -qa | grep java
```

使用这个命令就可以看到所以的Java文件，**.noarch**结尾的文件可以不用删除，其他文件使用下述命令进行删除。

```shell
rpm -e --nodeps [Java文件名]
```

将**[java文件名]**替换为对应的Java文件名就可以删除了。如果提示权限不够，则需要使用管理员权限。

以上过程之后，OpenJDK就删除完成了。

### OracleJDK安装

找到下载好的JDK安装包，我们知道，Linux系统万物皆是文件，所以所谓的安装过程其实就是文件的解压。

```shell
tar -zxvf [文件名]
```

同理，将 **[文件名]** 改成对应的JDK安装包的名称。之后我们就可以看到解压好的JDK文件，我们可以将其移动到我们希望安装的位置。一般都是放在 /usr 目录下。为了方便，我们先将JDK文件重命名为java，然后移动到 /usr 目录下。

```shell
mv [文件名] java
mv java /usr/
```

之后就可以配置环境变量了。

```shell
vi /etc/profile
```

这个命令是需要root权限的，建议进入root用户再进行处理。使用上述命令打开文件后，在文件最后写入Java的目录信息。

<img src='https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/java配置.jpg' style='zoom:80%'/>

这样Java就安装完成了。

```shell
java -version
```

<img src='https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/java-version.png' style='zoom:80%'/>

## 二、SSH免密登录

玩过GitHub的人应该都配置过免密登录。他是为了方便用户使用，避免每次使用都重新输入密码。

### SSH安装

```shell
ssh localhost
```

输入上述命令后，若显示

```
ssh: connect to host localhost port 22: Connection refused
```

则意味着没有安装SSH，我们需要先安装SSH。过程也很简单（Centos将apt命令改为yum命令）

```shell
apt-get update
sudo apt-get install openssh-server
sudo apt-get install openssh-server
```

接着启动SSH

```shell
sudo service ssh start
```

### SSH免密配置

```shell
ssh-keygen
```

输入上述命令之后，一路回车即可。它会在 /home/[用户名] 目录下生成一个隐藏的 .ssh文件夹，文件夹内保存着密钥信息。

```shell
cd /home/[用户名]/.ssh
touch authorized_keys
chmod 600 authorized_keys
cat id_rsa.pub >> authorized_keys 
```

执行时，将 **[用户名]** 改为自己的用户目录即可。

此时尝试

```shell
ssh localhost
```

发现无需密码，可以直接登录成功。

### SSH卸载

提供了一个卸载方法，以备不时之需。

```shell
sudo apt-get remove openssh-server
sudo apt-get remove openssh-client
```

## 三、Hadoop安装

本着Linux中万物皆文件的哲学，我们明白，所谓的安装就是解压二进制安装包，修改配置文件。

直接进入官网，下载自己想要Hadoop版本，我使用的2.7.7版本。下载完之后进行解压，然后修改称自己喜欢的名字，放到想安装的目录下。

```shell
tar -zxvf [hadoop安装包名]       # 解压
mv [hadoop文件名] hadoop        # 重命名
mv hadoop /home/hadoop/        #将文件移动到/home/hadoop目录下
```

### 文件配置

Hadoop安装的重点其实就是文件的配置。在hadoop文件的 etc/hadoop目录下，我们会看到很多.sh或者.xml结尾的配置文件。我们需要其中几个必选项。使用 vi 命令进入文件进行修改。例如：

```shell
vi core-site.xml
```

添加内容：

**1. core-site.xml**

```xml
<configuration>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:/home/hadoop/hadoop/tmp</value>   <!--你想存放临时文件的地方-->
        <description>Abase for other temporary directories.</description>
    </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value> 
    </property>
</configuration>
```

**2.  mapred-site.xml**

如果你只看到 mapred-site.xml.template文件，自己复制一个并重命名就可以。在下面的配置中，遇到同样的问题，可以采用相同的方法解决。

```shell
copy mapred-site.xml.template mapred-site.xml
vi mapred-site.xml
```

```xml
<configuration>
    <property>
        <name>mapred.job.tracker</name>
        <value>localhost:9001</value>
    </property>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

**3. hdfs-site.xml**

```xml
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value> <!--副本个数，可以按照自己的想法设置-->
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>file:/home/hadoop/hadoop/tmp/dfs/name</value>  <!--namenode数据-->
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>file:/home/hadoop/hadoop/tmp/dfs/data</value>  <!--datanode数据-->
    </property>
</configuration>
```

**4. hadoop-env.sh**

```sh
export JAVA_HOME=[java_path]
```

将 **[java_path]** 改为你自己java路径就可以。

**5.  yarn-site.xml**

```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

至此，一个伪分布式Hadoop就安装完成了。

### Hadoop的初始化

安装完成之后，需要进行集群初始化，当然这里我们没有集群，但是初始化也是必须的。

进入Hadoop文件目录。执行：

```shell
bin/hdfs namenode -format
```

之后会出现一连串信息，我们不用管他，中途没有出现**ERROR**关键字，我们的集群初始化就算成功了。

之后就可以启动Hadoop了。

```shell
sbin/start-dfs.sh
sbin/start-yarn.sh
```

没有出现报错则说明启动成功，输入jps

```shell
jps
```

<img src='https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/jps.jpg' style='zoom:80%'/>

#### 叮！！！

配置完成。

打开浏览器，输入地址  **localhost:50070**

<img src='https://github.com/luzhouxiaobai/Big-Data-Review/blob/master/file/mr50070.jpg' style='zoom:80%'/>

现在你就走出了Hadoop的新手村。

关闭也很简单

```shell
sbin/stop-all.sh
```

