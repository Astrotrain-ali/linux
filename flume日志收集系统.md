# flume日志收集系统
---
#####   Apache Flume 是一个从可以收集例如日志，事件等数据资源，并将这些数量庞大的数据从各项数据资源中集中起来存储的工具/服务，或者数集中机制。flume具有高可用，分布式，配置工具，其设计的原理也是基于将数据流，如日志数据从各种网站服务器上汇集起来存储到HDFS，HBase等集中存储器中。官网：http://flume.apache.org/FlumeUserGuide.html
#####   Flume agent 负责把外部事件流（数据流）传输到指定下一跳，agent包括source（数据源）、channel（传输通道）、sink（接收端）。Flume agent可以多跳级联，组成复杂的数据流。 Flume 支持多种类型的source：Avro数据源、Thrift数据源、Kafka数据源、NetCat数据源、Syslog数据源、文件数据源、自定义数据源等，可灵活地与应用系统集成，需要较少的开发代价。 Flume 能够与常见的大数据工具结合，支持多种sink：HDFS、Hive、HBase、Kafka等，将数据传输到这些系统，进行进一步分析处理。

####介绍
-   Source:（相当于一个来源）
    -   从数据发生器接收数据,并将接收的数据以Flume的event格式传递给一个或者多个通道channal,Flume提供多种数据接收的方式,比如Avro,Thrift,twitter1%等

-   Channel:（相当于一个中转）
    -   channal是一种短暂的存储容器,它将从source处接收到的event格式的数据缓存起来,直到它们被sinks消费掉,它在source和sink间起着一共桥梁的作用,channal是一个完整的事务,这一点保证了数据在收发的时候的一致性. 并且它可以和任意数量的source和sink链接. 支持的类型有: JDBC channel , File System channel , Memort channel等.

-   sink:（相当于最后的写出）
    -   sink将数据存储到集中存储器比如Hbase和HDFS,它从channals消费数据(events)并将其传递给目标地. 目标地可能是另一个sink,也可能HDFS,HBase.


#### 环境
-   操作系统：CentOS 7.2.1511 64位 
-   Flume版本：1.7.0

#### 安装
```
wget http://mirrors.tuna.tsinghua.edu.cn/apache/flume/1.8.0/apache-flume-1.8.0-bin.tar.gz
tar xzvf apache-flume-1.8.0-bin.tar.gz
mv apache-flume-1.8.0-bin /data/apps/flume_bin
修改conf/flume-env.sh  文件中的JDK目录,注意：JAVA_OPTS 配置  如果我们传输文件过大 报内存溢出时 需要修改这个配置项
./flume-ng version #验证安装是否成功  
export FLUME_HOME=/data/apps/flume_bin  #配置环境变量
```
#### Source、Channel、Sink有哪些类型
```
-   Flume Source
    -   Avro Source                         #支持Avro协议（实际上是Avro RPC），内置支持，一半用作代理与代理之间连接
    -   Thrift Source                       #支持Thrift协议，内置支持
    -   Exec Source                         #基于Unix的command在标准输出上生产数据
    -   JMS Source                          #从JMS系统（消息、主题）中读取数据
    -   Spooling Directory Source           #监控指定目录内数据变更
    -   Twitter 1% firehose Source          #通过API持续下载Twitter数据，试验性质
    -   Netcat Source                       #监控某个端口，将流经端口的每一个文本行数据作为Event输入
    -   Sequence Generator Source           #序列生成器数据源，生产序列数据
    -   Syslog Sources                      #读取syslog数据，产生Event，支持UDP和TCP两种协议
    -   HTTP Source                         #基于HTTP POST或GET方式的数据源，支持JSON、BLOB表示形式
    -   Legacy Sources                      #兼容老的Flume OG中Source（0.9.x版本）
-   Flume Channel
    -   Memory Channel                      #Event数据存储在内存中
    -   JDBC Channel                        #Event数据存储在持久化存储中，当前Flume Channel内置支持Derby
    -   File Channel                        #Event数据存储在磁盘文件中
    -   Spillable Memory Channel            #Event数据存储在内存中和磁盘上，当内存队列满了，会持久化到磁盘文件
    -   Pseudo Transaction Channel          #测试用途
    -   Custom Channel                      #自定义Channel实现
-   Flume Sink
    -   HDFS Sink                           #数据写入HDFS
    -   Logger Sink                         #数据写入日志文件
    -   Avro Sink                           #数据被转换成Avro Event，然后发送到配置的RPC端口上
    -   Thrift Sink                         #数据被转换成Thrift Event，然后发送到配置的RPC端口上
    -   IRC Sink                            #数据在IRC上进行回放
    -   File Roll Sink                      #存储数据到本地文件系统
    -   Null Sink                           #丢弃到所有数据
    -   HBase Sink                          #数据写入HBase数据库
    -   Morphline Solr Sink                 #数据发送到Solr搜索服务器（集群）
    -   ElasticSearch Sink                  #数据发送到Elastic Search搜索服务器（集群）
    -   Kite Dataset Sink                   #写数据到Kite Dataset，试验性质的
    -   Custom Sink                         #自定义Sink实现
```
#### 配置示例
【集群node1输出到node2】

