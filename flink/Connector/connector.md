流Connector
===
### 预定义的Source和Sink
Source包括读文件、socket、collections、iterators
Sink包括写文件、socket、stdout、stderr

### 封装Connector
Flink提供以下Connector负责和连接第三方存储或消息队列：

+ kafka(source/sink)
+ Cassandra
+ Einesis stream
+ Elasticsearch(sink)
+ Hadoop(sink)
+ RabbitMQ
+ NiFi
+ Twitter Streaming API

Apache Bahir提供以下Connector：

+ ActiveMQ
+ Flume
+ Redis(sink)
+ Akka
+ Netty(source)

### 其他连接Flink的方法
+ 异步外部数据源
+ Queryable State

### 容错保证
https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/connectors/guarantees.html

Source：

+ Kafka：exactly once
+ RabbitMQ：at most once (v 0.10) / exactly once (v 1.0)
+ Collection：exactly once
+ Files：exactly once
+ Sockets：at most once

Sink：

+ HDFS：exactly once
+ Elasticsearch：at least once
+ Kafka：at least once
+ File：at least once
+ Socket：at least once
+ Standard output：at least once
+ Redis：at least once

### Kafka Connector
Flink Kafka Consumer 支持Flink checkpoint机制，通过追踪offset，提供exactly once的语义
```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.8_2.11</artifactId>
  <version>1.6.1</version>
</dependency>
```
提供FlinkKafkaConsumer08、FlinkKafkaProducer08
##### Kafka Consumer
FlinkKafkaConsumer08构造器参数：

+ topic名
+ DeserializationSchema / KeyedDeserializationSchema用于反序列号
+ Properties：
	- bootstrap.servers：kafka broker list
	- zookeeper.connect：只有08版本需要zk
	- group.id：consumer group的id
```
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");
DataStream<String> stream = env
	.addSource(new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties));
```
##### DeserializationSchema
+ DeserializationSchema的T deserialize(byte[] message)负责反序列化
```
public interface DeserializationSchema<T> extends Serializable, ResultTypeQueryable<T> {
    T deserialize(byte[] var1) throws IOException;

    boolean isEndOfStream(T var1);
}

public interface ResultTypeQueryable<T> {
    TypeInformation<T> getProducedType();
}
```
+ AbstractDeserializationSchema抽象类，实现了getProducedType(...)方法
+ KeyedDeserializationSchema提供T deserialize(byte[] messageKey, byte[] message, String topic, int partition, long offset)

Flink封装了以下Schema：

+ TypeInformationSerializationSchema/TypeInformationKeyValueSerializationSchema
+ JsonDeserializationSchema/JSONKeyValueDeserializationSchema
+ AvroDeserializationSchema/

反序列失败后抛出异常可以重跑任务，返回null将跳过这条消息

##### Kafka Consumer配置开始消费的位置
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

FlinkKafkaConsumer08<String> myConsumer = new FlinkKafkaConsumer08<>(...);
myConsumer.setStartFromEarliest();     // start from the earliest record possible
myConsumer.setStartFromLatest();       // start from the latest record
myConsumer.setStartFromTimestamp(...); // start from specified epoch timestamp (milliseconds)
myConsumer.setStartFromGroupOffsets(); // the default behaviour

DataStream<String> stream = env.addSource(myConsumer);
...
```
```
Map<KafkaTopicPartition, Long> specificStartOffsets = new HashMap<>();
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 0), 23L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 1), 31L);
specificStartOffsets.put(new KafkaTopicPartition("myTopic", 2), 43L);

myConsumer.setStartFromSpecificOffsets(specificStartOffsets);
```

##### Kafka Consumer容错
offset随checkpoint写入，重跑时从offset重跑

##### Kafka Consumer Topic和Partition发现
+ Partition发现：初始化获取元数据后，partition发现会从最早的offset开始消费；为flink.partition-discovery.interval-millis配置非负数开启该功能
+ topic发现：通过表达式发现topic
```
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer011<String> myConsumer = new FlinkKafkaConsumer011<>(
    java.util.regex.Pattern.compile("test-topic-[0-9]"),
    new SimpleStringSchema(),
    properties);

