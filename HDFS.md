#HDFS的概念
#####HDFS有高容错性的特点，并且设计用来部署在低廉的（low-cost）硬件上；而且它提供高吞吐量（high throughput）来访问应用程序的数据，适合那些有着超大数据集（large data set）的应用程序。HDFS放宽了（relax）POSIX的要求，可以以流的形式访问（streaming access）文件系统中的数据。

-	linux中每个磁盘有默认的数据块大小，这是对磁盘操作的最小单位，通常512字节。HDFS同样也有块（Block）的概念，默认64MB/128MB,比磁盘块大得多。与单一的文件系统类似，HDFS上的文件系统也被划分成多个分块（Chunk）作为独立的存储单元。
	-	一个hadoop文件就是由一系列分散在不同的DataNode上的block组成。
-	HDFS数据块：HDFS上的文件被划分为块大小的多个分块，作为独立的存储单元，称为数据块，默认大小是64MB/127。 
	- 块相对较大，主要是把寻道时间最小化。如果一个块足够大，从硬盘传输数据的时间将远远大于寻找块起始位置的时间。这样使得HDFS的数据块速度和硬盘的传输速度更加接近。
-	HDFS的三个节点
	-	Namenode：HDFS的守护进程，用来管理文件系统的命名空间，负责记录文件是如何分割成数据块，以及这些数据块分别被存储到那些数据节点上，它的主要功能是对内存及IO进行集中管理。
	-	Datanode：文件系统的工作节点，根据需要存储和检索数据块，并且定期向namenode发送他们所存储的块的列表。
	-	Secondary Namenode：辅助后台程序，与NameNode进行通信，以便定期保存HDFS元数据的快照。

- HDFS的高可用性
	-  Hadoop的2.x发行版本在HDFS中增加了对高可用性（HA）的支持。在这一实现中，配置了一对活动-备用（active-standby）namenode。当活动namenode失效，备用namenode就会接管它的任务并开始服务于来自客户端的请求，不会有明显的中断。
	-  架构的实现包括：
		-  namenode之间通过高可用的共享存储实现编辑日志的共享。
		-  datanode同时向两个namenode发送数据块处理报告。
		-  客户端使用特定的机制来处理namenode的失效问题，这一机制对用户是透明的。
		-	故障转移控制器：管理着将活动namenode转移给备用namenode的转换过程，基于ZooKeeper并由此确保有且仅有一个活动namenode。每一个namenode运行着一个轻量级的故障转移控制器，其工作就是监视宿主namenode是否失效并在namenode失效时进行故障切换。

- 命令行接口

两个属性项： fs.default.name 用来设置Hadoop的默认文件系统，设置hdfs URL则是配置HDFS为Hadoop的默认文件系统。dfs.replication  设置文件系统块的副本个数

	-	文件系统的基本操作：hadoop fs -help可以获取所有的命令及其解释
		-	hadoop fs -ls /   列出hdfs文件系统根目录下的目录和文件
		-	hadoop fs -copyFromLocal <local path> <hdfs path> 从本地文件系统将一个文件复制到HDFS
		-	hadoop fs -rm -r <hdfs dir or file> 删除文件或文件夹及文件夹下的文件
		-	hadoop fs -mkdir <hdfs dir>在hdfs中新建文件夹