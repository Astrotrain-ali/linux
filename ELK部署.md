# ELK部署
---
-   集群介绍
    -   filebeat --> kafka --> logstash --> elastic --> kibana
    -   filebeat直接读取日志源文件并输出到kafka,kafka作为发布订阅，供logstash消费，logstash对日志进行过滤以后输出到es，es存储。kibana进行前段展示。
-   环境说明
```
操作系统版本：CentOS6.9 64位
ES版本：5.5.2
filebeat版本：5.5.2
kafka版本：2.12-0.10.2.1
kibana版本：5.5.2
logstash版本：5.5.2
node版本：6.11.2   # 8.4版本会导致安装插件失败
zookeeper版本：3.4.9
JDK版本：1.8.0_144
- 10.8.189.101        # ES/logstash/kafka
- 10.8.189.102        # ES/logstash/kafka
- 10.8.189.103        # ES/logstash/kibana/nginx
配置Java环境变量
# vim /etc/profile
    export JAVA_HOME=/usr/java/jdk1.8.0_144
    export PATH=$JAVA_HOME/bin:$PATH 
    export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar 
# source /etc/profile
```
-   资源包下载
```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.2.tar.gz
wget https://artifacts.elastic.co/downloads/logstash/logstash-5.5.2.tar.gz
wget https://artifacts.elastic.co/downloads/kibana/kibana-5.5.2-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-5.5.2-linux-x86_64.tar.gz
wget https://nodejs.org/dist/v6.11.2/node-v6.11.2-linux-x64.tar.xz
wget http://mirrors.hust.edu.cn/apache/zookeeper/zookeeper-3.4.9/zookeeper-3.4.9.tar.gz
wget http://mirror.bit.edu.cn/apache/kafka/0.10.2.1/kafka_2.12-0.10.2.1.tgz
```
### 安装配置es
-   解压压缩包，并移至/usr/local目录下
```
tar zxf elasticsearch-5.5.2.tar.gz
mv elasticsearch-5.5.2 /usr/local/elasticsearch
```
-   修改ES启动脚本中ES_JAVA_OPTS
```
脚本开头添加
ES_JAVA_OPTS="-Xms2g -Xmx2g"   #内存大小根据服务器实际情况分配
```
-   新增elastic组及用户, 因为ES不允许root用户启动
```
groupadd elastic
useradd elastic -g elastic
echo 'elastic' | passwd --stdin elastic
chown -R elastic:elastic /usr/local/elasticsearch
```
-   配置elasticsearch的配置文件
```
vim /usr/local/elasticsearch/config/elasticsearch.yml
  cluster.name: es-dev-cluster    # 集群名称
  node.name: elk-dev-101      # 节点名称 每个节点不同
  node.master: true       # 允许一个节点是否可以成为一个master节点 
  node.data: true         # 允许该节点存储数据(默认开启)
  bootstrap.memory_lock: true     
  bootstrap.system_call_filter: false    # centos7以下版本需要将这个参数设置为false
  network.host: 0.0.0.0
  http.port: 9200
  discovery.zen.ping.unicast.hosts: ["10.8.189.101:9300", "10.8.189.102:9300", "10.8.189.103:9300"]
  discovery.zen.minimum_master_nodes: 2
  discovery.zen.ping_timeout: 60s        # 网上大部分文章这个参数都写成了discovery.zen.ping.timeout
  http.cors.enabled: true
  http.cors.allow-origin: "*"
  su elastic
  /usr/local/elasticsearch/bin/elasticsearch -d #启动elasticsearch   不能使用root用户启动
```
-   安装NodeJS
```
yum -y install xz
tar xJf node-v8.4.0-linux-x64.tar.xz
mv node-v8.4.0-linux-x64 /usr/local/node
ln -s /usr/local/node/bin/node /usr/local/bin/
ln -s /usr/local/node/bin/npm /usr/local/bin/
```
-   安装ELK插件 建议安装在nginx监听的服务器上，两个插件只用安装在同一台机器即可
    -   ElasticSearch-Head 是一个与Elastic集群（Cluster）相交互的Web前台   
