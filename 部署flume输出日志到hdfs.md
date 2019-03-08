#部署flume输出日志到hdfs
###安装flume
	wget http://archive.apache.org/dist/flume/1.7.0/apache-flume-1.7.0-bin.tar.gz
	tar -zxf apache-flume-1.7.0-bin.tar.gz
	mv apache-flume-1.7.0-bin /usr/local/flume
###配置文件
	agent.sources = r1
	agent.channels = c1
	agent.sinks = s1

	agent.sources.r1.type = org.apache.flume.source.kafka.KafkaSource
	agent.sources.r1.zookeeperConnect = 10.235.200.13:2181 
	agent.sources.r1.topic = wgame_log
	agent.sources.r1.groupId = flume
	agent.sources.r1.kafka.consumer.timeout.ms = 100
	agent.sources.r1.interceptors = timestamp
	agent.sources.r1.interceptors.timestamp.type = org.apache.flume.interceptor.EventTimestampInterceptor$Builder
	agent.sources.r1.interceptors.timestamp.preserveExisting = false        
	agent.sources.r1.interceptors.timestamp.delimiter = 
	agent.sources.r1.interceptors.timestamp.dateIndex = 12
	agent.sources.r1.interceptors.timestamp.dateFormat = tsecond

	# The channel can be defined as follows.
	agent.sources.r1.channels = c1
	#Specify the channel the sink should use
	agent.sinks.s1.channel = c1


	agent.channels.c1.type = file 
	agent.channels.c1.checkpointDir = /data/flume/checkpoint 
	agent.channels.c1.dataDirs = /data/flume/data

	agent.sinks.s1.type = hdfs
	agent.sinks.s1.hdfs.fileType = DataStream
	agent.sinks.s1.hdfs.path = hdfs://10.235.102.193:9000/user/hive/warehouse/wgame_lang.db/log/tt=%Y%m%d
	agent.sinks.s1.hdfs.rollInterval = 3600 
	agent.sinks.s1.hdfs.rollSize = 4096000000
	agent.sinks.s1.hdfs.rollCount = 0
	agent.sinks.s1.hdfs.batchSize = 10000

###启动
	nohup ./flume-ng agent -c ../conf -f ../conf/flume-conf-lang.properties -n agent -Dflume.root.logger=INFO,console &
###相关启动报错
	
	Failed to start agent because dependencies were not found in classpath. Error follows.
	问题分析：
		缺少HDFS依赖库
	解决方法：
		从hadoop下拷贝相应的jar包到flume的lib下，但是这里并为起到作用，因为这里的Flume是独立安装，没有hadoop的环境变量，所以需要安装一个hadoop,然后配置hadoop环境变量即可
		tar xvf hadoop-2.8.0.tar.gz -C 
		mv hadoop-2.8.0 /usr/local/hadoop
		配置环境变量：
		vim /etc/profile.d/hadoop.sh
		export HADOOP_PREFIX="/usr/local/hadoop"
		export PATH=$PATH:$HADOOP_PREFIX/bin:$HADOOP_PREFIX/sbin
		export HADOOP_COMMON_HOME=${HADOOP_PREFIX}
		export HADOOP_HDFS_HOME=${HADOOP_PREFIX}
		export HADOOP_MAPRED_HOME=${HADOOP_PREFIX}
		export HADOOP_YARN_HOME=${HADOOP_PREFIX}

		source /etc/profile