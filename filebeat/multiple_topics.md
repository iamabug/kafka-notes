# 问题

filebeat 同时采集多个日志文件，且日志文件的格式有所差异，为了方便后续处理，可以输出到 Kafka 的不同 topic 中，那么，如何根据文件名或文件内容将日志输出到不同的 topic 呢？

例如：

1. 将 `error.log` 的的内容输出到 `error`，将 `info.log` 的内容输出到 `info`
2. 将符合正则的日志内容输出到 `type1`，不符合的内容输出到 `type2`

# 思路

> 参考链接1：https://www.elastic.co/guide/en/beats/filebeat/master/kafka-output.html
>
> 参考链接2：https://www.elastic.co/guide/en/beats/filebeat/master/defining-processors.html

1. kafka 输出插件有一个 topics 配置选项，提供一个 topic 列表，每个 topic 对应一个条件。
2. filebeat 支持的条件有很多，比如相等、包含、正则、范围、与或非等等。
3. filebeat 内部变量：source 表示文件名，message 表示文件的一行日志。

# 答案

1. 根据文件名输出到不同 topic 的参考配置：

   ```yaml
   filebeat.inputs:
       - type: log
         enabled: true
         paths:
             - /Users/iamabug/workspace/filebeat/test/*.log
   
   output.kafka:
       enabled: true
       hosts: ["localhost:9092"]
       codec.format:
           string: '%{[message]}'
       topics:
           - topic: "error"
             when.contains.source: "error"
           - topic: "info"
             when.contains.source: "info"
   ```

2. 根据文件内容输出到不同 topic 的参考配置：

   ```bash
   filebeat.inputs:
       - type: log
         enabled: true
         paths:
             - /Users/iamabug/workspace/filebeat/test/1.log
   
   output.kafka:
       enabled: true
       hosts: ["localhost:9092"]
       topic: "default"
       codec.format:
           string: '%{[message]}'
       topics:
           - topic: "type1"
             when.regexp.message: "type1.*"
           - topic: "type2"
             when.regexp.message: "type2.*"
   ```

   **值得说明的是：官方文档里虽然说支持正则表达式，如果尝试使用的话，会有如下报错：**

   ```bash
   Exiting: error loading config file: yaml: line 20: found unknown escape character
   ```

   在 elastic 论坛提交了这个问题：https://discuss.elastic.co/t/filebeat-kafka-out-plugin-error-when-configuring-multiple-topics-using-regex/215264，**更新：正则表达式需要使用单引号包起来。**

# 验证

## 根据文件名

1. 创建 info 和 error 两个 topic，在 filebeat 安装路径下启动 filebeat：

   ```bash
   ./filebeat -e -c filebeat.yml
   ```

2. 向 info.log 和 error.log 添加内容：

   ```bash
   echo this is an info message > info.log
   echo this is an error message > error.log
   ```

3. 消费 info 和 topic 这两个 topic：

   ![](https://tva1.sinaimg.cn/large/006tNbRwly1gaydva1shxj311e05pjsi.jpg)

   可以看到，不同文件的内容进入了不同的 topic。

## 根据文件内容

1. 创建 type1，type2 和 default 三个 topic，在 filebeat 安装路径下启动 filebeat：

   ```bash
   ./filebeat -e -c filebeat.yml
   ```

2. 向 1.log 添加不同格式的内容：

   ```bash
   echo "type1 xxx" >> 1.log
   echo "type2 yyy" >> 1.log
   echo "no type" >> 1.log
   ```

3. 消费三个 topic，观察输出：

   ![](https://tva1.sinaimg.cn/large/006tNbRwly1gayf30qeyij312c09fq4q.jpg)

   可以看到，type1 开头的日志进入了 type1，type2 开头的日志进入了 type2，其它的日志进入了 default。