```
它展现ES集群的拓扑结构，并且可以通过它来进行索引（Index）和节点（Node）级别的操作
它提供一组针对集群的查询API，并将结果以json和表格形式返回
它提供一些快捷菜单，用以展现集群的各种状态
```
-   在10.8.189.103上安装ElasticSearch-Head node8.4 npm5.3安装失败，GitHub搜索结果发现为npm版本太新导致，使用node6版本
```
cd /usr/local/elasticsearch  #此目录下权限必须为elastic，不然安装会报权限不对
git clone https://github.com/mobz/elasticsearch-head.git
cd elasticsearch-head
npm install -g grunt --registry=https://registry.npm.taobao.org  # 安装grunt
npm install # 安装head
/usr/local/elasticsearch/elasticsearch-head/node_modules/grunt/bin/grunt server &   # 启动head插件
访问地址为http://10.8.189.103:9100
```
-   安装bigdesk插件
    -   Bigdesk为Elastic集群提供动态的图表与统计数据
    -   在10.8.189.103上安装bigdesk
```
cd /usr/local/elasticsearch
git clone https://github.com/hlstudio/bigdesk
cd /usr/local/elasticsearch/bigdesk/_site
python -m SimpleHTTPServer &   # 启动bigdesk插件
访问地址http://10.8.189.103:8000
```
-   ES配置文件详细说明
```
#==================================================
# 配置文件中给出了三种配置高性能集群拓扑结构的模式,如下：
# 1. 如果你想让节点从不选举为主节点,只用来存储数据,可作为负载器
# node.master: false
# node.data: true
#
# 2. 如果想让节点成为主节点,且不存储任何数据,并保有空闲资源,可作为协调器
# node.master: true
# node.data: false
#
# 3. 如果想让节点既不称为主节点,又不成为数据节点,那么可将他作为搜索器,从节点中获取数据,生成搜索结果等
# node.master: false
# node.data: false
#==================================================
# 配置文件存储位置
#path.conf: /path/to/conf
# 数据存储位置(单个目录设置)
#path.data: /path/to/data
# 多个数据存储位置,有利于性能提升
#path.data: /path/to/data1,/path/to/data2
# 临时文件的路径
#path.work: /path/to/work
# 日志文件的路径
#path.logs: /path/to/logs
# 插件安装路径
#path.plugins: /path/to/plugins
#==================================================
# 设置索引的分片数,默认为5
#index.number_of_shards: 5
# 设置索引的副本数,默认为1:
#index.number_of_replicas: 1
# 配置文件中提到的最佳实践是,如果服务器够多,可以将分片提高,尽量将数据平均分布到大集群中去
# 同时,如果增加副本数量可以有效的提高搜索性能
# 需要注意的是,"number_of_shards" 是索引创建后一次生成的,后续不可更改设置
# "number_of_replicas" 是可以通过API去实时修改设置的
#==================================================
# 设置插件作为启动条件,如果一下插件没有安装,则该节点服务不会启动
#plugin.mandatory: mapper-attachments,lang-groovy
#==================================================
# 当JVM开始写入交换空间时（swapping）ElasticSearch性能会低下,你应该保证它不会写入交换空间
# 设置这个属性为true来锁定内存,同时也要允许elasticsearch的进程可以锁住内存,linux下可以通过 `ulimit -l unlimited` 命令
#bootstrap.mlockall: true
# 确保 ES_MIN_MEM 和 ES_MAX_MEM 环境变量设置为相同的值,以及机器有足够的内存分配给Elasticsearch
# 注意:内存也不是越大越好,一般64位机器,最大分配内存别才超过32G
#==================================================
# 设置绑定的ip地址,可以是ipv4或ipv6的,默认为0.0.0.0
#network.bind_host: 192.168.0.1
# 设置其它节点和该节点交互的ip地址,如果不设置它会自动设置,值必须是个真实的ip地址
#network.publish_host: 192.168.0.1
# 同时设置bind_host和publish_host上面两个参数
#network.host: 192.168.0.1
# 设置节点间交互的tcp端口,默认是9300
#transport.tcp.port: 9300
# 设置是否压缩tcp传输时的数据，默认为false,不压缩
#transport.tcp.compress: true
# 设置对外服务的http端口,默认为9200
#http.port: 9200
# 设置请求内容的最大容量,默认100mb
#http.max_content_length: 100mb
# 使用http协议对外提供服务,默认为true,开启
#http.enabled: false
#==================================================
############################## Network And HTTP #######################
# 设置绑定的ip地址,可以是ipv4或ipv6的,默认为0.0.0.0
#network.bind_host: 192.168.0.1
# 设置其它节点和该节点交互的ip地址,如果不设置它会自动设置,值必须是个真实的ip地址
#network.publish_host: 192.168.0.1
# 同时设置bind_host和publish_host上面两个参数
#network.host: 192.168.0.1
# 设置节点间交互的tcp端口,默认是9300
#transport.tcp.port: 9300
# 设置是否压缩tcp传输时的数据，默认为false,不压缩
#transport.tcp.compress: true
# 设置对外服务的http端口,默认为9200
#http.port: 9200
# 设置请求内容的最大容量,默认100mb
#http.max_content_length: 100mb
# 使用http协议对外提供服务,默认为true,开启
#http.enabled: false
################################### Gateway #############################
# gateway的类型,默认为local即为本地文件系统,可以设置为本地文件系统
#gateway.type: local
# 下面的配置控制怎样以及何时启动一整个集群重启的初始化恢复过程
# (当使用shard gateway时,是为了尽可能的重用local data(本地数据))
# 一个集群中的N个节点启动后,才允许进行恢复处理
#gateway.recover_after_nodes: 1
# 设置初始化恢复过程的超时时间,超时时间从上一个配置中配置的N个节点启动后算起
#gateway.recover_after_time: 5m
# 设置这个集群中期望有多少个节点.一旦这N个节点启动(并且recover_after_nodes也符合),
# 立即开始恢复过程(不等待recover_after_time超时)
#gateway.expected_nodes: 2
############################# Recovery Throttling #######################
# 下面这些配置允许在初始化恢复,副本分配,再平衡,或者添加和删除节点时控制节点间的分片分配
# 设置一个节点的并行恢复数
# 1.初始化数据恢复时,并发恢复线程的个数,默认为4
#cluster.routing.allocation.node_initial_primaries_recoveries: 4
#
# 2.添加删除节点或负载均衡时并发恢复线程的个数,默认为2
#cluster.routing.allocation.node_concurrent_recoveries: 2
# 设置恢复时的吞吐量(例如:100mb,默认为0无限制.如果机器还有其他业务在跑的话还是限制一下的好)
#indices.recovery.max_bytes_per_sec: 20mb
# 设置来限制从其它分片恢复数据时最大同时打开并发流的个数,默认为5
#indices.recovery.concurrent_streams: 5
# 注意: 合理的设置以上参数能有效的提高集群节点的数据恢复以及初始化速度
################################## Discovery ##########################
# 设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点.默认为1,对于大的集群来说,可以设置大一点的值(2-4)
#discovery.zen.minimum_master_nodes: 1
# 探查的超时时间,默认3秒,提高一点以应对网络不好的时候,防止脑裂
#discovery.zen.ping.timeout: 3s
# For more information, see
# <http://elasticsearch.org/guide/en/elasticsearch/reference/current/modules-discovery-zen.html>
# 设置是否打开多播发现节点.默认是true.
# 当多播不可用或者集群跨网段的时候集群通信还是用单播吧
#discovery.zen.ping.multicast.enabled: false
# 这是一个集群中的主节点的初始列表,当节点(主节点或者数据节点)启动时使用这个列表进行探测
#discovery.zen.ping.unicast.hosts: ["host1", "host2:port"]
# Slow Log部分与GC log部分略,不过可以通过相关日志优化搜索查询速度
############## Memory(重点需要调优的部分) ################
# Cache部分:
# es有很多种方式来缓存其内部与索引有关的数据.其中包括filter cache
# filter cache部分:
# filter cache是用来缓存filters的结果的.默认的cache type是node type.node type的机制是所有的索引内部的分片共享filter cache.node type采用的方式是LRU方式.即:当缓存达到了某个临界值之后，es会将最近没有使用的数据清除出filter cache.使让新的数据进入es.
# 这个临界值的设置方法如下：indices.cache.filter.size 值类型：eg.:512mb 20%。默认的值是10%。
# out of memory错误避免过于频繁的查询时集群假死
# 1.设置es的缓存类型为Soft Reference,它的主要特点是据有较强的引用功能.只有当内存不够的时候,才进行回收这类内存,因此在内存足够的时候,它们通常不被回收.另外,这些引用对象还能保证在Java抛出OutOfMemory异常之前,被设置为null.它可以用于实现一些常用图片的缓存,实现Cache的功能,保证最大限度的使用内存而不引起OutOfMemory.在es的配置文件加上index.cache.field.type: soft即可.
# 2.设置es最大缓存数据条数和缓存失效时间,通过设置index.cache.field.max_size: 50000来把缓存field的最大值设置为50000,设置index.cache.field.expire: 10m把过期时间设置成10分钟.
#index.cache.field.max_size: 50000
#index.cache.field.expire: 10m
#index.cache.field.type: soft
# field data部分&&circuit breaker部分：
# 用于field data 缓存的内存数量,主要用于当使用排序,faceting操作时,elasticsearch会将一些热点数据加载到内存中来提供给客户端访问,但是这种缓存是比较珍贵的,所以对它进行合理的设置.
# 可以使用值：eg:50mb 或者 30％(节点 node heap内存量),默认是：unbounded
#indices.fielddata.cache.size： unbounded
# field的超时时间.默认是-1,可以设置的值类型: 5m
#indices.fielddata.cache.expire: -1
# circuit breaker部分:
# 断路器是elasticsearch为了防止内存溢出的一种操作,每一种circuit breaker都可以指定一个内存界限触发此操作,这种circuit breaker的设定有一个最高级别的设定:indices.breaker.total.limit 默认值是JVM heap的70%.当内存达到这个数量的时候会触发内存回收
# 另外还有两组子设置：
#indices.breaker.fielddata.limit:当系统发现fielddata的数量达到一定数量时会触发内存回收.默认值是JVM heap的70%
#indices.breaker.fielddata.overhead:在系统要加载fielddata时会进行预先估计,当系统发现要加载进内存的值超过limit * overhead时会进行进行内存回收.默认是1.03
#indices.breaker.request.limit:这种断路器是elasticsearch为了防止OOM(内存溢出),在每次请求数据时设定了一个固定的内存数量.默认值是40%
#indices.breaker.request.overhead:同上,也是elasticsearch在发送请求时设定的一个预估系数,用来防止内存溢出.默认值是1
# Translog部分:
# 每一个分片(shard)都有一个transaction log或者是与它有关的预写日志,(write log),在es进行索引(index)或者删除(delete)操作时会将没有提交的数据记录在translog之中,当进行flush 操作的时候会将tranlog中的数据发送给Lucene进行相关的操作.一次flush操作的发生基于如下的几个配置
#index.translog.flush_threshold_ops:当发生多少次操作时进行一次flush.默认是 unlimited
#index.translog.flush_threshold_size:当translog的大小达到此值时会进行一次flush操作.默认是512mb
#index.translog.flush_threshold_period:在指定的时间间隔内如果没有进行flush操作,会进行一次强制flush操作.默认是30m
#index.translog.interval:多少时间间隔内会检查一次translog,来进行一次flush操作.es会随机的在这个值到这个值的2倍大小之间进行一次操作,默认是5s
#index.gateway.local.sync:多少时间进行一次的写磁盘操作,默认是5s
# 以上的translog配置都可以通过API进行动态的设置
```
### 安装配置zookeeper
部署在zk集群节点机器上
-   安装zookeeper
```
tar zxf zookeeper-3.4.9.tar.gz
mv zookeeper-3.4.9 /usr/local/zookeeper
mkdir /home/zk_data     #创建zk数据存放目录
echo 101 > /home/zk_data/myid   # 创建myid文件，三台机器上不可相同
```
-   配置zookeeper
```
cd /usr/local/zookeeper/conf
cp zoo_sample.cfg zoo.cfg
vim zoo.cfg
    tickTime=2000
    initLimit=10
    syncLimit=5
    dataDir=/home/zk_data
    clientPort=2181
    server.101=10.8.189.101:2888:3888
    server.102=10.8.189.102:2888:3888
    server.103=10.8.189.103:2888:3888
Zookeeper默认会将控制台信息输出到启动路径下的zookeeper.out中，显然在生产环境中我们不能允许Zookeeper这样做，通过如下方法，可以让Zookeeper输出按尺寸切分的日志文件：
修改conf/log4j.properties
zookeeper.root.logger=INFO, CONSOLE改为zookeeper.root.logger=INFO, ROLLINGFILE
修改bin/zkEnv.sh文件
ZOO_LOG4J_PROP="INFO,CONSOLE"改为ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
```
-   zk相关命令
```
/usr/local/zookeeper/bin/zkServer.sh start  #启动zk
/usr/local/zookeeper/bin/zkServer.sh status  #查看zk启动状态  正常三台节点，应有两台follower和一台leader
```
### 安装配置kafka
Kafka 是一个高吞吐量的分布式发布订阅日志服务，具有高可用、高性能、分布式、高扩展、持久性等特性。目前已经在各大公司中广泛使用。和之前采用 Redis 做轻量级消息队列不同，Kafka 利用磁盘作队列，所以也就无所谓消息缓冲时的磁盘问题。此外，如果公司内部已有 Kafka 服务在运行，logstash 也可以快速接入，免去重复建设的麻烦

