#日志系统部署-flume
-	下载安装

		wget http://archive.apache.org/dist/flume/1.7.0/apache-flume-1.7.0-bin.tar.gz
		tar -zxf apache-flume-1.7.0-bin.tar.gz
		sudo mv apache-flume-1.7.0-bin /usr/local/flume
		sudo chown -R hduser:hadoop /usr/local/flume

-	配置环境变量

		vim ~/.bashrc
		export FLUME_HOME=/usr/local/flume
		export PATH=$PATH:$FLUME_HOME/bin
		source ~/.bashrc
		cd /usr/local/flume/conf
		cp flume-env.sh.template flume-env.sh
		cp flume-conf.properties.template flume-conf.properties
		vim flume-env.sh
		export JAVA_HOME=/usr/java/jdk1.8.0_131
		export JAVA_OPTS="-Xms100m -Xmx200m -Dcom.sun.management.jmxremote"
-	配置

		# The configuration file needs to define the sources,
		# the channels and the sinks.
		# Sources, channels and sinks are defined per agent,
		# in this case called 'agent'
		agent.sources = r1
		agent.channels = c1
		agent.sinks = s1
		agent.sources.r1.type = org.apache.flume.source.kafka.KafkaSource
		agent.sources.r1.zookeeperConnect = 10.235.102.178:2181
		agent.sources.r1.topic = wgame_log
		agent.sources.r1.groupId = flume
		agent.sources.r1.kafka.consumer.timeout.ms = 100
		agent.sources.r1.interceptors = timestamp
		agent.sources.r1.interceptors.timestamp.type = org.apache.flume.interceptor.EventTimestampInterceptor$Builder
		agent.sources.r1.interceptors.timestamp.preserveExisting = false
		agent.sources.r1.interceptors.timestamp.delimiter = |
		agent.sources.r1.interceptors.timestamp.dateIndex = 12
		agent.sources.r1.interceptors.timestamp.dateFormat = tsecond
		
		# The channel can be defined as follows.
		agent.sources.r1.channels = c1
		#Specify the channel the sink should use
		agent.sinks.s1.channel = c1
		agent.channels.c1.type = memory
		agent.channels.c1.capacity = 10000
		agent.channels.c1.transactionCapacity = 1000
		agent.sinks.s1.type = hdfs
		agent.sinks.s1.hdfs.fileType = DataStream
		agent.sinks.s1.hdfs.path = hdfs://10.235.102.13:9000/user/hive/warehouse/wgame.db/log/tt=%Y%m%d
		agent.sinks.s1.hdfs.rollInterval = 60
		agent.sinks.s1.hdfs.rollSize = 67108864
		agent.sinks.s1.hdfs.rollCount = 1000
		agent.sinks.s1.hdfs.batchSize = 10000

-	启动

		nohup  ./flume-ng agent -n agent -c /usr/local/flume/conf/ -f /usr/local/flume/conf/flume-conf.properties &

-	关闭

		bin/flume-ng agent stop