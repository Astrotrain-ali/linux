#MapReduce
-	线性，可伸缩性编程。程序员需要编写 map函数 和 reduce函数。每个函数定义从一个键值对集合到另一个键值对集合的映射。
-	map函数：接受一个键值对（key-value pair），产生一组中间键值对。MapReduce框架会将map函数产生的中间键值对里键相同的值传递给一个reduce函数。
-	reduce函数：接受一个键，以及相关的一组值，将这组值进行合并产生一组规模更小的值（通常只有一个或零个值）。
-	MR框架是由一个单独运行在主节点上的JobTracker和运行在每个集群从节点上的TaskTracker共同组成。主节点负责调度构成一个作业的所有任务，这些任务分布在不同的不同的从节点上。主节点监视它们的执行情况，并重新执行之前失败的任务。从节点仅负责由主节点指派的任务。当一个Job被提交时，JobTracker接受到提交作业和配置信息之后，就会将配置信息等分发给从节点，同时调度任务并监控TaskTracker的执行。JobTracker可以运行于集群中的任意一台计算机上。TaskTracker负责执行任务，它必须运行在DataNode上，DataNode既是数据存储节点，也是计算节点。JobTracker将map任务和reduce任务分发给空闲的TaskTracker，这些任务并行运行，并监控任务运行的情况。如果JobTracker出了故障，JobTracker会把任务转交给另一个空闲的TaskTracker重新运行。

-	HDFS和MR共同组成Hadoop分布式系统体系结构的核心。HDFS在集群上实现了分布式文件系统，MR在集群上实现了分布式计算和任务处理。HDFS在MR任务处理过程中提供了文件操作和存储等支持，MR在HDFS的基础上实现了任务的分发、跟踪、执行等工作，并收集结果，二者相互作用，完成分布式集群的主要任务。

-	Hadoop上的并行应用程序开发是基于MR编程框架。MR编程模型原理：利用一个输入的key-value对集合来产生一个输出的key-value对集合。MR库通过Map和Reduce两个函数来实现这个框架。用户自定义的map函数接受一个输入的key-value对，然后产生一个中间的key-value对的集合。MR把所有具有相同的key值的value结合在一起，然后传递个reduce函数。Reduce函数接受key和相关的value结合，reduce函数合并这些value值，形成一个较小的value集合。通常我们通过一个迭代器把中间的value值提供给reduce函数（迭代器的作用就是收集这些value值），这样就可以处理无法全部放在内存中的大量的value值集合了。

#MapReduceV2-->Yarn
-	从业界使用分布式系统的变化趋势和 hadoop 框架的长远发展来看，MapReduce 的 JobTracker/TaskTracker 机制需要大规模的调整来修复它在可扩展性，内存消耗，线程模型，可靠性和性能上的缺陷。在过去的几年中，hadoop 开发团队做了一些 bug 的修复，但是最近这些修复的成本越来越高，这表明对原框架做出改变的难度越来越大。

-	为从根本上解决旧 MapReduce 框架的性能瓶颈，促进 Hadoop 框架的更长远发展，从 0.23.0 版本开始，Hadoop 的 MapReduce 框架完全重构，发生了根本的变化。新的 Hadoop MapReduce 框架命名为 MapReduceV2 或者叫 Yarn

####YARN基本架构
-	YARN是Hadoop 2.0中的资源管理系统，它的基本设计思想是将MRv1中的JobTracker拆分成了两个独立的服务：一个全局的资源管理器ResourceManager和每个应用程序特有的ApplicationMaster。其中ResourceManager负责整个系统的资源管理和分配，而ApplicationMaster负责单个应用程序的管理。

-	YARN总体上仍然是Master/Slave结构，在整个资源管理框架中，ResourceManager为Master，NodeManager为Slave，ResourceManager负责对各个NodeManager上的资源进行统一管理和调度。当用户提交一个应用程序时，需要提供一个用以跟踪和管理这个程序的ApplicationMaster，它负责向ResourceManager申请资源，并要求NodeManger启动可以占用一定资源的任务。由于不同的ApplicationMaster被分布到不同的节点上，因此它们之间不会相互影响。

