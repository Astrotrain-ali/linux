#日志系统部署-kafka
Kafka集群是把状态保存在Zookeeper中的，首先要搭建Zookeeper集群

-	下载

		su - root
		wget http://www-us.apache.org/dist/kafka/0.10.1.0/kafka_2.11-0.10.1.0.tgz
		tar zxvf kafka_2.11-0.10.1.0.tgz
		mv kafka_2.11-0.10.1.0 /usr/local/kafka

-	配置

		#有很多文件，这里可以发现有Zookeeper文件，我们可以根据Kafka内带的zk集群来启动，但是建议使用独立的zk集群
		cd /usr/local/kafka/config
		cp server.properties server.properties.bak
		vim server.properties
		#当前机器在集群中的唯一标识，和zookeeper的myid性质一样
		broker.id=0
		#当前kafka对外提供服务的端口默认是9092
		port=9092
		#这个参数默认是关闭的，在0.8.1有个bug，DNS解析问题，失败率的问题。
		host.name=10.235.102.178
		advertised.host.name=10.235.102.178
		#这个是borker进行网络处理的线程数
		num.network.threads=3
		#这个是borker进行I/O处理的线程数
		num.io.threads=8
		#发送缓冲区buffer大小，数据不是一下子就发送的，先回存储到缓冲区了到达一定的大小后在发送，能提高性能
		socket.send.buffer.bytes=102400
		#kafka接收缓冲区大小，当数据到达一定大小后在序列化到磁盘
		socket.receive.buffer.bytes=102400
		#这个参数是向kafka请求消息或者向kafka发送消息的请请求的最大数，这个值不能超过java的堆栈大小
		socket.request.max.bytes=104857600
		#消息存放的目录，这个目录可以配置为“，”逗号分割的表达式，上面的num.io.threads要大于这个目录的个数这个目录，如果配置多个目录，新创建的topic他把消息持久化的地方是，当前以逗号分割的目录中，那个分区数最少就放那一个
		log.dirs=/tmp/kafka-logs
		#默认的分区数，一个topic默认1个分区数
		num.partitions=256
		num.recovery.threads.per.data.dir=1
		log.flush.interval.messages=10000
		log.flush.interval.ms=1000
		#默认消息的最大持久化时间，168小时，7天
		log.retention.hours=168
		log.retention.bytes=1073741824
		#这个参数是：因为kafka的消息是以追加的形式落地到文件，当超过这个值的时候，kafka会新起一个文件
		log.segment.bytes=1073741824
		#每隔300000毫秒去检查上面配置的log失效时间（log.retention.hours=168 ），到目录查看是否有过期的消息如果有，删除
		log.retention.check.interval.ms=300000
		#设置zookeeper的连接端口
		zookeeper.connect=10.235.102.178:2181,10.235.102.246:2181,10.235.102.13:2181
		zookeeper.connection.timeout.ms=6000

-	启动

		./bin/kafka-server-start.sh -daemon config/server.properties

-	测试

		创建一个主题
		/usr/local/kafka/bin/kafka-topics.sh --create --zookeeper 10.235.102.178:2181 --replication-factor 2 --partitions 1 --topic xgame
		
		创建一个发布者
		/usr/local/kafka/bin/kafka-console-producer.sh --broker-list 10.235.102.178:9092 --topic xgame
		命令执行之后直接在控制台随意输入
		
		创建一个订阅者
		/usr/local/kafka/bin/kafka-console-consumer.sh --zookeeper 10.235.102.246:2181 --topic xgame --from-beginning
		命令执行之后会在控制台看到发布者发布的消息
		
		删除一个topic 
		/usr/local/kafka/bin/kafka-topics.sh --delete --zookeeper 10.235.102.178:2181 --topic xgame

