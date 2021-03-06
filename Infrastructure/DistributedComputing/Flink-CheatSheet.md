[![返回目录](https://parg.co/UCb)](https://github.com/wxyyxc1992/Awesome-CheatSheet)\

# Flink CheatSheet | Flink 实践技巧与机制浅析

# API

![image](https://user-images.githubusercontent.com/5803001/44439870-279eef80-a5f8-11e8-9012-5d082a2b8670.png)

## 关系型 API

关系型 API 其实是 Table API 和 SQL API 的统称：

- Table API：为 Java&Scala SDK 提供类似于 LINQ（语言集成查询）模式的 API（自 0.9.0 版本开始）

- SQL API：支持标准 SQL（自 1.1.0 版本开始）

关系型 API 作为一个统一的 API 层，既能够做到在 Batch 模式的表上进行可终止地查询并生成有限的结果集，同时也能做到在 Streaming 模式的表上持续地运行并生产结果流，并且在两种模式的表上的查询具有相同的语法跟语义。这其中最重要的概念是 Table，Table 与 DataSet、DataStream 紧密结合，DataSet 和 DataStream 都可以很容易地转换成 Table，同样转换回来也很方便。下面的代码段展示了采用关系型 API 编写 Flink 程序的示例：

```scala
val tEnv = TableEnvironment.getTableEnvironment(env)
//配置数据源
val customerSource = CsvTableSource.builder()
  .path("/path/to/customer_data.csv")
  .field("name", Types.STRING).field("prefs", Types.STRING)
  .build()

//将数据源注册为一个Table
tEnv.registerTableSource(”cust", customerSource)

//定义你的table程序（在一个Flink程序中Table API和SQL API可以混用）
val table = tEnv.scan("cust").select('name.lowerCase(), myParser('prefs))
val table = tEnv.sql("SELECT LOWER(name), myParser(prefs) FROM cust")

//转换为DataStraem
val ds: DataStream[Customer] = table.toDataStream[Customer]
```

Flink 并没有自己去实现转换、SQL 的解析、执行计划的生成、优化等操作，它将一些“不擅长”的任务转交给了 Apache Calcite。整体架构如下图：

![image](https://user-images.githubusercontent.com/5803001/44439926-5ddc6f00-a5f8-11e8-9e12-99d3a23c9a96.png)

Apache Calcite 是一个 SQL 解析与查询优化框架（这个定义是从 Flink 关注的视角来看，Calcite 官方的定义为动态的数据管理框架），目前已被许多项目选择用来解析并优化 SQL 查询，比如：Drill、Hive、Kylin 等。可以从 DataSet、DataStream 以及 Table Source 等多种渠道来创建 Table，Table 相关的一些信息比如 schema、数据字段及类型等信息统一被注册并存放到 Calcite Catalog 中。这些信息将为 Table & SQL API 提供元数据。接着往下看，Table API 跟 SQL 构建的查询将被翻译成共同的逻辑计划表示，逻辑计划将作为 Calcite 优化器的输入。优化器结合逻辑计划以及特定的后端（DataSet、DataStream）规则进行翻译和优化，随之产生不同的计划。计划将通过代码生成器，生成特定的后端程序。后端程序的执行将返回 DataSet 或 DataStream。
