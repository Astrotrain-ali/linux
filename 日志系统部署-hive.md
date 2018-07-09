#日志系统部署-hive

-	下载安装

		su - hduser
		wget https://archive.apache.org/dist/hive/hive-1.2.1/apache-hive-1.2.1-bin.tar.gz
		tar zxvf apache-hive-1.2.1-bin.tar.gz
		sudo mv apache-hive-1.2.1-bin /usr/local/hive
		sudo chown -R hduser:hadoop /usr/local/hive

-	环境变量

		vim ~/.bashrc
		export HIVE_HOME=/usr/local/hive
		export PATH=:${HIVE_HOME}/bin:$PATH
		vim /usr/local/hive/conf/hive-env.sh
		HADOOP_HOME=/usr/local/hadoop/
		export HIVE_CONF_DIR=/usr/local/hive/conf
		export HIVE_AUX_JARS_PATH=/usr/local/hive/lib

-	安装配置Mysql-server

		yum -y install mariadb-server mariadb
		systemctl start mariadb.service
		systemctl enable mariadb.service
		mysql_secure_installation
		create database hive DEFAULT CHARACTER SET utf8;
		grant all PRIVILEGES on *.* TO 'hive'@'localhost' IDENTIFIED BY '123456' WITH GRANT OPTION;
		yum install unzip
		cd /usr/local/hive/lib
		wget http://www.java2s.com/Code/JarDownload/mysql/mysql-connector-java-commercial-5.1.7-bin.jar.zip
		unzip mysql-connector-java-commercial-5.1.7-bin.jar.zip
		rm -Rf mysql-connector-java-commercial-5.1.7-bin.jar.zip

-	配置 hive-site.xml文件

		cd /usr/local/hive/conf
		cp hive-default.xml.template hive-site.xml
		<configuration>
		    <property>
		        <name>javax.jdo.option.ConnectionURL</name>
		        <value>jdbc:mysql://localhost/metastore?createDatabaseIfNotExist=true</value>
		        <description>为Hive创建的访问MySQL的用户的库空间</description>
		    </property>
		    <property>
		        <name>javax.jdo.option.ConnectionDriverName</name>
		        <value>com.mysql.jdbc.Driver</value>
		        <description> MySQL的JDBC驱动（需要把mysql的驱动包放到目录 <HIVE_HOME>/lib 中）</description>
		    </property>
		    <property>
		        <name>javax.jdo.option.ConnectionUserName</name>
		        <value>hive</value>
		        <description>为Hive创建的访问MySQL的用户</description>
		    </property>
		    <property>
		        <name>javax.jdo.option.ConnectionPassword</name>
		        <value>123456</value>
		        <description>password for connecting to mysql server</description>
		    </property>
		</configuration>

-	启动

		hive