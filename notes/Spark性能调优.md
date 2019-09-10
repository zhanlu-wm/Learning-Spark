# Spark任务并行度调优
## 1. 概念：
## 2. 配置：
## 3. 编码：






---------------------------------------------

Hadoop MapReduce

    map任务具有数据本地化优势，因而一个mapper任务通常（不是一定）被安排运行在它所负责处理的输入分片所在的节点上。
    reduce并不具有数据本地化优势，一个reducer任务的输入通常来自所有的mapper的输出。

---------------------------------------------

    可以很容易的从多个节点并行读取的格式被称为“可分割”的格式，txt格式的数据文件是可分割的，各种压缩格式的数据文件，一些是可分割的，一些是不可分割的。SparkContext.textFile()可以处理压缩过的输入，但是无法对压缩过的输入（即使该压缩格式是可分割的）进行分割读取，因而，在处理一个较大的压缩文件时，只能由单一节点读取整个压缩文件，这很容易造成性能瓶颈。因而，如果要读取单个压缩过的输入，最好不要考虑使用Spark的封装，而是使用newAPIHadoopFile或者hadoopFile，并指定正确的压缩编解码格式。

---------------------------------------------

    资源并行度：executor&core 数量

    数据并行度：数据分区数量

    Spark中的资源并行度影响因素（以Spark on YARN为例）
    1. executor-cores
    2. --num-executors(spark-submit命令选项)/spark.executor.instances(sparkConf配置项)/SPARK_EXECUTOR_INSTANCES(spark-env环境变量)
    3. spark.dynamicAllocation.initialExecutors
    * spark动态资源分配（！！！）

    Spark中的数据并行度影响因素
    1. spark.default.parallelism（即：sc.defaultParallelism）
    2. sc.defaultMinPartitions方法返回值
    3. 在Spark输入数据读取方法(如textFile)中指定的minPartitions
    4. 在Spark算子中指定的numPartitions
    5. 输入数据文件的物理分块情况

|选项|默认值|描述|
|---------------------------------------------|-----------|------------------------------------------------------------------------|
|spark.dynamicAllocation.initialExecutors	|spark.dynamicAllocation.minExecutors	|Initial number of executors to run if dynamic allocation is enabled. <br/>If `--num-executors` (or `spark.executor.instances`) is set and larger than this value, it will be used as the initial number of executors.|
|spark.default.parallelism	|For distributed shuffle operations like reduceByKey and join, the largest number of partitions in a parent RDD. For operations like parallelize with no parent RDDs, it depends on the cluster manager: <br/>Local mode: number of cores on the local machine <br/>Mesos fine grained mode: 8<br/>Others: total number of cores on all executor nodes or 2, whichever is larger|Default number of partitions in RDDs returned by transformations like join, reduceByKey, and parallelize when not set by user.|
|


    【Spark许多自带的封装好的函数（比如textFile()、sequenceFile()）都是使用旧式Hadoop API实现的】

    Spark应用程序的输入，以典型的textFile方法为例：textFile(path: String, minPartitions: Int = defaultMinPartitions): RDD[String]，minPartitions参数指定了要切分的最小输入分片数量，该参数可以使用外部传入的值，也可以默认使用defaultMinPartitions方法计算得到的值，采用后一种方式时，得到的minPartitions值最大为2。textFile方法对输入数据的切片，最终是通过调用MRv1中的FileInputFormat类的getSplits(JobConf job, int numSplits)方法实现，textFile方法的minPartitions参数值，最终传递给getSplits方法的numSplits参数，作为FileInputFormat的切片算法实现的参考值（该值只是参考值，实际返回的分片数量并不一定等于该值）。==》根据已进行的一系列测试表明，最终的逻辑切片总数不会小于输入数据文件的物理分片总数(该结论未必是绝对)，具体数量还是要依赖getSplits方法的切片算法实现。


    【概念区分】： 文件block（物理分片）和输入分片inputSplit（逻辑分片）的概念区别
    【切片算法】： mapreduce的mapper输入切片机制：MRv1与MRv2的文件切片API实现的区别：注意切片算法的实现与哪些因素或配置项有关系

---------------------------------------------

