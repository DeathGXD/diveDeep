RDD是一个弹性的分布式数据集，代表了一个不可变的，分区的，可以被并行操作的元素集合，Spark中基本的数据抽象。**本质上说，RDD就是一个只读的，分区记录的集合**。这句话的意思是：1.首先RDD是可以看做是一个集合。2.RDD是只读的，不可修改。3.RDD中存放的是数据的分区地址，比如访问的是HDFS上的数据，那么RDD中存放的就是数据在HDFS上对应的每一个block的地址，每一个block对应一个分区地址。  

RDD是弹性的，指的是由于RDD lineage graph的帮组，RDD具有容错性并且能够重新计算由于节点失败造成丢失的或者损坏的分区。  
RDD是分区的，指的是RDD中的数据分布在集群中的多个节点中。  
RDD是数据集，指的是RDD是分区数据的集合。  

除了以上明显的特性之外，RDD还有下面一些额外的特性：  
* 存储于内存：RDD会尽可能长时间的存储于内存
* 不可变(只读)：RDD一旦被创建，就不可以被改变，只可以通过转换操作创建出新的RDD
* 懒加载：RDD内部的数据是不可用或者不可以被转换，除非一个action操作被触发
* 可缓存：你可以将RDD都存放在持久化存储中，比如内存，磁盘等
* 并行：可以并行的处理数据
* 类型化：RDD中的记录具有类型，比如：RDD[Long]，RDD[String]
* 分区：RDD中的数据是被分区的，分割成多个逻辑分区，并且分布在集群中的多个节点中
* 位置灵活：RDD可以定义放置策略来计算分区  

RDD主要包含五个特性：</br>
>  1.RDD的分区列表</br>
>  2.RDD的优先位置列表</br>
>  3.RDD之间的依赖关系列表</br>
>  4.RDD的分区计算函数</br>
>  5.RDD的分区函数，只对key-value RDD</br>

下面通过源码具体分析RDD五个特性的原理：

###  1.RDD的分区列表  
![Alt text](/Spark/Images/RDD.png)  

RDD是一个分区的数据集合，倒不如说RDD就是一个包含多个数据分区记录的集合。由上图可以看出，一个RDD包含多个分区。再看RDD源码：  
```scala
protected def getPartitions: Array[Partition]  
```  

RDD分区的多少涉及到对这个RDD计算的并行度。所以RDD的分区是非常重要的。RDD中一个分区对应着一个inputSplit或者说是一个block，所以RDD的分区个数只与数据的block的个数相关。我们可以查看源码：  
```scala
/**
* Read a text file from HDFS, a local file system (available on all nodes), or any
* Hadoop-supported file system URI, and return it as an RDD of Strings.
*/
def textFile(
  path: String,
  minPartitions: Int = defaultMinPartitions): RDD[String] = withScope {
  assertNotStopped()
  hadoopFile(path, classOf[TextInputFormat], classOf[LongWritable], classOf[Text],
  minPartitions).map(pair => pair._2.toString).setName(path)
}
```  
如果不指定RDD的分区数，defaultMinPartitions就是RDD的默认分区，而defaultMinPartitions是多少呢？  
```scala
/**
* Default min number of partitions for Hadoop RDDs when not given by user
* Notice that we use math.min so the "defaultMinPartitions" cannot be higher than 2.
* The reasons for this are discussed in https://github.com/mesos/spark/pull/718
*/
def defaultMinPartitions: Int = math.min(defaultParallelism, 2)
```  
 从上面源码可以看出，RDD的默认分区为defaultParallelism和2的最小值，也就是说RDD的默认分区不允许超过2。从注释可以看出在(https://github.com/mesos/spark/pull/718)讨论了这么设计的原因。那么defaultParallelism的值是从哪里来的呢？继续跟进源码  
```scala
/** Default level of parallelism to use when not given by user (e.g. parallelize and makeRDD). */
def defaultParallelism: Int = {
  assertNotStopped()
  taskScheduler.defaultParallelism
}
```  
defaultParallelism由taskScheduler.defaultParallelism决定，是由TaskScheduler的defaultParallelism方法决定，而TaskScheduler是一个trait，所以我们需要看TaskScheduler的实现类TaskSchedulerImpl的defaultParallelism方法，具体如下：  
```scala
override def defaultParallelism(): Int = backend.defaultParallelism()
```  
由上可以看出backend应该是SchedulerBackend，而SchedulerBackend也是一个trait，我们也要去它的实现类查看CoarseGrainedSchedulerBackend中查看defaultParallelism方法，具体如下：  
```scala
override def defaultParallelism(): Int = {
conf.getInt("spark.default.parallelism", math.max(totalCoreCount.get(), 2))
}
```  

OK，到现在为止终于看到设置defaultParallelism的参数了，是spark.default.parallelism，如果没有设置spark.default.parallelism，那么取当时executor所在机器的CPU核数与2的最大值，一般都应该大于2才对。  

从上面代码看出不管是取CPU核数，还是设置spark.default.parallelism参数，最后在SparkContext内部都会设置为不大于2的defaultMinPartitions,可以在显示的指定RDD分区数，进而覆盖defaultMinPartitions的值。那么显示指定的分区数真的有用吗？我们继续查看RDD的getPartitions，由于RDD是一个抽象类，所以我们要去实现类中查看，这里我们查看HadoopRDD。HadoopRDD的getPartitions方法实现如下:  
```scala
override def getPartitions: Array[Partition] = {
  val jobConf = getJobConf()
  // add the credentials here as this can be called before SparkContext initialized
  SparkHadoopUtil.get.addCredentials(jobConf)
  val inputFormat = getInputFormat(jobConf)

  // 获得一个包含inputSplits的数组，从而通过inputSplits.size获得有多少个inputSplits
  // 根据一个inputSplits对应一个partition来说，进而得知RDD中有多少个partition
  val inputSplits = inputFormat.getSplits(jobConf, minPartitions)

  // 通过该行代码可以看出，一个inputSplits对应着一个partition
  val array = new Array[Partition](inputSplits.size)
  for (i <- 0 until inputSplits.size) {
    array(i) = new HadoopPartition(id, i, inputSplits(i))
  }
  array
}
```  
由上诉代码可以看出，RDD的分区数是由inputSplit或者说是block个数决定。一般不指定RDD的分区，RDD的分区数就是block的个数。若是指定了分区数，如果指定的分区数大于或者等于block数，那么以指定的分区个数为准。但是，多出来的分区记录为空，没有数据。若是指定的分区记录小于block个数，那么与block个数为准。因此得出结论，RDD的分区个数只与block
相关。具体为何会有空的inputSplit，可以看一下inputFormat.getSplits(jobConf, minPartitions)的具体实现，后续再分析。  
  
###  2.RDD的优先位置列表  
RDD的优先位置就是RDD分区对应的的block的位置，该位置影响Spark任务调度的位置。RDD优先位置函数是getPreferredLocations(split: Partition): Seq[String]，返回的是一个列表，里面包含了inputsplit所在位置的hostname，如果列表为空，说明inputsplit在所有节点的磁盘上都有数据。getPreferredLocations底层是调用了InputSplit.getLocations研究源码  

###  3.RDD之间的依赖关系列表  
RDD之间的依赖主要有两种，窄依赖和宽依赖。RDD窄依赖指的子RDD中的Partition和父RDD中的Partition是一对一的关系，RDD宽依赖指的是子RDD中的Partition与父RDD中的Partition是多对一的关系。  

###  4.RDD的分区计算函数  


###  5.RDD的分区函数  











RDD的分析到这里就这里就结束了!!!  
