Checkpointing
===
flink的function和operator都是有状态的，为了实现容错，flink为state创建了Checkpoint

### 前提
* 可回放的Source
* State可持久存储

### 开启、配置Checkpointing
checkpointing默认是关闭的，通过enableCheckpointing(n)打开，n是checkpoint的间隔。其他参数包括：

+ exactly-once vs. at-least-once
+ checkpoint超时时间
+ 两次checkpoint之间的最小时间
+ 并发checkpoint的数量
+ 外部checkpoint
+ checkpoint出错时任务继续/报错

```
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// enable externalized checkpoints which are retained after job cancellation
env.getCheckpointConfig().enableExternalizedCheckpoints(ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```

### 相关配置
通过conf/flink-conf.yaml配置

+ state.backend：状态后端
+ state.backend.async：异步快照
+ state.backend.fs.memory-threshold：State数据文件的最小大小
+ state.backend.incremental：增量快照
+ state.backend.local-recovery
+ state.checkpoints.dir：checkpoint文件目录
+ state.checkpoints.num-retained：全量checkpoint数量
+ state.savepoints.dir：savepoint的目录
+ taskmanager.state.local.root-dirs

### 选择State backend
默认，State存储在TaskManager的内存中，checkpoint存在JobManager的内存中。
StreamExecutionEnvironment.setStateBackend(…)

### 循环Job的State checkpoint
不支持循环job

### 重启策略
