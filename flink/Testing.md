Testing
===

### 单元测试
单元测试主要测试函数中的业务逻辑

比如测试：
```
public class SumReduce implements ReduceFunction<Long> {

    @Override
    public Long reduce(Long value1, Long value2) throws Exception {
        return value1 + value2;
    }
}
```
```
public class SumReduceTest {

    @Test
    public void testSum() throws Exception {
        // instantiate your function
        SumReduce sumReduce = new SumReduce();

        // call the methods that you have implemented
        assertEquals(42L, sumReduce.reduce(40L, 2L));
    }
}
```

### 集成测试
需要一个本地的Flink小集群

依赖以下内容：
```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-test-utils_2.11</artifactId>
  <version>1.6.1</version>
</dependency>
```

比如测试：
```
public class MultiplyByTwo implements MapFunction<Long, Long> {

    @Override
    public Long map(Long value) throws Exception {
        return value * 2;
    }
}
```
```
public class ExampleIntegrationTest extends AbstractTestBase {

    @Test
    public void testMultiply() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // configure your test environment
        env.setParallelism(1);

        // values are collected in a static variable
        CollectSink.values.clear();

        // create a stream of custom elements and apply transformations
        env.fromElements(1L, 21L, 22L)
                .map(new MultiplyByTwo())
                .addSink(new CollectSink());

        // execute
        env.execute();

        // verify your results
        assertEquals(Lists.newArrayList(2L, 42L, 44L), CollectSink.values);
    }

    // create a testing sink
    private static class CollectSink implements SinkFunction<Long> {

        // must be static
        public static final List<Long> values = new ArrayList<>();

        @Override
        public synchronized void invoke(Long value) throws Exception {
            values.add(value);
        }
    }
}
```

### 测试checkpoint和state处理
https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/stream/testing.html#testing-checkpointing-and-state-handling