####重要组件
-	ResourceManager(RM);
	-	RM是一个全局的资源管理器,负责整个系统的资源管理和分配。它主要由两个组件构成：调度器（Scheduler）和应用程序管理器（Applications Manager，ASM）。
		-	调度器（分配Container）
			-	调度器根据容量、队列等限制条件（如每个队列分配一定的资源，最多执行一定数量的作业等），将系统中的资源分配给各个正在运行的应用程序。需要注意的是，该调度器是一个“纯调度器”，它不再从事任何与具体应用程序相关的工作，比如不负责监控或者跟踪应用的执行状态等，也不负责重新启动因应用执行失败或者硬件故障而产生的失败任务，这些均交由应用程序相关的ApplicationMaster完成。调度器仅根据各个应用程序的资源需求进行资源分配，而资源分配单位用一个抽象概念“资源容器”（Resource Container，简称Container）表示，Container是一个动态资源分配单位，它将内存、CPU、磁盘、网络等资源封装在一起，从而限定每个任务使用的资源量。此外，该调度器是一个可插拔的组件，用户可根据自己的需要设计新的调度器，YARN提供了多种直接可用的调度器，比如Fair Scheduler和Capacity Scheduler等。
		-	应用程序管理器
			-	应用程序管理器负责管理整个系统中所有应用程序，包括应用程序提交、与调度器协商资源以启动ApplicationMaster、监控ApplicationMaster运行状态并在失败时重新启动它等。

-	ApplicationMaster（AM）
	
	-	用户提交的每个应用程序均包含一个AM，主要功能包括：
		-	与RM调度器协商以获取资源（以Container表示）
		-	将得到的任务进一步分配给内部的任务
		-	与NM通信以启动/停止任务
		-	监控所有任务运行状态，并在任务失败时重新为任务申请资源以重启任务

-	NodeManager（NM）
	-	NM是每个节点上的资源和任务管理器。一方面，它定时地向RM汇报本节点的资源使用情况和Container运行状态；另一方面，它接受并处理来自AM的Container启动/停止等各种请求。

-	Container
	-	Container是YARN中的资源抽象，它封装了某个节点上的多维资源，如CPU、内存、磁盘、网络等。当AM向RM申请资源时，RM向AM返回的资源便是用Container表示的。YARN会为每个任务分配一个Container，且该任务只能使用该Container中描述的资源。Container是一个动态资源划分单位，是根据应用程序的需求自动生成的。目前，YARN仅支持CPU和内存两种资源。

####YARN工作流程
-	运行在YARN上的应用程序主要分为两类：短应用程序和长应用程序。其中，短应用程序是指一定时间内可运行完成并正常退出的应用程序，如MapReduce作业、Spark DAG作业等。长应用程序是指不出意外，永不终止运行的应用程序，通常是一些服务，比如Storm Service（包括Nimbus和Supervisor两类服务），HBase Service（包括HMaster和RegionServer两类服务）等，而它们本身作为一种框架提供编程接口供用户使用。尽管这两类应用程序作业不同，一类直接运行数据处理程序，一类用于部署服务（服务之上再运行数据处理程序），但运行在YARN上的流程是相同的。
-	当用户向YARN中提交一个应用程序后，YARN将分两个阶段运行该应用程序：第一阶段是启动ApplicationMaster。第二阶段是由ApplicationMaster创建应用程序，为它申请资源，并监控它的整个运行过程，直到运行完成。具体如下：
	-	用户向YARN中提交应用程序，其中包括ApplicationMaster程序、启动ApplicationMaster的命令、用户程序等。
	-	ResourceManager为该应用程序分配第一个Container，并与对应的NodeManager通信，要求它在这个Container中启动应用程序的ApplicationMaster。
	-	ApplicationMaster首先向ResourceManager注册，这样用户就可以直接通过ResourceManager查看应用程序的运行状态，然后它将为各个任务申请资源，并监控它的运行状态，直到运行结束，即重复步骤4~7。
	-	ApplicationMaster采用轮询的方式通过RPC协议向ResourceManager申请和领取资源。
	-	一旦ApplicationMaster申请到资源后，便与对应的NodeManager通信，要求它启动任务。
	-	NodeManager为任务设置好运行环境（包括环境变量、JAR包、二进制程序等）后，将任务启动命令写到一个脚本中，并通过运行该脚本启动任务。
	-	各个任务通过某个RPC协议向ApplicationMaster汇报自己的状态和进度，以让ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务。
	-	应用程序运行完成后，ApplicationMaster向ResourceManager注销并关闭自己。