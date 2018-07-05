#hadoop日志系统部署
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
		fs.defaultFS ： 这个属性用来指定namenode的hdfs协议的文件系统通信地址，可以指定一个主机+端口，也可以指定为一个namenode服务（这个服务内部可以有多台namenode实现ha的namenode服务
		hadoop.tmp.dir : hadoop集群在工作的时候存储的一些临时文件的目录
-	修改 hdfs-site.xml 文件

		vim /usr/local/hadoop/etc/hadoop/hdfs-site.xml
		dfs.namenode.name.dir：namenode数据的存放地点。也就是namenode元数据存放的地方，记录了hdfs系统中文件的元数据。
		dfs.datanode.data.dir： datanode数据的存放地点。也就是block块存放的目录了。
		dfs.replication：hdfs的副本数设置。也就是上传一个文件，其分割为block块后，每个block的冗余副本个数，默认配置是3。
		dfs.secondary.http.address：secondarynamenode 运行节点的信息，和 namenode 不同节点
