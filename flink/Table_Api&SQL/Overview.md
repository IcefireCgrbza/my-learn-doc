Table Api 和 SQL
===

Flink对SQL的支持使用了Apache Calcite。对流或批使用Table Api&SQL只需要切换依赖即可

### 依赖
```
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-table_2.11</artifactId>
  <version>1.6.1</version>
</dependency>
```

### Table Api&SQL的程序结构
```

// for batch programs use ExecutionEnvironment instead of StreamExecutionEnvironment
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// create a TableEnvironment
// for batch programs use BatchTableEnvironment instead of StreamTableEnvironment
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register a Table
tableEnv.registerTable("table1", ...)            // or
tableEnv.registerTableSource("table2", ...);     // or
tableEnv.registerExternalCatalog("extCat", ...);

// create a Table from a Table API query
Table tapiResult = tableEnv.scan("table1").select(...);
// create a Table from a SQL query
Table sqlResult  = tableEnv.sqlQuery("SELECT ... FROM table2 ... ");

// emit a Table API result Table to a TableSink, same for SQL result
tapiResult.writeToSink(...);

// execute
env.execute();
```

### 创建TableEnvironment
TableEnvironment负责：

+ 在内部catalog中注册Table
+ 注册外部catalog
+ 执行SQL
+ 注册用户自定义函数
+ 把DataStream/DataSet转换成Table
+ 持有ExecutionEnvironment或StreamExecutionEnvironment

```

// ***************
// STREAMING QUERY
// ***************
StreamExecutionEnvironment sEnv = StreamExecutionEnvironment.getExecutionEnvironment();
// create a TableEnvironment for streaming queries
StreamTableEnvironment sTableEnv = TableEnvironment.getTableEnvironment(sEnv);

// ***********
// BATCH QUERY
// ***********
ExecutionEnvironment bEnv = ExecutionEnvironment.getExecutionEnvironment();
// create a TableEnvironment for batch queries
BatchTableEnvironment bTableEnv = TableEnvironment.getTableEnvironment(bEnv);
```

### 在Catalog中注册Tables

TableEnvironment的catalog中包含通过名称注册的Table

可以通过以下方式注册input table：

+ Table对象（Table Api 或 SQL查询的结果）
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table is the result of a simple projection query 
Table projTable = tableEnv.scan("X").select(...);

// register the Table projTable as table "projectedX"
tableEnv.registerTable("projectedTable", projTable);
```
+ TableSource，对接外部数据源
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSource
TableSource csvSource = new CsvTableSource("/path/to/file", ...);

// register the TableSource as table "CsvTable"
tableEnv.registerTableSource("CsvTable", csvSource);
```
+ DataStream或DataSet


使用TableSink注册Output Table：
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create a TableSink
TableSink csvSink = new CsvTableSink("/path/to/file", ...);

// define the field names and types
String[] fieldNames = {"a", "b", "c"};
TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};

// register the TableSink as table "CsvSinkTable"
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, csvSink);
```

### 注册外部Catalog
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// create an external catalog
ExternalCatalog catalog = new InMemoryExternalCatalog();

// register the ExternalCatalog catalog
tableEnv.registerExternalCatalog("InMemCatalog", catalog);
```

### 查询Table
##### Table Api
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table

// scan registered Orders table
Table orders = tableEnv.scan("Orders");
// compute revenue for all customers from France
Table revenue = orders
  .filter("cCountry === 'FRANCE'")
  .groupBy("cID, cName")
  .select("cID, cName, revenue.sum AS revSum");

// emit or convert Table
// execute query
```

##### SQL
查询样例：
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register Orders table

// compute revenue for all customers from France
Table revenue = tableEnv.sqlQuery(
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// emit or convert Table
// execute query
```
更新样例：
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// register "Orders" table
// register "RevenueFrance" output table

// compute revenue for all customers from France and emit to "RevenueFrance"
tableEnv.sqlUpdate(
    "INSERT INTO RevenueFrance " +
    "SELECT cID, cName, SUM(revenue) AS revSum " +
    "FROM Orders " +
    "WHERE cCountry = 'FRANCE' " +
    "GROUP BY cID, cName"
  );

