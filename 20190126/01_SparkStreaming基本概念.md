# Ⅰ | SparkStreaming基本概念

## 大纲

### 1、批处理时间间隔

### 2、窗口时间间隔

### 3、滑动时间间隔

### 4、DStream基本概念

```
参考地址：https://mp.weixin.qq.com/s/YR7WEHkTLeH8qvZIXuuPJQ
```

# 0.0 SparkStreaming

```
1、Spark Streaming是Spark上的一个流式处理框架，可以面向海量数据实现高吞吐量、高容错的实时计算。

2、Spark Streaming支持多种类型数据源，包括Kafka、Flume、trwitter、zeroMQ、Kinesis以及TCP sockets等。

3、Spark Streaming实时接收数据流，并按照一定的时间间隔将连续的数据流拆分成一批批离散的数据集；然后应用诸如map、reducluce、join和window等丰富的API进行复杂的数据处理；最后提交给Spark引擎进行运算，得到批量结果数据，因此其也被称为准实时处理系统。
```

# 0.1 SparkStreaming与Storm的对比

```
1、目前应用最广泛的大数据流式处理框架是Storm。Spark Streaming 最低0.5~2s做一次处理（而Storm最快可达0.1s），在实时性和容错方面不如Storm。

2、然而Spark Streaming的集成性非常好，通过RDD不仅能够与Spark上的所有组件无缝衔接共享数据，还能非常容易地与Kafka、Flume等分布式日志收集框架进行集成；

3、同时Spark Streaming的吞吐量非常高，远远优于Storm的吞吐量。所以虽然Spark Streaming的处理延迟高于Storm，但是在集成性与吞吐量方面的优势使其更适用于大数据背景。
```

# 一、批处理时间间隔

```
1、在Spark Streaming中，对数据的采集是实时、逐条进行的，但是对数据的处理却是分批进行的。
```

```
2、因此，Spark Streaming需要设定一个时间间隔，将该时间间隔内采集到的数据统一进行处理，这个间隔称为批处理时间间隔。
```

```
3、也就是说对于源源不断的数据，Spark Streaming是通过切分的方式，先将连续的数据流进行离散化处理。
```

```
4、数据流每被切分一次，对应生成一个RDD，每个RDD都包含了一个时间间隔内所获取到的所有数据，因此数据流被转换为由若干个RDD构成的有序集合。
```

```
5、而批处理时间间隔决定了Spark Streaming需要多久对数据流切分一次。
```

```
6、Spark Streaming是Spark上的组件，其获取的数据和数据上的操作最终仍以Spark作业的形式在底层的Spark内核中进行计算。
```

```
7、因此批处理时间间隔不仅影响数据处理的吞吐量，同时也决定了Spark Streaming向Spark提交作业的频率和数据处理的延迟。
```

```
8、需要注意的是，批处理时间间隔的设置会伴随Spark Streaming应用程序的整个生命周期，无法在程序运行期间动态修改，所以需要综合考虑实际应用场景中的数据流特点和集群的处理性能等多种因素进行设定。
```

# 二、窗口时间间隔

```
1、窗口时间间隔又称为窗口长度，它是一个抽象的时间概念，决定了Spark Streaming对RDD序列进行处理的范围与粒度
```

```
2、即用户可以通过设置窗口长度来对一定时间范围内的数据进行统计和分析。
```

```
3、如果设批处理时间设为1s，窗口时间间隔为3s，其中每个实心矩形表示Spark Streaming每1秒钟切分出的一个RDD，若干个实心矩形块表示一个以时间为序的RDD序列，而透明矩形框表示窗口时间间隔。易知窗口内RDD的数量最多为3个，即Spark Streming 每次最多对3个RDD中的数据进行统计和分析。
```

```
4、对于窗口时间间隔还需要注意以下几点：
	* 在系统启动后的前3s内，因进入窗口的RDD不足3个，但是随着时间的推移，最终窗口将被填满。
	
	* 不同窗口内所包含的RDD可能会有重叠，即当前窗口内的数据可能被其后续若干个窗口所包含，因此在一些应用场景中，对于已经处理过的数据不能立即删除，以备后续计算使用。
	
	* 窗口时间间隔必须是批处理时间间隔的整数倍。
```

# 三、滑动时间间隔

```
1、滑动时间间隔决定了Spark Streaming对数据进行统计与分析的频率，多出现在与窗口相关的操作中。

2、滑动时间间隔是基于批处理时间间隔提出的，其必须是批处理时间间隔的整数倍。在默认的情况下滑动时间间隔设置为与批处理时间间隔相同的值。

3、如果批处理时间间隔为1s，窗口间隔为3s，滑动时间间隔为2s，其含义是每隔2s对过去3s内产生的3个RDD进行统计分析。
```

# 四、DStream的基本概念

