Event Time
===

Flink提供不一样的时间：
+ Processing time：机器处理的系统时间。最高效，延迟最低。不保证事件延迟到达、不同Operator上的窗口不统一
+ Event time：Event生成的时间。保证Event顺序，保证一致的结果。需要指定如何生成Watermarks
+ Ingestion time：Event进入Flink的时间，由source operator生成。比Processing time代价高，但能保证结果一致。无法处理乱序和迟到的Event，能自动生成时间戳和Watermarks
![](images/EventTime1.png)

设置使用哪种时间
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);

// alternatively:
// env.setStreamTimeCharacteristic(TimeCharacteristic.IngestionTime);
// env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<MyEvent> stream = env.addSource(new FlinkKafkaConsumer09<MyEvent>(topic, schema, props));

stream
    .keyBy( (event) -> event.getUser() )
    .timeWindow(Time.hours(1))
    .reduce( (a, b) -> a.add(b) )
    .addSink(...);
```

Event Time 和 Watermarks
---
Watermarks用于衡量Event Time进度。Watermarks是数据流的一部分，通过携带时间戳t，说明流已经到达了t的时间节点，Operator调整内部时钟，将所有小于t的Event都被丢弃。这对乱序的流非常重要
![](images/Watermarks1.png)

并行流中的Watermarks
每个Source Operator独立生成Watermarks。当Watermarks到达Operator，向下游生成一个新的Watermarks。Operator消费多个输入流时，使用多个流中最小的Watermarks

迟到的事件
---
允许一定程度的延迟