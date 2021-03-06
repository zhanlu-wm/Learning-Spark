# Chapter_5 数据读取与保存

    Spark支持多种输入输出源。Spark本身基于Hadoop生态圈构建，可以通过Hadoop MapReduce所使用的InputFormat和OutputFormat接口访问数据。
    本章会介绍三类常见的数据源：

*   文件格式与文件系统

        对于存储在本地文件系统或分布式文件系统中的数据，Spark可以访问多种不同的文件格式： 文本文件、JSON、SequenceFile、protocol buffer等

*   Spark SQL中的结构化数据源

        Spark SQL模块，它针对包括JSON和Apache Hive在内的结构化数据源

*   数据库与键值存储

        JDBC源、Hbase等

## 1. 文件格式

        Spark支持的一些常见格式：
        -------------------------------------------------------------------------------------------
        格式名称                结构化               备注
        -------------------------------------------------------------------------------------------
        文本文件                否                  普通的文本文件，每行一条记录
        JSON                   半结构化             常见的基于文本的格式，半结构化；大多数库都要求每行一条记录
        CSV                    是                  非常常见的基于文本的格式，通常在电子表格应用中使用
        SequenceFiles          是                  一种用于键值对数据的常见 Hadoop 文件格式
        Protocol buffers       是                  一种快速、节约空间的跨语言格式
        对象文件                是                  用来将 Spark 作业中的数据存储下来以让共享的代码读取。
                                                  改变类的时候它会失效，因为它依赖于Java序列化。
        -------------------------------------------------------------------------------------------

#### 1. 文本文件

*   读取文本文件

        def textFile(path: String, minPartitions: Int): RDD[String]
        def wholeTextFiles(path: String, minPartitions: Int): RDD[(String, String)]

        path：该参数指定了输入文件的路径，可以是一个明确的文件或者目录，也可以是一个文件路径通配符（如 part-*.txt）
        minPartitions：该参数用来控制分区数(输入分片数)，它作为Spark进行输入切片时的参考值，实际得到的结果RDD的分区数不一定等于该值。
        wholeTextFiles：该方法会返回一个pairRDD，其中键是输入文件的文件名。当需要知道数据的各部分分别来自哪个文件时，或者需要对每个输入文件整体处理时，如果文件足够小，我们可以使用该方法读取输入数据。

*   保存文本文件

        def saveAsTextFile(path: String): Unit

        该方法接收一个路径，并将RDD中的内容都输入到路径对应的文件中。Spark将传入的路径作为目录对待，会在那个目录下输出多个文件。目录由Spark自动创建，事先不能存在。

#### 2. JSON

#### 3. 逗号分隔值与制表符分隔值

#### 4. SequenceFile

#### 5. 对象文件

#### 6. Hadoop输入输出格式

        def hadoopFile[K, V](
            path: String,
            inputFormatClass: Class[_ <: InputFormat[K, V]],
            keyClass: Class[K],
            valueClass: Class[V],
            minPartitions: Int = defaultMinPartitions): RDD[(K, V)] 

        def newAPIHadoopFile[K, V, F <: NewInputFormat[K, V]](
            path: String,
            fClass: Class[F],
            kClass: Class[K],
            vClass: Class[V],
            conf: Configuration = hadoopConfiguration): RDD[(K, V)]

#### 7. 文件压缩

    可以很容易的从多个节点并行读取的格式被称为“可分割”的格式，txt格式的数据文件是可分割的，各种压缩格式的数据文件，一些是可分割的，一些是不可分割的。SparkContext.textFile()可以处理压缩过的输入，但是无法对压缩过的输入（即使该压缩格式是可分割的）进行分割读取，因而，在处理一个较大的压缩文件时，只能由单一节点读取整个压缩文件，这很容易造成性能瓶颈。因而，如果要读取单个压缩过的输入，最好不要考虑使用Spark的封装，而是使用newAPIHadoopFile或者hadoopFile，并指定正确的压缩编解码格式。

## 2. 文件系统

## 3. Spark SQL中结构化数据

## 4. 数据库
