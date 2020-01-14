FROM ubuntu:18.04

# 安装必要工具，logstash 运行需要 JDK
RUN apt-get update && \
    apt-get install -y gnupg apt-transport-https wget openjdk-8-jdk && \

# 使用国内镜像，但有可能还是有点慢
RUN wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add - && \
    echo "deb https://mirrors.tuna.tsinghua.edu.cn/elasticstack/7.x/apt stable main" | tee -a /etc/apt/sources.list.d/elastic-7.x.list && \
    apt-get update --fix-missing && \
    apt-get install logstash

# 安装路径为 /usr/share/logstash，日志路径为 /var/log/logstash