```
1、DStream是Spark Streaming的一个基本抽象，它以离散化的RDD序列的形式近似描述了连续的数据流。

2、DStream本质上是一个以时间为键，RDD为值的哈希表，保存了按时间顺序产生的RDD，而每个RDD封装了批处理时间间隔内获取到的数据。

3、Spark Streaming每次将新产生的RDD添加到哈希表中，而对于已经不再需要的RDD则会从这个哈希表中删除，所以DStream也可以简单地理解为以时间为键的RDD的动态序列。设批处理时间间隔为1s，下图为4s内产生的DStream示意图。
```

# Ⅱ | Spark Streaming编程模式与案例分析

# 一、Spark Streaming编程模式

```scala
Spark Streaming应用程序在功能结构上通常包含以下五部分

1、导入Spark Streaming相关包：Spark Streaming作为Spark框架上的一个组件，具有很好的集成性。在开发Spark Streaming应用程序时，只需导入Spark Streaming相关包，无需额外的参数配置。

2、创建StreamingContext对象：同Spark应用程序中的SparkContext对象一样， StreamingContext对象是Spark Streaming应用程序与集群进行交互的唯一通道，其中封装了Spark集群的环境信息和应用程序的一些属性信息。在该对象中通常需要指明应用程序的运行模式（eg:setMaster）、设定应用程序名称（setAppName）、设定批处理时间间隔（val ssc = new StreamingContext(conf, Seconds(1)) ），其中批处理时间间隔需要根据用户的需求和集群的处理能力进行适当地设置。

3、创建InputDStream：Spark Streaming需要根据数据源类型选择相应的创建DStream的方法。Spark Streaming同时支持多种不同的数据源类型，其中包括Kafka、Flume、HDFS/S3、Kinesis和Twitter等数据源。

4、操作DStream：对于从数据源得到的DStream，用户可以调用丰富的操作对其进行处理。

5、启动与停止Spark Streaming应用程序：在启动Spark Streaming应用程序之前，DStream上所有的操作仅仅是定义了数据的处理流程，程序并没有真正连接上数据源，也没有对数据进行任何操作，当ssc.start()启动后程序中定义的操作才会真正开始执行。
```

# 二、stateful应用案例





## 1、功能需求

```
监听网络中某节点上指定端口传输的数据流（slave1节点9999端口的英文文本数据，以逗号间隔单词），每5秒分别统计各单词的累计出现次数。
```

## 2、代码实现

```scala
import org.apache.spark.{SparkContext, SparkConf}
import org.apache.spark.streaming.{Milliseconds,Seconds, StreamingContext}
import org.apache.spark.streaming.StreamingContext._
object StatefulWordCount {
  def main(args:Array[String]): Unit ={
/*定义更新状态方法，参数values为当前批处理时间间隔内各单词出现的次数，state为以往所有批次各单词累计出现次数。*/
    val updateFunc=(values: Seq[Int],state:Option[Int])=>{
		val currentCount=values.foldLeft(0)(_+_)
		val previousCount=state.getOrElse(0)
		Some(currentCount+previousCount)
    }
    val conf=new SparkConf().
setAppName("StatefulWordCount").

setMaster("spark://192.168.149.132:7077")
    val sc=new SparkContext(conf)
//创建StreamingContext，Spark Steaming运行时间间隔为5秒。
    val ssc=new StreamingContext(sc, Seconds(5))
/*使用updateStateByKey时需要checkpoint持久化接收到的数据。在集群模式下运行时，需要将持久化目录设为HDFS上的目录。*/
	ssc.checkpoint("hdfs://master:9000/user/dong/input/StatefulWordCountlog")
/*通过Socket获取指定节点指定端口的数据创建DStream，其中节点与端口分别由参数args(0)和args(1)给出。*/
    val lines=ssc.socketTextStream(args(0),args(1).toInt)
    val words=lines.flatMap(_.split(","))
    val wordcounts=words.map(x=>(x,1))
//使用updateStateByKey来更新状态，统计从运行开始以来单词总的次数。
    val stateDstream=wordcounts.updateStateByKey[Int](updateFunc)
    stateDstream.print()
    ssc.start()
    ssc.awaitTermination()
  }
}
```

# 三、window应用案例

## 1、功能需求

```
监听网络中某节点上指定端口传输的数据流（slave1节点上9999端口的英文文本数据，以逗号间隔单词），每10秒统计前30秒各单词累计出现的次数。

window应用案例同时涉及批处理时间间隔、窗口时间间隔与滑动时间间隔。
```

## 2、代码实现

```scala
//导包
import org.apache.spark.{SparkContext, SparkConf}
import org.apache.spark.streaming.StreamingContext._
import org.apache.spark.streaming._
import org.apache.spark.storage.StorageLevel

object WindowWordCount {
  def main(args:Array[String]) ={
    //初始化
    val conf=new SparkConf().setAppName("WindowWordCount").
setMaster("spark://master:7077")
    val sc=new SparkContext(conf)
    val ssc=new StreamingContext(sc, Seconds(5))	//每5秒产生一个RDD
    
    //设置存储目录
    ssc.checkpoint("hdfs://master:9000/user/dong/WindowWordCountlog")
    
    //
    val lines=ssc.socketTextStream("master",9999,StorageLevel.MEMORY_ONLY_SER)
    val words= lines.flatMap(_.split(","))
	
    /*采用reduceByKeyAndWindow操作进行叠加处理，窗口时间间隔:30 滑动时间间隔:10*/
    val wordcounts=words.map(x=>(x,1)).
      reduceByKeyAndWindow((a:Int,b:Int)=>(a+b),Seconds(30),Seconds(10))
    
    wordcounts.print()
    
    //启动
    ssc.start()
    ssc.awaitTermination()
  }
}
```

