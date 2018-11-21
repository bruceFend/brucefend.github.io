---
layout: post
title: 项目总结2- highAnalyseProcess
category: 技术
tags: DWA sparkStreaming Kafka 
keywords: DWA sparkStreaming Kafka
description: 进入数据组后的第二个项目
---

### 背景

客户每次调用我们的产品都会产生日志信息，有部分日志（带特定关键字）会通过flume写入到kafka的特定topic，作为后续数据分析的“原材料”

**简要逻辑：**

1. 生产往`flume`写入日志，通过配置`flume`将特定消息写入`kafka:myTopic`，
2. `sparkStreaming`工程（本项目）读`kafka`指定`myTopic`的消息
3. 读`redis`中的配置信息，对`log`处理后，写入`HBase`表
4. 往`kafka:myTopic2`写推送消息

**涉及到的技术：**
1. 创建`sparkStreaming`流（`kafkaStreaming`）
2. `spark`读`redis`
3. `spark`写`HBase`
4. `spark`写`kafka`


### 核心代码

**SparkStreaming初始化**
```
// 初始化
val conf = new SparkConf().setAppName("myStreamingApp")
val duration = Seconds(10) // 10s作为窗口
val ssc = new StreamingContext(conf, duration)
ssc.checkpoint("checkpoint") // 这里设置得并不合适，应该在rdd缓存起来之后，见《Spark技术内幕》

//【从kafka中创建streaming】【得到Map[String,String]】
val kafkaStream = KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder](ssc, kafkaParams, topics).map(_._2)
// 过滤关键字
val requestMsg = kafkaStream.filter(_.contains(“keyWord”))

// 【业务逻辑】
requestMsg.foreachRDD(rdd => {
    // do what you want

})

ssc.start()
ssc.awaitTermination()
```

**Spark读redis**
```
import redis.clients.jedis.{Jedis, JedisPool, JedisCluster, HostAndPort}

def setJedisCluster(): JedisCluster = {
    var jc: JedisCluster = null
    try {
        jc = new JedisCluster(hosts, redisTimeout, redirections)
    } catch {
        case e:Excetpion =>
        LOG.error("something wrong: {}",e.getMessage)
        throw new Exception("JedisCluster create failed")
    }
}

def Jhget(key: String, field: String): String = {
    var res: String = ""
    if( jedisCluster != null && key != null && field != null ){
       try {
            res = jedisCluster.hget(key,field)
       } catch {
           case e:Exception =>
           LOG.error("something wrong")
           throw new Exception("Jedis hget failed.", e)
       }
    }
    res
}
```

**Spark写HBase**

`Scala`写入`HBase`的两种方法：

