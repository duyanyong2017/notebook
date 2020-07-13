# groupByKey、reduceByKey、aggregateByKey

官网解释：

| Transformation                                               | Meaning                                                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **groupByKey**([*numPartitions*])                            | When called on a dataset of (K, V) pairs, returns a dataset of (K, Iterable<V>) pairs.<br/>**Note:** If you are grouping in order to perform an aggregation (such as a sum or average) over each key, using `reduceByKey` or `aggregateByKey` will yield much better performance.<br/>**Note:** By default, the level of parallelism in the output depends on the number of partitions of the parent RDD. You can pass an optional `numPartitions` argument to set a different number of tasks. |
| **reduceByKey**(*func*, [*numPartitions*])                   | When called on a dataset of (K, V) pairs, returns a dataset of (K, V) pairs where the values for each key are aggregated using the given reduce function *func*, which must be of type (V,V) => V. Like in `groupByKey`, the number of reduce tasks is configurable through an optional second argument. |
| **aggregateByKey**(*zeroValue*)(*seqOp*, *combOp*, [*numPartitions*]) | When called on a dataset of (K, V) pairs, returns a dataset of (K, U) pairs where the values for each key are aggregated using the given combine functions and a neutral "zero" value. Allows an aggregated value type that is different than the input value type, while avoiding unnecessary allocations. Like in `groupByKey`, the number of reduce tasks is configurable through an optional second argument. |

## groupByKey

groupByKey实现WordCount

~~~scala
scala> val rdd = sc.parallelize(List(1,1,2,2,3,3)).map((_,1))
rdd: org.apache.spark.rdd.RDD[(Int, Int)] = MapPartitionsRDD[1] at map at <console>:24

scala> rdd.groupByKey()
res0: org.apache.spark.rdd.RDD[(Int, Iterable[Int])] = ShuffledRDD[2] at groupByKey at <console>:26

scala> rdd.groupByKey().foreach(println)
[Stage 0:==============>                                            (2 + 6) / 8](2,CompactBuffer(1, 1))
(3,CompactBuffer(1, 1))
(1,CompactBuffer(1, 1))

scala> rdd.groupByKey().collect.foreach(println)
(1,CompactBuffer(1, 1))
(2,CompactBuffer(1, 1))
(3,CompactBuffer(1, 1))

scala> rdd.groupByKey().map(x => (x._1,x._2.sum)).collect.foreach(println)
(1,2)
(2,2)
(3,2)
~~~

## groupByKey源码

~~~scala
def groupByKey(numPartitions: Int): RDD[(K, Iterable[V])] = self.withScope {
    groupByKey(new HashPartitioner(numPartitions))
  }

def groupByKey(partitioner: Partitioner): RDD[(K, Iterable[V])] = self.withScope {
    // groupByKey shouldn't use map side combine because map side combine does not
    // reduce the amount of data shuffled and requires all map side data be inserted
    // into a hash table, leading to more objects in the old gen.
    val createCombiner = (v: V) => CompactBuffer(v)
    val mergeValue = (buf: CompactBuffer[V], v: V) => buf += v
    val mergeCombiners = (c1: CompactBuffer[V], c2: CompactBuffer[V]) => c1 ++= c2
    val bufs = combineByKeyWithClassTag[CompactBuffer[V]](
      createCombiner, mergeValue, mergeCombiners, partitioner, mapSideCombine = false)
    bufs.asInstanceOf[RDD[(K, Iterable[V])]]
  }
~~~

## reduceByKey

reduceByKey实现WordCount

~~~scala
scala> val rdd = sc.parallelize(List(1,1,2,2,3,3)).map((_,1))
rdd: org.apache.spark.rdd.RDD[(Int, Int)] = MapPartitionsRDD[8] at map at <console>:24

scala> rdd.reduceByKey(_+_).collect.foreach(println)
(1,2)
(2,2)
(3,2)
~~~

#### reduceByKey源码

~~~scala
def reduceByKey(func: (V, V) => V): RDD[(K, V)] = self.withScope {
    reduceByKey(defaultPartitioner(self), func)
  }
/**
   * Merge the values for each key using an associative and commutative reduce function. This will
   * also perform the merging locally on each mapper before sending results to a reducer, similarly
   * to a "combiner" in MapReduce.
   */
  def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] = self.withScope {
    combineByKeyWithClassTag[V]((v: V) => v, func, func, partitioner)
  }

/**
   * :: Experimental ::
   * Generic function to combine the elements for each key using a custom set of aggregation
   * functions. Turns an RDD[(K, V)] into a result of type RDD[(K, C)], for a "combined type" C
   *
   * Users provide three functions:
   *
   *  - `createCombiner`, which turns a V into a C (e.g., creates a one-element list)
   *  - `mergeValue`, to merge a V into a C (e.g., adds it to the end of a list)
   *  - `mergeCombiners`, to combine two C's into a single one.
   *
   * In addition, users can control the partitioning of the output RDD, and whether to perform
   * map-side aggregation (if a mapper can produce multiple items with the same key).
   *
   * @note V and C can be different -- for example, one might group an RDD of type
   * (Int, Int) into an RDD of type (Int, Seq[Int]).
   */
  @Experimental
  def combineByKeyWithClassTag[C](
      createCombiner: V => C,
      mergeValue: (C, V) => C,
      mergeCombiners: (C, C) => C,
      partitioner: Partitioner,
      mapSideCombine: Boolean = true,
      serializer: Serializer = null)(implicit ct: ClassTag[C]): RDD[(K, C)] = self.withScope {
    require(mergeCombiners != null, "mergeCombiners must be defined") // required as of Spark 0.9.0
    if (keyClass.isArray) {
      if (mapSideCombine) {
        throw new SparkException("Cannot use map-side combining with array keys.")
      }
      if (partitioner.isInstanceOf[HashPartitioner]) {
        throw new SparkException("HashPartitioner cannot partition array keys.")
      }
    }
    val aggregator = new Aggregator[K, V, C](
      self.context.clean(createCombiner),
      self.context.clean(mergeValue),
      self.context.clean(mergeCombiners))
    if (self.partitioner == Some(partitioner)) {
      self.mapPartitions(iter => {
        val context = TaskContext.get()
        new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
      }, preservesPartitioning = true)
    } else {
      new ShuffledRDD[K, V, C](self, partitioner)
        .setSerializer(serializer)
        .setAggregator(aggregator)
        .setMapSideCombine(mapSideCombine)
    }
  }
