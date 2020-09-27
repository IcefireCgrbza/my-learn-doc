生成Timestamps / Watermarks
===

只有Event Time需要指定如何生成Timestamp和Watermarks，首先流处理程序需要使用Event Time
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);
```

### 提取Timestamps
有两种方式提取Timestamps，生成Watermarks：
+ 在数据流的Source中
+ 通过Timestamp Assigner和Watermark Generator
Timestamp和Watermark都是毫秒级的

#### Source Function生成Timestamp和Watermarks
Source可以直接从流中的数据提取Timestamp，还可以发送Watermark。这样就不需要Timestamp Assigner了。如果使用了Timestamp Assigner，将覆盖Source提供的。
例子：
```
@Override
public void run(SourceContext<MyType> ctx) throws Exception {
	while (/* condition */) {
		MyType next = getNext();
		ctx.collectWithTimestamp(next, next.getEventTimestamp());

		if (next.hasWatermarkTime()) {
			ctx.emitWatermark(new Watermark(next.getWatermarkTime()));
		}
	}
}
```

#### Timestamp Assigners / Watermark Generators
Timestamp Assigner获取流并生成一个带Timestamp和Watermark的新的流。
通常，Timestamp Assigner在流处理的最前面，但不是严格要求这么做。场景的例子，MapFunction和FilterFunction可以作用在Timestamp Assigner的前面。也就是说，Timestamp Assigner必须在使用Event Time的第一个Operator之前指定
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.readFile(
        myFormat, myFilePath, FileProcessingMode.PROCESS_CONTINUOUSLY, 100,
        FilePathFilter.createDefaultFilter(), typeInfo);

DataStream<MyEvent> withTimestampsAndWatermarks = stream
        .filter( event -> event.severity() == WARNING )
        .assignTimestampsAndWatermarks(new MyTimestampsAndWatermarks());

withTimestampsAndWatermarks
        .keyBy((event) -> event.getGroup())
        .timeWindow(Time.seconds(10))
        .reduce( (a, b) -> a.add(b) )
        .addSink(...);
```

#### Periodic Watermarks
AssignerWithPeriodicWatermarks周期性提取Timestamp并生成Watermarks
通过ExecutionConfig.setAutoWatermarkInterval(...)定义生成Watermarks的时间间隔，Assigner的getCurrentWatermark()每次都会被调用，只有当返回的Watermarks不是null且大于之前的Watermark时，才会提交这个watermark
```
/**
 * 生成的Watermark = 当前到达最大的Event Time - 3.5second
 */
public class BoundedOutOfOrdernessGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

    private final long maxOutOfOrderness = 3500; // 3.5 seconds

    private long currentMaxTimestamp;

    @Override
    public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
        long timestamp = element.getCreationTime();
        currentMaxTimestamp = Math.max(timestamp, currentMaxTimestamp);
        return timestamp;
    }

    @Override
    public Watermark getCurrentWatermark() {
        // return the watermark as current highest timestamp minus the out-of-orderness bound
        return new Watermark(currentMaxTimestamp - maxOutOfOrderness);
    }
}

/**
 * 生成的Watermark = 当前时间 - 5 second
 */
public class TimeLagWatermarkGenerator implements AssignerWithPeriodicWatermarks<MyEvent> {

	private final long maxTimeLag = 5000; // 5 seconds

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark getCurrentWatermark() {
		// return the watermark as current time minus the maximum time lag
		return new Watermark(System.currentTimeMillis() - maxTimeLag);
	}
}
```

#### Punctuated Watermarks
AssignerWithPunctuatedWatermarks先调用extractTimestamp(...)提取Timestamp，然后将数据和Timestamp传给checkAndGetNextWatermark(...)，由这个方法判断是否生成Watermark
返回null或小于当前Watermark则不采用
```
public class PunctuatedAssigner implements AssignerWithPunctuatedWatermarks<MyEvent> {

	@Override
	public long extractTimestamp(MyEvent element, long previousElementTimestamp) {
		return element.getCreationTime();
	}

	@Override
	public Watermark checkAndGetNextWatermark(MyEvent lastElement, long extractedTimestamp) {
		return lastElement.hasWatermarkMarker() ? new Watermark(extractedTimestamp) : null;
	}
}
``` 

#### Timestamps per Kafka Partition
使用kafka作为数据源时，每个kafka分区有一个Event Time模式（上升的），当消费时，多个分区可能被并行处理，这样对破坏单分区的模式
我们可以使用Kafka-partition-aware watermark generation。使用这个特性，watermarks在kafka消费者内部生成，每个分区的Watermarks合并会像流做shuffles时的合并一样
例如，如果kafka分配的Event Timestamp严格上升，使用AscendingTimestampExtractor可以保证完美的整体Watermark
```
FlinkKafkaConsumer09<MyType> kafkaSource = new FlinkKafkaConsumer09<>("myTopic", schema, props);
kafkaSource.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyType>() {

    @Override
    public long extractAscendingTimestamp(MyType element) {
        return element.eventTimestamp();
    }
});

DataStream<MyType> stream = env.addSource(kafkaSource);
```

#### Flink提供的Timestamp Extractors和Watermark Emitters
+ AscendingTimestampExtractor：上升的时间戳
```
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new AscendingTimestampExtractor<MyEvent>() {

        @Override
        public long extractAscendingTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
```
+ BoundedOutOfOrdernessTimestampExtractor：可容忍固定延迟
```
DataStream<MyEvent> stream = ...

DataStream<MyEvent> withTimestampsAndWatermarks =
    stream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor<MyEvent>(Time.seconds(10)) {

        @Override
        public long extractTimestamp(MyEvent element) {
            return element.getCreationTime();
        }
});
```