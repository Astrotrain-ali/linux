#日志系统部署-hbase+zookeeper
###zookper安装

-	三台机器操作下载

		http://zookeeper.apache.org/releases.html#download

-	安装

		tar zxf zookeeper-3.4.9.tar.gz
		mv zookeeper-3.4.9 /usr/local/zookeeper
		echo "
		10.235.102.178 zk1
		10.235.102.246 zk2
		10.235.102.13 zk3" >> /etc/hosts

-	创建zk数据存放目录

		mkdir /data/zookeeper

-	创建myid文件，三台机器上不可相同

		echo 1 > /data/zookeeper/myid
		echo 2 > /data/zookeeper/myid
		echo 3 > /data/zookeeper/myid 

-	配置

		cd /usr/local/zookeeper/conf
		cp zoo_sample.cfg zoo.cfg
		vim zoo.cfg
		#Zookeeper CS通信心跳数：服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳。
		tickTime=2000
		#LF初始通信时限：集群中的follower服务器(F)与leader服务器(L)之间 初始连接 时能容忍的最多心跳数（tickTime的数量）
		initLimit=10
		#LF同步通信时限：集群中的follower服务器(F)与leader服务器(L)之间 请求和应答 之间能容忍的最多心跳数（tickTime的数量）。
		syncLimit=5
		dataDir=/data/zookeeper
		clientPort=2181
		server.1=10.235.102.178:2888:3888
		server.2=10.235.102.246:2888:3888
		server.3=10.235.102.13:2888:3888
		
		#Zookeeper默认会将控制台信息输出到启动路径下的zookeeper.out中
		#显然在生产环境中我们不能允许Zookeeper这样做，通过如下方法，可以让Zookeeper输出按尺寸切分的日志文件：
		#修改conf/log4j.properties
		zookeeper.root.logger=INFO, CONSOLE改为zookeeper.root.logger=INFO, ROLLINGFILE　　　　　　
		#修改bin/zkEnv.sh文件
		ZOO_LOG4J_PROP="INFO,CONSOLE"改为ZOO_LOG4J_PROP="INFO,ROLLINGFILE"

-	zk启动测试

		#启动zk
		/usr/local/zookeeper/bin/zkServer.sh start 
		#查看zk启动状态  正常三台节点，应有两台follower和一台leader 
		/usr/local/zookeeper/bin/zkServer.sh status  


###hbase安装
-	在主节点操作

		cd /usr/local/
		wget https://archive.apache.org/dist/hbase/hbase-0.98.8/hbase-0.98.8-hadoop2-bin.tar.gz
		tar xzf hbase-0.98.8-hadoop2-bin.tar.gz
		mv hbase-0.98.8-hadoop2 hbase
		chown -R hduser:hadoop hbase

-	修改 hbase-env.sh 文件

		cd /usr/local/hbase/conf
		#修改 vim hbase-env.sh
		export JAVA_HOME=/usr/java/jdk1.8.0_131
		export HBASE_MANAGES_ZK=true
		#需要注意的地方是 ZooKeeper的配置。这与 hbase-env.sh 文件相关，文件中 HBASE_MANAGES_ZK 环境变量用来设置是使用hbase默认自带的 Zookeeper还是使用独立的ZooKeeper。HBASE_MANAGES_ZK=false 时使用独立的，为true时使用默认自带的。

-	修改 hbase-site.xml 文件

		<configuration>
		    //Here you have to set the path where you want HBase to store its files.
		    <property>
		        <name>hbase.rootdir</name>
		        <value>hdfs://master:9000/hbase</value>
		        <description>The directory shared by region servers. Should be fully-qualified to include the filesystem to use. E.g: hdfs://NAMENODE_  </property>
		    <property>
		        <name>hbase.master</name>
		        <value>hdfs://master:60000</value>
		        <description>The host and port that the HBase master runs at.</description>
		    </property>
		    <property>
		        <name>hbase.cluster.distributed</name>
		        <value>true</value>
		        <description>The mode the cluster will be in. Possible values are
		        false: standalone and pseudo-distributed setups with managed Zookeeper
		        true: fully-distributed with unmanaged Zookeeper Quorum (see hbase-env.sh)</description>
		    </property>
		    <property>
		        <name>hbase.zookeeper.quorum</name>
		        <value>10.235.102.178,10.235.102.246,10.235.102.13</value>
		    </property>
		    <property>
		        <name>zookeeper.session.timeout</name>
		        <value>60000</value>
		    </property>
		    <property>
		        <name>hbase.zookeeper.property.clientPort</name>
		        <value>2181</value>
		    </property>
		    <property>
		        <name>hbase.zookeeper.property.dataDir</name>
		        <value>/zookeeper</value>
		        <description>zk的配置，snapshot存放的目录，默认是${hbase.tmp.dir}/zookeeper；</description>
		    </property>
		</configuration>

-	修改 regionservers 文件设置slave

		vim regionservers
		node1
		node2

-	拷贝hbase目录到其他结点机器上

		scp -r hbase root@node1:/usr/local/
		scp -r hbase root@node2:/usr/local/

-	启动

		进入master所在机器在的终端
		cd /usr/local/hbase/bin
		./start-hbase.sh
		./hbase-daemon.sh start thrift2

-	查看hbase状态

		http://master:60010
		http://slave:60020

-	关闭

		进入master所在机器在的终端
		cd /usr/local/hbase/bin
		./hbase-daemon.sh stop thrift2
		./stop-hbase.sh

