# 问题

Kafka 要求接收的日志是 JSON 格式（否则下游的 Flink 不能正常处理），但第三方软件的日志格式修改不便，如何通过 logstash 将日志处理成 JSON 格式？

# 思路

> 参考链接：https://www.elastic.co/guide/en/logstash/7.5/plugins-filters-grok.html

1. logstash 的 grok 插件和 dissect 插件都可以从结构凌乱的日志中提取信息，但 grok 基于正则表达式，而 dissect 基于分割符，grok 功能更强大，也更可靠，但也更耗资源。
2. grok 有两种模式匹配的方法，一是使用预定义的模式，二是使用自定义的模式。预定义模式的匹配方式为：`%{SYNTAX:SEMANTIC}`，其中 `SYNTAX` 为模式名称，匹配成功时对应字符串将被存储在 `SEMANTIC` 变量中；自定义模式的匹配方式为：`(?<field_name>pattern)`，其中 `pattern` 是正则表达式，`field_name` 为匹配成功后存储数据的变量。
3. 在自定义模式串的方法下，`field_name` 可以是 `[obj][key1]` 形式，这样相当于创建一个名为 `obj` 的对象，并将匹配到的字符串存储到这个对象的 `key1` 字段中，相当于字典或 HashMap，可以直接输出为 JSON 格式。

# 答案

假设一条日志样例为：

```bash
[2020-01-14 15:00:00] INFO iamabug is not happy
```

希望将它处理成如下格式：

```bash
{"timestamp": "2020-01-14 15:00:00", "level": "INFO", "message": "iamabug is not happy"}
```

参考配置为：

```bash
input {
	file {
		path => ["/Users/iamabug/1.log"]
	}
}
filter {
	grok {
		match => {
            "message" => "\[(?<[res][timestamp]>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\]\s+(?<[res][level]>\w+)\s+(?<[res][message]>.*)"
		}
	}
}

output {
	kafka {
        topic_id => "test"
		codec => line { format => "%{[res]}" }
        bootstrap_servers => "localhost:9092"
	}
}
```

# 验证

1. 启动 Kafka，创建名为 `test` 的 topic，或者开启自动创建，然后消费 `test`：

   ```bash
   kafka-console-consumer.sh --topic test --bootstrap-server localhost:9092
   ```

2. 安装（Docker 中的安装方式参见 [Dockerfile](https://github.com/iamabug/kafka-notes/blob/master/logstash/setup/Dockerfile)）并启动 logstash，假设配置文件放在 logstash 安装目录下的 config 目录下，启动命令为：

   ```bash
   bin/logstash --path.config config/logstash.conf
   ```

3. 向 `/Users/iamabug/1.log`  添加两条日志，第二条日志中包含双引号：

   ```bash
    echo '[2020-01-14 15:00:00] INFO iamabug is not happy' >> ~/1.log
    echo '[2020-01-14 15:00:00] INFO "I want to go home", says iamabug' >> ~/1.log
   ```

4. 查看 Kafka 命令行输出：

   ![](https://tva1.sinaimg.cn/large/006tNbRwly1gaw50rf4obj312h041dgo.jpg)

   可以看到，双引号会自动进行转义。
