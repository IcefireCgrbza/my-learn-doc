Connect to External Systems
===

### 依赖

+ Connectors
	- Filesystem
	- Apache Kafka：flink-connector-kafka-{version}
+ Formats
	- CSV
	- JSON：flink-json
	- Apache Avro：flink-avro
	
### Overview
连接外部系统需要定义：

+ Connector
+ Data Format：外部系统的数据格式
+ Table Schema：动态表schema
+ Update Mode：Append、Retract、Upsert
+ Source or Sink
```
tableEnvironment
  .connect(...)
  .withFormat(...)
  .withSchema(...)
  .inAppendMode()
  .registerTableSource("MyTable")
```

例子：连接kafka，读取Avro数据

```
//JAVA

tableEnvironment
  // declare the external system to connect to
  .connect(
    new Kafka()
      .version("0.10")
      .topic("test-input")
      .startFromEarliest()
      .property("zookeeper.connect", "localhost:2181")
      .property("bootstrap.servers", "localhost:9092")
  )

  // declare a format for this system
  .withFormat(
    new Avro()
      .avroSchema(
        "{" +
        "  \"namespace\": \"org.myorganization\"," +
        "  \"type\": \"record\"," +
        "  \"name\": \"UserMessage\"," +
        "    \"fields\": [" +
        "      {\"name\": \"timestamp\", \"type\": \"string\"}," +
        "      {\"name\": \"user\", \"type\": \"long\"}," +
        "      {\"name\": \"message\", \"type\": [\"string\", \"null\"]}" +
        "    ]" +
        "}" +
      )
  )

  // declare the schema of the table
  .withSchema(
    new Schema()
      .field("rowtime", Types.SQL_TIMESTAMP)
        .rowtime(new Rowtime()
          .timestampsFromField("ts")
          .watermarksPeriodicBounded(60000)
        )
      .field("user", Types.LONG)
      .field("message", Types.STRING)
  )

  // specify the update-mode for streaming tables
  .inAppendMode()

  // register as source, sink, or both and under a name
  .registerTableSource("MyUserTable");
```
```
//YAML


tables:
  - name: MyUserTable      # name the new table
    type: source           # declare if the table should be "source", "sink", or "both"
    update-mode: append    # specify the update-mode for streaming tables

    # declare the external system to connect to
    connector:
      type: kafka
      version: "0.10"
      topic: test-input
      startup-mode: earliest-offset
      properties:
        - key: zookeeper.connect
          value: localhost:2181
        - key: bootstrap.servers
          value: localhost:9092

    # declare a format for this system
    format:
      type: avro
      avro-schema: >
        {
          "namespace": "org.myorganization",
          "type": "record",
          "name": "UserMessage",
            "fields": [
              {"name": "ts", "type": "string"},
              {"name": "user", "type": "long"},
              {"name": "message", "type": ["string", "null"]}
            ]
        }

    # declare the schema of the table
    schema:
      - name: rowtime
        type: TIMESTAMP
        rowtime:
          timestamps:
            type: from-field
            from: ts
          watermarks:
            type: periodic-bounded
            delay: "60000"
      - name: user
        type: BIGINT
      - name: message
        type: VARCHAR
```

