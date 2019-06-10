# Chapter_4 键值对操作
    Spark为包含键值对类型的RDD提供了一些专有操作，这类RDD称为Pair RDD。
## 1. 创建Pair RDD
    Spark中创建Pair RDD的方式有多种：
    （1）很多存储键值对的数据格式在读取时会直接返回由其键值对数据组成的Pair RDD；
    （2）基于已有的RDD可以通过调用map()函数经过一定转化得到Pair RDD，传给map()的函数需要返回键值对。

## 2. Pair RDD的转化操作
    Pair RDD可以使用所有标准RDD上可用的转化操作。
    Pair RDD上专用的一些转化操作如下：
        reduceByKey
        groupByKey
        combineByKey
        mapValues
        flatMapValues
        keys
        values
        sortByKey
    针对两个Pair RDD的转化操作如下：
        subtractByKey
        join
        rightOuterJoin
        leftOuterJoin
        cogroup
### 1. 聚合操作
    reduceByKey
    foldByKey
    combineByKey(原理??????)

    有很多函数可以进行基于键的数据合并。它们中的大多数都是在combineByKey()的基础上实现的，为用户提供了更简单的接口。在Spark中使用这些专用的聚合函数，键值对操作始终要比手动将数据分组(groupByKey???)再归约快很多。
#### 并行度调优
    每个RDD都有固定数目的分区，分区决定了在RDD上执行操作时的并行度。
    执行聚合或分组操作时，Spark默认会根据集群的大小推断出一个有意义的分区数（如何推断？？？），但我们也可以要求Spark使用指定的分区数，以便帮我们获取更好的性能表现。
    前面所述的大多数转化操作函数都能接收第二个参数，用来指定分组结果或聚合结果的RDD分区数。
    有时，我们希望在除分组操作和聚合操作之外的操作中也能改变RDD的分区，对此，Spark提供了reparation()函数，它会把数据通过网络进行混洗，并创建出新的分区集合。对数据进行重新分区时代价相对比较大的操作，Spark也提供了一个优化版的reparation()，叫做coalesce(),可以使用Java或Scala中的rdd.partitions.size()以及Python中的rdd.getNumPartitions查看RDD的分区数，并确保调用coalesce()时将RDD合并到比现在的分区数更少的分区中。

### 2. 数据分组
    对单个RDD数据分组：
        groupByKey
            RDD[K, V] => RDD[K, Iterable[V]]
        groypBy
            可以用于未成对的数据上，也可以根据除键相同以外的条件进行分组。它可以接收一个函数，对源RDD中的每个元素使用该函数，将返回结果作为键再进行分组。
    对多个共享同一个键的RDD分组：
        cogroup
            RDD[K, V], RDD[K, W] => RDD[K, (Iterable[V], Iterable[W])]
            cogroup()提供了为多个RDD进行数据分组的方法。cogroup()不仅可以用于实现连接操作，还可以用来求键的交集。除此之外，cogroup()还能同时应用于三个及以上的RDD。

### 3. 连接
    join
    rightOuterJoin
    leftOuterJoin

### 4. 数据排序
    sortByKey
        接收一个叫作ascending的参数，表示我们是否想要让结果按升序排序（默认值为true）。有时我们也可以提供一个自定义的比较函数，实现完全自定义的排序依据。

## 3. Pair RDD的行动操作
    countByKey
    collectAsMap
    lookup

## 4. 数据分区（进阶）
    本节讨论的Spark特性是对数据集在节点间的分区进行控制。
    分布式程序中，通信的代价是很大的，因此控制数据分布以获得最少的网络传输可以极大的提升整体性能。Spark程序可以通过控制RDD分区方式来减少通信开销。





 