## spark streaming应用在经历实际大数据时出现的问题

### 场景1 
> 如果不对应用进行限速设置，一般遇到数据洪流的时候，应用会出现一个或者多个的超大*batch*。
导致的结果就是应用一直会去处理这个（或者多个）超大的*batch*，而随后的*batch*延时处理。

解决方案：
```
  一般采用背压（backpressure）和限制每个分区的速度的方式对*spark streaming*应用进行限速
  spark.streaming.backpressure.enabled:true
  spark.streaming.kafka.maxRatePerPartition:n (record/s)
```

处理效果对比:

限速之前的*batch*:

![处理之前](../pictures/before.png "处理之前的batch")

限速之后的*batch*:

![处理之后](../pictures/after.png "处理之后的batch")

