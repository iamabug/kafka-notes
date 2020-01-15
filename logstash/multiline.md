# 问题

有时一条日志会跨多行，比如 Java 的错误 stacktrace，logstash 如何将多行日志当作一条进行处理？

例如，想要把这样的日志：

```bash
[2020-01-15] ERROR some exception:
	second exception
	third exception
```

处理成如下形式

```bash
[2020-01-15] ERROR some exception:\n	second exception\n	third exception
```

# 思路

> 参考链接：https://www.elastic.co/guide/en/logstash/current/plugins-codecs-multiline.html

1. multiline 插件可以实现该功能，但是输入不能是来自多个主机的（比如各种 beat），因为多个主机的日志会混在一起，产生未知的结果。
2. multiline 插件的基本原理是：当一行日志匹配或不匹配某个预定义的模式时，将这条日志与上一条或下一条日志合并成一条。
3. multiline 有三个重要参数：pattern，negate，what。pattern 是模式字符串，可以是正则表达式，也可以是预定义的模式名；negate 的可选值为 true 或者 false，true 表示匹配了 pattern 进行合并，false 表示不匹配 pattern 时进行合并；what 的可选值为 previous 或者 next，previous 表示合并到上一行，next 表示合并到下一行。

# 答案

参考配置如下：

```bash
input {
    file {
        path => ["/Users/iamabug/1.log"]
        codec => multiline {
        		# 可以不用写完整的正则表达式，只写开头，能够区分是换行的日志即可
            pattern => "^\[\d{4}-\d{2}-\d{2}\]\s+[a-zA-Z]+\s+[\s\S]+"
            negate => true
            what => "previous"
        }
    }
}

output {
		# 为了测试方便，直接输出在终端
    stdout {}
}
```

# 验证

1. 在 logstash 安装目录执行如下命令：

   ```bash
   bin/logstash --path.config config/logstash.conf
   ```
   
2. 向 `/Users/iamabug/1.log` 末尾添加如下日志：

  ```bash
  [2020-01-15] ERROR some exception:
  second exception
  third exception
  [2020-01-15] INFO no problem
  ```

  注意：这里额外添加了一条符合模式的单行日志，否则 logstash 会一直等待。

3. 观察 logstash 的输出：

  ![](https://tva1.sinaimg.cn/large/006tNbRwly1gax9kqnzw8j311p09a0u2.jpg)

  可以看到，多行日志被使用换行符拼接成了一条日志，存储在 `message` 中。