+++
date = '2025-06-14T21:53:59+08:00'
draft = false
title = 'consul'
+++

## docker 里面安装和启动consul
1. 创建网络
```shell
docker network create consul_network
```
2. 指定consul使用的网络
```shell
docker run -d --name consul --network consul_network -p 8500:8500 -p 8300:8300 -p 8301:8301 -p 8302:8302 -p 8600:8600/udp hashicorp/consul agent -dev -client=0.0.0.0
```
3. 其他服务启动时，指定跟consul在同一个网络
4. 

