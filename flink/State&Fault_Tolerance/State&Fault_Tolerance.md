State & Fault Tolerance
===
State使用State backend存储状态，用于

+ 实现checkpoints容错，并提供savepoints
+ 在并行实例中重新分配任务和状态
+ 可从外部查询

### Keyed State 和 Operator State
+ Keyed State：用于KeyedStream的function和operator上。可看做（operator, key）。Keyed State被合成Key Groups，Key Groups是Flink中可被重新分配的最小单位，运行时每个Operator分到一个或多个Key Groups
+ Operator State：用于Operator

### Raw and Managed State
+ Managed State：Flink控制的数据结构
+ Raw State：自定义的数据结构，字节表示

### Managed Keyed State
这种类型的State使用于KeyedStream，通过stream.keyBy(..)生成，有以下类型：

+ ValueState<T>：update(T)更新，T value()获取
+ ListState<T>：add(T)、addAll(List<T>)新增，Iterable<T> get()获取，update(List<T>)重写
+ ReducingState<T>：存储reduce的结果，接口与ListState类似，使用ReduceFunction聚合数据
+ AggregatingState<IN, OUT>：存储聚合结果，接口与ListState类似，使用AggregateFunction聚合数据。不同于ReducingState，聚合结果和输入类型不同
+ FoldingState<T, ACC>：存储聚合结果，接口与ListState类似，使用FoldFunction聚合数据。不同于ReducingState，聚合结果和输入类型不同。废弃
+ MapState<UK, UV>：存储k-v，put(UK, UV)、putAll(Map<UK, UV>)新增，get(UK)获取，entries()、keys()、values()获取迭代器

获取State处理，需要新建StateDescriptor

+ ValueState<T> getState(ValueStateDescriptor<T>)
+ ReducingState<T> getReducingState(ReducingStateDescriptor<T>)
+ ListState<T> getListState(ListStateDescriptor<T>)
+ AggregatingState<IN, OUT> getAggregatingState(AggregatingState<IN, OUT>)
+ FoldingState<T, ACC> getFoldingState(FoldingStateDescriptor<T, ACC>)
+ MapState<UK, UV> getMapState(MapStateDescriptor<UK, UV>)

从RuntimeContext中获取State，只能用于rich function
```
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();

        // update the count
        currentSum.f0 += 1;

        // add the second field of the input value
        currentSum.f1 += input.f1;

        // update the state
        sum.update(currentSum);

        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(Configuration config) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}), // type information
                        Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}

// this can be used in a streaming program like this (assuming we have a StreamExecutionEnvironment env)
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(0)
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
```

### State Time-To-Live (TTL)
需要StateTtlConfig

+ TTL刷新时机：
	* StateTtlConfig.UpdateType.OnCreateAndWrite：新建和写入时
	* StateTtlConfig.UpdateType.OnReadAndWrite：读取、写入时
+ 读取策略：
	* StateTtlConfig.StateVisibility.NeverReturnExpired：超时不返回
	* StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp：超时但未被清理可返回
```
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
```
需要读取一次才会触发清理，可以主动请求全量的state快照，从而触发全量清理
```
StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Time.seconds(1))
    .cleanupFullSnapshot()
    .build();
```

### Managed Operator State
实现以下两个接口：
#### CheckpointedFunction
* void snapshotState(FunctionSnapshotContext context) throws Exception：checkpoint时调用，可用于保存状态到State
* void initializeState(FunctionInitializationContext context) throws Exception：初始化或者是从checkpoint恢复时调用，FunctionInitializationContext是上下文，可用于获取/新建State。isRestored()方法获取是否从Checkpoint恢复的信息

支持list风格的managed operator state，保存了List结构的序列化对象，互相之间都是独立的，有利于扩容时重分配。重分配方案定义如下：
	
+ Even-split redistribution：重新分配给每个Operator。FunctionInitializationContext.getOperatorStateStore()
+ Union redistribution：每个Operator获取全量。FunctionInitializationContext.getUnionListState()

Sink在发送前缓存的例子：
```
public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
                   CheckpointedFunction {

    private final int threshold;

    private transient ListState<Tuple2<String, Integer>> checkpointedState;

    private List<Tuple2<String, Integer>> bufferedElements;

    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<String, Integer> value) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() == threshold) {
            for (Tuple2<String, Integer> element: bufferedElements) {
                // send it to the sink
            }
            bufferedElements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.clear();
        for (Tuple2<String, Integer> element : bufferedElements) {
            checkpointedState.add(element);
        }
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
            new ListStateDescriptor<>(
                "buffered-elements",
                TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));

        checkpointedState = context.getOperatorStateStore().getListState(descriptor);

        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }
}
```

#### ListCheckpointed
list风格，even-split redistribution的CheckpointedFunction：

+ List<T> snapshotState(long checkpointId, long timestamp) throws Exception;
+ void restoreState(List<T> state) throws Exception;

有状态的Source Function：

```
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements ListCheckpointed<Long> {

    /**  current offset for exactly once semantics */
    private Long offset;

    /** flag for job cancellation */
    private volatile boolean isRunning = true;

    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();

        while (isRunning) {
            // output and state update are atomic
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

    @Override
    public List<Long> snapshotState(long checkpointId, long checkpointTimestamp) {
        return Collections.singletonList(offset);
    }

    @Override
    public void restoreState(List<Long> state) {
        for (Long s : state)
            offset = s;
    }
}
```