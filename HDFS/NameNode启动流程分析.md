### 一切从脚本开始  
NameNode的启动是从脚本开始的，那么就先来简单分析一些脚本。首先HDFS的服务相关的脚本都放在$HADOOP_HOME/sbin目录下，HDFS的启动脚本为start-dfs.sh。那么首先看一下脚本，脚本中NameNode启动的关键代码如下：  
```shell
NAMENODES=$($HADOOP_PREFIX/bin/hdfs getconf -namenodes)

echo "Starting namenodes on [$NAMENODES]"

"$HADOOP_PREFIX/sbin/hadoop-daemons.sh" \
  --config "$HADOOP_CONF_DIR" \
  --hostnames "$NAMENODES" \
  --script "$bin/hdfs" start namenode $nameStartOpt
```  
从上诉代码可以看到，真正启动NameNode的命令是：
```shell
"$bin/hdfs" start namenode $nameStartOpt
```   
所以我们再去看一下hdfs脚本。该脚本存放在$HADOOPHOME/bin目录下。摘录其中关于启动NameNode部分的关键代码如下：
```shell
COMMAND=$1
shift
......

```  
......  

### 有图有真相  
具体代码分析之前，我们先来看一张比较直观的NameNode的启动流程图：  
![image](/HDFS/Images/hdfs-namenode-start.png)

下面我们再具体的分析NameNode的启动的源码。  

### 一切从main函数开始  
main函数是程序的入口，NameNode也不例外。那么我们先从NameNode的main函数看起。  
```java

```
