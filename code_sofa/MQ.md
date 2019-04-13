# MQ

<br />

## Kafka

<br />

### 工作流程

<img width="60%" height="480px" src="/note/_v_images/code_sofa/MQ/kafka-process-1.png"/>

<br />

```test
Producers往Brokers里面的指定Topic中写消息, Consumers从Brokers里面拉去指定Topic的消息, 然后进行业务处理。
图中有两个topic(topic-0和topic-1), topic-0有两个partition, topic-1有一个partition, 三副本备份.
```

<br />

#### partition

```test
当存在多副本的情况下，会尽量把多个副本，分配到不同的broker上。kafka会为partition选出一个leader，之后所有该partition的
请求，实际操作的都是leader，然后再同步到其他的follower。当一个broker歇菜后，所有leader在该broker上的partition都会重新
选举，选出一个leader。（这里不像分布式文件存储系统那样会自动进行复制保持副本数）

然后这里就涉及两个细节：怎么分配partition，怎么选leader。

关于partition的分配，还有leader的选举，总得有个执行者。在kafka中，这个执行者就叫controller。kafka使用zk在broker中选出
一个controller，用于partition分配和leader选举。
```

<br />

##### partition分配

```test
1. 将所有Broker（假设共n个Broker）和待分配的Partition排序
2. 将第i个Partition分配到第（i mod n）个Broker上 （这个就是leader）
3. 将第i个Partition的第j个Replica分配到第（(i + j) mode n）个Broker上
```

<br />

##### leader容灾

```test
controller会在Zookeeper的/brokers/ids节点上注册Watch，一旦有broker宕机，它就能知道。当broker宕机后，controller就会
给受到影响的partition选出新leader。controller从zk的/brokers/topics/[topic]/partitions/[partition]/state中，读取
对应partition的ISR（in-sync replica已同步的副本）列表，选一个出来做leader。

选出leader后，更新zk，然后发送LeaderAndISRRequest给受影响的broker，让它们改变知道这事。为什么这里不是使用zk通知，而是
直接给broker发送rpc请求，我的理解可能是这样做zk有性能问题吧。
```

<br />

### 安装步骤

<br />

#### Zookeeper 安装(前置软件)

```text
1. 下载地址: http://mirrors.hust.edu.cn/apache/zookeeper/ (建议下载稳定版, stable版本)
   假设下载到本地位置(E:\ProgramFiles\zookeeper, 路径中不要有空格:像Program Files就不行, 会出现 "找不到或无法加载主类 Files\kafka.logs" 异常)

2. 新建一个 dataDir(存储zookeeper运行数据) 文件夹, 假设路径为 E:\ProgramFiles\zookeeper\data
   复制 E:\ProgramFiles\zookeeper\conf\zoo_sample.cfg 改名为 zoo.cfg, 并编辑里面内容
   dataDir=E:\\ProgramFiles\\zookeeper\\data; (如果想修改Zookeeper端口, 修改 clientPort 配置即可)

3. 添加 Zookeeper 环境变量, Win+R -> sysdm.cpl -> 高级 -> 环境变量;
   新增变量： ZOOKEEPER_HOME -- E:\ProgramFiles\zookeeper
   添加 PATH 路径：%ZOOKEEPER_HOME%\bin

4. 运行 Zookeeper, cmd -> zkserver
```

<img width="100%" height="350px" src="/note/_v_images/code_sofa/MQ/zookeeper-start-1.png"/>

<br />

#### Kafka 安装

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

<img width="100%" height="350px" src="/note/_v_images/code_sofa/MQ/kafka-start-1.png"/>

<br />

### 彻底删除 topic

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

### 随笔

