---
date: 2019-04-30
title: "flink-wordcount-debug"
author: "邓子明"
tags:
    - kafka
    - 源码
categories:
    - kafka源码
comment: true
---

# 代码

```
DataStream<String> text = env.fromElements(WordCountData.WORDS);

DataStream<Tuple2<String, Integer>> counts = text.flatMap(new Tokenizer()).keyBy(0).sum(1);

counts.print();

env.execute("Streaming WordCount");
```

# env.fromElements(WordCountData.WORDS)

暂且忽略反射得到数据类型的步骤，直接到 `fromCollection(Arrays.asList(data), typeInfo);`

```java
public <OUT> DataStreamSource<OUT> fromCollection(Collection<OUT> data, TypeInformation<OUT> typeInfo) {
	
	SourceFunction<OUT> function = new FromElementsFunction<>(typeInfo.createSerializer(getConfig()), data);
	return addSource(function, "Collection Source", typeInfo).setParallelism(1);
}
```

第一步骤得到一个 FromElementsFunction ，大概看一下 FromElementsFunction 的继承体系：

```java
SourceFunction (org.apache.flink.streaming.api.functions.source)
	ProcessingTimeServiceSource in StreamSourceOperatorLatencyMetricsTest (org.apache.flink.streaming.runtime.operators)
	MockSourceFunction in StreamTaskTest (org.apache.flink.streaming.runtime.tasks)
	SocketTextStreamFunction (org.apache.flink.streaming.api.functions.source)
	FiniteTestSource (org.apache.flink.streaming.util)
	CheckpointingNonParallelSourceWithListState in MigrationTestUtils (org.apache.flink.test.checkpointing.utils)
	FromIteratorFunction (org.apache.flink.streaming.api.functions.source)
	NonSerializableTupleSource in StreamingOperatorsITCase (org.apache.flink.test.streaming.api)
	Generator in StreamingFileSinkProgram ()
	SimpleStringGenerator in CheckpointedStreamingProgram (org.apache.flink.test.classloading.jar)
	FailingCollectionSource (org.apache.flink.table.runtime.utils)
	Generator in BucketingSinkTestProgram (org.apache.flink.streaming.tests)
	SimpleSource in AsyncIOExample (org.apache.flink.streaming.examples.async)
	RichSourceFunction (org.apache.flink.streaming.api.functions.source)
	FileMonitoringFunction (org.apache.flink.streaming.api.functions.source)
	RandomFibonacciSource in IterateExample (org.apache.flink.streaming.examples.iteration)
	WaitingTestSourceFunction in RollingSinkITCase (org.apache.flink.streaming.connectors.fs)
	TupleSource in StreamingOperatorsITCase (org.apache.flink.test.streaming.api)
	PythonGeneratorFunction (org.apache.flink.streaming.python.api.functions)
	FromElementsFunction (org.apache.flink.streaming.api.functions.source)
... 后面的太多放不下
```

都实现 SourceFunction 接口，接口里一个 run 一个 cancel 。
FromElementsFunction 直接实现了 SourceFunction，它的构造方法传入了一个 elements 的 Iterable，
它的 run 方法循环调用 next = serializer.deserialize(input) 和 ctx.collect(next) 。
可以看出它的 run 方法就 类似 spark 的 RDD 的 compute 方法。
这里的 ctx.collect(next) 类似 mapreduce 的 context.write() 方法。

然后调用 `addSource(function, "Collection Source", typeInfo).setParallelism(1)`

