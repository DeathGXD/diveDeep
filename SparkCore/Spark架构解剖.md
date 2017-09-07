
![image](/SparkCore/Images/Spark-Architecture.png)


### Spark核心组件  


* SparkContext  
    SparkContext是Spark程序的入口


* Driver  
    Driver是Spark程序的控制器



* DAGScheduler  
    DAGScheduler是高级调度器，是逻辑执行计划


* TaskScheduler  
    TaskScheduler是物理执行计划


* SchedulerBackend  
    SchedulerBackend是调度层和集群之间进行资源交互的桥梁


* ExecutorBackend  
    负责与TaskScheduler交互，更新任务的执行状态


* Executor  
    Executor负责具体的喏任务的执行

* Task  
    Spark中最小的执行单元