[详细解析](https://blog.csdn.net/tangdong3415/article/details/53432166/)

[详细解析](https://www.jianshu.com/p/d3e963ff8b70)

<br />

#### 1

```test
zookeeper只能启动单数，比如1台 、3台、7台等等，不能偶数台，偶数台的话假设有两台，那么只有一台机器再运行，因为如果是偶数
的话，选举出来的管理者有可能两个borker得到的票数相同，奇数的话就不会出现这个情况
```

<br />

#### 2

```test
在 kafka中,我们可以认为一个group是一个“订阅者”，一个Topic中的每个partions，只会被一个“订阅者”中的一个consumer消费，不
过一个 consumer可以消费多个partitions中的消息（消费者数据小于Partions的数量时）。注意：kafka的设计原理决定，对于一
个topic，同一个group中不能有多于partitions个数的consumer同时消费，否则将意味着某些consumer将无法得到消息。
```

<br />

## Rabbit

<br />

### 安装步骤

<br />

#### Erlang 安装(前置软件)

```text
1. 下载地址: http://www.erlang.org/downloads
   假设下载到本地位置(E:\Program Files\MQ\erl10.3)

2. 添加 Erlang 环境变量, Win+R -> sysdm.cpl -> 高级 -> 环境变量;
   新增变量： ERLANG_HOME -- E:\Program Files\MQ\erl10.3
   添加 PATH 路径：%ERLANG_HOME%\bin

3. 测试 Erlang 安装情况, cmd -> erl
```

<img width="80%" src="/note/_v_images/code_sofa/MQ/erlang-start-1.png"/>

<br />

#### Rabbit 安装

```text
1. 下载地址: http://www.rabbitmq.com/download.html
   假设下载到本地位置(E:\Program Files\MQ\RabbitMQ Server)

2. 添加 Zookeeper 环境变量, Win+R -> sysdm.cpl -> 高级 -> 环境变量;
   新增变量： RABBITMQ_SERVER -- E:\Program Files\MQ\RabbitMQ Server\rabbitmq_server-3.7.13
   添加 PATH 路径：%RABBITMQ_SERVER%\sbin

3. 安装 RabbitMQ 服务:
   激活rabbitmq_management: cmd -> rabbitmq-plugins.bat enable rabbitmq_management
   激活后显示:
        Enabling plugins on node rabbit@ScorpionSestar:
        rabbitmq_management
        The following plugins have been configured:
          rabbitmq_management
          rabbitmq_management_agent
          rabbitmq_web_dispatch
        Applying plugin configuration to rabbit@ScorpionSestar...
        The following plugins have been enabled:
          rabbitmq_management
          rabbitmq_management_agent
          rabbitmq_web_dispatch

        started 3 plugins.

4. 启动 RabbitMQ 服务：
    1. cmd -> rabbitmq-service.bat start (win启动RabbitMQ服务)
    2. cmd -> rabbitmqctl.bat start_app (关闭RabbitMQ页面管理插件)
    或者
    1. net start RabbitMQ (直接启动服务)

   启动异常:
   1. 出现：
   ‘
    RabbitMQ service is already present - only updating service parameters
    E:\ProgramFiles\MQ\erl10.3\erts-10.3\bin\erlsrv: Warning, could not set correct interactive mode. RabbitMQ Error: 句柄无效
    E:\ProgramFiles\MQ\erl10.3\erts-10.3\bin\erlsrv: Warning, could not set correct service description (comment) RabbitMQError: 句柄无效
   ’
   直接删除注册表HKEY_LOCAL_MACHINE/SOFTWARE/Ericsson/Erlang/ErlSrv下的项清掉，
   删除文件: C:\Users\Sestar\.erlang.cookie 和 C:\Users\Sestar\AppData\Roaming\RabbitMQ
   然后重新已管理员身份安装erlang和RabbitMQ

   2. 出现:
   ‘ Error: unable to connect to node rabbit@xxx: nodedown ’
   将 C:\Windows\System32\config\systemprofile\.erlang.cookie 和 C:\Users\Sestar\.erlang.cookie 和 C:\Windows\.erlang.cookie
   三个文件内容统一后重试

5. 打开 RabbitMQ 管理页面
   url: http://localhost:15672/     account: guest/guest

6. 关闭 RabbitMQ 服务:
    1. cmd -> rabbitmqctl.bat stop_app 关闭 management 插件
    2. cmd -> rabbitmq-service.bat stop 关闭 RabbitMQ 服务
```

<br />

### 终止 epmd.exe(erlang/bin保护进程)

```text
管理员权限进入 cmd -> 执行命令行 taskkill /f /t /im "epmd.exe"

/f 强制关闭程序
/t 强制关闭相关子程序
/im 应用程序名称
```

<br />

## ActiveMQ

<br />

### 安装步骤

```text
1. 下载地址: http://activemq.apache.org/download-archives.html
   假设下载到本地位置(E:\ProgramFiles\MQ\ActiveMQ)

2. 启动 ActiveMQ 服务:
   启动 activemq.bat (直接点击运行 E:\ProgramFiles\MQ\ActiveMQ\bin\win64\activemq.bat)

3. 打开 ActiveMQ 管理页面
   url: http://localhost:8161/admin/  account: admin/admin

4. 关闭 ActiveMQ 服务:
   终止 activemq.bat 的命令执行程序
```

```text
ActiveMQ正常启动日志:

wrapper  | Launching a JVM...
jvm 1    | Wrapper (Version 3.2.3) http://wrapper.tanukisoftware.org
jvm 1    |   Copyright 1999-2006 Tanuki Software, Inc.  All Rights Reserved.
jvm 1    |
jvm 1    | Java Runtime: Oracle Corporation 1.8.0_161 D:\Program Files\Java\jdk1.8.0_161\jre
jvm 1    |   Heap sizes: current=123904k  free=112758k  max=932352k
jvm 1    |     JVM args: -Dactivemq.home=../.. -Dactivemq.base=../.. -Djavax.net.ssl.keyStorePassword=password
-Djavax.net.ssl.trustStorePassword=password -Djavax.net.ssl.keyStore=../../conf/broker.ks
-Djavax.net.ssl.trustStore=../../conf/broker.ts -Dcom.sun.management.jmxremote -Dorg.apache.activemq.UseDedicatedTaskRunner=true 
-Djava.util.logging.config.file=logging.properties -Dactivemq.conf=../../conf -Dactivemq.data=../../data 
-Djava.security.auth.login.config=../../conf/login.config -Xmx1024m -Djava.library.path=../../bin/win64 -Dwrapper.key=d0G_hWgr27GV83Rg 
-Dwrapper.port=32000 -Dwrapper.jvm.port.min=31000 -Dwrapper.jvm.port.max=31999 -Dwrapper.pid=16948 -Dwrapper.version=3.2.3 
-Dwrapper.native_library=wrapper -Dwrapper.cpu.timeout=10 -Dwrapper.jvmid=1
jvm 1    | Extensions classpath:
jvm 1    |   [..\..\lib,..\..\lib\camel,..\..\lib\optional,..\..\lib\web,..\..\lib\extra]
jvm 1    | ACTIVEMQ_HOME: ..\..
jvm 1    | ACTIVEMQ_BASE: ..\..
jvm 1    | ACTIVEMQ_CONF: ..\..\conf
jvm 1    | ACTIVEMQ_DATA: ..\..\data
jvm 1    | Loading message broker from: xbean:activemq.xml
jvm 1    |  INFO | Refreshing org.apache.activemq.xbean.XBeanBrokerFactory$1@43b35966: startup date [Mon Apr 01 14:19:43 CST 2019]; root of context hierarchy
jvm 1    |  INFO | PListStore:[E:\ProgramFiles\MQ\ActiveMQ\bin\win64\..\..\data\localhost\tmp_storage] started
jvm 1    |  INFO | Using Persistence Adapter: KahaDBPersistenceAdapter[E:\ProgramFiles\MQ\ActiveMQ\bin\win64\..\..\data\kahadb]
jvm 1    |  INFO | KahaDB is version 5
jvm 1    |  INFO | Recovering from the journal ...
jvm 1    |  INFO | Recovery replayed 888 operations from the journal in 0.049 seconds.
jvm 1    |  INFO | Apache ActiveMQ 5.9.1 (localhost, ID:ScorpionSestar-12684-1554099585112-0:1) is starting
jvm 1    |  INFO | Listening for connections at: tcp://ScorpionSestar:61616?maximumConnections=1000&wireFormat.maxFrameSize=104857600
jvm 1    |  INFO | Connector openwire started
jvm 1    |  INFO | Listening for connections at: amqp://ScorpionSestar:5672?maximumConnections=1000&wireFormat.maxFrameSize=104857600
jvm 1    |  INFO | Connector amqp started
jvm 1    |  INFO | Listening for connections at: stomp://ScorpionSestar:61613?maximumConnections=1000&wireFormat.maxFrameSize=104857600
jvm 1    |  INFO | Connector stomp started
jvm 1    |  INFO | Listening for connections at: mqtt://ScorpionSestar:1883?maximumConnections=1000&wireFormat.maxFrameSize=104857600
jvm 1    |  INFO | Connector mqtt started
jvm 1    |  INFO | Listening for connections at ws://ScorpionSestar:61614?maximumConnections=1000&wireFormat.maxFrameSize=104857600
jvm 1    |  INFO | Connector ws started
jvm 1    |  INFO | Apache ActiveMQ 5.9.1 (localhost, ID:ScorpionSestar-12684-1554099585112-0:1) started
jvm 1    |  INFO | For help or more information please see: http://activemq.apache.org
jvm 1    |  INFO | ActiveMQ WebConsole available at http://localhost:8161/
jvm 1    |  INFO | Initializing Spring FrameworkServlet 'dispatcher'
jvm 1    |  INFO | jolokia-agent: No access restrictor found at classpath:/jolokia-access.xml, access to all MBeans is allowed
```

