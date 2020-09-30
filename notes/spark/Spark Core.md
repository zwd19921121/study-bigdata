# Spark Core

## 1 RDD概述

### 1.1 什么是 RDD

RDD（Resilient Distributed Dataset）叫做弹性分布式数据集，是Spark中最基本的数据抽象。

在代码中是一个抽象类，它代表一个弹性的、不可变、可分区、里面的元素可并行计算的集合。

### 1.2 RDD 的 5 个主要属性(property)

* A list of partitions

​       多个分区. 分区可以看成是数据集的基本组成单位.

​       对于 RDD 来说, 每个分区都会被一个计算任务处理, 并决定了并行计算的粒度.

​       用户可以在创建 RDD 时指定 RDD 的分区数, 如果没有指定, 那么就会采用默认值. 默认值就是程序所分配到的 CPU Coure 的数目.

​       每个分配的存储是由BlockManager 实现的. 每个分区都会被逻辑映射成 BlockManager 的一个 Block, 而这个 Block 会被一个 Task 		负责计算.

* A function for computing each split

​       计算每个切片(分区)的函数.

​       Spark 中 RDD 的计算是以分片为单位的, 每个 RDD 都会实现 compute 函数以达到这个目的.

* A list of dependencies on other RDDs

​       与其他 RDD 之间的依赖关系

​       RDD 的每次转换都会生成一个新的 RDD, 所以 RDD 之间会形成类似于流水线一样的前后依赖关系. 在部分分区数据丢失时, Spark 可		以通过这个依赖关系重新计算丢失的分区数据, 而不是对 RDD 的所有分区进行重新计算.

* Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)

​       对存储键值对的 RDD, 还有一个可选的分区器.

​       只有对于 key-value的 RDD, 才会有 Partitioner, 非key-value的 RDD 的 Partitioner 的值是 None. Partitiner 不但决定了 RDD 的本区	   数量, 也决定了 parent RDD Shuffle 输出时的分区数量.

* Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)

​       存储每个切片优先(preferred location)位置的列表. 比如对于一个 HDFS 文件来说, 这个列表保存的就是每个 Partition 所在文件块的	   位置. 按照“移动数据不如移动计算”的理念, Spark 在进行任务调度的时候, 会尽可能地将计算任务分配到其所要处理数据块的存储位置.

### 1.3 理解RDD

一个 RDD 可以简单的理解为一个分布式的元素集合.

RDD 表示只读的分区的数据集，对 RDD 进行改动，只能通过 RDD 的转换操作, 然后得到新的 RDD, 并不会对原 RDD 有任何的影响

在 Spark 中, 所有的工作要么是创建 RDD, 要么是转换已经存在 RDD 成为新的 RDD, 要么在 RDD 上去执行一些操作来得到一些计算结果.

每个 RDD 被切分成多个分区(partition), 每个分区可能会在集群中不同的节点上进行计算.

#### 1.3.1 RDD 特点

##### 1.3.1.1 弹性

* 存储的弹性：内存与磁盘的自动切换；

* 容错的弹性：数据丢失可以自动恢复；

* 计算的弹性：计算出错重试机制；

* 分片的弹性：可根据需要重新分片。

##### 1.3.1.2 分区

RDD 逻辑上是分区的，每个分区的数据是抽象存在的，计算的时候会通过一个compute函数得到每个分区的数据。

如果 RDD 是通过已有的文件系统构建，则compute函数是读取指定文件系统中的数据，如果 RDD 是通过其他 RDD 转换而来，则 compute函数是执行转换逻辑将其他 RDD 的数据进行转换。

##### 1.3.1.3   只读

RDD 是只读的，要想改变 RDD 中的数据，只能在现有 RDD 基础上创建新的 RDD。

由一个 RDD 转换到另一个 RDD，可以通过丰富的转换算子实现，不再像 MapReduce 那样只能写map和reduce了。

RDD 的操作算子包括两类，

* 一类叫做transformation，它是用来将 RDD 进行转化，构建 RDD 的血缘关系；

* 另一类叫做action，它是用来触发 RDD 进行计算，得到 RDD 的相关计算结果或者 保存 RDD 数据到文件系统中.

##### 1.3.1.4   依赖(血缘)

RDDs 通过操作算子进行转换，转换得到的新 RDD 包含了从其他 RDDs 衍生所必需的信息，RDDs 之间维护着这种血缘关系，也称之为依赖。

