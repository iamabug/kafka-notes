# 问题

使用 filebeat 发送日志内容到 Kafka，然后使用 Flink 对日志进行解析，但 Flink 解析日志的代码对日志格式有要求，而日志是由第三方代码产生的，不方便直接修改，因此想要**通过 filebeat 在发送日志到 Kafka 之前对日志进行修改，使其符合预期的格式**。

# 思路

>  参考链接：https://www.elastic.co/guide/en/beats/filebeat/current/dissect.html#dissect

filebeat 提供了 `dissect` 模块，可以根据用户定义的模式将每行日志的内容进行解析，并写入各个变量中，官方示例：

```yaml
processors:
- dissect:
    tokenizer: "%{key1} %{key2}"
    field: "message"
    target_prefix: "dissect"
```

如果日志内容是 `hello world`，那么经过 `dissect` 处理后，`dissect.key1` 的值为 `hello`，`dissect.key2` 的值为 `world`。变量默认都会带有 `dissect` 前缀。

注意：如果日志不符合 `tokenizer` 定义的格式，则不会产生 `dissect.key1` 和 `dissect.key2` 这两个变量。

# 答案

将原始日志格式为 `[时间戳] 名字 动作` 处理成 JSON 格式的参考配置如下：

```yaml
filebeat.inputs:
    - type: log
      enabled: true
      paths:
          - /Users/iamabug/1.log

processors:
    - dissect:
        tokenizer: "[%{timestamp}] %{name} %{action}"
        field: "message"

output.kafka:
    enabled: true
    version: "2.0.0"
    topic: "test"
    hosts: ["localhost:9092"]
    codec.format:
        string: '{"timestamp": "%{[dissect.timestamp]}", "name": "%{[dissect.name]}", "action": "%{[dissect.action]}"}'
```

注意，`tokenizer` 的模式字符串有以下两个特点：

1. 模式字符串中的单个空格，可以匹配实际日志的多个空格；
2. 模式字符串中可以包含非变量的字符，比如上方配置的 `[]`，相应地，日志也必须包含 `[]`。

# 验证

1. （安装并）启动 Kafka，创建名为 test 的 topic，或者开启自动创建，创建命令为：

   ```bash
   kafka-topics.sh --create --bootstrap-server localhost:9092 --replication-factor 1 --partitions 1 --topic test
   ```

2. 在 filebeat 的安装目录下，创建 filebeat.yml 并进行如上配置，启动 filebeat：

   ```bash
   ./filebeat -e -c filebeat.yml
   ```

3. 向日志文件末尾添加三行内容，分别是**符合模式的、不符合模式的、符合模式且带有多余空格的**：

   ```bash
   echo [2020-01-13] iamabug write >> ~/1.log
   echo bad log >> ~/1.log
   echo [2020-01-14] iamabug eat breakfast >> ~/1.log
   ```

4. 在命令行消费 test，截图如下：

   ![](https://tva1.sinaimg.cn/large/006tNbRwly1gavp3bg8syj310z0310tc.jpg)

   根据运行结果，可以确认以下两点：

   * 不符合 `tokenizer` 模式的日志不会生成相应的变量；
   * 多余的空格会被最后一个变量捕获。

5. 这种方法用来进行通用的日志处理比较合适，比如增添内容等，其实不是很适合生成 JSON，因为它是通过暴力拼接字符串构建 JSON，比如当日志内容本身带有引号时，将不会输出一个合法 JSON，比如输入内容为：

   ```bash
   [2020-01-14] "iamabug sleep
   ```

   输出到 Kafka 的内容为：

   ```bash
   {"timestamp": "2020-01-14", "name": ""iamabug", "action": "sleep"}
   ```

   它不是一个JSON，所以使用这种方法时需要注意它的限制，当然，如果你确定它不会包含引号也可以用它来生成 JSON。