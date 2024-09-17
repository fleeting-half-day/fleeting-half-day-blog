---
title: "Docker 搭建 ELK 8.11.0"
date: 2023-08-27
metaAlignment: center
categories:
- docker
tags:
- ELK
- ES
---

<!--more-->

#### 一、创建网络

```
docker network create elastic
```
#### 二、安装 Elasticsearch

```shell
# 拉取镜像
docker pull elasticsearch:8.11.0
# 运行镜像
docker run -itd --name es --net elastic -p 9200:9200 -p 9300:9300 -m 1GB elasticsearch:8.11.0
```

#### 三、安装 Kibana

* 运行 Kibana

```shell
# 拉取镜像
docker pull kibana:8.11.0
# 运行镜像
docker run -itd --name kibana --net elastic -p 5601:5601 kibana:8.11.0
```

* 配置 Kibana

浏览器访问：[http://127.0.0.1:5601](http://127.0.0.1:5601)
第一次访问需要配置访问令牌，和验证码

```shell
# 随机生成 elastic 密码
docker exec -it es /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
# 也可以进入 es 服务设置自定义密码
# 交互式运行 es
docker exec -it es /bin/bash
# 设置 elastic 用户密码
>bin/elasticsearch-reset-password --username elastic -i


# 注册 Kibana 令牌
docker exec -it es /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
# 生成验证码
docker exec -it kibana bin/kibana-verification-code

# 也可以在 es 启动日志中查看 elastic 密码、Kibana 令牌、HTTP CA 证书 SHA-256 指纹：
docker logs -f es
```
![es log](/images/other/es_log.png)


#### 四、安装 Logstash
* 配置 Logstash 环境

创建 config、pipeline 目录，把 es 生成的 http 证书拷贝到 config 文件夹。

```shell
# 把 es 集群生成的 HTTP CA 证书拷贝到 config 文件夹
cd config
docker cp es:/usr/share/elasticsearch/config/certs .
```
在 config 目录下创建 logstash.yml、pipeline.yml 配置文件，logstash.yml配置如下：
```
http.host: "0.0.0.0"
# 配置xpack monitoring
# 开启监控
xpack.monitoring.enabled: true
# 配置es集群 注意更换成自己的虚拟机的ip
xpack.monitoring.elasticsearch.hosts: ["https://127.0.0.1:9200"]
# 用户名和密码  自行在kibana上更改密码
xpack.monitoring.elasticsearch.username: logstash_system
xpack.monitoring.elasticsearch.password: lroot123
# https通信使用的证书，该证书拷贝自es集群
xpack.monitoring.elasticsearch.ssl.certificate_authority: "/usr/share/logstash/config/certs/http_ca.crt"
# http ca的指纹，es   更换成自己的http ca指纹
xpack.monitoring.elasticsearch.ssl.ca_trusted_fingerprint: 0209798656a65d579d816d6f537e6a157ee5b8c0544aa731f9ba85e7ff878cf0
pipeline.id: main
```

pipeline.yml 配置：
```
- pipeline.id: main
  path.config: "/usr/share/logstash/pipeline"
```

* 配置管道

在 pipeline 目录下创建 *.conf 文件，配置 input、filter、output
> 详细配置参考官网：
> * [input plugins](https://www.elastic.co/guide/en/logstash/8.11/input-plugins.html)
> * [Filter plugins](https://www.elastic.co/guide/en/logstash/8.11/filter-plugins.html)
> * [Output plugins](https://www.elastic.co/guide/en/logstash/8.11/output-plugins.html)

示例如下：
```
# 控制台标准输入输出
input {
    stdin { }
}
output {
    stdout { }
}
```

* 运行 logstash

```shell
# 拉取镜像
docker pull logstash:8.11.0
# 运行 logstash，rm 参数作用：运行结束删除容器
docker run --rm -it --net elastic -v ~/docker/logstash/config/:/usr/share/logstash/config/ -v ~/docker/logstash/pipeline/:/usr/share/logstash/pipeline/ logstash:8.11.0
# 或
docker run -itd --name logstash --net elastic -v ~/docker/logstash/config/:/usr/share/logstash/config/ -v ~/docker/logstash/pipeline/:/usr/share/logstash/pipeline/ logstash:8.11.0
```

> 官网文档:
> [Elasticsearch、Kibana](https://www.elastic.co/guide/en/elasticsearch/reference/8.11/docker.html)
> [Logstash](https://www.elastic.co/guide/en/logstash/current/docker.html)