### Table Schema
+ Table Schema字段和Data Format一一对应
```
.withSchema(
  new Schema()
    .field("MyField1", Types.INT)     // required: specify the fields of the table (in this order)
    .field("MyField2", Types.STRING)
    .field("MyField3", Types.BOOLEAN)
)
```
```
schema:
  - name: MyField1    # required: specify the fields of the table (in this order)
    type: INT
  - name: MyField2
    type: VARCHAR
  - name: MyField3
    type: BOOLEAN
```
+ Time & 指定Table Schema字段和Data Format的映射
```
.withSchema(
  new Schema()
    .field("MyField1", Types.SQL_TIMESTAMP)
      .proctime()      // optional: declares this field as a processing-time attribute
    .field("MyField2", Types.SQL_TIMESTAMP)
      .rowtime(...)    // optional: declares this field as a event-time attribute
    .field("MyField3", Types.BOOLEAN)
      .from("mf3")     // optional: original field in the input that is referenced/aliased by this field
)
```
```
schema:
  - name: MyField1
    type: TIMESTAMP
    proctime: true    # optional: boolean flag whether this field should be a processing-time attribute
  - name: MyField2
    type: TIMESTAMP
    rowtime: ...      # optional: wether this field should be a event-time attribute
  - name: MyField3
    type: BOOLEAN
    from: mf3         # optional: original field in the input that is referenced/aliased by this field
```
+ Rowtime Attribute指定timestamp extractors和watermark strategies
```

// Converts an existing LONG or SQL_TIMESTAMP field in the input into the rowtime attribute.
.rowtime(
  new Rowtime()
    .timestampsFromField("ts_field")    // required: original field name in the input
)

// Converts the assigned timestamps from a DataStream API record into the rowtime attribute 
// and thus preserves the assigned timestamps from the source.
// This requires a source that assigns timestamps (e.g., Kafka 0.10+).
.rowtime(
  new Rowtime()
    .timestampsFromSource()
)

// Sets a custom timestamp extractor to be used for the rowtime attribute.
// The extractor must extend `org.apache.flink.table.sources.tsextractors.TimestampExtractor`.
.rowtime(
  new Rowtime()
    .timestampsFromExtractor(...)
)
```
```
# Converts an existing BIGINT or TIMESTAMP field in the input into the rowtime attribute.
rowtime:
  timestamps:
    type: from-field
    from: "ts_field"                 # required: original field name in the input

# Converts the assigned timestamps from a DataStream API record into the rowtime attribute 
# and thus preserves the assigned timestamps from the source.
rowtime:
  timestamps:
    type: from-source
```
```
// Sets a watermark strategy for ascending rowtime attributes. Emits a watermark of the maximum 
// observed timestamp so far minus 1. Rows that have a timestamp equal to the max timestamp
// are not late.
.rowtime(
  new Rowtime()
    .watermarksPeriodicAscending()
)

// Sets a built-in watermark strategy for rowtime attributes which are out-of-order by a bounded time interval.
// Emits watermarks which are the maximum observed timestamp minus the specified delay.
.rowtime(
  new Rowtime()
    .watermarksPeriodicBounded(2000)    // delay in milliseconds
)

// Sets a built-in watermark strategy which indicates the watermarks should be preserved from the
// underlying DataStream API and thus preserves the assigned watermarks from the source.
.rowtime(
  new Rowtime()
    .watermarksFromSource()
)
```
```
# Sets a watermark strategy for ascending rowtime attributes. Emits a watermark of the maximum 
# observed timestamp so far minus 1. Rows that have a timestamp equal to the max timestamp
# are not late.
rowtime:
  watermarks:
    type: periodic-ascending

# Sets a built-in watermark strategy for rowtime attributes which are out-of-order by a bounded time interval.
# Emits watermarks which are the maximum observed timestamp minus the specified delay.
rowtime:
  watermarks:
    type: periodic-bounded
    delay: ...                # required: delay in milliseconds

# Sets a built-in watermark strategy which indicates the watermarks should be preserved from the
# underlying DataStream API and thus preserves the assigned watermarks from the source.
rowtime:
  watermarks:
    type: from-source
```
+ Type
```
VARCHAR
BOOLEAN
TINYINT
SMALLINT
INT
BIGINT
FLOAT
DOUBLE
DECIMAL
DATE
TIME
TIMESTAMP
ROW(fieldtype, ...)              # unnamed row; e.g. ROW(VARCHAR, INT) that is mapped to Flink's RowTypeInfo
                                 # with indexed fields names f0, f1, ...
ROW(fieldname fieldtype, ...)    # named row; e.g., ROW(myField VARCHAR, myOtherField INT) that
                                 # is mapped to Flink's RowTypeInfo
POJO(class)                      # e.g., POJO(org.mycompany.MyPojoClass) that is mapped to Flink's PojoTypeInfo
ANY(class)                       # e.g., ANY(org.mycompany.MyClass) that is mapped to Flink's GenericTypeInfo
ANY(class, serialized)           # used for type information that is not supported by Flink's Table & SQL API
```

### Update Modes
+ Append Mode
+ Retract Mode
+ Upsert Mode
```
tables:
  - name: ...
    update-mode: append    # otherwise: "retract" or "upsert"
```

### Table Connectors
##### File System Connector
```
.connect(
  new FileSystem()
    .path("file:///path/to/whatever")    // required: path to a file or directory
)
```
```
connector:
  type: filesystem
  path: "file:///path/to/whatever"    # required: path to a file or directory
```
##### Kafka Connector
```
.connect(
  new Kafka()
    .version("0.11")    // required: valid connector versions are "0.8", "0.9", "0.10", and "0.11"
    .topic("...")       // required: topic name from which the table is read

    // optional: connector specific properties
    .property("zookeeper.connect", "localhost:2181")
    .property("bootstrap.servers", "localhost:9092")
    .property("group.id", "testGroup")

    // optional: select a startup mode for Kafka offsets
    .startFromEarliest()
    .startFromLatest()
    .startFromSpecificOffsets(...)

    // optional: output partitioning from Flink's partitions into Kafka's partitions
    .sinkPartitionerFixed()         // each Flink partition ends up in at-most one Kafka partition (default)
    .sinkPartitionerRoundRobin()    // a Flink partition is distributed to Kafka partitions round-robin
    .sinkPartitionerCustom(MyCustom.class)    // use a custom FlinkKafkaPartitioner subclass
)
```
```
connector:
  type: kafka
  version: "0.11"     # required: valid connector versions are "0.8", "0.9", "0.10", and "0.11"
  topic: ...          # required: topic name from which the table is read

  properties:         # optional: connector specific properties
    - key: zookeeper.connect
      value: localhost:2181
    - key: bootstrap.servers
      value: localhost:9092
    - key: group.id
      value: testGroup

  startup-mode: ...   # optional: valid modes are "earliest-offset", "latest-offset",
                      # "group-offsets", or "specific-offsets"
  specific-offsets:   # optional: used in case of startup mode with specific offsets
    - partition: 0
      offset: 42
    - partition: 1
      offset: 300

  sink-partitioner: ...    # optional: output partitioning from Flink's partitions into Kafka's partitions
                           # valid are "fixed" (each Flink partition ends up in at most one Kafka partition),
                           # "round-robin" (a Flink partition is distributed to Kafka partitions round-robin)
                           # "custom" (use a custom FlinkKafkaPartitioner subclass)
  sink-partitioner-class: org.mycompany.MyPartitioner  # optional: used in case of sink partitioner custom
```

