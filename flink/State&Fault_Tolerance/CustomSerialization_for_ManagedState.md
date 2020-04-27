Custom Serialization for Managed State
===

### 使用自定义序列化
flink将自定义序列化的类名作为元数据写入State，因此不要使用匿名类
```
public class CustomTypeSerializer extends TypeSerializer<Tuple2<String, Integer>> {...};

ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "state-name",
        new CustomTypeSerializer());

checkpointedState = getRuntimeContext().getListState(descriptor);
```

### 序列化器升级与兼容
TypeSerializer提供以下两个接口

+ snapshotConfiguration()在checkpoint时作为元数据写入State
+ 从checkpoint恢复时，ensureCompatibility(TypeSerializerConfigSnapshot configSnapshot)检查序列化器是否兼容
```
public abstract TypeSerializerConfigSnapshot snapshotConfiguration();
public abstract CompatibilityResult ensureCompatibility(TypeSerializerConfigSnapshot configSnapshot);
```

##### 实现snapshotConfiguration
TypeSerializerConfigSnapshot为判断序列化器是否兼容提供信息
read和write方法定义了怎样将配置写入快照和怎样从快照读取，默认实现是写入version，配置快照从getVersion获取version
```
public abstract TypeSerializerConfigSnapshot extends VersionedIOReadableWritable {
  public abstract int getVersion();
  public void read(DataInputView in) {...}
  public void write(DataOutputView out) {...}
}
```

##### 实现ensureCompatibility
做以下事情：

+ 确认是否兼容，重新配置使其兼容
+ 不兼容迁移状态

返回：

+ CompatibilityResult.compatible(): 兼容，或者重新配置为兼容
+ CompatibilityResult.requiresMigration(): 不兼容，使用之前的序列化器反序列化成对象，再使用新的序列化器迁移State
+ CompatibilityResult.requiresMigration(TypeDeserializer deserializer): 同上，用于找不到之前的序列化器的场景