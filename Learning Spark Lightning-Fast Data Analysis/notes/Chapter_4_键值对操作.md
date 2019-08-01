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
    分区并不总是有好处的，如果给定的RDD只需要被扫描一次，那么我们完全没有必要事先对其进行分区处理。只有当数据集多次在诸如连接这种基于键的操作中使用时，分区才会有帮助。
    Spark中所有键值对RDD都可以进行分区，系统会根据一个针对键的函数对元素进行分组，Spark可以确保属于同一组的键出现在同一个节点上，例如，可以使用哈希分区将一个RDD分成100个分区，此时，键的哈希值对100取模的结果相同（即，属于同一组）的记录被放在一个节点上。此外也可以使用范围分区法，将键在同一范围的记录放在一个节点上。

    关于合理分区的优势，以join操作为例，默认情况下，连接操作会将两个RDD中的所有键的哈希值求出来，将哈希值相同的数据通过网络传输到同一台机器上，然后在那台机器上对所有的键相同的记录进行连接操作，该过程是一个shuffle过程，开销较大，如果参与连接的两个RDD没有事先按键进行分区，那么后续每次使用这两个RDD进行连接操作时，都需要重新执行分区过程，对两个RDD的数据都需要进行跨节点的shuffle操作，产生较多的网络数据传输，从而带来较大的性能消耗，相反，如果参与连接的两个RDD中的一个已事先按键进行了分区，那么后续再次使用两个RDD进行连接操作时，就可以直接采用已有的分区结构，只对另一个RDD的数据进行跨节点的shuffle操作，从而减少网络数据传输，降低开销。

#### 1. 获取RDD的分区方式

    可以使用RDD的partitioner属性来获取RDD的分区方式：
    它返回一个scala.Option对象，可以通过get()方法来获取其中的值，得到一个scala.Partitioner对象。
    在Spark shell中使用partitioner属性不仅是检验各种Spark操作如何影响分区方式的一种好办法，还可以用来在你的程序中检查想要使用的操作是否会生成正确的结果。

#### 2. 从RDD分区获益的操作

    Spark的许多操作都引入了将数据根据键跨节点进行混洗的过程，所有这些操作都能从数据分区机制中获益，就Spark1.0而言，能够从数据分区中获益的操作有：
    cogroup()、groupWith()、join()、leftOuterJoin()、rightOuterJoin()、groupByKey()、reduceByKey()、combineByKey()、lookup()等

    对于像 reduceByKey() 这样只作用于单个 RDD 的操作，运行在未分区的 RDD 上的时候会导致每个键的所有对应值都在每台机器上进行本地计算，只需要把本地最终归约出的结果值从各工作节点传回主节点， 所以原本的网络开销就不算大。而对于诸如 cogroup() 和 join() 这样的二元操作，预先进行数据分区会导致其中至少一个 RDD（使用已知分区器的那个 RDD）不发生数据混洗。如果两个 RDD 使用同样的分区方式， 并且它们还缓存在同样的机器上（比如一个 RDD 是通过 mapValues() 从另一个 RDD 中创建出来的，这两个RDD 就会拥有相同的键和分区方式），那么跨节点的数据混洗就不会发生了。

#### 3. 影响分区方式的操作

    Spark内部知道各种操作会如何影响分区方式，并将 会对数据进行分区的操作的 结果RDD自动设置为对应的分区器。
    转化操作的结果也可能不会按已知的分区方式分区，这时输出的RDD可能就会没有设置分区器。当对一个哈希分区的键值对RDD调用map操作时，得到的结果RDD中就不会设置有分区器（RDD.partitioner=None）。
    Spark提供了另外两个操作mapValues()和flatMapValues()作为在键值对RDD上进行map操作的替代方法，它们可以保证每个二元组的键保持不变。

    下面列出了所有会为生成的结果RDD设好分区方式的操作：
    cogroup()、groupWith()、join()、leftOuterJoin()、
    rightOuterJoin()、groupByKey()、reduceByKey()、
    combineByKey()、partitionBy()、sort()、
    mapValues()（如果父RDD有分区方式的话）、
    flatMapValues()（如果父RDD有分区方式的话），
    filter()（如果父RDD有分区方式的话）。
    其他所有的操作生成的结果都不会存在特定的分区方式。

    对一个RDD执行partitionBy操作后得到的结果RDD，如果后续需要复用，那么应该对该结果RDD调用persist()方法进行缓存，避免重新对原始RDD执行分区操作。

    最后，对于二元操作，输出数据的分区方式取决于父RDD的分区方式。 ...略... 【？？？？？？】

#### 4. 自定义分区方式

    Spark提供的HashPartitioner和RangePartitioner已经能够满足大多数应用场景，但Spark也运行我们提供一个自定义的Partitioner对象来控制RDD的分区方式。
    
    要实现自定义的Partitioner，需要继承org.apache.spark.Partitioner类并实现以下三个方法：
    * numPartitions: Int ：返回创建出的分区数；
    * getPartition(key: Any): Int ：返回给定键对应的分区编号（0到numPartitions-1）（注意保证该方法的实现永远返回一个非负数）；
    * equals() ：Java判断相等性的标准方法。Spark需要用这个方法来检查你的分区器对象是否和其他分区器实例相同，这样Spark才可以判断两个RDD的分区方式是否相同。