如下图所示，依赖包括两种，

* 一种是窄依赖，RDDs 之间分区是一一对应的，

* 另一种是宽依赖，下游 RDD 的每个分区与上游 RDD(也称之为父RDD)的每个分区都有关，是多对多的关系。

![依赖](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-依赖.png)

##### 1.3.1.5   缓存

如果在应用程序中多次使用同一个 RDD，可以将该 RDD 缓存起来，该 RDD 只有在第一次计算的时候会根据血缘关系得到分区的数据，在后续其他地方用到该 RDD 的时候，会直接从缓存处取而不用再根据血缘关系计算，这样就加速后期的重用。

如下图所示，RDD-1 经过一系列的转换后得到 RDD-n 并保存到 hdfs，RDD-1 在这一过程中会有个中间结果，如果将其缓存到内存，那么在随后的 RDD-1 转换到 RDD-m 这一过程中，就不会计算其之前的 RDD-0 了。

![缓存](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-缓存.png)

##### 1.3.1.6   checkpoint

虽然 RDD 的血缘关系天然地可以实现容错，当 RDD 的某个分区数据计算失败或丢失，可以通过血缘关系重建。

但是对于长时间迭代型应用来说，随着迭代的进行，RDDs 之间的血缘关系会越来越长，一旦在后续迭代过程中出错，则需要通过非常长的血缘关系去重建，势必影响性能。

为此，RDD 支持checkpoint 将数据保存到持久化的存储中，这样就可以切断之前的血缘关系，因为checkpoint 后的 RDD 不需要知道它的父 RDDs 了，它可以从 checkpoint 处拿到数据。

## 2 RDD 编程

### 2.1 RDD 编程模型

在 Spark 中，RDD 被表示为对象，通过对象上的方法调用来对 RDD 进行转换。

经过一系列的transformations定义 RDD 之后，就可以调用 actions 触发 RDD 的计算

action可以是向应用程序返回结果(count, collect等)，或者是向存储系统保存数据(saveAsTextFile等)。

在Spark中，只有遇到action，才会执行 RDD 的计算(即延迟计算)，这样在运行时可以通过管道的方式传输多个转换。

要使用 Spark，开发者需要编写一个 Driver 程序，它被提交到集群以调度运行 Worker.Driver 中定义了一个或多个 RDD，并调用 RDD 上的 action，Worker 则执行 RDD 分区计算任务

### 2.2 RDD的创建

在 Spark 中创建 RDD 的方式可以分为 3 种：

* 从集合中创建 RDD

* 从外部存储创建 RDD

* 从其他 RDD 转换得到新的 RDD。

#### 2.2.1  从集合中创建 RDD

##### 2.2.1.1   使用parallelize函数创建

```scala
scala> val arr = Array(10,20,30,40,50,60)
arr: Array[Int] = Array(10, 20, 30, 40, 50, 60)

scala> val rdd1 = sc.parallelize(arr)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at parallelize at <console>:26
```

##### 2.2.1.2   使用makeRDD函数创建

```scala
makeRDD和parallelize是一样的.
scala> val rdd1 = sc.makeRDD(Array(10,20,30,40,50,60))
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at makeRDD at <console>:24
```

说明:

* 一旦 RDD 创建成功, 就可以通过并行的方式去操作这个分布式的数据集了.

* parallelize和makeRDD还有一个重要的参数就是把数据集切分成的分区数.

* Spark 会为每个分区运行一个任务(task). 正常情况下, Spark 会自动的根据你的集群来设置分区数

#### 2.2.2  从外部存储创建 RDD

Spark 也可以从任意 Hadoop 支持的存储数据源来创建分布式数据集.

可以是本地文件系统, HDFS, Cassandra, HVase, Amazon S3 等等.

Spark 支持 文本文件, SequenceFiles, 和其他所有的 Hadoop InputFormat.

```scala
scala> var distFile = sc.textFile("words.txt")
distFile: org.apache.spark.rdd.RDD[String] = words.txt MapPartitionsRDD[1] at textFile at <console>:24
scala> distFile.collect
res0: Array[String] = Array(atguigu hello, hello world, how are you, abc efg)
```

说明:

1. url可以是本地文件系统文件, hdfs://..., s3n://...等等