# Ⅲ | 性能考量

```
1、在开发Spark Streaming应用程序时，要结合集群中各节点的配置情况尽可能地提高数据处理的实时性。

2、在调优的过程中，一方面要尽可能利用集群资源来减少每个批处理的时间；另一方面要确保接收到的数据能及时处理掉。
```

## 1、运行时间优化

### 1.1 设置合理的批处理时间和窗口大小

```
1、Spark Streaming中作业之间通常存在依赖关系，后面的作业必须确保前面的作业执行结束后才能提交

2、若前面的作业的执行时间超过了设置的批处理时间间隔，那么后续的作业将无法按时提交执行，造成作业的堵塞。

3、也就是说若想Spark Streaming应用程序稳定地在集群中运行，对于接收到的数据必须尽快处理掉。

4、例如若设定批处理时间为1秒钟，那么系统每1秒钟生成一个RDD，如果系统计算一个RDD的时间大于1秒，那么当前的RDD还没来得及处理，后续的RDD已经提交上来在等待处理了，这就产生了堵塞。因此需要设置一个合理的批处理时间间隔以确保作业能够在这个批处理时间间隔时间内结束。

5、许多实验数据表明，500毫秒对大多Spark Streaming应用而言是较好的批处理时间间隔。
```

```
6、类似地，对于窗口操作，滑动时间间隔对于性能也有很大的影响。当单批次数据计算代价过高时，可以考虑适当增大滑动时间间隔.
```

```
7、对于批处理时间和窗口大小的设定，并没有统一的标准。通常是先从一个比较大的批处理时间（10秒左右）开始，然后不断地使用更小的值进行对比测试。如果Spark Streaming用户界面中显示的处理时间保持不变，则可以进一步设定更小的值；如果处理时间开始增加，则可能已经达到了应用的极限，再减小该值则可能会影响系统的性能。
```

### 1.2 提高并行度

```
1、提高并行度也是一种减少批处理所消耗时间的常见方法。

2、有以下三种方式可以提高并行度。
	2.1 一种方法是增加接收器数目。
	——> 如果获取的数据太多，则可能导致单个节点来不及对数据进行读入与分发，使得接收器成为系统瓶颈。这时可以通过创建多个输入DStream来增加接收器数目，然后再使用union来把数据合并为一个数据源。
	
	2.2 第二种方法是将收到的数据显式地重新分区。
	——> 如果接收器数目无法再增加，可以通过使用DStream.repartition、spark.streaming.blocklnterval等参数显式地对Dstream进行重新分区。
	
	2.3 第三种方法是提高聚合计算的并行度。
	——> 对于会导致shuffle的操作，例如reduceByKey、reduceByKeyAndWindow等操作，可通过显示设置更高的行度参数确保更为充分地使用集群资源。
```

## 2、内存使用与垃圾回收











### 2.1 控制批处理时间间隔内的数据量

```
1、Spark Streaming会把批处理时间间隔内获取到的所有数据存放在Spark内部可用的内存中。

2、因此必须确保在当前节点上SparkStreaming可用的内存容量至少能容下一个批处理时间间隔内所有的数据。

3、比如一个批处理时间间隔是1秒，但是1秒产生了1GB的数据，那么要确保当前的节点上至少有可供SparkStreaming使用的1GB内存。
```

### 2.2 及时清理不再使用的数据

```
1、对于内存中处理过的、不再需要的数据应及时清理，以确保Spark Streaming能够拥有足够的内存空间可以使用。

2、一种方法是可以通过设置合理的spark.cleaner.ttl时长来及时清理超时的无用数据，但该方法应慎重使用，以免后续数据在需要时被错误清理。

3、另一种方法是将spark.streaming.unpersist设置为true，系统将自动清理已经不需要的RDD。该方法能显著减少RDD对内存的需要，同时潜在地提高GC的性能。此外用户还可以通过配置参数streamingContext.remember为数据设置更长的保留时间。
```

### 2.3 减少序列化与反序列化的负担

```scala
1、SparkStreaming默认将接收到的数据序列化后放入内存，以减少内存使用。

2、序列化和反序列化需要更多的CPU资源，因此使用适当的序列化工具（例如Kryo）和自定义的序列化接口可以更高效地使用CPU。

//设置序列化器为KryoSerializer
conf.set("spark.serializer","org.apache.spark.serializer.KyroSerializer")

//注册要序列化的自定义类
conf.registerKyroClasses(Array(classOf[MyClass1],classOf[MyClass2]))

3、除了使用更好的序列化工具外还可以结合压缩机制，通过配置spark.rdd.compress，以CPU的时间开销来换取内存资源，降低GC开销。
```

