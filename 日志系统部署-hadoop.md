#日志系统部署-hadoop
#环境规划：

-	centos7.3

		10.235.102.178_master,hadoop
		10.235.102.246_node1,hadoop
		10.235.102.13_node2,hadoop



-	系统用户：


		groupadd hadoop
		useradd hduser
		usermod -g hadoop hduser
		passwd hduser

-	hosts：

		127.0.0.1 localhost
		10.235.102.178 master
		10.235.102.246 node1
		10.235.102.13 node2

-	主机互信：
		
		hduser
		mkdir ~/.ssh
		cd ~/.ssh
		ssh-keygen -t rsa
		cat id_rsa.pub >> authorized_keys
		chmod 600 authorized_keys
		ssh-copy-id hduser@hostip

-	sudo权限
	
		vim /etc/sudoers
		root    ALL=(ALL)       ALL
		hduser ALL=(ALL) NOPASSWD:ALL
		
		sed -i '/^root/a\hduser ALL=(ALL) NOPASSWD:ALL' /etc/sudoers

-	jdk
	
		#卸载旧版的open-jdk
		yum -y remove java*
		#检验
		java -version
		#sun jdk下载
		wget --no-check-certificate --no-cookies \
		--header "Cookie: oraclelicense=accept-securebackup-cookie" \
		http://download.oracle.com/otn-pub/java/jdk/8u131-b11/d54c1d3a095b4ff2b6607d096fa80163/jdk-8u131-linux-x64.rpm
		#安装
		rpm -ivh jdk-8u131-linux-x64.rpm
		#检验
		java -version

#hadoop集群安装
-	安装

		su - hduser
		wget https://archive.apache.org/dist/hadoop/core/hadoop-2.6.0/hadoop-2.6.0.tar.gz
		tar xzf hadoop-2.6.0.tar.gz
		sudo mv hadoop-2.6.0 /usr/local/hadoop
		sudo chown -R hduser:hadoop /usr/local/hadoop
-	环境变量

		su - hduser
		vim $HOME/.bashrc
		export HADOOP_HOME=/usr/local/hadoop
		export JAVA_HOME=/usr/java/jdk1.8.0_131
		export PATH=${JAVA_HOME}/bin:${PATH}
		source $HOME/.bashrc
-	配置数据存储目录

		su - hduser
		sudo mkdir -p /hadoop/hadoopinfra/hdfs/namenode
		sudo mkdir -p /hadoop/hadoopinfra/hdfs/datanode
		sudo chown -R hduser:hadoop /hadoop
-	修改 hadoop-env.sh 文件

		vim /usr/local/hadoop/etc/hadoop/hadoop-env.sh
		export JAVA_HOME=/usr/java/jdk1.8.0_131

-	修改 core-site.xml 文件
	
		vim /usr/local/hadoop/etc/hadoop/core-site.xml
		<configuration>
		<property>
		<name>fs.defaultFS</name>
		<value>hdfs://master:9000/</value>
		<description>NameNode URI</description>
		</property>
		</configuration>

		fs.defaultFS ： 这个属性用来指定namenode的hdfs协议的文件系统通信地址，可以指定一个主机+端口，也可以指定为一个namenode服务（这个服务内部可以有多台namenode实现ha的namenode服务
		hadoop.tmp.dir : hadoop集群在工作的时候存储的一些临时文件的目录
-	修改 hdfs-site.xml 文件

		vim /usr/local/hadoop/etc/hadoop/hdfs-site.xml
		<configuration>
		<property>
		<name>dfs.replication</name>
		<value>3</value>
		</property>
		<property>
		<name>dfs.name.dir</name>
		<value>file:///hadoop/hadoopinfra/hdfs/namenode</value>
		</property>
		<property>
		<name>dfs.data.dir</name>
		<value>file:///hadoop/hadoopinfra/hdfs/datanode</value>
		</property>
		</configuration>
		dfs.namenode.name.dir：namenode数据的存放地点。也就是namenode元数据存放的地方，记录了hdfs系统中文件的元数据。
		dfs.datanode.data.dir： datanode数据的存放地点。也就是block块存放的目录了。
		dfs.replication：hdfs的副本数设置。也就是上传一个文件，其分割为block块后，每个block的冗余副本个数，默认配置是3。
		dfs.secondary.http.address：secondarynamenode 运行节点的信息，和 namenode 不同节点

