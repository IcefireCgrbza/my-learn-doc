Side Outputs
===

Side Outputs指的是主流外的输出流，可以通过主流复制并过滤生成

使用Side Outputs需要定义OutputTag：
```
// this needs to be an anonymous inner class, so that we can analyze the type
OutputTag<String> outputTag = new OutputTag<String>("side-output") {};
```
可以从以下函数提交数据：
+ ProcessFunction
+ CoProcessFunction
+ ProcessWindowFunction
+ ProcessAllWindowFunction

```
DataStream<Integer> input = ...;

final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = input
  .process(new ProcessFunction<Integer, Integer>() {

      @Override
      public void processElement(
          Integer value,
          Context ctx,
          Collector<Integer> out) throws Exception {
        // emit data to regular output
        out.collect(value);

        // emit data to side output
        ctx.output(outputTag, "sideout-" + String.valueOf(value));
      }
    });
```

获取Side Output：
```
final OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<Integer> mainDataStream = ...;

DataStream<String> sideOutputStream = mainDataStream.getSideOutput(outputTag);
```