~~~

#### groupByKey与reduceByKey区别

- reduceByKey的性能比groupByKey好，因为发生shuffle的数据小一些，减少了数据拉取次数和网络IO、磁盘IO

- 通过传入的参数我们可以发现两者最大的不同是mapSideCombine参数的不同。mapSideCombine参数是否进行map端的本地聚合，groupByKey的mapSideCombine默认值为**false**，表示不进行map的本地聚合，reduceByKey的mapSideCombine默认值为**true**，表示进行map的本地聚合。

- 我们通过MapReduce的shuffle过程可以知道shuffle发生在reduce task 拉去 map task处理的结果数据的过程间，所以在map端进行一次数据的本地聚合能够优化shuffle

- 采用groupByKey时，由于它不接受函数，Spark只能先将所有的键值对（key-value）都移动，这样的后果时集群节点之间的开销很大

  **groupByKey的shuffle过程**

![image](https://upload-images.jianshu.io/upload_images/9221434-2b56dda8a902c294.png?imageMogr2/auto-orient/strip|imageView2/2/w/761/format/webp)

​	**reduceByKey的shuffle过程**

![image](https://upload-images.jianshu.io/upload_images/9221434-bea9f6da3a33c706.png?imageMogr2/auto-orient/strip|imageView2/2/w/429/format/webp)

## aggregateByKey

#### 源码

~~~scala
def aggregateByKey[U: ClassTag](zeroValue: U, numPartitions: Int)(seqOp: (U, V) => U,
      combOp: (U, U) => U): RDD[(K, U)] = self.withScope {
    aggregateByKey(zeroValue, new HashPartitioner(numPartitions))(seqOp, combOp)
  }
~~~

首先看到的是 “泛型” 声明。懂 Java 的同学直接把这个 " **[U: ClassTag]** " 理解成是一个泛型声明就好了。如果您不是很熟悉 Java 语言，那我们只需要知道这个 U 表示我们的 aggregate 方法**只能接受某一种类型的输入值**，至于到底是哪种类型，要看您在具体调用的时候给了什么类型。

然后我们来看看 aggregate 的参数列表。明显这个 aggregate 方法是一个柯里化函数。柯里化的知识不在本篇文章讨论的范围之内。如果您还不了解柯里化的概念，那在这里简单地理解为是**通过多个圆括号来接受多个输入参数就可以了**。

 

然后我们来看看第 1 部分，即上面蓝色加粗的 " **(zeroValue: U)** " 。这个表示它接受一个**任意类型**的输入参数，变量名为 zeroValue 。这个值就是**初值**，至于这个初值的作用，姑且不用理会，等到下一小节通过实例来讲解会更明了，在这里只需要记住它是一个 “只使用一次” 的值就好了。

 

第 2 部分，我们还可以再把它拆分一下，因为它里面其实有两个参数。笔者认为 Scala 语法在定义多个参数时，辨识度比较弱，不睁大眼睛仔细看，很难确定它到底有几个参数。

 

首先是第 1 个参数 " **seqOp: (U, T) => U** " 它是一个函数类型，以一个输入为任意两个类型 U, T 而输出为 U 类型的函数作为参数。这个函数会先被执行。这个参数函数的作用是为每一个分片（ slice ）中的数据遍历应用一次函数。换句话说就是假设我们的输入数据集（ RDD ）有 1 个分片，则只有一个 seqOp 函数在运行，假设有 3 个分片，则有三个 seqOp 函数在运行。可能有点难以理解，不过没关系，到后面结合实例就很容易理解了。

 

另一个参数 " **combOp: (U, U) => U** " 接受的也是一个函数类型，以输入为任意类型的两个输入参数而输出为一个与输入同类型的值的函数作为参数。这个函数会在上面那个函数执行以后再执行。这个参数函数的输入数据来自于第一个参数函数的输出结果，这个函数仅会执行 1 次，它是用来最终聚合结果用的。同样这里搞不懂没关系，下一小节的实例部分保证让您明白。

最后是上面这个红色加粗的 " **: U** " 它是 aggregate 方法的返回值类型，也是泛型表示。



#### 说明

第一个参数是， 给每一个分区中的每一种key一个初始值
第二个是个函数， Seq Function， 这个函数就是用来先对每个分区内的数据按照 key 分别进行定义进行函数定义的操作
第三个是个函数， Combiner Function， 对经过 Seq Function 处理过的数据按照 key 分别进行进行函数定义的操作

也可以自定义分区器, 分区器有默认值

#### 整个流程就是

在 kv 对的 RDD 中，按 key 将 value 进行分组合并，合并时，将每个 value 和初始值作为 seq 函数的参数，进行计算，返回的结果作为一个新的 kv 对，然后再将结果按照 key 进行合并，最后将每个分组的 value 传递给 combine 函数进行计算（先将前两个 value 进行计算，将返回结果和下一个 value 传给 combine 函数，以此类推），将 key 与计算结果作为一个新的 kv 对输出 



https://www.cnblogs.com/chorm590/p/spark_201904201159.html