-	修改 mapred-site.xml 文件
		
		vim /usr/local/hadoop/etc/hadoop/mapred-site.xml
		<?xml version="1.0" encoding="UTF-8"?>
		<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
		<configuration>
		<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
		</property>
		</configuration>
		mapreduce.framework.name：指定mr框架为yarn方式,Hadoop二代MP也基于资源管理系统Yarn来运行 。 

-	修改yarn-site.xml文件

		vim /usr/local/hadoop/etc/hadoop/yarn-site.xml
		<configuration>
		<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
		</property>
		<property>
		<name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
		<value>org.apache.hadoop.mapred.ShuffleHandler</value>
		</property>
		<property>
		<name>yarn.resourcemanager.resource-tracker.address</name>
		<value>master:8031</value>
		</property>
		<property>
		<name>yarn.resourcemanager.scheduler.address</name>
		<value>master:8030</value>
		</property>
		<property>
		<name>yarn.resourcemanager.address</name>
		<value>master:8032</value>
		</property>
		</configuration>

		yarn.resourcemanager.hostname：yarn总管理器的IPC通讯地址

 		yarn.nodemanager.aux-services：集群为 MapReduce 程序提供的 shuffle 服务

-	修改slaves文件
	
		10.235.102.178
		10.235.102.246
		10.235.102.13
		或者
		master
		node1
		node2

-	拷贝hadoop目录到其他机器上

		重点强调： 每台服务器中的hadoop安装包的目录必须一致， 安装包的配置信息还必须保持一致
		重点强调： 每台服务器中的hadoop安装包的目录必须一致， 安装包的配置信息还必须保持一致
		重点强调： 每台服务器中的hadoop安装包的目录必须一致， 安装包的配置信息还必须保持一致
		
		cd /usr/local/
		scp -r hadoop/ hduser@node1:/home/hduser/
		scp -r hadoop/ hduser@node2:/home/hduser/

-	第一次启动前先格式化

		在master机器上执行下面命令
		cd /usr/local/hadoop/
		bin/hadoop namenode -format

-	启动

		在master机器上执行下面命令
		cd /usr/local/hadoop/
		sbin/start-all.sh
		所有机器结点上的进程会自动启动起来

-	关闭

		cd /usr/local/hadoop/
		sbin/stop-all.sh

-	验证

		web访问http://master:50070 HDFS页面
		web访问http://master:8088  YARN页面

![](https://i.imgur.com/1WDW4sY.png)

		此处踩坑，因为主机名采用x-x-x-x格式内外地址命名，在进行namenode初始化的时候会异常解析到一个外网地址，而且只显示一个node地址，如下图；

![](https://i.imgur.com/5yKOPlx.png)

		随即更改主机命名方式，然后重新初始化namenode。正确解析得到地址如下；

![](https://i.imgur.com/UeBQodj.png)


####Hadoop的简单使用

		cd /usr/local/hadoop/bin/
		#在HDFS上创建一个文件夹/test/input
		./hadoop fs -mkdir -p /test/input
		#查看
		./hadoop fs -ls /
		./hadoop fs -ls /test
		#上传文件
		#先创建
		vim ~/text.txt
		#上传到HDFS的/test/input文件夹中
		./hadoop fs -put ~/text.txt /test/input
		#查看是否上传成功
		./hadoop fs -ls /test/input
		#下载文件
		#将刚刚上传的文件下载到~/data文件夹中
		./hadoop fs -get /test/input/text.txt ~/data
####运行一个mapreduce的例子程序： wordcount
		hadoop jar /usr/local/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.6.0.jar wordcount /test/input /test/output

![](https://i.imgur.com/1onwaj0.png)

![](https://i.imgur.com/qYXIg3R.png)

		#查看结果

![](https://i.imgur.com/0QvHlRW.png)