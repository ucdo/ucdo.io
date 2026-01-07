+++
date = '2025-06-14T21:53:59+08:00'
draft = false
title = 'My First Post - docker'
+++

## docker

1. 需要特别注意的时。如果时docker想连接宿主机的ip。需要通过 `host.docker.internal:port`
2. 我这里是为了做一个关于consul访问我宿主机的服务，检查服务的健康状态的东西

## 常用命令
### 查看镜像
docker ps -a

### 查看某个容器有没有问题
dokcer logs [查看镜像 命令中输出的container_id]

### 安装服务

#### redis
```shell 
dokcer run -p 6379:6379 -d redis:latest redis-server
```

#### 查看某个容器在docker内的ip地址
```shell
 docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_id
```