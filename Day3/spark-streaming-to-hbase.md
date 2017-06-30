### Programming Environment

* Spark Streaming 1.6.0-cdh5.9.1
* Scala 2.10.4
* Kafka 2.0.2-1.2.0.2
* HBase 1.2.0-cdh5.9.1

### Connection Initialization and Kafka Broker List

The code to initialize a connection with Kafka cluster go as follows:

````
val topics = Set(args(1))
val kafkaParams = Map[String, String](
  "metadata.broker.list" -> args(0))

val messages = KafkaUtils.createDirectStream[
  String, String, StringDecoder, StringDecoder](
  ssc,kafkaParams,topics)
````

In the code above, we pass the Kafka broker list as a command line arguments. When experimenting, we just pass ONE Kafka broker. We will at least pass 2 brokers for high availability.

### Managing Kafka Offsets

When using the direct mode of Kafka connection in Spark Streaming, we will have to deal to the offset management things in our program code. That means to acquire the offset value, store it in a temporary variable, and save it to a reliable storage external.

The following is the code to do that:

````
var offsets = Array.empty[OffsetRange]
messages.transform {rdd =>
  offsets = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
  ...
  PROCESSING BUSINESS REQUIREMENTS;
  ...
  for (o <- offsets) {
    SAVING offsets into somewhere reliable;
  }
  rdd
}
````

### Writing to HBase

````
def writeToHBase(rdd:RDD[String],tabName:String):Unit = {
  rdd.foreachPartition {msgs =>
    val config = HBaseConfiguration.create()
    val tab = new HTable(config,tabName)
    msgs.foreach {msg =>
      val Array(id,s)=msg.split("\\u0001",2)
      val put = new Put(toBytes(id))
      put.add(toBytes("cf"),toBytes("value"),toBytes(s))
      tab.put(put)
    }
    tab.flushCommits()
    tab.close()
  }
}
````

When writing data to HBase, one of the potential performance issue is to open/close HBase connection every time it writes a record into it. Some of the methods to deal with it include: 1) creating a connection in a RDD partition basis, or 2) using connection pools.

The above is by creating a connection in a RDD partiton basis.

### The whole Program

````
package com.cloudera.fceboot.lab
import kafka.serializer.StringDecoder
import org.apache.spark.SparkConf
import org.apache.spark.rdd.RDD
import org.apache.spark.streaming._
import org.apache.spark.streaming.kafka._
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.client._
import org.apache.hadoop.hbase.util.Bytes.toBytes

object KafkaStreamer {
  def writeToHBase(rdd:RDD[String],tabName:String):Unit = {
    rdd.foreachPartition {msgs =>
      val config = HBaseConfiguration.create()
      val tab = new HTable(config,tabName)
      msgs.foreach {msg =>
        val Array(id,s)=msg.split("\\u0001",2)
        val put = new Put(toBytes(id))
        put.add(toBytes("cf"),toBytes("value"),toBytes(s))
        tab.put(put)
      }
      tab.flushCommits()
      tab.close()
    }
  }

  def main(args: Array[String]):Unit = {
    if(args.length<2) {
      System.err.println("Invalid arguments.")
      System.exit(2)
    }

    val sparkConf = new SparkConf().setAppName("kafka-streamer")
    sparkConf.set("spark.broadcast.compress","false")
    sparkConf.set("spark.shuffle.compress","false")
    sparkConf.set("spark.shuffle.spill.compress","false")
    sparkConf.set("spark.io.compress.codec","lzf")
    val ssc = new StreamingContext(sparkConf, Seconds(3))

    val topics = Set(args(1))
    val kafkaParams = Map[String, String](
      "metadata.broker.list" -> args(0))

    val messages = KafkaUtils.createDirectStream[
      String, String, StringDecoder, StringDecoder](
      ssc,kafkaParams,topics)
    var offsets = Array.empty[OffsetRange]
    messages.transform { rdd =>
      offsets = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
      rdd
    }.map(_._2).foreachRDD {rdd =>
      writeToHBase(rdd,args(2))
      //rdd.foreach(println)

      println("-====== Kafka Offsets Information ======-")
      for (o <- offsets) {
        println(s"topic=${o.topic} partition=${o.partition} offset_from=${o.fromOffset} offset_until=${o.untilOffset}")
      }
    }

    ssc.start()
    ssc.awaitTermination()
  }
}
````
