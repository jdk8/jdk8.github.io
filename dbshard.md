# 分库分表

如何设计一个业务无感的分库分表组件❓

如下是jdbc的接口，实现有Druid、mybatis等。

| 类/接口               | 作用                 | 使用场景                  |
| :-------------------- | :------------------- | :------------------------ |
| **DataSource**        | 连接工厂，管理连接池 | 应用启动时配置，全局使用  |
| **Connection**        | 数据库会话           | 事务管理，创建 Statement  |
| **Statement**         | 执行静态SQL          | 简单查询（有SQL注入风险） |
| **PreparedStatement** | 预编译SQL            | 参数化查询（安全，推荐）  |
| **CallableStatement** | 调用存储过程         | 执行存储过程              |
| **ResultSet**         | 保存查询结果         | 遍历和处理查询数据        |

正常的流程是：

1️⃣从DataSource中获取Connection

2️⃣从Connection获取各种类型的Statement（传入sql）

3️⃣Statement执行sql

需要在正常的流程上包装一层分库分表的流程，也就是实现ShardDataSource，ShardConnection，ShardPreparedStatement等。

ShardPreparedStatement去执行👇🏻流程：

解析sql → 获取tableName和sql参数 → 得到tableName对应的分库分表规则，将sql参数映射到实际的表和库名 → 生成分表和分库名，改写sql → 到对应的分库上去执行sql。

❓如何到对应分库去执行❓

📢业务中可以记录分库名到原始DataSource（分库）的映射。

❓如何将sql参数➕分表规则映射到对应的分库和分表❓

**分表**：

* 分片：分N个片，比如：table_name_001、table_name_002、table_name_003、...、table_name_255
* 分区：按照时间（年、月份、天）分区，比如：table_name_2020、table_name_2021、...、table_name_2025
* 分片+分区：分N个片，每个片再按照时间去分区，比如：table_name_001_2020、table_name_001_2021、table_name_002_2020、table_name_002_2021、...、table_name_255_2020、...、table_name_255_2021

下面是分表规则的一个示例：

````json
{
    "table_shard_rule": {
        "table": "table_name",
        "shardCount": 256,
        "shardField": "user_id",
        "partitionField": "create_time",
        "partitionFormat": "yyyy",
        "partitionRetentionCount": 3
    }
}
````

执行sql流程❓

匹配rule，table_shard_rule会对应一个原始DataSource（单库），就是说匹配到了这个规则，就会将sql到这个库（DataSource）去执行。

**分库**

分库的规则同分表。示例如下，分库名为:db1、db_2、db3、db4

````json
{
    "db_shard_rule": {
        "table": "table_name",
        "shardCount": 4,
        "shardField": "user_id"
    }
}
````

执行sql流程❓

匹配rule，db_shard_rule会生成sql对应的分库名，table_shard_rule会对应一个原始DataSource map，key是分库名，value是原始DataSource。最终sql会根据分库名找到对应的DataSource，然后去执行sql。

## 不停机数据迁移

如果当前是单表且有数据，则需要将单表数据迁移到分表上。迁移前需要明确几点：

1. 数据是否增、删、改？
2. 是否强一致性读，也就是不能读到旧数据。

在发版前需要尽量的迁移现有数据，需要做的工作有：

1. 数据批量迁移到分表
2. 订阅单表binlog，更新同步到分表（binlog要覆盖步骤1的区间）

发版的代码开启双写，也就是说，对单表的更新和对分表的更新放在一个事务中，并且通过开关控制读单表数据（因为尚未发版的节点还是写单表），当全部节点发布结束后，然后切换开关读分表数据，最后关闭双写。
