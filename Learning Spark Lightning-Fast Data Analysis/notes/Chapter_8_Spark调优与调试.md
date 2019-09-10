# Chapter_8 Spark调优与调试

## 1. 使用SparkConf配置Spark

对Spark进行性能调优，通常就是修改Spark应用的运行时配置选项。Spark中最主要的配置机制是通过SparkConf类对Spark进行配置。当创建出一个SparkContext时，就需要创建出一个SparkConf的实例。

修改Spark配置项的方式有多种：

1. 在代码中直接配置；
2. 在执行spark-submit提交任务时通过脚本选项进行配置；
3. 在spark默认配置文件中配置（默认情况下，spark-submit脚本会在Spark安装目录中找到conf/spark-defaults.conf文件，尝试读取该文件中的键值对数据）；
上述三种方式以及Spark中默认配置的生效优先级是**依次递减**的，同一配置项在多处设置时，代码中的设置优先生效。

常用的Spark配置项的值：
|选项|默认值|描述|
|---------------------------------------------|-----------|------------------------------------------------------------------------|
|spark.executor.memory<br/>(--executor-memory)|512m       |为每个执行器进程分配的内存，格式与JVM内存字符串格式一样（例如512m，2g）。|
|spark.executor.cores<br/>(--executor-cores)  |1          |在YARN模式下，spark.executor.cores会为每个任务分配指定数目的核心。|
|spark.cores.max<br/>(--total-executor-cores) |无         |在独立模式和Mesos模式下，spark.core.max设置了所有执行器进程使用的核心总数的上限。|
|spark.speculation                            |false      |设为true时开启任务预测执行机制。当出现比较慢的任务时，这种机制会在另外的节点上也尝试执行该任务的一个副本。<br/>打开此选项会帮助减少大规模集群中个别较慢的任务带来的影响。|
spark.storage.blockManagerTimeoutIntervalMs   |45000      |内部用来通过超时机制追踪执行器进程是否存活的阈值。对于会引发长时间垃圾回收(GC)暂停的作业，需要把这个值调到100秒（对应值为100000）以上来防止失败。在Spark将来的版本中，这个配置项可能会被一个统一的超时设置所取代，所以请注意检索最新文档。|
|spark.executor.extraJavaOptions<br/>spark.executor.extraClassPath<br/>spark.executor.extraLibraryPath|（空） |这三个参数用来自定义如何启动执行器进程的JVM，分别用来添加额外的Java参数、classpath以及程序库路径。使用字符串来设置这些参数（例如 spark.executor.extraJavaOptions="- XX:+PrintGCDetailsXX:+PrintGCTimeStamps"）。请注意，虽然这些参数可以让你自行添加执行器程序的classpath，我们还是推荐使用spark-submit的--jars标记来添加依赖，而不是使用这几个选项。|
|spark.serializer|org.apache.spark.serializer.JavaSerializer|指定用来进行序列化的类库，包括通过网络传输数据或缓存数据时的序列化。默认的Java序列化对于任何可以被序列化的Java对象都适用，但是速度很慢。我们推荐在追求速度时使用org.apache.spark.serializer.KryoSerializer并且对Kryo进行适当的调优。该项可以配置为任何org.apache.spark.Serializer的子类。|
|spark.[X].port|（任意值） |用来设置运行Spark应用时用到的各个端口。这些参数对于运行在可靠网络上的集群是很有用的。有效的X包括driver、fileserver、broadcast、replClassServer、blockManager，以及 executor|
|spark.eventLog.enabled |false |设为true时，开启事件日志机制，这样已完成的Spark作业就可以通过历史服务器（history server）查看。|
|spark.eventLog.dir |file:///tmp/spark-events|指开启事件日志机制时，事件日志文件的存储位置。这个值指向的路径需要设置到一个全局可见的文件系统中，比如HDFS|
|

## 2. Spark执行的组成部分：作业、任务和步骤




