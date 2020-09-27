### The Broadcast State Pattern
Broadcast state需要被广播到所有下游任务中，下游任务使用它来处理另一个流的数据
```
//假设一个场景，一个Shape流，里面存有Color，另一个Rule流，指示怎样对Shape流进行处理
//比如，相同Color，rectangle和triangle连着

// key the shapes by color
KeyedStream<Item, Color> colorPartitionedStream = shapeStream
                        .keyBy(new KeySelector<Shape, Color>(){...});
                        
// a map descriptor to store the name of the rule (string) and the rule itself.
MapStateDescriptor<String, Rule> ruleStateDescriptor = new MapStateDescriptor<>(
			"RulesBroadcastState",
			BasicTypeInfo.STRING_TYPE_INFO,
			TypeInformation.of(new TypeHint<Rule>() {}));
		
// broadcast the rules and create the broadcast state
BroadcastStream<Rule> ruleBroadcastStream = ruleStream
                        .broadcast(ruleStateDescriptor);
                        
//非BroadcastStream可以通过connect方法，传入BroadcastStream可以连接两个流，得到KeyedBroadcastProcessFunction或BroadcastProcessFunction，通过process方法，传入CoProcessFunction，执行匹配逻辑
DataStream<Match> output = colorPartitionedStream
                 .connect(ruleBroadcastStream)
                 .process(
                     
                     // type arguments in our KeyedBroadcastProcessFunction represent: 
                     //   1. the key of the keyed stream
                     //   2. the type of elements in the non-broadcast side
                     //   3. the type of elements in the broadcast side
                     //   4. the type of the result, here a string
                     
                     new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                         // my matching logic
                     }
                 )
```

### BroadcastProcessFunction  和 KeyedBroadcastProcessFunction
+ BroadcastProcessFunction：接口源码可以简单读一下
```
public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {

    /**
     * 处理非广播的流
     */
    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    /**
     * 处理广播的流
     */
    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;
}
```
+ KeyedBroadcastProcessFunction：接口源码可以简单读一下
```
public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

    public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception;
}
```

ReadOnlyContext和Context的共性

+ 都可以读取BroadcastState: ctx.getBroadcastState(MapStateDescriptor<K, V> stateDescriptor)
+ 允许查询Timestamp：ctx.timestamp()
+ 获取Watermark: ctx.currentWatermark()
+ 获取Processing Time：ctx.currentProcessingTime()
+ 输出：ctx.output(OutputTag<X> outputTag, X value)

不同：

+ Context可读写
+ ReadOnlyContext只可读
+ KeyedBroadcastProcessFunction的processElement()方法的ReadOnlyContext可获取TimerService，可用于注册Event/Processing Time的timer。timer触发调用onTimer()，OnTimerContext与ReadOnlyContext类似，额外提供获取event time类型和key的接口
+ processBroadcastElement()的Context包含方法applyToKeyedState(StateDescriptor<S, VS> stateDescriptor,KeyedStateFunction<KS, S> function)，提供函数用于State

```
new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {

    // store partial matches, i.e. first elements of the pair waiting for their second element
    // we keep a list as we may have many first elements waiting
    private final MapStateDescriptor<String, List<Item>> mapStateDesc =
	    new MapStateDescriptor<>(
	        "items",
	        BasicTypeInfo.STRING_TYPE_INFO, 
	        new ListTypeInfo<>(Item.class));

    // identical to our ruleStateDescriptor above
    private final MapStateDescriptor<String, Rule> ruleStateDescriptor = 
        new MapStateDescriptor<>(
    	    "RulesBroadcastState",
    		BasicTypeInfo.STRING_TYPE_INFO,
    		TypeInformation.of(new TypeHint<Rule>() {}));

	@Override
	public void processBroadcastElement(Rule value, 
	                                    Context ctx, 
	                                    Collector<String> out) throws Exception {
	    ctx.getBroadcastState(ruleStateDescriptor).put(value.name, value);
	}

	@Override
	public void processElement(Item value, 
	                           ReadOnlyContext ctx, 
	                           Collector<String> out) throws Exception {

        final MapState<String, List<Item>> state = getRuntimeContext().getMapState(mapStateDesc);
        final Shape shape = value.getShape();
    
        for (Map.Entry<String, Rule> entry: 
                ctx.getBroadcastState(ruleStateDescriptor).immutableEntries()) {
            final String ruleName = entry.getKey();
            final Rule rule = entry.getValue();
    
            List<Item> stored = state.get(ruleName);
            if (stored == null) {
                stored = new ArrayList<>();
            }
    
            if (shape == rule.second && !stored.isEmpty()) {
                for (Item i : stored) {
                    out.collect("MATCH: " + i + " - " + value);
                }
                stored.clear();
            }
    
            // there is  no else{} to cover if rule.first == rule.second
            if (shape.equals(rule.first)) {
                stored.add(value);
            }
    
            if (stored.isEmpty()) {
                state.remove(ruleName);
            } else {
                state.put(ruleName, stored);
            }
        }
	}
}
```