// execute query
```
### 提交Table
通过TableSink提交Table，批Table可用BatchTableSink，流Table可用AppendStreamTableSink、RetractStreamTableSink、UpsertStreamTableSink
使用以下方法提交Table：
+ Table.writeToSink(TableSink sink)
+ Table.insertInto(String sinkTable)：从TableEnvironment中找TableSink
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// compute a result Table using Table API operators and/or SQL queries
Table result = ...

// create a TableSink
TableSink sink = new CsvTableSink("/path/to/file", fieldDelim = "|");

// METHOD 1:
//   Emit the result Table to the TableSink via the writeToSink() method
result.writeToSink(sink);

// METHOD 2:
//   Register the TableSink with a specific schema
String[] fieldNames = {"a", "b", "c"};
TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, sink);
//   Emit the result Table to the registered TableSink via the insertInto() method
result.insertInto("CsvSinkTable");

// execute the program// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// compute a result Table using Table API operators and/or SQL queries
Table result = ...

// create a TableSink
TableSink sink = new CsvTableSink("/path/to/file", fieldDelim = "|");

// METHOD 1:
//   Emit the result Table to the TableSink via the writeToSink() method
result.writeToSink(sink);

// METHOD 2:
//   Register the TableSink with a specific schema
String[] fieldNames = {"a", "b", "c"};
TypeInformation[] fieldTypes = {Types.INT, Types.STRING, Types.LONG};
tableEnv.registerTableSink("CsvSinkTable", fieldNames, fieldTypes, sink);
//   Emit the result Table to the registered TableSink via the insertInto() method
result.insertInto("CsvSinkTable");

// execute the program
```

### 翻译、执行查询
Table Api&SQL 查询会被翻译成DataStream或DataSet程序，翻译经历两个阶段：

+ 优化
+ 翻译为DataStream或DataSet程序

Table Api或SQL查询翻译的时机：
+ Table提交到TableSink
+ TableEnvironment.sqlUpdate
+ Table转换为DataStream或DataSet

### 整合DataStream和DataSet Api
DataStream或DataSet可以转换为Table，反之亦然

##### 注册DataStream或DataSet为Table
```
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// register the DataStream as Table "myTable" with fields "f0", "f1"
tableEnv.registerDataStream("myTable", stream);

// register the DataStream as table "myTable2" with fields "myLong", "myString"
tableEnv.registerDataStream("myTable2", stream, "myLong, myString");
```
##### DataStream或DataSet转换为Table
```
// get StreamTableEnvironment
// registration of a DataSet in a BatchTableEnvironment is equivalent
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// Convert the DataStream into a Table with default fields "f0", "f1"
Table table1 = tableEnv.fromDataStream(stream);

// Convert the DataStream into a Table with fields "myLong", "myString"
Table table2 = tableEnv.fromDataStream(stream, "myLong, myString");
```
##### Table转换为DataStream或DataSet
转换类型：

+ Row
+ POJO
+ Case Class
+ Tuple
+ Atomic Type

转换模式：

+ Append Mode：动态Table只有插入操作
+ Retract Mode：通过一个标记为插入和删除操作编码

```
// get StreamTableEnvironment. 
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into an append DataStream of Row by specifying the class
DataStream<Row> dsRow = tableEnv.toAppendStream(table, Row.class);

// convert the Table into an append DataStream of Tuple2<String, Integer> 
//   via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataStream<Tuple2<String, Integer>> dsTuple = 
  tableEnv.toAppendStream(table, tupleType);

// convert the Table into a retract DataStream of Row.
//   A retract stream of type X is a DataStream<Tuple2<Boolean, X>>. 
//   The boolean field indicates the type of the change. 
//   True is INSERT, false is DELETE.
DataStream<Tuple2<Boolean, Row>> retractStream = 
  tableEnv.toRetractStream(table, Row.class);
```
```
// get BatchTableEnvironment
BatchTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Table with two fields (String name, Integer age)
Table table = ...

// convert the Table into a DataSet of Row by specifying a class
DataSet<Row> dsRow = tableEnv.toDataSet(table, Row.class);

// convert the Table into a DataSet of Tuple2<String, Integer> via a TypeInformation
TupleTypeInfo<Tuple2<String, Integer>> tupleType = new TupleTypeInfo<>(
  Types.STRING(),
  Types.INT());
DataSet<Tuple2<String, Integer>> dsTuple = 
  tableEnv.toDataSet(table, tupleType);
```

