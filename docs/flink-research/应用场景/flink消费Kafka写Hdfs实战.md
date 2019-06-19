## flink消费Kafka写Hdfs实战

### 背景
在我们用实时计算处理实时报表需求的时候，往往到最后都需要进行数据校对。这个时候我们就需要将消息队列（Apache Kafka）的数据落地（HDFS），然后校对实时计算指标与离线计算指标，这种方式也就是我们经常听到的大数据里面的Labmda架构。下面跟大家分享一下怎么使用flink将Kafka里面的数据写入到Hdfs。

### 操作步骤

#### 前置条件
首先，我们得添加flink提供的一个connector，这个connector名叫*flink-connector-filesystem_2.11*，添加依赖的方式如下：

```java
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-filesystem_2.11</artifactId>
    <version>${flink.version}</version>
</dependency>
```

#### Flink BucketingSink
在*flink-connector-filesystem_2.11*里面提供一个数据汇（BucketingSink），我们可以利用这个数据汇来将数据写入Hdfs，它是通过数据到达数据汇的时间作为分桶的条件，例如数据A到达数据汇的时间是12:45，则被分配到2019-6-19--12这个桶里面，而当数据B到达数据汇的时间是13:01，则被分配到2019-6-19--13这哥桶里面。
```java
  DataStream<String> input = ...;
  input.addSink(new BucketingSink("path/to/store"));
```

此外，BucketingSink还提供很多高级设置，例如可以配置自定义的bucket、writer和批的大小。默认情况下，BucketingSink是通过数据到达系统的时间进行切分的，并使用“yyyy-MM-dd--HH”的时间格式来命名桶。
```java
DateTimeBucketer line 93

	@Override
	public Path getBucketPath(Clock clock, Path basePath, T element) {
		String newDateTimeString = dateFormatter.format(new Date(clock.currentTimeMillis()));
		return new Path(basePath + "/" + newDateTimeString);
	}
```
这个时间格式格式化当前的系统时间来形成一个桶的路径，当遇到一个新的时间后会创建一个新的桶。每个分桶本身包含若干分区文件，分区的数量与数据汇（BucketingSink）的并行度相同，也就是说，每个并行的BucketingSink实例都会创建它的分区文件，当分区文件过大时，BucketingSink会紧接着它的分区文件再创建一个新的分区文件。有趣是时，BucketingSink会周期检测不活跃的桶，不活跃的桶会被刷新并关闭。默认情况下，BucketingSink会每分钟检查一遍是否活跃，并关闭超过一分钟没有写入数据的分桶，我们也可以自定义检查周期和不活跃的时间阈值
  - BucketingSink.setInactiveBucketCheckInterval()
  - BucketingSink.setInactiveBucketThreshold()

最后，我们介绍一下自定义writer、bucket以及设置批的大小
```java
  DataStream<String> input = ...;
  
  BucketingSink<String> sink = new BucketingSink("path/to/store");
  sink.setBucketer(new DateTimeBucketer<String>("yyyy-MM-dd--HH"));
  sink.setWriter(new SequenceFileWriter<IntWriter, Text>());
  sink.setBatchSize(600 * 1024 * 1024) // 600M
  
  input.addSink(sink);
```

写入到文件系统中文件的格式：/{basePath}/{date-time}/part-{parallel-task}-count
  - basePath： 设置的文件路径
  - date-time：分桶器格式化的时间字符串
  - parallel-task： 并行BucketingSink的索引
  - count：分区文件的运行编号，这个运行编号是由于分区文件大小超过设置的大小导致的
  
