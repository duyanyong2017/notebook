转换操作

RDD 的转换操作是返回新的 RDD 的操作。转换出来的 RDD 是惰性求值的，只有在行动操作中用到这些 RDD 时才会被计算。

许多转换操作都是针对各个元素的，也就是说，这些转换操作每次只会操作 RDD 中的一个元素，不过并不是所有的转换操作都是这样的。表 1 描述了常用的 RDD 转换操作。

表 1 RDD转换操作（rdd1={1, 2, 3, 3}，rdd2={3,4,5})
函数名	作用	示例	结果
map()	将函数应用于 RDD 的每个元素，返回值是新的 RDD	rdd1.map(x=>x+l)	{2,3,4,4}
flatMap()	将函数应用于 RDD 的每个元素，将元素数据进行拆分，变成迭代器，返回值是新的 RDD	rdd1.flatMap(x=>x.to(3))	{1,2,3,2,3,3,3}
filter()	函数会过滤掉不符合条件的元素，返回值是新的 RDD	rdd1.filter(x=>x!=1)	{2,3,3}
distinct()	将 RDD 里的元素进行去重操作	rdd1.distinct()	(1,2,3)
union()	生成包含两个 RDD 所有元素的新的 RDD	rdd1.union(rdd2)	{1,2,3,3,3,4,5}
intersection()	求出两个 RDD 的共同元素	rdd1.intersection(rdd2)	{3}
subtract()	将原 RDD 里和参数 RDD 里相同的元素去掉	rdd1.subtract(rdd2)	{1,2}
cartesian()	求两个 RDD 的笛卡儿积	rdd1.cartesian(rdd2)	{(1,3),(1,4)……(3,5)}



对一个数据为List(1,2,3)的RDD进行基于基本的RDD转化操作

| 函数名     | 目的                                                         | 实例                      | 结果            |
| ---------- | ------------------------------------------------------------ | ------------------------- | --------------- |
| map()      | 将函数应用于RDD中的每个元素，将返回值构成新的RDD             | rdd.map(x => x + 1)       | {2,3,4,5}       |
| flatMap()  | 将元素应用于RDD中的每个元素，将返回的迭代器中所有内容构成新的RDD。通常用来切分单词 | rdd.flatMap(x => x.to(3)) | {1,2,3,2,3,3,3} |
| filter()   | 返回一个通过传给filter()的函数过滤掉不符合条件的元素，返回一个新的RDD | rdd.filter(x => x != 1)   | {2,3}           |
| distinct() | 将 RDD 里的元素进行去重操作                                  | rdd1.distinct()           | (1,2,3)         |
|            |                                                              |                           |                 |
|            |                                                              |                           |                 |
|            |                                                              |                           |                 |
|            |                                                              |                           |                 |
|            |                                                              |                           |                 |
|            |                                                              |                           |                 |
|            |                                                              |                           |                 |



对一个数据为List(1,2,3)和List(3,4,5)的RDD进行基于两个RDD的转化操作

| 函数名  | 目的                             | 实例             | 结果          |
| ------- | -------------------------------- | ---------------- | ------------- |
| union() | 生成一个包含两个RDD所有元素的RDD | rdd.union(other) | {1,2,3,3,4,5} |
|         |                                  |                  |               |
|         |                                  |                  |               |
|         |                                  |                  |               |
|         |                                  |                  |               |

3. 行动操作

行动操作用于执行计算并按指定的方式输出结果。行动操作接受 RDD，但是返回非 RDD，即输出一个值或者结果。在 RDD 执行过程中，真正的计算发生在行动操作。表 2 描述了常用的 RDD 行动操作。

表 2 RDD 行动操作（rdd={1,2,3,3}）
函数名	作用	示例	结果
collect()	返回 RDD 的所有元素	rdd.collect()	{1,2,3,3}
count()	RDD 里元素的个数	rdd.count()	4
countByValue()	各元素在 RDD 中的出现次数	rdd.countByValue()	{(1,1),(2,1),(3,2})}
take(num)	从 RDD 中返回 num 个元素	rdd.take(2)	{1,2}
top(num)	从 RDD 中，按照默认（降序）或者指定的排序返回最前面的 num 个元素	rdd.top(2)	{3,3}
reduce()	并行整合所有 RDD 数据，如求和操作	rdd.reduce((x,y)=>x+y)	9
fold(zero)(func)	和 reduce() 功能一样，但需要提供初始值	rdd.fold(0)((x,y)=>x+y)	9
foreach(func)	对 RDD 的每个元素都使用特定函数	rdd1.foreach(x=>printIn(x))	打印每一个元素
saveAsTextFile(path)	将数据集的元素，以文本的形式保存到文件系统中	rdd1.saveAsTextFile(file://home/test)	 
saveAsSequenceFile(path)	将数据集的元素，以顺序文件格式保存到指 定的目录下	saveAsSequenceFile(hdfs://home/test)	 