-   在10.8.189.101和10.8.189.102两台机器上部署kafka做集群
```
tar zxf kafka_2.12-0.10.2.1.tgz 
mv kafka_2.12-0.10.2.1 /usr/local/kafka
cd /usr/local/kafka
mkdir /home/data     # 创建消息文件存储的路径
```
-   编辑kafka配置文件
```
vim /usr/local/kafka/config/server.properties
broker.id=101        # brokerid 每台机器不同
delete.topic.enable=true
num.network.threads=3
num.io.threads=8
post=9092
hostname=10.8.189.101
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/home/data
num.partitions=6
num.recovery.threads.per.data.dir=1
log.retention.hours=72
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
kafka默认启动以hostname识别
# vim /etc/hosts
    10.8.189.101 elk1
    10.8.189.102 elk2
    10.8.189.103 elk3
```
-   kafka启动
```
cd /usr/local/kafka/
nohup /usr/local/kafka/bin/kafka-server-start.sh /usr/local/kafka/config/server.properties &
```
-   测试zk和kafka之间的连通性
```
#在10.8.189.101上创建一个topics主题
/usr/local/kafka/bin/kafka-topics.sh --create --zookeeper 10.8.189.101:2181 --replication-factor 2 --partitions 1 --topic lvfang
    --replication-factor 2 #复制两份
    --partitions 1 #创建1个分区
    --topic #主题为shuaige
#在10.8.189.101上创建一个发布者
/usr/local/kafka/bin/kafka-console-producer.sh --broker-list 10.8.189.101:9092 --topic lvfang
#在10.8.189.102上创建一个订阅者
/usr/local/kafka/bin/kafka-console-consumer.sh --zookeeper 10.8.189.102:2181 --topic lvfang --from-beginning
# 删除一个topic 
# bin/kafka-topics.sh --delete --zookeeper 10.8.189.101:2181 --topic topics_name
# 如果还不能删除, 可以到zookeeper中去干掉它
# cd /usr/local/zookeeper-3.4.10/
# bin/zkCli.sh
# ls /brokers/topics            # 查看topic
# rmr /brokers/topics/test1     # 删除topic
```
在发布者的终端中输入字符信息，可以在订阅者上看到输出。这个时候kafka就可以用作消息队列的收发了

