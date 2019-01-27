SparkSQL的join实现

## 大纲

### 1、SparkSQL的总体流程

### 2、Join基本要素

### 3、Join基本实现流程

### 4、inner join

### 5、left outer join

### 6、right outer join

### 7、full outer join

### 8、left semi join

### 9、left anti join

## 参考网址

```
Spark SQL 之 Join 实现 :

http://sharkdtu.com/posts/spark-sql-join.html


SparkSQL中的三种Join及其具体实现（broadcast join、shuffle hash join和sort merge join）
https://blog.csdn.net/wlk_328909605/article/details/82933552

SparkSQL – 有必要坐下来聊聊Join
http://hbasefly.com/2017/03/19/sparksql-basic-join/
```

# 一、SparkSQL的总体流程

```
1、一般地，我们有两种方式使用SparkSQL	
```

```
2、一种是直接写sql语句，这个需要有元数据库支持，例如Hive等，

3、另一种是通过DataSet/DataFrame编写Spark应用程序。
	此时，sql语句被语法解析(SQL AST)成查询计划，或者我们通过Dataset/DataFrame提供的APIs组织成查询计划，查询计划分为两大类：逻辑计划和物理计划，这个阶段通常叫做逻辑计划，经过语法分析(Analyzer)、一系列查询优化(Optimizer)后得到优化后的逻辑计划，最后被映射成物理计划，转换成RDD执行。
```