##### Table Schema和数据类型映射
+ 基于位置的映射：数据类型没有字段名
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field names "myLong" and "myInt"
Table table = tableEnv.fromDataStream(stream, "myLong, myInt");
```
+ 基于字段名称的映射
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, Integer>> stream = ...

// convert DataStream into Table with default field names "f0" and "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field "f1" only
Table table = tableEnv.fromDataStream(stream, "f1");

// convert DataStream into Table with swapped fields
Table table = tableEnv.fromDataStream(stream, "f1, f0");

// convert DataStream into Table with swapped fields and field names "myInt" and "myLong"
Table table = tableEnv.fromDataStream(stream, "f1 as myInt, f0 as myLong");
```
+ 原子类型
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Long> stream = ...

// convert DataStream into Table with default field name "f0"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with field name "myLong"
Table table = tableEnv.fromDataStream(stream, "myLong");
```
+ Tuple和Case Classes（Scala）
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Long, String>> stream = ...

// convert DataStream into Table with default field names "f0", "f1"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myLong", "myString" (position-based)
Table table = tableEnv.fromDataStream(stream, "myLong, myString");

// convert DataStream into Table with reordered fields "f1", "f0" (name-based)
Table table = tableEnv.fromDataStream(stream, "f1, f0");

// convert DataStream into Table with projected field "f1" (name-based)
Table table = tableEnv.fromDataStream(stream, "f1");

// convert DataStream into Table with reordered and aliased fields "myString", "myLong" (name-based)
Table table = tableEnv.fromDataStream(stream, "f1 as 'myString', f0 as 'myLong'");
```
+ POJO
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// Person is a POJO with fields "name" and "age"
DataStream<Person> stream = ...

// convert DataStream into Table with default field names "age", "name" (fields are ordered by name!)
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed fields "myAge", "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "age as myAge, name as myName");

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, "name");

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName");
```
+ Row
```
// get a StreamTableEnvironment, works for BatchTableEnvironment equivalently
StreamTableEnvironment tableEnv = TableEnvironment.getTableEnvironment(env);

// DataStream of Row with two fields "name" and "age" specified in `RowTypeInfo`
DataStream<Row> stream = ...

// convert DataStream into Table with default field names "name", "age"
Table table = tableEnv.fromDataStream(stream);

// convert DataStream into Table with renamed field names "myName", "myAge" (position-based)
Table table = tableEnv.fromDataStream(stream, "myName, myAge");

// convert DataStream into Table with renamed fields "myName", "myAge" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName, age as myAge");

// convert DataStream into Table with projected field "name" (name-based)
Table table = tableEnv.fromDataStream(stream, "name");

// convert DataStream into Table with projected and renamed field "myName" (name-based)
Table table = tableEnv.fromDataStream(stream, "name as myName");
```

### 查询优化
Flink使用Apache Calcite优化并翻译查询。优化包括projection、filter push-down、subquery decorrelation和各种类型的查询重写，没有优化表连接的顺序

提供CalciteConfig可配置优化规则
```
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tEnv = TableEnvironment.getTableEnvironment(env);

DataStream<Tuple2<Integer, String>> stream1 = env.fromElements(new Tuple2<>(1, "hello"));
DataStream<Tuple2<Integer, String>> stream2 = env.fromElements(new Tuple2<>(1, "hello"));

Table table1 = tEnv.fromDataStream(stream1, "count, word");
Table table2 = tEnv.fromDataStream(stream2, "count, word");
Table table = table1
  .where("LIKE(word, 'F%')")
  .unionAll(table2);

String explanation = tEnv.explain(table);
System.out.println(explanation);
```
```
== Abstract Syntax Tree ==
LogicalUnion(all=[true])
  LogicalFilter(condition=[LIKE($1, 'F%')])
    LogicalTableScan(table=[[_DataStreamTable_0]])
  LogicalTableScan(table=[[_DataStreamTable_1]])

== Optimized Logical Plan ==
DataStreamUnion(union=[count, word])
  DataStreamCalc(select=[count, word], where=[LIKE(word, 'F%')])
    DataStreamScan(table=[[_DataStreamTable_0]])
  DataStreamScan(table=[[_DataStreamTable_1]])

== Physical Execution Plan ==
Stage 1 : Data Source
  content : collect elements with CollectionInputFormat

Stage 2 : Data Source
  content : collect elements with CollectionInputFormat

  Stage 3 : Operator
    content : from: (count, word)
    ship_strategy : REBALANCE

    Stage 4 : Operator
      content : where: (LIKE(word, 'F%')), select: (count, word)
      ship_strategy : FORWARD

      Stage 5 : Operator
        content : from: (count, word)
        ship_strategy : REBALANCE
```