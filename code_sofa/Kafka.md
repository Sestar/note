# kafka

<br />

## 工作流程

<center><img width="60%" height="480px" src="/note/_v_images/code_sofa/kafka/kafka-process-1.png"/></center>

<br />

```test
Producers往Brokers里面的指定Topic中写消息, Consumers从Brokers里面拉去指定Topic的消息, 然后进行业务处理。
图中有两个topic(topic-0和topic-1), topic-0有两个partition, topic-1有一个partition, 三副本备份.
```

<br />

### partition

```test
当存在多副本的情况下，会尽量把多个副本，分配到不同的broker上。kafka会为partition选出一个leader，之后所有该partition的
请求，实际操作的都是leader，然后再同步到其他的follower。当一个broker歇菜后，所有leader在该broker上的partition都会重新
选举，选出一个leader。（这里不像分布式文件存储系统那样会自动进行复制保持副本数）

然后这里就涉及两个细节：怎么分配partition，怎么选leader。

关于partition的分配，还有leader的选举，总得有个执行者。在kafka中，这个执行者就叫controller。kafka使用zk在broker中选出
一个controller，用于partition分配和leader选举。
```

<br />

#### partition分配

```test
1. 将所有Broker（假设共n个Broker）和待分配的Partition排序
2. 将第i个Partition分配到第（i mod n）个Broker上 （这个就是leader）
3. 将第i个Partition的第j个Replica分配到第（(i + j) mode n）个Broker上
```

<br />

#### leader容灾

```test
controller会在Zookeeper的/brokers/ids节点上注册Watch，一旦有broker宕机，它就能知道。当broker宕机后，controller就会
给受到影响的partition选出新leader。controller从zk的/brokers/topics/[topic]/partitions/[partition]/state中，读取
对应partition的ISR（in-sync replica已同步的副本）列表，选一个出来做leader。

选出leader后，更新zk，然后发送LeaderAndISRRequest给受影响的broker，让它们改变知道这事。为什么这里不是使用zk通知，而是
直接给broker发送rpc请求，我的理解可能是这样做zk有性能问题吧。
```

<br />

## 安装步骤

<br />

### Zookeeper 安装(前置软件)

```text
1. 下载地址: http://mirrors.hust.edu.cn/apache/zookeeper/ (建议下载稳定版, stable版本)
   假设下载到本地位置(E:\ProgramFiles\zookeeper)

2. 新建一个 dataDir(存储zookeeper运行数据) 文件夹, 假设路径为 E:\ProgramFiles\zookeeper\data
   复制 E:\ProgramFiles\zookeeper\conf\zoo_sample.cfg 改名为 zoo.cfg, 并编辑里面内容
   dataDir=E:\\ProgramFiles\\zookeeper\\data; (如果想修改Zookeeper端口, 修改 clientPort 配置即可)

3. 添加 Zookeeper 环境变量, Win+R -> sysdm.cpl -> 高级 -> 环境变量;
   新增变量： ZOOKEEPER_HOME -- E:\ProgramFiles\zookeeper
   添加 PATH 路径：%ZOOKEEPER_HOME%\bin

4. 运行 Zookeeper, cmd -> zkserver
```

<center><img width="100%" height="350px" src="/note/_v_images/code_sofa/kafka/zookeeper-start-1.png"/></center>

<br />

### Kafka 安装

```test
1. 下载地址：http://kafka.apache.org/downloads.html/  假设下载到本地位置(E:\ProgramFiles\kafka)

2. 新建一个 log 文件夹, 假设路径为 E:\ProgramFiles\kafka\logs
   编辑 E:\ProgramFiles\kafka\config\server.properties 里面 log.dirs=E:\\Program Files\\kafka\\logs
   默认Zookeeper的端口是2181, 如果有修改也需要配置 zookeeper.connect

3. 修改 E:\ProgramFiles\kafka\bin\windows\kafka-run-class.bat 内容, CLASSPATH 加上 ""
   原： set COMMAND=%JAVA% %KAFKA_HEAP_OPTS% %KAFKA_JVM_PERFORMANCE_OPTS% %KAFKA_JMX_OPTS% %KAFKA_LOG4J_OPTS% -cp %CLASSPATH% %KAFKA_OPTS% %*
   改： set COMMAND=%JAVA% %KAFKA_HEAP_OPTS% %KAFKA_JVM_PERFORMANCE_OPTS% %KAFKA_JMX_OPTS% %KAFKA_LOG4J_OPTS% -cp "%CLASSPATH%" %KAFKA_OPTS% %*

4. 启动 Kafka 前, 必须保证 Zookeeper 已经运行成功, 在 Kafka 安装路径下 (E:\ProgramFiles\kafka) 运行 cmd,
   执行命令 .\bin\windows\kafka-server-start.bat .\config\server.properties
```

<center><img width="100%" height="350px" src="/note/_v_images/code_sofa/kafka/kafka-start-1.png"/></center>

<br />

## 彻底删除 topic

<br />

```text
1、删除kafka存储目录（server.properties文件log.dirs配置，默认为"/tmp/kafka-logs"）相关topic目录。

2. 启动 Zookeeper Server: 执行 /zookeeper/bin/zkServer.cmd

2、删除 topic: 执行
    ../kafka/bin/windows/kafka-topics.bat --delete --zookeeper localhost:2181 --topic [topic_name]
      如果执行上面的命令行, server.properties没有配置delete.topic.enable=true,
      那么此时的删除并不是真正的删除，而是把topic标记为：marked for deletion
      可以通过命令：./bin/windows/kafka-topics.bat --zookeeper 192.168.100.1:2181 --list 来查看所有topic

3.此时你若想真正删除它，可以如下操作：
（1）启动zookeeper客户端：../zookeeper/bin/zkCli.cmd
（2）继续执行命令行找到topic所在的目录：ls /brokers/topics
（3）选择要删除的topic：rmr /brokers/topics/【topic name】即可，此时topic被彻底删除。
     但是继续执行 ls /admin/delete_topics 还是可以查到删除的topic
```

<br />

## 随笔

[详细解析](https://blog.csdn.net/tangdong3415/article/details/53432166/)

[详细解析](https://www.jianshu.com/p/d3e963ff8b70)

<br />

### 1

```test
zookeeper只能启动单数，比如1台 、3台、7台等等，不能偶数台，偶数台的话假设有两台，那么只有一台机器再运行，因为如果是偶数
的话，选举出来的管理者有可能两个borker得到的票数相同，奇数的话就不会出现这个情况
```

<br />

### 2

```test
在 kafka中,我们可以认为一个group是一个“订阅者”，一个Topic中的每个partions，只会被一个“订阅者”中的一个consumer消费，不
过一个 consumer可以消费多个partitions中的消息（消费者数据小于Partions的数量时）。注意：kafka的设计原理决定，对于一
个topic，同一个group中不能有多于partitions个数的consumer同时消费，否则将意味着某些consumer将无法得到消息。
```