![img](http://sharkdtu.com/images/spark-sql-overview.png)

更多关于SparkSQL的解析与执行请参考文章[【sql的解析与执行】](http://www.cnblogs.com/hseagle/p/3752917.html)。对于语法解析、语法分析以及查询优化，本文不做详细阐述，本文重点介绍Join的物理执行过程。

# 二、Join基本要素

如下图所示，Join大致包括三个要素：Join方式、Join条件以及过滤条件。其中过滤条件也可以通过AND语句放在Join条件中。

![img](http://sharkdtu.com/images/spark-sql-join-overview.png)

```
Spark支持所有类型的Join，包括：
    inner join
    left outer join
    right outer join
    full outer join
    left semi join
    left anti join
下面分别阐述这几种Join的实现。
```

# 三、Join基本实现流程

## 3.0 基本概念

```
1、总体上来说，Join的基本实现流程如下图所示

2、Spark将参与Join的两张表抽象为流式遍历表(streamIter)和查找表(buildIter)，通常streamIter为大表，buildIter为小表

3、我们不用担心哪个表为streamIter，哪个表为buildIter，这个spark会根据join语句自动帮我们完成。
```

![img](http://sharkdtu.com/images/spark-sql-join-basic.png)

```
1、在实际计算时，spark会基于streamIter来遍历，每次取出streamIter中的一条记录rowA，根据Join条件计算keyA，然后根据该keyA去buildIter中查找所有满足Join条件(keyB==keyA)的记录rowBs，并将rowBs中每条记录分别与rowAjoin得到join后的记录，最后根据过滤条件得到最终join的记录。

2、从上述计算过程中不难发现，对于每条来自streamIter的记录，都要去buildIter中查找匹配的记录，所以buildIter一定要是查找性能较优的数据结构。

3、spark提供了三种join实现：sort merge join、broadcast join以及shuffle hash join。
```

## 3.1、sort merge join实现

```
如下图所示，整个过程分为三个步骤： 
1. shuffle阶段：将两张大表根据join key进行重新分区，两张表数据会分布到整个集群，以便分布式并行处理 
2. sort阶段：对单个分区节点的两表数据，分别进行排序 
3. merge阶段：对排好序的两张分区表数据执行join操作。join操作很简单，分别遍历两个有序序列，碰到相同join key就merge输出，否则取更小一边 
```

![004](http://hbasefly.com/wp-content/uploads/2017/03/004.png)

```
在遍历streamIter时，对于每条记录，都采用顺序查找的方式从buildIter查找对应的记录，由于两个表都是排序的，每次处理完streamIter的一条记录后，对于streamIter的下一条记录，只需从buildIter中上一次查找结束的位置开始查找，所以说每次在buildIter中查找不必重头开始，整体上来说，查找性能还是较优的。
```

## 3.2、broadcast join实现

```
1、为了能具有相同key的记录分到同一个分区，我们通常是做shuffle

2、那么如果buildIter是一个非常小的表，那么其实就没有必要大动干戈做shuffle了，直接将buildIter广播到每个计算节点，然后将buildIter放到hash表中
```

```
如下图所示，broadcast join可以分为两步： 
1. broadcast阶段
	* 将小表广播分发到大表所在的所有主机。
	* 广播算法可以有很多，最简单的是先发给driver，driver再统一分发给所有executor
	
2. hash join阶段：在每个executor上执行单机版hash join；

不用做shuffle，可以直接在一个map中完成，通常这种join也称之为map join。
```

![002](http://hbasefly.com/wp-content/uploads/2017/03/002.png)

### 3.2.1 broadcast join条件

```
当buildIter的估计大小不超过参数spark.sql.autoBroadcastJoinThreshold设定的值(默认10M)，那么就会自动采用broadcast join，否则采用sort merge join。
```

## 3.3 、shuffle hash join实现

```
1、在大数据条件下如果一张表很小，执行join操作最优的选择无疑是broadcast join，效率最高。但是一旦小表数据量增大，广播所需内存、带宽等资源必然就会太大，broadcast hash join就不再是最优方案。
```

```
2、此时可以按照join key进行分区，根据key相同必然分区相同的原理，就可以将大表join分而治之，划分为很多小表的join，充分利用集群资源并行化。如下图所示，shuffle hash join也可以分为两步： 
    * shuffle阶段：分别将两个表按照join key进行分区，将相同join key的记录重分布到同一节点，两张表的数据会被重分布到集群中所有节点。这个过程称为shuffle 
   
   * hash join阶段：每个分区节点上的数据执行hash join。 
```

![003](http://hbasefly.com/wp-content/uploads/2017/03/003.png)

### 3.3.1 shuffle hash join的条件

```
1、buildIter总体估计大小超过spark.sql.autoBroadcastJoinThreshold设定的值，即不满足broadcast join条件

2、开启尝试使用hash join的开关，spark.sql.join.preferSortMergeJoin=false

3、每个分区的平均大小不超过spark.sql.autoBroadcastJoinThreshold设定的值，即shuffle read阶段每个分区来自buildIter的记录要能放到内存中

4、streamIter的大小是buildIter三倍以上
```

## 3.4、小总结

```
仔细分析的话会发现，sort-merge join的代价并不比shuffle hash join小，反而是多了很多。那为什么SparkSQL还会在两张大表的场景下选择使用sort-merge join算法呢？这和Spark的shuffle实现有关，目前spark的shuffle实现都适用sort-based shuffle算法，因此在经过shuffle之后partition数据都是按照key排序的。因此理论上可以认为数据经过shuffle之后是不需要sort的，可以直接merge。 
```

# 四、不同Join方式的实现流程。

## 4.1 inner join

```
inner join是一定要找到左右表中满足join条件的记录，我们在写sql语句或者使用DataFrame时，可以不用关心哪个是左表，哪个是右表，在spark sql查询优化阶段，spark会自动将大表设为左表，即streamIter，将小表设为右表，即buildIter。这样对小表的查找相对更优。其基本实现流程如下图所示，在查找阶段，如果右表不存在满足join条件的记录，则跳过。
```

![img](http://sharkdtu.com/images/spark-sql-inner-join.png)

## 4.2 left outer join

```
left outer join是以左表为准，在右表中查找匹配的记录，如果查找失败，则返回一个所有字段都为null的记录。我们在写sql语句或者使用DataFrmae时，一般让大表在左边，小表在右边。其基本实现流程如下图所示。
```

![img](http://sharkdtu.com/images/spark-sql-leftouter-join.png)



## 4.3 right outer join

```
right outer join是以右表为准，在左表中查找匹配的记录，如果查找失败，则返回一个所有字段都为null的记录。所以说，右表是streamIter，左表是buildIter，我们在写sql语句或者使用DataFrame时，一般让大表在右边，小表在左边。其基本实现流程如下图所示。
```

![img](http://sharkdtu.com/images/spark-sql-rightouter-join.png)

## 4.4 full outer join

```
1、full outer join仅采用sort merge join实现

2、左边和右表既要作为streamIter，又要作为buildIter，其基本实现流程如下图所示。
```

![img](http://sharkdtu.com/images/spark-sql-fullouter-join.png)

```
由于左表和右表已经排好序，首先分别顺序取出左表和右表中的一条记录，比较key，如果key相等，则joinrowA和rowB，并将rowA和rowB分别更新到左表和右表的下一条记录；如果keyA<keyB，则说明右表中没有与左表rowA对应的记录，那么joinrowA与nullRow，紧接着，rowA更新到左表的下一条记录；如果keyA>keyB，则说明左表中没有与右表rowB对应的记录，那么joinnullRow与rowB，紧接着，rowB更新到右表的下一条记录。如此循环遍历直到左表和右表的记录全部处理完。
```

## 4.5 left semi join

```
left semi join是以左表为准，在右表中查找匹配的记录，如果查找成功，则仅返回左边的记录，否则返回null，其基本实现流程如下图所示。
```

![img](http://sharkdtu.com/images/spark-sql-semi-join.png)

## 4.6 left anti join

```
left anti join与left semi join相反，是以左表为准，在右表中查找匹配的记录，如果查找成功，则返回null，否则仅返回左边的记录，其基本实现流程如下图所示。
```

![img](http://sharkdtu.com/images/spark-sql-anti-join.png)