DataStream<String> stream = env.addSource(myConsumer);
...
```

##### Kafka Consumer offset提交策略配置
flink kafka consumer容错不依赖向kafka broker提交offset，只用于暴露consumer进度到监控

+ 当不开启checkpointing时，通过Properties配置，enable.auto.commit (or auto.commit.enable for Kafka 0.8) / auto.commit.interval.ms
+ 开启checkpointing时，向kafka broker提交的offset与checkpoint保持一致，setCommitOffsetsOnCheckpoints(boolean)

##### Kafka Consumer Timestamp提取、Watermark提交
如下定义timestamp提取和watermark生成：
```
Properties properties = new Properties();
properties.setProperty("bootstrap.servers", "localhost:9092");
// only required for Kafka 0.8
properties.setProperty("zookeeper.connect", "localhost:2181");
properties.setProperty("group.id", "test");

FlinkKafkaConsumer08<String> myConsumer =
    new FlinkKafkaConsumer08<>("topic", new SimpleStringSchema(), properties);
myConsumer.assignTimestampsAndWatermarks(new CustomWatermarkEmitter());

DataStream<String> stream = env
	.addSource(myConsumer)
	.print();
```

##### Kafka Producer
```
DataStream<String> stream = ...;

FlinkKafkaProducer011<String> myProducer = new FlinkKafkaProducer011<String>(
        "localhost:9092",            // broker list
        "my-topic",                  // target topic
        new SimpleStringSchema());   // serialization schema

// versions 0.10+ allow attaching the records' event timestamp when writing them to Kafka;
// this method is not available for earlier Kafka versions
myProducer.setWriteTimestampToKafka(true);

stream.addSink(myProducer);
```
构造器变量：

+ 自定义属性Properties
+ 自定义分片路由FlinkKafkaPartitioner
+ 自定义序列化

##### Kafka Producer分片方案
+ 默认使用FlinkFixedPartitioner，每个子任务写到同一个kafka partition
+ 使用FlinkKafkaPartitioner实现自定义分片器
+ 传null，按key分片数据

##### Kafka Producer容错
+ Kafka 0.8：不支持容错
+ Kafka 0.9 and 0.10：
	- 开启checkpoint
	- setLogFailuresOnly(boolean)：必须设置false，否则感知不到失败
	- setFlushOnCheckpoint(boolean)：设置为true，保证checkpoint成功前所有数据写入到kafka。注意kafka ack也可能丢数据，这取决于kafka配置
	- retry默认设置为0，领导节点变更会导致丢数据。在生产环境中，频繁的broker变更需要设置重试
	- kafka没有事务性的producer，因此无法保证exactly-once
+ Kafka 0.11：可提供exactly-once保证，通过设置FlinkKafkaProducer011的semantic参数：
	- Semantic.NONE：不保证
	- Semantic.AT_LEAST_ONCE：像FlinkKafkaProducer010，保证不丢数据
	- Semantic.EXACTLY_ONCE：使用kafka事务机制保证exactly-once
	
注意事项：
+ 事务超时会导致丢数据
+ kafka broker的transaction.max.timeout.ms默认设置为15分钟，FlinkKafkaProducer011默认1小时，因此要调高kafka的事务超时
+ KafkaConsumer read_committed模式将被未提交的事务阻塞读请求

##### 使用kafka timestamp和Flink event time
+ 设置EventTime：StreamExecutionEnvironment.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
+ 生成Watermark：assignTimestampsAndWatermarks
+ 使用kafka timestamp：
```
public long extractTimestamp(Long element, long previousElementTimestamp) {
    return previousElementTimestamp;
}
```
+ 写出记录的timestamp
```
FlinkKafkaProducer010.FlinkKafkaProducer010Configuration config = FlinkKafkaProducer010.writeToKafkaWithTimestamps(streamWithTimestamps, topic, new SimpleStringSchema(), standardProps);
config.setWriteTimestampToKafka(true);
```

##### Kafka Connector Metrics
Flink导出kafka的内部指标作为Kafka Connector的metrics

##### Enabling Kerberos 认证
https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/connectors/kafka.html#enabling-kerberos-authentication-for-versions-09-and-above-only

### Elasticsearch Connector
#####Elasticsearch Sink
Es5：
```
import org.apache.flink.api.common.functions.RuntimeContext;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.connectors.elasticsearch.ElasticsearchSinkFunction;
import org.apache.flink.streaming.connectors.elasticsearch.RequestIndexer;
import org.apache.flink.streaming.connectors.elasticsearch5.ElasticsearchSink;

import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.client.Requests;

import java.net.InetAddress;
import java.net.InetSocketAddress;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

DataStream<String> input = ...;

Map<String, String> config = new HashMap<>();
config.put("cluster.name", "my-cluster-name");
// This instructs the sink to emit after every element, otherwise they would be buffered
config.put("bulk.flush.max.actions", "1");

