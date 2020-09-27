Operator
===

Operator将多个数据流转换成一个

### 数据流转换
+ Map：DataStream → DataStream，一对一转换
```
DataStream<Integer> dataStream = //...
dataStream.map(new MapFunction<Integer, Integer>() {
    @Override
    public Integer map(Integer value) throws Exception {
        return 2 * value;
    }
});
```
+ FlatMap：DataStream → DataStream，一对多转换
```
dataStream.flatMap(new FlatMapFunction<String, String>() {
    @Override
    public void flatMap(String value, Collector<String> out)
        throws Exception {
        for(String word: value.split(" ")){
            out.collect(word);
        }
    }
});
```
+ Filter：DataStream → DataStream，过滤
```
dataStream.filter(new FilterFunction<Integer>() {
    @Override
    public boolean filter(Integer value) throws Exception {
        return value != 0;
    }
});
```
+ KeyBy：DataStream → KeyedStream：按key的hash分片，相同key被分到同一个分区
```
dataStream.keyBy("someKey") // Key by field "someKey"
dataStream.keyBy(0) // Key by the first element of a Tuple
```
+ Reduce KeyedStream → DataStream：连接将当前元素和最后reduce的结果，提交新的reduce
```
keyedStream.reduce(new ReduceFunction<Integer>() {
    @Override
    public Integer reduce(Integer value1, Integer value2)
    throws Exception {
        return value1 + value2;
    }
});
```
+ Fold：KeyedStream → DataStream，同reduce，类型不同
```
DataStream<String> result =
  keyedStream.fold("start", new FoldFunction<Integer, String>() {
    @Override
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
  });
```
+ Aggregations：KeyedStream → DataStream
```
keyedStream.sum(0);
keyedStream.sum("key");
keyedStream.min(0);
keyedStream.min("key");
keyedStream.max(0);
keyedStream.max("key");
keyedStream.minBy(0);
keyedStream.minBy("key");
keyedStream.maxBy(0);
keyedStream.maxBy("key");
```
+ Window：KeyedStream → WindowedStream
```
dataStream.keyBy(0).window(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
```
+ WindowAll：DataStream → AllWindowedStream
```
dataStream.windowAll(TumblingEventTimeWindows.of(Time.seconds(5))); // Last 5 seconds of data
```
+ Window Apply：WindowedStream → DataStream，AllWindowedStream → DataStream，聚合窗口数据为流
```
windowedStream.apply (new WindowFunction<Tuple2<String,Integer>, Integer, Tuple, Window>() {
    public void apply (Tuple tuple,
            Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});

// applying an AllWindowFunction on non-keyed window stream
allWindowedStream.apply (new AllWindowFunction<Tuple2<String,Integer>, Integer, Window>() {
    public void apply (Window window,
            Iterable<Tuple2<String, Integer>> values,
            Collector<Integer> out) throws Exception {
        int sum = 0;
        for (value t: values) {
            sum += t.f1;
        }
        out.collect (new Integer(sum));
    }
});
```
+ Window Reduce：WindowedStream → DataStream，对流reduce
```
windowedStream.reduce (new ReduceFunction<Tuple2<String,Integer>>() {
    public Tuple2<String, Integer> reduce(Tuple2<String, Integer> value1, Tuple2<String, Integer> value2) throws Exception {
        return new Tuple2<String,Integer>(value1.f0, value1.f1 + value2.f1);
    }
});
```
+ Window Fold：WindowedStream → DataStream，聚合窗口
```
windowedStream.fold("start", new FoldFunction<Integer, String>() {
    public String fold(String current, Integer value) {
        return current + "-" + value;
    }
});
```
+ Aggregations on windows：WindowedStream → DataStream，同上
```
windowedStream.sum(0);
windowedStream.sum("key");
windowedStream.min(0);
windowedStream.min("key");
windowedStream.max(0);
windowedStream.max("key");
windowedStream.minBy(0);
windowedStream.minBy("key");
windowedStream.maxBy(0);
windowedStream.maxBy("key");
```
+ Union：DataStream* → DataStream，合并多个流
```
dataStream.union(otherStream1, otherStream2, ...);
```
+ Window Join：DataStream,DataStream → DataStream，join多个流
```
dataStream.join(otherStream)
    .where(<key selector>).equalTo(<key selector>)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new JoinFunction () {...});
```
+ Interval Join：KeyedStream,KeyedStream → DataStream，按key join多个流
```
// this will join the two streams so that
// key1 == key2 && leftTs - 2 < rightTs < leftTs + 2
keyedStream.intervalJoin(otherKeyedStream)
    .between(Time.milliseconds(-2), Time.milliseconds(2)) // lower and upper bound
    .upperBoundExclusive(true) // optional
    .lowerBoundExclusive(true) // optional
    .process(new IntervalJoinFunction() {...});
```
+ Window CoGroup：DataStream,DataStream → DataStream
```
dataStream.coGroup(otherStream)
    .where(0).equalTo(1)
    .window(TumblingEventTimeWindows.of(Time.seconds(3)))
    .apply (new CoGroupFunction () {...});
```
+ Connect：DataStream,DataStream → ConnectedStreams，连接两个流，保留类型
```
DataStream<Integer> someStream = //...
DataStream<String> otherStream = //...

ConnectedStreams<Integer, String> connectedStreams = someStream.connect(otherStream);
```
+ CoMap, CoFlatMap：ConnectedStreams → DataStream，对连接的流做map、flatMap
```
connectedStreams.map(new CoMapFunction<Integer, String, Boolean>() {
    @Override
    public Boolean map1(Integer value) {
        return true;
    }

    @Override
    public Boolean map2(String value) {
        return false;
    }
});
connectedStreams.flatMap(new CoFlatMapFunction<Integer, String, String>() {

   @Override
   public void flatMap1(Integer value, Collector<String> out) {
       out.collect(value.toString());
   }

   @Override
   public void flatMap2(String value, Collector<String> out) {
       for (String word: value.split(" ")) {
         out.collect(word);
       }
   }
});
```
+ Split：DataStream → SplitStream，一个流拆分成多个流
```
SplitStream<Integer> split = someDataStream.split(new OutputSelector<Integer>() {
    @Override
    public Iterable<String> select(Integer value) {
        List<String> output = new ArrayList<String>();
        if (value % 2 == 0) {
            output.add("even");
        }
        else {
            output.add("odd");
        }
        return output;
    }
});
```
+ Select：SplitStream → DataStream，从拆分的流中选择一个流
```
SplitStream<Integer> split;
DataStream<Integer> even = split.select("even");
DataStream<Integer> odd = split.select("odd");
DataStream<Integer> all = split.select("even","odd");
```
+ Iterate：DataStream → IterativeStream → DataStream，迭代应用map逻辑，满足一定条件的元素反馈回迭代流中，其余元素输出到下游
```
IterativeStream<Long> iteration = initialStream.iterate();
DataStream<Long> iterationBody = iteration.map (/*do something*/);
DataStream<Long> feedback = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Integer value) throws Exception {
        return value > 0;
    }
});
iteration.closeWith(feedback);
DataStream<Long> output = iterationBody.filter(new FilterFunction<Long>(){
    @Override
    public boolean filter(Integer value) throws Exception {
        return value <= 0;
    }
});
```
+ Extract Timestamps：DataStream → DataStream，提取时间戳
```
stream.assignTimestamps (new TimeStampExtractor() {...});
```
+ Project：DataStream → DataStream，转换Truple类型的流
```
DataStream<Tuple3<Integer, Double, String>> in = // [...]
DataStream<Tuple2<String, Integer>> out = in.project(2,0);
```

### 物理分片
+ Custom partitioning：自定义分片
```
dataStream.partitionCustom(partitioner, "someKey");
dataStream.partitionCustom(partitioner, 0);
```
+ Random partitioning：随机分片
```
dataStream.shuffle();
```
+ Rebalancing (Round-robin partitioning)：通过轮询均衡的分片
```
dataStream.rebalance();
```
+ Rescaling：同一个上游Operator被均衡的分片到多个下游Operator集合中
```
dataStream.rescale();
```
+ Broadcasting：广播
```
dataStream.broadcast();
```

### 任务链和资源组
两个转换在同一个线程上性能更好，默认情况Flink会自动连接任务。也可以通过Api控制任务链，这些Api只能用于转换后。
禁用任务链：StreamExecutionEnvironment.disableOperatorChaining()
一个资源组是一个slot，可以手动隔离slot，从而隔离operator

+ 开启一个新的任务链，连接两个操作
```
someStream.filter(...).map(...).startNewChain().map(...);
```
+ 禁用连接
```
someStream.map(...).disableChaining();
```
+ 设置slot sharing group，同样slot sharing group上的操作运行在同一个slot上，不同slot sharing group上的操作运行在不同slot上
```
someStream.filter(...).slotSharingGroup("name");
```