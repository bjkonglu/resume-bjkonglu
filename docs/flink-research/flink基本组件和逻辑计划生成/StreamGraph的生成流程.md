## StreamGraph的生成流程
> 文章基于[flink作业提交流程](./flink作业提交流程.md)， 建议先阅读flink作业提交流程文章再阅读该文章。该篇文章主要介绍Flink任务如何构建StreamGraph。

### 一个例子
介绍一个从Kafka消费，经过Flink处理，然后又写回Kafka的例子[完整的代码](https://github.com/apache/flink/blob/release-1.6/flink-examples/flink-examples-streaming/src/main/scala/org/apache/flink/streaming/scala/examples/kafka/Kafka010Example.scala)。我们看一下代码里面主要逻辑：
```scala
    val env = ...
    val kafkaConsumer = ...
    val messageStream = env.addSource(kafkaConsumer)
      .map(in => prefix + in)
    val kafkaProducer = ...
    messageStream.addSink(kafkaProducer)
    
    env.execute("Kafka010Demo")
```
在上篇文章中，我们知道Flink的客户端最后会通过Java的反射机制调用应用的代码的*main(String[] args)* 方法， 即会执行上述代码块逻辑。上述代码从整体来划分，可以分为三部分：
  - 应用上下文环境的创建，env
  - StreamGraph的构建过程
  - JobGraph的构建和提交过程
 
我们主要介绍StreamGraph的构建过程，在上述代码块中stream主要三个组件：
  - source: kafkaConsumer
  - map: Map
  - sink: kafkaProducer
  
在执行*env.addSource(kafkaConsumer)* 时，底层实现如下，主要是返回DataStreamSource
```java
public <OUT> DataStreamSource<OUT> addSource(SourceFunction<OUT> function, String sourceName, TypeInformation<OUT> typeInfo) {

		if (typeInfo == null) {
			if (function instanceof ResultTypeQueryable) {
				typeInfo = ((ResultTypeQueryable<OUT>) function).getProducedType();
			} else {
				try {
					typeInfo = TypeExtractor.createTypeInfo(
							SourceFunction.class,
							function.getClass(), 0, null, null);
				} catch (final InvalidTypesException e) {
					typeInfo = (TypeInformation<OUT>) new MissingTypeInfo(sourceName, e);
				}
			}
		}

		boolean isParallel = function instanceof ParallelSourceFunction;

		clean(function);
		StreamSource<OUT, ?> sourceOperator;
		if (function instanceof StoppableFunction) {
			sourceOperator = new StoppableStreamSource<>(cast2StoppableSourceFunction(function));
		} else {
			sourceOperator = new StreamSource<>(function);
		}

		return new DataStreamSource<>(this, typeInfo, sourceOperator, isParallel, sourceName);
}
```
接着，便是执行*map(..)* 操作，具体的执行逻辑如下：
```java
  1.
  public <R> SingleOutputStreamOperator<R> map(MapFunction<T, R> mapper) {

		TypeInformation<R> outType = TypeExtractor.getMapReturnTypes(clean(mapper), getType(),
				Utils.getCallLocationName(), true);

		return transform("Map", outType, new StreamMap<>(clean(mapper)));
	}
  2.
  public <R> SingleOutputStreamOperator<R> transform(String operatorName, TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

		// read the output type of the input Transform to coax out errors about MissingTypeInfo
		transformation.getOutputType();

    // 将当前转换操作链接前一个转换操作
		OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
				this.transformation,
				operatorName,
				operator,
				outTypeInfo,
				environment.getParallelism());

		@SuppressWarnings({ "unchecked", "rawtypes" })
		SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);

    // 然后将转换操作（除去source操作）存入转换列表容器
		getExecutionEnvironment().addOperator(resultTransform);

		return returnStream;
	}
```
最后便是*addSink(..)* 操作，具体的执行逻辑如下：
```java
public DataStreamSink<T> addSink(SinkFunction<T> sinkFunction) {

		// read the output type of the input Transform to coax out errors about MissingTypeInfo
		transformation.getOutputType();

		// configure the type if needed
		if (sinkFunction instanceof InputTypeConfigurable) {
			((InputTypeConfigurable) sinkFunction).setInputType(getType(), getExecutionConfig());
		}

		StreamSink<T> sinkOperator = new StreamSink<>(clean(sinkFunction));

    // 将当前转换操作链接前一个转换操作
		DataStreamSink<T> sink = new DataStreamSink<>(this, sinkOperator);
    // 然后将转换操作（除去source操作）存入转换列表容器
		getExecutionEnvironment().addOperator(sink.getTransformation());
		return sink;
	}
```