#### 不同的算子对RDD分区的影响（shuffle对RDD分区的影响）：

    map操作会导致新的RDD失去父RDD的分区信息（使新RDD的partitioner=None，map及类似的操作理论上可能会修改每条记录的键），得到的新RDD的分区数量则仍与父RDD保持一致

    reduceByKey操作会对父RDD进行重新混洗分区得到新RDD，其中父RDD的分区数量为numPartitions：

        1. 设置了spark.default.parallelism参数；
        如果父RDD没有按键进行分区（例如一个刚通过map操作生成的键值对RDD，RDD.partitioner=None），则会创建新的哈希partitioner并使用spark.default.parallelism作为新RDD的分区数量进行哈希分区；
        如果父RDD已经按照键进行分区（例如一个刚通过partitionBy操作生成的键值对RDD），则会使用父RDD的partitioner和分区数量numPartitions对新RDD进行分区
        1. 没有设置spark.default.parallelism参数；
        如果父RDD没有按键进行分区（例如一个刚通过map操作生成的键值对RDD，RDD.partitioner=None），则会创建新的哈希partitioner并使用父RDD的分区数量numPartitions作为新RDD的分区数量进行哈希分区；
        如果父RDD已经按照键进行分区（例如一个刚通过partitionBy操作生成的键值对RDD），则会使用父RDD的partitioner和分区数量numPartitions对新RDD进行分区

    join操作会对父RDD进行重新混洗分区得到新RDD：

        1. 设置了spark.default.parallelism参数；
        如果两个父键值对RDD都没有按键进行分区，则会创建新的哈希partitioner并使用spark.default.parallelism作为新RDD的分区数量进行哈希分区；
        如果其中一个父键值对RDD已经按照键进行分区，则会使用该父RDD的partitioner和分区数量对新RDD进行分区
        如果两个父键值对RDD都已经按键进行分区，则会选择分区数量较大的父RDD的partitioner和分区数量对新RDD进行分区
        1. 没有设置spark.default.parallelism参数；
        如果两个父键值对RDD都没有按键进行分区，则会创建新的哈希partitioner并使用两个父RDD的两个分区数量中较大的那个值作为新RDD的分区数量进行哈希分区；
        如果其中一个父键值对RDD已经按照键进行分区，则会使用该父RDD的partitioner和分区数量对新RDD进行分区
        如果两个父键值对RDD都已经按键进行分区，则会选择分区数量较大的父RDD的partitioner和分区数量对新RDD进行分区

    partitionBy（partitioner（num））：对键值对RDD使用指定的partitioner，分成num个分区

    repartition（num）：将RDD的数据随机打乱并分成设定的分区数目

    为保证足够的并行度，在对算子（如partitionBy）设置分区数目时，应至少保证分区数目等于应用可用的cpu核心(vcore)总数


    问题：在对Pair RDD进行分区时，以哈希分区为例，如果设置的分区数大于数据集本身所有键的可能取值数量（例如key的可能取值只有1到5五种，但指定的分区数是10），那么必然导致一部分分区数据量较多，另一部分分区数据量为0，另外通常情况下我们设置的并行度会大于可用的vcore资源数，那么实际运行时，是否意味着必然有一部分任务(task)是空转，启动后很快就结束，另一部分则要处理较多的数据，花费较多的时间？更甚者，如果数据集键的可能取值数量甚至小于分配给作业的vcore资源数量（假定没有开启动态资源分配），那么这时是否意味着将会有一部分vcore被作业占用但又处于无事可做的状态？

    问题：不开启动态资源分配时，分配给一个作业的executor，在作业负载低时，一些executor会空闲，这些空闲executor是会自动回收，还是保持空闲状态一直被占用？

##### 以下为新RDD创建时，得到默认的partitioner的方法

```scala
  /**
  * Choose a partitioner to use for a cogroup-like operation between a number of RDDs.
  *
  * If any of the RDDs already has a partitioner, choose that one.
  *
  * Otherwise, we use a default HashPartitioner. For the number of partitions, if
  * spark.default.parallelism is set, then we'll use the value from SparkContext
  * defaultParallelism, otherwise we'll use the max number of upstream partitions.
  *
  * Unless spark.default.parallelism is set, the number of partitions will be the
  * same as the number of partitions in the largest upstream RDD, as this should
  * be least likely to cause out-of-memory errors.
  *
  * We use two method parameters (rdd, others) to enforce callers passing at least 1 RDD.
  */
  def defaultPartitioner(rdd: RDD[_], others: RDD[_]*): Partitioner = {
    val rdds = (Seq(rdd) ++ others)
    val hasPartitioner = rdds.filter(_.partitioner.exists(_.numPartitions > 0))
    if (hasPartitioner.nonEmpty) {
      hasPartitioner.maxBy(_.partitions.length).partitioner.get
    } else {
      if (rdd.context.conf.contains("spark.default.parallelism")) {
        new HashPartitioner(rdd.context.defaultParallelism)
      } else {
        new HashPartitioner(rdds.map(_.partitions.length).max)
      }
    }
  }
```

---------------------------------------------

Spark的shuffle机制

不同算子的shuffle过程

---------------------------------------------

数据倾斜

---------------------------------------------
