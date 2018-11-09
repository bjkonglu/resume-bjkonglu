## spark streaming应用在经历实际大数据时出现的问题

### 场景1 
> 如果不对应用进行限速设置，一般遇到数据洪流的时候，应用会出现一个或者多个的超大*batch*。
导致的结果就是应用一直会去处理这个（或者多个）超大的*batch*，而随后的*batch*延时处理。

解决方案：
```
  一般采用背压（backpressure）和限制每个分区的速度的方式对spark streaming应用进行限速
  spark.streaming.backpressure.enabled:true
  spark.streaming.kafka.maxRatePerPartition:n (record/s)
```

处理效果对比:

限速之前的*batch*:

![处理之前](../../pics/rate-limiting-before.png "处理之前的batch")

限速之后的*batch*:

![处理之后](../../pics/rate-limiting-after.png "处理之后的batch")

### 场景2
> 增加应用的资源(number.executors,executor.cores)，但是应用的每个批次(*batch*)的处理时间反而没有减少，
还有个现象是应用没有能充分使用executor（例如应用本身占有120个executor，但是最多只能使用50个executor。）

解决方案:
```
  每个spark streaming应用的每个批次(batch)处于"活动"状态的Task数量取决于RDD的partition数量,
  不提高应用的分区数是无法提 任务的并行度，继而无法充分使用扩充的资源(number.executor)。
  比较推荐的解决方案是增加数据源的分区数。
```
处理效果对比：

增加分区(*batch*)之前:

![处理之前](../../pics/add-partition-before.png "未增加数据源分区数量时的处理时间和延时图")


### 场景3
> 在实际应用中，spark streaming应用会出现接入多个数据源(*InputDStream*)和多个输出操作(*OutputDStream*)的需求，
这个时候spark streaming引擎会会遍历输出操作(*OutputDStream*)，然后处理这个输出操作的的*RDD*，依次迭代完所有的
输出操作。

### 场景4
> 在实际应用中，spark streaming应用通过参数(spark.executor.instances)设置executor的个数，但是在spark UI上看到
executor多于设置的数量？

