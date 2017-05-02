在MapReduce中，一个YARN的应用被称为一个Job。MapReduce提供的Application Master实现被称为MRAppMaster。YARN相关的概念请自行补习。  

上图展示了MapReduce任务执行的时间轴:  
* Map阶段：多个Map任务被执行
* Reduce阶段：多个Reduce任务被执行  

注意：Reduce阶段可能在Map阶段结束之前就被执行。因此，Map任何和Reduce任务之间可能存在交叉。  

### Map阶段  
我们现在将我们讨论的焦点放在Map阶段。对此，一个关键的决策是Application Master需要给当前的任务启动多少个MapTask。  

#### 用户给了我们什么？  

让我们后退一步。当一个client提交应用的时候，多种信息被提供给YARN的底层。特别：  
* 一个配置：这个可能只是部分参数(一些参数可能没有被用户指定)，假设是这样的话，job将会使用默认值。注意：那些默认值可能是Hadoop提供者的一个选择，就像Amanzon
* 一个JAR文件：  
    * 一个map()实现  
    * 一个combiner实现  
    * 一个reduce()实现
* input和output信息：  
    * input目录：是在HDFS上的目录？还是S3？有多少文件？  
    * output目录：我们将output存放在哪？在HDFS还是S3？  

input目录内部包含的文件的数量是用来决定一个任务中Map Tasks的数量的。

#### 有多少Map Task？  

Application Master将会为每一个map split运行一个Map Task。通常，每个input file都会有一个map split。如果input file很大(比HDFS的block
size大)，那么我们会有两个或者更多的map split关联到相同的input file。下面是FileInputSplit类内部方法getSplits()的伪代码：  
```java
num_splits = 0
for each input file f:
   remaining = f.length
   while remaining / split_size > split_slope:
      num_splits += 1
      remaining -= split_size
```  
where:  
```java
split_slope = 1.1
split_size =~ dfs.blocksize
```  

注意：配置参数mapreduce.job.maps在MRv2中已经被忽略(在过去，它仅仅是一个提示)。  

#### MapTask运行  

MapReduce任务的Application Master向ResourceManager请求Job需要的containers：每个MapTask(每个map split)需要一个MapTask container。  

MapTask请求的container尝试去利用本地的map split，也就是说尝试在map split所在的节点启动container。 Application Master请求：  
* container启动的NodeManager节点与map split存储的DataNode节点位于同一台机器(由于HDFS的备份策略，一个map split可能会被存储在多个节点上)  
* 否则，container启动的NodeManager节点与map split存储的DataNode节点位于相同机架
* 否则，container启动在集群任何的NodeManager节点  

这仅仅是对Resource Manager的一个暗示。如果建议的分配与Resource Manager的目标冲突，那么Resource Manager将会自由的忽略数据本地性。  

当一个Container分配给Application Master之后，MapTask就会被运行。  

#### Map阶段：执行场景的例子  

下面是一个Map阶段可能的执行场景：  
* 有两个NodeManager：每个NodeManager有2GB的RAM(NM的容量)，并且每个MapTask要求1GB，我们可以并行的在每个NodeManager运行2个container(这是最好的场景，但是ResourceManager可能有不同的决定)  
* 没有其他的YARN应用运行在集群上  
* 我们的job有8个map split(比如：在input目录中有7个文件，但是仅仅其中一个比HDFS block size要大，但是比两个block要小，所以我们要将大的文件切分成两个map split)，我们需要运行8个Map Tasks。  

#### Map Task执行时间轴  

现在让我们关注在一个单独的Map Task。下面是一个Map Task的执行时间轴：
* INIT阶段：设置Map Task
* EXECUTION阶段：对于map split内部的每一个(key, value) tuple(key是文件偏移量，value为每一行的内容)，都会执行map()函数
* SPILLING阶段：map阶段的output会被存储在内存缓冲区，当缓冲区几乎满了的时候(默认是80%)，那么我们会并行的运行spilling阶段，为了删除缓冲区的数据
* SHUFFLE阶段：在spilling阶段结束后，我们会合并所有的map任务的output文件，并且打包给reduce阶段  

#### MapTask：INIT  

在Map任务的INIT阶段，我们会：  
1. 创建一个上下文(TaskAttemptContext.class)  
2. 创建一个用户实现的Map实例(Mapper.class)  
3. 设置输入(比如：InputFormat.class, InputSplit.class, RecordReader.class)  
4. 设置输出(NewOutputCollector.class)  
5. 创建一个mapper上下文(MapContext.class, Mapper.Context.class)   
6. 实例化输入  
7. 创建一个SplitLineReader.class对象  
8. 创建一个HdfsDataInputStream.class对象  

#### MapTask：EXECUTION  

map任务的执行阶段通过Mapper类的run方法执行。用户可以去重写它，但是默认情况下它将通过调用setup方法开始，setup方法默认情况下不会做一些有用的事，但是用户可以为了设置Task而去重写它(比如：初始化类的变量)。在setup方法被调用之后，map split文件中的每一个(key, value) tuple都会触发map()方法的执行。因此，map()方法接受一个key、一个value和一个mapper context作为参数。使用context，map任务会存储它的输出到buffer中。  

注意：map split


#### MapTask：SPILLING  