-   node1
    -   编辑一个配置文件node.conf

【node.conf】

```
# 指定Agent的组件名称
#a1.sources = r1    
a1.sinks = k1
a1.channels = c1

# 指定Flume source(要监听的路径)
a1.sources = udp
a1.sources.udp.type = com.mutantbox.flume.plugins.source.epolludp.UDPSource #此处为开发人员做的一个类
a1.sources.udp.bind = localhost #本地监听地址，用于本地只直接接收发往本地的日志
a1.sources.udp.port = 5515  #本地监听udp端口
a1.sources.udp.maxsize = 20480  #数据size
a1.sources.udp.maxthreads = 8 #进程数
a1.sources.udp.reuseport = 4

# 指定Flume sink
# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.channel = c1
a1.sinks.k1.hostname = 52.202.182.194 #node2的地址，即把node1接收到的数据发送到node2
a1.sinks.k1.port = 4545 #日志存储时候占用的端口，此处因为输出到node1所以不必理会
a1.sinks.k1.batch-size = 1000

# 指定Flume channe
# Use a channel which buffers events in memory
a1.channels.c1.type = SPILLABLEMEMORY
a1.channels.c1.memoryCapacity = 1000000
a1.channels.c1.overflowCapacity = 100000000
a1.channels.c1.byteCapacity = 5000000000
a1.channels.c1.checkpointDir = /data/flume_data/checkpoint #channe存储用到目录
a1.channels.c1.dataDirs = /data/flume_data/data #channe存储用到目录

# 绑定source和sink到channel上
# Bind the source and sink to the channel
#a1.sources.r1.channels = c1
a1.sources.udp.channels = c1

a1.sinks.k1.channel = c1
```
-   node2
    -   编辑一个配置文件node.conf和store.conf,node.conf做集群连接，store用于收集的日志落地

【node.conf】
```
#a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources = udp
a1.sources.udp.type = com.mutantbox.flume.plugins.source.epolludp.UDPSource
a1.sources.udp.bind = localhost 
a1.sources.udp.port = 5515
a1.sources.udp.maxsize = 20480
a1.sources.udp.maxthreads = 8
a1.sources.udp.reuseport = 4


# Describe the sink
a1.sinks.k1.type = avro
a1.sinks.k1.channel = c1
a1.sinks.k1.hostname = localhost #此处配置直接存储到本地的4545端口，然后stort.conf配置source
a1.sinks.k1.port = 4545
a1.sinks.k1.batch-size = 1000

# Use a channel which buffers events in memory
a1.channels.c1.type = SPILLABLEMEMORY
a1.channels.c1.memoryCapacity = 1000000
a1.channels.c1.overflowCapacity = 10000000
a1.channels.c1.byteCapacity = 2000000000
a1.channels.c1.checkpointDir = /data/flume_data/checkpoint
a1.channels.c1.dataDirs = /data/flume_data/data


# Bind the source and sink to the channel
#a1.sources.r1.channels = c1
a1.sources.udp.channels = c1

a1.sinks.k1.channel = c1
    
```
【store.conf】
```
#a1.sources = r1
a1.sinks = k1
a1.channels = c1

a1.sources = r1
a1.sources.r1.type = avro
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0    #接收数据来源
a1.sources.r1.port = 4545
  

a1.sinks.k1.type = com.mutantbox.flume.plugins.sink.file.FileSink
a1.sinks.k1.file.path = /data/udplog/%{msgRid}/%Y-%m-%d%z  #数据存储路径
a1.sinks.k1.file.filePrefix = log-%{msgType}-%H-
a1.sinks.k1.file.txnEventMax = 1000
a1.sinks.k1.file.maxOpenFiles = 50 
a1.sinks.k1.file.rollInterval = 1800
a1.sinks.k1.timeZone = America/Los_Angeles
  
# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000000
a1.channels.c1.transactionCapacity = 1000
a1.channels.c1.keep-alive = 30 

  
# Bind the source and sink to the channel
#a1.sources.r1.channels = c1
a1.sources.udp.channels = c1

a1.sinks.k1.channel = c1
```
#### 启动
先启动node2，再启动node1
-   node2
```
nohup /data/apps/flume_bin/bin/flume-ng agent -c /data/apps/flume_bin/conf/ -f /data/apps/flume_bin/conf/store.conf -n a1 -Dflume.root.logger=INFO,console -Dflume.monitoring.type=http -Dflume.monitoring.port=34546 > /data/logs/flume/store.log 2>&1 &
nohup /data/apps/flume_bin/bin/flume-ng agent -c /data/apps/flume_bin/conf/ -f /data/apps/flume_bin/conf/node.conf -n a1 -Dflume.root.logger=INFO,console -Dflume.monitoring.type=http -Dflume.monitoring.port=34545 > /data/logs/flume/node.log 2>&1 &
```
-   node1
```
nohup /data/apps/flume_bin/bin/flume-ng agent -c /data/apps/flume_bin/conf/ -f /data/apps/flume_bin/conf/node.conf -n a1 -Dflume.root.logger=INFO,console -Dflume.monitoring.type=http -Dflume.monitoring.port=34545 > /data/logs/flume/node.log 2>&1 &
```