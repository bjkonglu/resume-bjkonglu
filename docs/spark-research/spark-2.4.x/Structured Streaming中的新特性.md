## Structured Streaming中的新特性
### 背景
昨天，也就是11.08，Spark发布了2.4.0版本。其中包含很多有用的新特性，让我最感兴趣的新特性就是*Flexible Streaming Sink*。在2.4.0版本未发布之前，使用Structured Streaming构建实时计算应用时，有个痛点就是需要用户自定一些sink，例如实时写入MySQL、HBase的组件。这样导致开发难度比较大，调试起来比较费劲。不过在2.4.0版本中，Spark终于解决了这个痛点，推出了*Flexible Streaming Sink*。

### Flexible Streaming Sink简介
针对许多扩展的存储系统，Spark已经有批处理的Connector，但是这些扩展的存储系统不是都有流的sink。*Flexible Streaming Sink*就是用来处理这个问题的，*streamingDF.writeStream.foreachBatch(...)* 允许用户使用batch data writer来写每个micbatch，举个例子🌰
```scala
//stream write into mysql
streamingDF.writeStream
.foreachBatch { (batchDF: DataFrame, batchId: Long) =>

    batchDF.write       // Use mysql batch data source to write streaming out
      .mode("append")
      .jdbc(url, tableName, prop)
  }
```
此外，用户也可以使用上述接口将micro-batch output写入其他Spark暂不支持的存储系统中，同时*foreachBatch*可以避免重复计算，将数据写入到不同的存储。
```scala
streamingDF.writeStream.foreachBatch { (batchDF: DataFrame, batchId: Long) =>
  batchDF.cache()
  batchDF.write.format(...).save(...)  // location 1
  batchDF.write.format(...).save(...)  // location 2
  batchDF.uncache()
}
```

### 后记
Spark2.4.0版本中的SS还是提升不少的，针对数据中的ETL场景还是非常适用的 