```java
public <OUT> DataStreamSource<OUT> addSource(SourceFunction<OUT> function, String sourceName, TypeInformation<OUT> typeInfo) {

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

我们先只看 StreamSource ，他继承自 AbstractUdfStreamOperator 继承自 AbstractStreamOperator 实现了 StreamOperator ，

```java
StreamOperator
	TwoInputStreamOperator (org.apache.flink.streaming.api.operators)
	OneInputStreamOperator (org.apache.flink.streaming.api.operators)
	AbstractStreamOperator (org.apache.flink.streaming.api.operators)
		WatermarkReifier in testWatermarkForwarding() in SideOutputITCase (org.apache.flink.test.streaming.runtime)
		TestOperator in OneInputStreamTaskTest (org.apache.flink.streaming.runtime.tasks)
		AggregatingTestOperator in AbstractQueryableStateTestBase (org.apache.flink.queryablestate.itcases)
		TestingStreamOperator in OneInputStreamTaskTest (org.apache.flink.streaming.runtime.tasks)
		WindowOperator (org.apache.flink.table.runtime.window)
		AbstractUdfStreamOperator (org.apache.flink.streaming.api.operators)
		CheckingTimelyStatefulOperator in LegacyStatefulJobSavepointMigrationITCase (org.apache.flink.test.checkpointing.utils)
		ValidatingOperator in OperatorChainTest (org.apache.flink.streaming.runtime.tasks)
		DuplicatingOperator in OneInputStreamTaskTest (org.apache.flink.streaming.runtime.tasks)
...太多了
```

相关方法我们也不知道，所以暂且不看，只能看出 run 方法比较核心。调用了 `StreamSource userFunction.run(ctx)`，
也就是上面的 FromElementsFunction 的 run。

然后返回 `new DataStreamSource<>(this, typeInfo, sourceOperator, isParallel, sourceName)`

DataStreamSource 继承自  SingleOutputStreamOperator 继承自 DataStream 。
```java
DataStream (org.apache.flink.streaming.api.datastream)
	DataStreamMock in Elasticsearch6UpsertTableSinkFactoryTest (org.apache.flink.streaming.connectors.elasticsearch6)
	SingleOutputStreamOperator (org.apache.flink.streaming.api.datastream)
		IterativeStream (org.apache.flink.streaming.api.datastream)
		DataStreamSource (org.apache.flink.streaming.api.datastream)
	KeyedStream (org.apache.flink.streaming.api.datastream)
	SplitStream (org.apache.flink.streaming.api.datastream)
	DataStreamMock in KafkaTableSourceSinkFactoryTestBase (org.apache.flink.streaming.connectors.kafka)
```

DataStream 是比较关键的一个类，类似 spark 的 RDD，他有两个属性：

```java
	protected final StreamExecutionEnvironment environment;
	protected final StreamTransformation<T> transformation;
```

它持有 environment 对象，还有一个 StreamTransformation 对象，这个对象类似 spark RDD 的 compute 方法，包含了用户的逻辑。
StreamTransformation 实现各不相同，有读文件的，由用户自定义的，有保存的，分区的，shuffle 的，继承体系如下：

```java
StreamTransformation (org.apache.flink.streaming.api.transformations)
	SplitTransformation (org.apache.flink.streaming.api.transformations)
	StreamTransformationMock in KafkaTableSourceSinkFactoryTestBase (org.apache.flink.streaming.connectors.kafka)
	SinkTransformation (org.apache.flink.streaming.api.transformations)
	PartitionTransformation (org.apache.flink.streaming.api.transformations)
	UnionTransformation (org.apache.flink.streaming.api.transformations)
	TwoInputTransformation (org.apache.flink.streaming.api.transformations)
	MockStreamTransformation in InfiniteStringsGenerator in DataGenerators (org.apache.flink.streaming.connectors.kafka.testutils)
	StreamTransformationMock in Elasticsearch6UpsertTableSinkFactoryTest (org.apache.flink.streaming.connectors.elasticsearch6)
	FeedbackTransformation (org.apache.flink.streaming.api.transformations)
	OneInputTransformation (org.apache.flink.streaming.api.transformations)
	SourceTransformation (org.apache.flink.streaming.api.transformations)
	SelectTransformation (org.apache.flink.streaming.api.transformations)
	SideOutputTransformation (org.apache.flink.streaming.api.transformations)
	CoFeedbackTransformation (org.apache.flink.streaming.api.transformations)
````

到目前为止我们接触了 4 几个继承体系。

首先是 SourceFunction，就是对我们的自定义 UDF 函数的封装。

然后是 StreamOperator 应该也是处理的逻辑，有的是对 Function 的封装，也有的是别的处理逻辑，例如 join 等。

然后是 DataStream ，这个就类似 spark 的 RDD，内部有一个 StreamTransformation 属性，包含了处理的逻辑。

