+++
date = '2025-07-29T15:54:12+08:00'
draft = true
title = 'Network'
+++

## 常用命令

1. 查看网络 `docker network ls`
2. 创建网络 `docker network create xxx_network`
3. 已经启动的服务加入到网络中 `docker network connect xxx_network container_name/id`
4. 启动的时候加入网络 以goods-api服务为例:
   `docker run --name goods-api --network project_network -p 8082:8082 gods-api:v1.0.0`

## 网络问题

### 记一次docker部署遇到的网络问题

#### 背景

docker中运行了nacos，consul，且在本地运行时能够正常访问。放进docker里面就出问题了

#### 折腾

1. 部署到容器内部后发现并不能够访问`nacos`和`consul`。`consul`的`gRPC健康检查`失败
2. 尝试使用`172.17.0.1`发现能够正常访问`nacos`，但是`consul`就没法用了
3. 尝试分别创建`nacos_network` 以及`consul_network`,然后服务运行失败之后，就`手动`将这两个网络加入到`goods-service`
4. 然后发现`goods-api`又没办法请求到`goods-service`
5. `核心是：所有的服务都要在同一个网络才能够相互访问` ，而且`需要根据服务的名称进行访问`
6. 而且之前的配置的nacos的ip或者consul的`ip`需要改成他们自己的`容器名称`

#### 解决方案

创建一个project_network,都加入这个网络就行了。

#### 总结

**"同一网络 + 用服务名访问 = Docker 服务互通的根本解法。"**

### 记录一次docker build的网络问题

#### 背景

需要将服务编译进docker进行运行

#### 问题和折腾

1. 在执行到 RUN go mod tidy 的时候发现，有些包拉去失败
2. 多半是代理的问题。然后就搜七牛云的go代理

#### 解决

在一阶段构建的时候，将代理配置在go mod tidy 之前

``` 
ENV GOPROXY=https://goproxy.cn,direct
ENV GO111MODULE=on
```

#### 总结

代理问题，轻松解决