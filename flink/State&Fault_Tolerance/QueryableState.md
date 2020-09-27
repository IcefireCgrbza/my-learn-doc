QueryableState
===

QueryableState用于从外部系统查询Flink的内部状态，功能还在升级中

### 架构
QueryableStateClient：从Flink外部提交查询请求的客户端
QueryableStateClientProxy：运行在TaskManager上的proxy，负责接收QueryableStateClient的查询，和JobManager交互，获取查询目标所在的TaskManager，请求该TaskManager上的QueryableStateServer获取响应
QueryableStateServer：运行在TaskManager上，负责这个TaskManager上的状态

### 开启Queryable State
+ Flink开启Queryable State：需要将flink-queryable-state-runtime_2.11-1.6.1.jar放到lib目录下，日志中有"Started the Queryable State Proxy Server @ ..."说明开启成功
+ 使用QueryableStateStream或stateDescriptor.setQueryable(String queryableStateName)声明State可查询

##### QueryableStateStream
KeyedStream上调用以下方法返回QueryableStateStream，作为Sink供外部查询使用
```
// ValueState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ValueStateDescriptor stateDescriptor)

// Shortcut for explicit ValueStateDescriptor variant
QueryableStateStream asQueryableState(String queryableStateName)

// FoldingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    FoldingStateDescriptor stateDescriptor)

// ReducingState
QueryableStateStream asQueryableState(
    String queryableStateName,
    ReducingStateDescriptor stateDescriptor)
```

##### 受Flink管理的State
使用StateDescriptor.setQueryable(String queryableStateName)设置Flink管理的State为可读
```
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
                "average", // the state name
                TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {})); // type information
descriptor.setQueryable("query-name"); // queryable state name
```

### 从外部查询State
首先需要QueryableStateClient的帮助类，依赖以下jar
```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-core</artifactId>
  <version>1.6.1</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-queryable-state-client-java_2.11</artifactId>
  <version>1.6.1</version>
</dependency>
```
使用proxy的hostname和端口初始化QueryableStateClient
```
QueryableStateClient client = new QueryableStateClient(tmHostname, proxyPort);
```
通过这个方法查询，
```
CompletableFuture<S> getKvState(
    JobID jobId,
    String queryableStateName,
    K key,
    TypeInformation<K> keyTypeInfo,
    StateDescriptor<S, V> stateDescriptor)
```

#####例子
声明State可查询，这个State存储了次数和总和
```
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    private transient ValueState<Tuple2<Long, Long>> sum; // a tuple containing the count and the sum

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {
        Tuple2<Long, Long> currentSum = sum.value();
        currentSum.f0 += 1;
        currentSum.f1 += input.f1;
        sum.update(currentSum);

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
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {})); // type information
        descriptor.setQueryable("query-name");
        sum = getRuntimeContext().getState(descriptor);
    }
}
```
查询
```
QueryableStateClient client = new QueryableStateClient(tmHostname, proxyPort);

// the state descriptor of the state to be fetched.
ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
        new ValueStateDescriptor<>(
          "average",
          TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {}));

CompletableFuture<ValueState<Tuple2<Long, Long>>> resultFuture =
        client.getKvState(jobId, "query-name", key, BasicTypeInfo.LONG_TYPE_INFO, descriptor);

// now handle the returned value
resultFuture.thenAccept(response -> {
        try {
            Tuple2<Long, Long> res = response.get();
        } catch (Exception e) {
            e.printStackTrace();
        }
});
```

### 配置
在QueryableStateOptions中定义配置：

+ query.server.ports: QueryableStateServer的端口，单一端口: “9123”, 端口范围: “50100-50200”, 端口列表: “50100-50200,50300-50400,51234”. 默认端口 9067.
+ query.server.network-threads: QueryableStateServer的网络线程数
+ query.server.query-threads: QueryableStateServer的处理线程数
Proxy
+ query.proxy.ports: proxy的端口，单一端口: “9123”, 端口范围: “50100-50200”, 端口列表: “50100-50200,50300-50400,51234”. 默认端口 9069.
+ query.proxy.network-threads: proxy的网络线程数
+ query.proxy.query-threads: proxy的处理线程数