1. [PutRDD.saveAsHadoopDataset()](https://mapr.com/blog/spark-streaming-hbase/)
```
val jobConfig: Jobconfiguration= new JobConf(HBCONF, this.getClass)
jobConfig.setOutputFormat(classOf[TableOutputFormat])
jobConfig.set(TableOutputFormat.OUTPUT_TABLE,"talbeName")
PutRDD.saveAsHadoopDataset(jobConfig)
```

2. [hTable.put(put)](https://blog.csdn.net/liyongke89/article/details/51991132)
```
val conn = HBaseUtils.getConn()
val myTable = conn.getTable(TableName.valueOf("tableName")).asInstanceOf[HTable]
myTable.setAutoFlush(false, false)
myTable.setWriteBufferSize( 300 * 1024 * 1024 )
val put = new Put(Bytes.toBytes(rowkey))
put.addColumn(..., ...)
......
myTable.put(put)
try {
    myTbale.flushCommits()
} catch {
    case e:Exception => Log.error("something wrong: {}",e.getMessage)
} finally {
    myTable.close()
}
// hBaseTable 需要关闭。虽然经过测试没有发现线程泄露的问题，但还是有隐患？
```

**Spark写kafka**

// Main函数广播kafkaProducer或者广播配置，注意只能广播***只读变量*** 
```
val brokerListBroadcast: Broadcast[String] = ssc.sparkContext.broadcast("my.broker.list")
val topicBroadcast: Broadcast[String] = ssc.sparkContext.broadcast("myTopic")

requestMsg.forEachRDD(rdd => {
    if(!rdd.isEmpty){
        val kafkaProducer = KafkaProducer.getInstance(brokerListBroadcast.value)
        kafkaProducer.send(topicBroadcast.value, "msg")
    }
})
```

// KafkaProducer.java 接收广播变量
```
import kafka.javaapi.producer.Producer;
improt kafka.producer.KeyedMessage;
import kafka.producer.ProducerConfig;

private static KafkaProducer instance = null;

public static synchronized KafkaProducer getInstance(String brokerList) {
    if (instance == null) {
        Properties props = new Properties();
        props.put(METADATA_BROKER_LIST_KEY, brokerList);
        props.put(SERIALIZER_CLASS_KEY, "serializer.class");
        props.put("kafka.message.CompressionCodec","1");
        props.put("client.id","streaming-kafka-output");

        ProducerConfig conf = new ProducerConfig(props);
        instance = new Producer(conf);
    }
    return instance;
}
```

### 采坑和反思：

1、写入`HBase`时，从日志中看到写入速度非常慢（大概5条记录/秒），由于忘了之前的map等操作都是懒操作，
因此错误地以为写HBase的方法有问题，后面发现实际是rdd在map操作时有连接redis的代码，
而且每条记录都会**创建一次`redis`的`connection`**，导致在后面插入时缓慢

2、提示`java.lang.NoSuchMethodError: org.apache.spark.sql.functions$.count(Ljava/lang/String:)Lorg/apache/spark/sql/TypedColumn`
错误因为`spark-sql`的版本问题，之前是1.5.1，改成1.6.3不行(运行环境就是1.5.1的版本)

3、将处理结果往`kafka`写消息的时候，出现了线程泄露的问题。一开始以为是`kafkaProducer`的创建问题，
但后面通过注释代码的方式定位到是sqlContext对象的创建问题。之前用的是 `val sqlContext = new SQLContext(sparkContext)`
改成下面的写法后问题解决 `val sqlContext = SQLContext.getOrCreate(sparkContext)`
【在调试的时候，其实发现了管理的web页面出现了SQL、SQL1、SQL2这些菜单，也就是每处理一批数据都会出现一个新的sqlContext，
只是当时不知道这里是有问题的】

4、各种序列化问题，`kafkaProducer/sparkContext`

5、没有记录kafka的offset信息，应用重启后，无法从之前断掉的位置开始，导致数据丢失。
【[方法一](https://blog.csdn.net/high2011/article/details/53706446?winzoom=1)、[方法二](https://blog.csdn.net/high2011/article/details/79848882)，实际工程参考了方法一，
将`offset`的信息记录在`redis`中，每一批次数据处理完后更新`offset`】

```scala
// 原本使用的方法 KafkaUtils.scala #395
 KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder](ssc, kafkaParams, topics)

// 新的方法 KafkaUtils.scala #349
 KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder, (String, String)](ssc,kafkaParams, fromOffsets, (mmd: MessageAndMetadata[String, String]) => (mmd.key, mmd.message))

kafkaStream.foreachRDD( rdd=>{
    // do something
    updateOffsetInRedis(rdd)
})

```

`updateOffsetInRedis`中关键的地方在于类型转换`val offsetRanges = rdd.asInstanceOf[HasOffsetRanges].offsetRanges`
在方法一和二中都有详细代码。之所以不用方法二的原因之一是因为不知道它具体写在哪儿，无法手动设置（第一次运行的时候需要）

6、应用跑一段时间后会变慢，甚至完全卡住不动。跑了一周后发现应用在处理某一批消息的时候完全不动，但是日志没有异常。
猜测的原因是跑另外一个批处理的程序时，占有过多内存，导致实时处理卡住，但是批处理程序跑完后也没有恢复。
【这里对各种参数的调优还不理解，需要继续学习】

7、程序运行起来后，会在集群的磁盘中写入缓存(`/hadoop/data14/nm/localdir/usercache/hdUser/appcache/application_yourAppId`)
而且每次重启应用后，都不确定会写在哪个节点上，导致磁盘会被撑爆，这里应该跟spark任务的运行机制有关，需要之后再去了解。

#### TODO：

1、优化代码/重构系统（考虑参考bitmap的方式来重新设计表，加快计算速度同时降低代码的复杂度）
（优化计算步骤、缓存和`checkpoint`的设置）

2、深入了解spark任务的运行机制