最后是 StreamTransformation，它的子类一共十几个，是 DataStream 的属性。

知道了这些以后，我们再看一遍代码，熟悉一下他们之间的关系。

## text.flatMap(new Tokenizer())

这是对 DataStream 类的一次转换，生成一个新的 DataStream，
我们查看核心的代码就一行：`transform("Flat Map", outType, new StreamFlatMap<>(clean(flatMapper)))`

第一步 new StreamFlatMap ，继承体系和 StreamSource 一样。
StreamSource 是 Stream 的 源头，所以它有一个 run 方法。而 StreamFlatMap 是对 Stream 的转换，核心方法是  processElement 。

第二步是调用 transform 
```java
@PublicEvolving
public <R> SingleOutputStreamOperator<R> transform(String operatorName, TypeInformation<R> outTypeInfo, OneInputStreamOperator<T, R> operator) {

	// read the output type of the input Transform to coax out errors about MissingTypeInfo
	transformation.getOutputType();

	OneInputTransformation<T, R> resultTransform = new OneInputTransformation<>(
			this.transformation,
			operatorName,
			operator,
			outTypeInfo,
			environment.getParallelism());

	@SuppressWarnings({ "unchecked", "rawtypes" })
	SingleOutputStreamOperator<R> returnStream = new SingleOutputStreamOperator(environment, resultTransform);

	getExecutionEnvironment().addOperator(resultTransform);

	return returnStream;
}
```

首先是 OneInputTransformation，和 上面已经介绍过的 SourceTransformation 一样，继承自 StreamTransformation。

然后返回一个 SingleOutputStreamOperator，和上面的 DataStreamSource 一样，继承自 DataStream。

这里可以看出还是上面的四个继承体系。

### TypeInformation

上面我们的分析都忽略了获得结果类型的操作，但是这在后面的分析是很有用的，从这一点可以看出 flink 整体设计还是不如 spark 的，
还需要手动获得信息。

第一次是：`typeInfo = TypeExtractor.getForObject(data[0])`，完整的方法：

```java
private <X> TypeInformation<X> privateGetForObject(X value) {
	checkNotNull(value);

	// check if type information can be produced using a factory
	final ArrayList<Type> typeHierarchy = new ArrayList<>();
	typeHierarchy.add(value.getClass());
	final TypeInformation<X> typeFromFactory = createTypeInfoFromFactory(value.getClass(), typeHierarchy, null, null);
	if (typeFromFactory != null) {
		return typeFromFactory;
	}

	// check if we can extract the types from tuples, otherwise work with the class
	if (value instanceof Tuple) {
		Tuple t = (Tuple) value;
		int numFields = t.getArity();
		if(numFields != countFieldsInClass(value.getClass())) {
			// not a tuple since it has more fields.
			return analyzePojo((Class<X>) value.getClass(), new ArrayList<Type>(), null, null, null); // we immediately call analyze Pojo here, because
			// there is currently no other type that can handle such a class.
		}

		TypeInformation<?>[] infos = new TypeInformation[numFields];
		for (int i = 0; i < numFields; i++) {
			Object field = t.getField(i);

			if (field == null) {
				throw new InvalidTypesException("Automatic type extraction is not possible on candidates with null values. "
						+ "Please specify the types directly.");
			}

			infos[i] = privateGetForObject(field);
		}
		return new TupleTypeInfo(value.getClass(), infos);
	}
	else if (value instanceof Row) {
		Row row = (Row) value;
		int arity = row.getArity();
		for (int i = 0; i < arity; i++) {
			if (row.getField(i) == null) {
				LOG.warn("Cannot extract type of Row field, because of Row field[" + i + "] is null. " +
					"Should define RowTypeInfo explicitly.");
				return privateGetForClass((Class<X>) value.getClass(), new ArrayList<Type>());
			}
		}
		TypeInformation<?>[] typeArray = new TypeInformation<?>[arity];
		for (int i = 0; i < arity; i++) {
			typeArray[i] = TypeExtractor.getForObject(row.getField(i));
		}
		return (TypeInformation<X>) new RowTypeInfo(typeArray);
	}
	else {
		return privateGetForClass((Class<X>) value.getClass(), new ArrayList<Type>());
	}
}
```




