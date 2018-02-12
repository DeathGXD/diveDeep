Job是什么？具体如何使用的？为什么这么用？源码的具体分析。  


Spark中的job是提交给DAGScheduler的最顶层的工作层，用于计算一个action操作的结果，也就是说一个action对应着一个Job。job和application的区别是，application是用户通过Spark的API编写的应用，一个application中可以包含多个action，也就是说一个application可以对应多个job。  


计算一个job也就是计算通过action提交执行的RDD上的分区。一个job中的分区数依赖于stage的类型：ResultStage或者ShuffleMapStage。job开始于一个单一目标的RDD，也就是没有父RDD。但是该RDD最终是可以包含目标血缘图中的其它的RDD。  


父stage是ShffuleMapStage的实例，也就是说父stage只可能是ShuffleMapStage，ResultStage是一个job最后的stage。  

在Spark内部，job是用ActiveJob的实例来表示的。job有两种类型：
Map-stage job：在downstream阶段被提交之前，为ShuffleMapStage计算map输出文件
Result job：计算job中的ResultStage

job通过使用finished字段，来跟踪已经计算完成的分区  

job的实例类ActiveJob非常简单，下面来看一下ActiveJob的代码：  
```scala
private[spark] class ActiveJob(
    val jobId: Int,     // Job的唯一标识符
    val finalStage: Stage,    //
    val callSite: CallSite,
    val listener: JobListener,
    val properties: Properties) {

  /**
   * Number of partitions we need to compute for this job. Note that result stages may not need
   * to compute all partitions in their target RDD, for actions like first() and lookup().
   */
  val numPartitions = finalStage match {
    case r: ResultStage => r.partitions.length
    case m: ShuffleMapStage => m.rdd.partitions.length
  }

  /** Which partitions of the stage have finished */
  val finished = Array.fill[Boolean](numPartitions)(false)

  var numFinished = 0
}
```  
