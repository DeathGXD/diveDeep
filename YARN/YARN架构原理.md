在YARN之前，Hadoop有JobTracker进程和TaskTracker进程。其中JobTracker负责资源的调度，任务的管理和监控，包括处理失败的任务，任务的记录等。TaskTacker负责具体的任务运行。基于JobTracker这种方法有很多弊端，比如说，1.可扩展性瓶颈，集群扩展到4000+节点就已经成为上限。2.集群资源共享和分配过于灵活，基于slot(槽)方法，每个节点上的槽的个数固定，不管集群有多大或者多小的任务，造成了资源的利用率低下。正因为Hadoop 1.0中MapReduce的种种问题，进而催生了YARN。  

YARN，全称Yet Another Resource Negotiator，另一种资源协调者。针对Hadoop 1.0中MapReduce中诸多弊端，尤其是JobTracker的单点故障，负责资源的管理，任务的调度等多职责的压力，以及MapReduce集群资源利用率低，造成大量资源浪费等问题，直接造成了对Hadoop架构的重写，将Hadoop 1.0中MapReduce中对资源的管理和任务的调度剥离了出来，诞生了一个全新的资源了管理框架YARN。YARN的诞生还有Hadoop 1.0存在一个问题，就是Hadoop 1.0中只能运行MapReduce任务，不能运行其他分布式应用，  

YARN的核心思想是将资源管理和任务的调度/监控分离出来，成为一个独立的进程。具体的做法就是有一个全局的ResourceManager进程，负责整个集群的资源管理，每个应用一个ApplicationMaster，负责每个应用的调度和监控，和每个节点上一个NodeManager进程，负责管理每个节点上的资源和具体的任务运行。这是主要的架构设计，下面会介绍具体的架构细节。  

首先我们看一张Hadoop官网给出的一张YARN架构图：  
![image](/YARN/Images/yarn-architecture.png)  

从图中我们看出了YARN同样遵循了主从架构设计(Master/Slave)，ResourceManager是主节点，NodeManager是从节点。图中出现的Container代表了资源的抽象，比如内存，CPU，磁盘等。一个Container中包含了固定数量的内存，CPU等资源，也可以认为是资源的集合。所以申请资源的本质就是启动Container，然后由Container执行具体任务。ApplicationMaster也是一个特有的Container。YARN简单的运行流程是：客户端提交应用到YARN集群，首先会与ResourceManager进行通信，获取一个application，然后再到NodeManager上启动一个ApplicationMaster(简称AM)，AM启动成功之后，AM会与ResourceManager进行通信，申请运行任务的资源，申请成功之后，会到NodeManager上启动对应的container，负责任务的执行。

















### ResourceManager  
ResourceManager有两个核心的组件：调度器(Scheduler)和应用管理器(ApplicationManager)。




### NodeManager  









### ApplicationMaster  




### Container  