-   kafka 基本概念
```
Topic 主题，声明一个主题，producer指定该主题发布消息，订阅该主题的consumer对该主题进行消费

Partition 每个主题可以分为多个分区，每个分区对应磁盘上一个目录，分区可以分布在不同broker上，producer在发布消息时，可以通过指定partition key映射到对应分区，然后向该分区发布消息，在无partition key情况下，随机选取分区，一段时间内触发一次(比如10分钟)，这样就保证了同一个producer向同一partition发布的消息是顺序的。 消费者消费时，可以指定partition进行消费，也可以使用high-level-consumer api,自动进行负载均衡，并将partition分给consumer，一个partition只能被一个consumer进行消费

Consumer 消费者，可以多实例部署，可以批量拉取，有两类API可供选择：一个simpleConsumer，暴露所有的操作给用户，可以提交offset、fetch offset、指定partition fetch message；另外一个high-level-consumer(ZookeeperConsumerConnector)，帮助用户做基于partition自动分配的负载均衡，定期提交offset，建立消费队列等。simpleConsumer相当于手动挡，high-level-consumer相当于自动挡。

simpleConsumer：无需像high-level-consumer那样向zk注册brokerid、owner，甚至不需要提交offset到zk，可以将offset提交到任意地方比如(mysql,本地文件等)。

high-level-consumer：一个进程中可以启多个消费线程，一个消费线程即是一个consumer，假设A进程里有2个线程(consumerid分别为1，2)，B进程有2个线程(consumerid分别为1，2)，topic1的partition有5个，那么partition分配是这样的：
partition1 ---> A进程consumerid1
partition2 ---> A进程consumerid1
partition3 ---> A进程consumerid2
partition4 ---> B进程consumer1
partition5 ---> B进程consumer2

Group High-level-consumer可以声明group，每个group可以有多个consumer，每group各自管理各自的消费offset，各个不同group之间互不关联影响。

由于目前版本消费的offset、owner、group都是consumer自己通过zk管理，所以group对于broker和producer并不关心，一些监控工具需要通过group来监控，simpleComsumer无需声明group。
```
-   kafka配置详解
```
broker.id=1
    #当前机器在集群中的唯一标识，和zookeeper的myid性质一样
port=19092 
    #当前kafka对外提供服务的端口默认是9092
host.name=192.168.1.224
    #这个参数默认是关闭的，在0.8.1有个bug，DNS解析问题，失败率的问题。
num.network.threads=3 
    #这个是borker进行网络处理的线程数
num.io.threads=8 
    #这个是borker进行I/O处理的线程数
log.dirs=/usr/local/kafka/kafka_2.11-0.9.0.1/kafka_log
    #消息存放的目录，这个目录可以配置为“，”逗号分割的表达式，上面的num.io.threads要大于这个目录的个数这个目录，如果配置多个目录，新创建的topic他把消息持久化的地方是，当前以逗号分割的目录中，那个分区数最少就放那一个
socket.send.buffer.bytes=102400 
    #发送缓冲区buffer大小，数据不是一下子就发送的，先回存储到缓冲区了到达一定的大小后在发送，能提高性能
socket.receive.buffer.bytes=102400 
    #kafka接收缓冲区大小，当数据到达一定大小后在序列化到磁盘
socket.request.max.bytes=104857600 
    #这个参数是向kafka请求消息或者向kafka发送消息的请请求的最大数，这个值不能超过java的堆栈大小
num.partitions=1 
    #默认的分区数，一个topic默认1个分区数
log.retention.hours=168 
    #默认消息的最大持久化时间，168小时，7天
message.max.byte=5242880 
    #消息保存的最大值5M
default.replication.factor=2 
    #kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务
replica.fetch.max.bytes=5242880 
    #取消息的最大直接数
log.segment.bytes=1073741824 
    #这个参数是：因为kafka的消息是以追加的形式落地到文件，当超过这个值的时候，kafka会新起一个文件
log.retention.check.interval.ms=300000 
    #每隔300000毫秒去检查上面配置的log失效时间（log.retention.hours=168 ），到目录查看是否有过期的消息如果有，删除
log.cleaner.enable=false 
    #是否启用log压缩，一般不用启用，启用的话可以提高性能
zookeeper.connect=192.168.1.224:2181,192.168.1.225:2181,192.168.1.226:1218 
    #设置zookeeper的连接端口
```
### 安装配置logstash
```
tar zxf logstash-5.5.2.tar.gz
mv /opt/elk/logstash-5.5.2 /usr/local/logstash
```
-   配置broker (可以不用配置，使用filebeat上传数据流至kafka)
```
cd /usr/local/logstash/config
vim beat_to_kafka.conf
    input {
      beats {
        port => 5044
      }
    }
    
    filter {
      
    }
    
    # topic_id改成按beat中配置的document_type来输出到不同的topic中, 供kibana分组过滤用
    output {
      kafka {
        bootstrap_servers => "10.8.189.101:9092,10.8.189.102:9092,10.8.189.103:9092"
        topic_id => '%{[type]}'
      }
}

```
-   配置indexer集群
```
cd /usr/local/logstash/config
vim kafka_to_es.conf
    input {
      kafka {
        bootstrap_servers => "10.8.189.101:9092,10.8.189.102:9092,10.8.189.103:9092"
        group_id => "logstash"
        topics => ["drds-sql","drds-slow","sc_user","sc_channel","sc_order","sc_inventory","sc_message","sc_file","sc_marketing","rms",'scm','engineering']
        consumer_threads => 50
        decorate_events => true
      }
    }
    
    filter {
    
    }
    
    output {
      elasticsearch {
        hosts => ["10.8.189.101:9200","10.8.189.102:9200","10.8.189.103:9200"]
        index => "logstash-%{+YYYY.MM.dd.hh}"
        manage_template => true
        template_overwrite => true
        template_name => "drdsLogstash"
        flush_size => 50000
        idle_flush_time => 10
      }
}
```
-   启动logstash
```
### /usr/local/logstash/bin/logstash -f /usr/local/logstash/config/beat_to_kafka.conf > /var/log/beat_to_kafka.log &
/usr/local/logstash/bin/logstash -f /usr/local/logstash/config/kafka_to_es.conf > /var/log/kafka_to_es.log &

### /usr/local/logstash/bin/logstash -f /usr/local/logstash/config/beat_to_kafka.conf --path.data /home/logstash_data/beat_to_kafka > /var/log/beat_to_kafka.log &
/usr/local/logstash/bin/logstash -f /usr/local/logstash/config/kafka_to_es.conf --path.data /home/logstash_data/kafka_to_es > /var/log/kafka_to_es.log &

```
-   启动多个配置文件
```
mkdir -p /usr/local/logstash/conf.d
将需要启动的配置文件，全部放置在该目录下
/usr/local/logstash/bin/logstash -f /usr/local/logstash/conf.d --path.data /home/logstash_data/kafka_to_es > /var/log/kafka_to_es.log &
```