### Table Formats
##### CSV
```
.withFormat(
  new Csv()
    .field("field1", Types.STRING)    // required: ordered format fields
    .field("field2", Types.TIMESTAMP)
    .fieldDelimiter(",")              // optional: string delimiter "," by default 
    .lineDelimiter("\n")              // optional: string delimiter "\n" by default 
    .quoteCharacter('"')              // optional: single character for string values, empty by default
    .commentPrefix('#')               // optional: string to indicate comments, empty by default
    .ignoreFirstLine()                // optional: ignore the first line, by default it is not skipped
    .ignoreParseErrors()              // optional: skip records with parse error instead of failing by default
)
```
```
format:
  type: csv
  fields:                    # required: ordered format fields
    - name: field1
      type: VARCHAR
    - name: field2
      type: TIMESTAMP
  field-delimiter: ","       # optional: string delimiter "," by default 
  line-delimiter: "\n"       # optional: string delimiter "\n" by default 
  quote-character: '"'       # optional: single character for string values, empty by default
  comment-prefix: '#'        # optional: string to indicate comments, empty by default
  ignore-first-line: false   # optional: boolean flag to ignore the first line, by default it is not skipped
  ignore-parse-errors: true  # optional: skip records with parse error instead of failing by default
```
##### JSON
```
.withFormat(
  new Json()
    .failOnMissingField(true)   // optional: flag whether to fail if a field is missing or not, false by default

    // required: define the schema either by using type information which parses numbers to corresponding types
    .schema(Type.ROW(...))

    // or by using a JSON schema which parses to DECIMAL and TIMESTAMP
    .jsonSchema(
      "{" +
      "  type: 'object'," +
      "  properties: {" +
      "    lon: {" +
      "      type: 'number'" +
      "    }," +
      "    rideTime: {" +
      "      type: 'string'," +
      "      format: 'date-time'" +
      "    }" +
      "  }" +
      "}"
    )

    // or use the table's schema
    .deriveSchema()
)
```
```
format:
  type: json
  fail-on-missing-field: true   # optional: flag whether to fail if a field is missing or not, false by default

  # required: define the schema either by using a type string which parses numbers to corresponding types
  schema: "ROW(lon FLOAT, rideTime TIMESTAMP)"

  # or by using a JSON schema which parses to DECIMAL and TIMESTAMP
  json-schema: >
    {
      type: 'object',
      properties: {
        lon: {
          type: 'number'
        },
        rideTime: {
          type: 'string',
          format: 'date-time'
        }
      }
    }

  # or use the table's schema
  derive-schema: true
```
JSON schema和Flink SQL的类型映射：
+ object：Row
+ boolean：BOOLEAN
+ array：ARRAY[_]
+ number：DECIMAL
+ integer：DECIMAL
+ string：VARCHAR
+ string with format: date-time：TIMESTAMP
+ string with format: date：Date
+ string with format: time：TIME
+ string with encoding: base64：ARRAY[TINYINT]
+ null：NULL(unsupported yet)

##### Apache Avro
```
.withFormat(
  new Avro()

    // required: define the schema either by using an Avro specific record class
    .recordClass(User.class)

    // or by using an Avro schema
    .avroSchema(
      "{" +
      "  \"type\": \"record\"," +
      "  \"name\": \"test\"," +
      "  \"fields\" : [" +
      "    {\"name\": \"a\", \"type\": \"long\"}," +
      "    {\"name\": \"b\", \"type\": \"string\"}" +
      "  ]" +
      "}"
    )
)
```
```
format:
  type: avro

  # required: define the schema either by using an Avro specific record class
  record-class: "org.organization.types.User"

  # or by using an Avro schema
  avro-schema: >
    {
      "type": "record",
      "name": "test",
      "fields" : [
        {"name": "a", "type": "long"},
        {"name": "b", "type": "string"}
      ]
    }
```

### Further TableSource and TableSink
https://ci.apache.org/projects/flink/flink-docs-release-1.6/dev/table/connect.html#further-tablesources-and-tablesinks
JDBCAppendTableSink：只支持Append Mode
```
JDBCAppendTableSink sink = JDBCAppendTableSink.builder()
  .setDrivername("org.apache.derby.jdbc.EmbeddedDriver")
  .setDBUrl("jdbc:derby:memory:ebookshop")
  .setQuery("INSERT INTO books (id) VALUES (?)")
  .setParameterTypes(INT_TYPE_INFO)
  .build();

Table table = ...
table.writeToSink(sink);
```