List<InetSocketAddress> transportAddresses = new ArrayList<>();
transportAddresses.add(new InetSocketAddress(InetAddress.getByName("127.0.0.1"), 9300));
transportAddresses.add(new InetSocketAddress(InetAddress.getByName("10.2.3.1"), 9300));

input.addSink(new ElasticsearchSink<>(config, transportAddresses, new ElasticsearchSinkFunction<String>() {
    public IndexRequest createIndexRequest(String element) {
        Map<String, String> json = new HashMap<>();
        json.put("data", element);
    
        return Requests.indexRequest()
                .index("my-index")
                .type("my-type")
                .source(json);
    }
    
    @Override
    public void process(String element, RuntimeContext ctx, RequestIndexer indexer) {
        indexer.add(createIndexRequest(element));
    }
}));
```
Es6：
```
import org.apache.flink.api.common.functions.RuntimeContext;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.connectors.elasticsearch.RequestIndexer;
import org.apache.flink.streaming.connectors.elasticsearch6.ElasticsearchSink;

import org.apache.http.HttpHost;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.client.Requests;

import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

DataStream<String> input = ...;

List<HttpHost> httpHost = new ArrayList<>();
httpHosts.add(new HttpHost("127.0.0.1", 9200, "http"));
httpHosts.add(new HttpHost("10.2.3.1", 9200, "http"));

// use a ElasticsearchSink.Builder to create an ElasticsearchSink
ElasticsearchSink.Builder<String> esSinkBuilder = new ElasticsearchSink.Builder<>(
    httpHosts,
    new ElasticsearchSinkFunction<String>() {
        public IndexRequest createIndexRequest(String element) {
            Map<String, String> json = new HashMap<>();
            json.put("data", element);
        
            return Requests.indexRequest()
                    .index("my-index")
                    .type("my-type")
                    .source(json);
        }
        
        @Override
        public void process(String element, RuntimeContext ctx, RequestIndexer indexer) {
            indexer.add(createIndexRequest(element));
        }
    }
);

// configuration for the bulk requests; this instructs the sink to emit after every element, otherwise they would be buffered
builder.setBulkFlushMaxActions(1);

// provide a RestClientFactory for custom configuration on the internally created REST client
builder.setRestClientFactory(
  restClientBuilder -> {
    restClientBuilder.setDefaultHeaders(...)
    restClientBuilder.setMaxRetryTimeoutMillis(...)
    restClientBuilder.setPathPrefix(...)
    restClientBuilder.setHttpClientConfigCallback(...)
  }
);

// finally, build and add the sink to the job's pipeline
input.addSink(esSinkBuilder.build());
```

##### 容错
开启Checkpoint可保证至少一次的语义，Flink在checkpoint时等待所有BulkRequest完成

##### 处理失败的Elasticsearch请求
实现ActionRequestFailureHandler自定义处理失败请求
```
DataStream<String> input = ...;

input.addSink(new ElasticsearchSink<>(
    config, transportAddresses,
    new ElasticsearchSinkFunction<String>() {...},
    new ActionRequestFailureHandler() {
        @Override
        void onFailure(ActionRequest action,
                Throwable failure,
                int restStatusCode,
                RequestIndexer indexer) throw Throwable {

            if (ExceptionUtils.containsThrowable(failure, EsRejectedExecutionException.class)) {
                // full queue; re-add document for indexing
                indexer.add(action);
            } else if (ExceptionUtils.containsThrowable(failure, ElasticsearchParseException.class)) {
                // malformed document; simply drop request without failing sink
            } else {
                // for all other failures, fail the sink
                // here the failure is simply rethrown, but users can also choose to throw custom exceptions
                throw failure;
            }
        }
}));
```
onFailure在BulkProcessor失败后调用，默认情况下，BulkProcessor会在失败后重试8次并进行指数回避
+ NoOpFailureHandler：默认实现
+ RetryRejectedExecutionFailureHandler：由于队列满导致的拒绝可重试

##### 配置BulkProcess
+ bulk.flush.max.actions：缓存最大批次
bulk.flush.max.size.mb：缓存最大大小
bulk.flush.interval.ms:：缓存时间
bulk.flush.backoff.enable：重试时是否使用回避策略
bulk.flush.backoff.type: 指数回避/常数回避
bulk.flush.backoff.delay：回避时间，常数回避是固定的回避时间，指数回避是基准回避时间
bulk.flush.backoff.retries: 重试次数