### filebeat
-   Filebeat是一个日志文件托运工具，在你的服务器上安装客户端后，filebeat会监控日志目录或者指定的日志文件，追踪读取这些文件（追踪文件的变化，不停的读），并且转发这些信息到elasticsearch或者logstarsh中存放。
-   当你开启filebeat程序的时候，它会启动一个或多个探测器（prospectors）去检测你指定的日志目录或文件，对于探测器找出的每一个日志文件，filebeat启动收割进程（harvester），每一个收割进程读取一个日志文件的新内容，并发送这些新的日志数据到处理程序（spooler），处理程序会集合这些事件，最后filebeat会发送集合的数据到你指定的地点。   

-   安装
```
wget https://download.elastic.co/beats/filebeat/filebeat-1.3.0-x86_64.tar.gz
tar -zxvf filebeat-1.3.0-x86_64.tar.gz
```
-   配置
```
vim filebeat.yml
默认监控日志配置
filebeat: 
  prospectors:
  -
      paths:
        - /var/log/*.log
      input_type: log 

根据需求做修改
filebeat:
  spool_size: 1024            # 最大可以攒够 1024 条数据一起发送出去
  idle_timeout: "5s"          # 否则每 5 秒钟也得发送一次
  registry_file: ".filebeat"  # 文件读取位置记录文件，默认在filebeat启动目录的data目录下保存。所以如果换一个目录保存.filebeat 会导致重复传输！

filebeat.config.prospectors:
  path: configs/*.yml      # filebeat的其他配置文件路径
  reload.enabled: true
  reload.period: 12h     # 开启12小时间隔重载配置文件


filebeat.prospectors:

- input_type: log
  paths:
    - /data/log/nginx/access*.log   # 读取的日志文件路径，支持文件名的通配符
  tail_files: true
  encoding: utf-8
  document_type: nginxaccess
  scan_frequency: 60s   # 60s会检测一次文件变化
  fields:
    type: "nginxaccess"  # 消费的kafka主题

- input_type: log
  paths:
    - /data/log/nginx/error*.log
  tail_files: true
  encoding: utf-8
  document_type: nginxerror
  scan_frequency: 60s
  fields:
    type: "nginxerror"


output.kafka:
  hosts: ["10.8.189.101:9092","10.8.189.102:9092"]    # broker kafka集群中的机器
  topic: '%{[type]}'
  use_type: true 
  partition.round_robin:
    reachable_only: true     # 只把消息发布到可用的分区上
  workers: 3
  required_acks: 1   # 经纪人要求的ACK可靠性水平 0 =无响应，1 =等待本地提交，-1 =等待所有副本提交
  compression: gzip   # Sets the output compression codec. Must be one of none, snappy and gzip. The default is gzip
  max_message_bytes: 1000000
```
-   启动
```
nohup ./filebeat -e -c filebeat.yml >/dev/null 2>&1 &
```
