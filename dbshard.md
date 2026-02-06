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

如何到对应分库去执行❓

业务中可以记录分库名到原始DataSource（分库）的映射。





