2. 如果是使用的本地文件系统的路径, 则必须每个节点都要存在这个路径

3. 所有基于文件的方法, 都支持目录, 压缩文件, 和通配符(*). 例如: 

```shell
textFile("/my/directory"), textFile("/my/directory/*.txt"), and textFile("/my/directory/*.gz").
```

4. textFile还可以有第二个参数, 表示分区数. 默认情况下, 每个块对应一个分区.(对 HDFS 来说, 块大小默认是 128M). 可以传递一个大于块数的分区数, 但是不能传递一个比块数小的分区数.

5. 关于读取文件和保存文件的其他知识, 后面专门的章节介绍.

#### 2.2.3  从其他 RDD 转换得到新的 RDD

就是通过 RDD 的各种转换算子来得到新的 RDD.

### 2.3 RDD 的转换(transformation)

在 RDD 上支持 2 种操作:

1. transformation

从一个已知的 RDD 中创建出来一个新的 RDD 例如: map就是一个transformation.

2. action

在数据集上计算结束之后, 给驱动程序返回一个值. 例如: reduce就是一个action.

本节学习 RDD 的转换操作, Action操作下节再学习.

在 Spark 中几乎所有的transformation操作都是懒执行的(lazy), 也就是说transformation操作并不会立即计算他们的结果, 而是记住了这个操作.

只有当通过一个action来获取结果返回给驱动程序的时候这些转换操作才开始计算.

这种设计可以使 Spark 运行起来更加的高效.

默认情况下, 你每次在一个 RDD 上运行一个action的时候, 前面的每个transformed RDD 都会被重新计算.

但是我们可以通过persist (or cache)方法来持久化一个 RDD 在内存中, 也可以持久化到磁盘上, 来加快访问速度. 后面有专门的章节学习这种持久化技术.

根据 RDD 中数据类型的不同, 整体分为 2 种 RDD:

1. Value类型

2. Key-Value类型(其实就是存一个二维的元组)

#### 2.3.1 Value 类型

##### 2.3.1.1 map(func)

作用: 返回一个新的 RDD, 该 RDD 是由原 RDD 的每个元素经过函数转换后的值而组成. 就是对 RDD 中的数据做转换.

案例:

创建一个包含1-10的的 RDD，然后将每个元素*2形成新的 RDD

```scala
scala > val rdd1 = sc.parallelize(1 to 10)
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at parallelize at <console>:24
// 得到一个新的 RDD, 但是这个 RDD 中的元素并不是立即计算出来的
scala> val rdd2 = rdd1.map(_ * 2)
rdd2: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[1] at map at
<console>:26

// 开始计算 rdd2 中的元素, 并把计算后的结果传递给驱动程序
scala> rdd2.collect
res0: Array[Int] = Array(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
```

##### 2.3.1.2   mapPartitions(func)

作用: 类似于map(func), 但是是独立在每个分区上运行.所以:Iterator<T> => Iterator<U>

假设有N个元素，有M个分区，那么map的函数的将被调用N次,而mapPartitions被调用M次,一个函数一次处理所有分区。

```scala
scala> val source = sc.parallelize(1 to 10)
source: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[5] at parallelize at <console>:24

scala> source.mapPartitions(it => it.map(_ * 2))
res7: org.apache.spark.rdd.RDD[Int] = MapPartitionsRDD[6] at mapPartitions at <console>:27

scala> res7.collect
res8: Array[Int] = Array(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
```

##### 2.3.1.3  mapPartitionsWithIndex(func)

作用: 和mapPartitions(func)类似. 但是会给func多提供一个Int值来表示分区的索引. 所以func的类型是:(Int, Iterator<T>) => Iterator<U>

```scala
scala> val rdd1 = sc.parallelize(Array(10,20,30,40,50,60))
rdd1: org.apache.spark.rdd.RDD[Int] = ParallelCollectionRDD[0] at parallelize at <console>:24

scala> rdd1.mapPartitionsWithIndex((index, items) => items.map((index, _)))
res8: org.apache.spark.rdd.RDD[(Int, Int)] = MapPartitionsRDD[3] at mapPartitionsWithIndex at <console>:27

scala> res8.collect
res9: Array[(Int, Int)] = Array((0,10), (0,20), (0,30), (1,40), (1,50), (1,60))
```

