在MapReduce中，一个YARN的应用被称为一个Job。MapReduce提供的Application Master实现被称为MRAppMaster。YARN相关的概念请自行补习。  

上图展示了MapReduce任务执行的时间轴:  
* Map阶段：多个Map任务被执行
* Reduce阶段：多个Reduce任务被执行  

注意：Reduce阶段可能在Map阶段结束之前就被执行。因此，Map任务和Reduce任务之间可能存在交叉。  

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

注意：map split是一块接着一块的被获取(例如：64KB)并且每个块会被分割成多个(key, value) tuple(例如：使用SplitLineReader.class).这个动作是在Mapper.Context.nextKeyValue方法内完成的。  

当map split都被处理完成了，run方法内部会调用clean方法，默认情况下，clean内没有任何动作会被执行，但是用户可以决定是否需要重写这个方法。  

#### MapTask：SPILLING  
就如EXECUTING阶段所看见的一样，map会将它的ouput写入到一个环形内存缓冲区(MapTask.MapOutputBuffer).缓冲区的大小是确定的，由配置参数ma pmapreduce.task.io.sort.mb决定(默认是100MB)。  

每当这个环形内存缓冲区差不多要满的时候(默认：mapreduce.map.sort.spill.percent 80%), SPILLING阶段就会被执行(使用一个独立的线程并行的执行)。如果spilling线程很慢并且缓冲区被装满达到了100%，那么map()任务将不会被执行并且不得不等待。  

SPILLING线程会执行下面的动作：
1. 它会创建一个SpillReader和FSOutputStream(本地文件系统)
2. 在内存中对缓冲区中使用的块进行排序：输出的tuples通过(partitionIdx, key)使用快速排序算法进行排序
3. 排序后的输出会以分区为单位进行分割：job中的每个Reduce任务对应着一个分区
4. 分割后的分区数据会被连续的写入到本地文件中  

#### 有多少Reduce任务？  
job中的Reduce任务的数量是由mapreduce.job.reduces配置参数决定的。  

#### 与输出tuple相关联的partitionIdx是什么？
一个输出tuple的partitionIdx是一个分区的索引。它是由Mapper.Context.write()确定的：  
```java
partitionIdx = (key.hashCode() & Integer.MAX_VALUE) % numReducers
```  
它是作为输出tuple的元数据与输出tuple一起被存储在环形内存缓冲区中。用户可以自定义分区器通过设置mapreduce.job.partitioner.class参数。  
#### 我们何时使用combiner？
如果用户指定了一个combiner，那么SPILLING线程在将输出tuple写入到文件中之前，会先在tuple中的每个分区执行combiner。主要会进行如下操作：  
1. 创建一个用户的Reduce.class实例(一个特定的实例用来combiner)
2. 创建一个Reduce.Context：输出文件将会被存储在本地文件系统
3. 执行Reduce.run()  

combiner使用的是与标准的reduce函数相同的实现，因此可以将combiner看做是本地的reducer(map端的reduce)。  

### MapTask：执行结束
在执行阶段的结束，SPILLING线程最后将会被触发。更多细节：  
1. 排序和分割剩下的没有被spill的tuple
2. 开始SHUFFLE阶段  

注意：每次缓冲区几乎要满的时候，我们会生成一个spill文件(SpillRecord + 输出文件)。每个spill文件会包含多个分区(segments)。  

#### MapTask：SHUFFLE

### Reduce阶段  
#### ReduceTask启动  
MRAppMaster等到mapreduce.job.reduce.slowstart.completedmaps(5%)MapTask完成之后，那么ReduceTask会定期的执行:  


### YARN与MapReduce交互
