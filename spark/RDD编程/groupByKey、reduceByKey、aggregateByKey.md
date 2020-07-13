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
