+++
date = '2025-06-20T09:59:34+08:00'
draft = false
title = 'Naocs'
+++

## 安装+
linux
```shell 
docker run --name nacos-standalone \
-e MODE=standalone \
-e NACOS_AUTH_TOKEN=SyyMx74CiQ3ciBQaTBpo33oVr0xY0t9ab1jL44aaaeXceNhr \
-e NACOS_AUTH_IDENTITY_KEY=nacos \
-e NACOS_AUTH_IDENTITY_VALUE=nacos \
-p 8080:8080 \
-p 8848:8848 \
-p 9848:9848 \
-d nacos/nacos-server:latest
```
windows cmd
```shell
docker run --name nacos-standalone ^
-e MODE=standalone ^
-e NACOS_AUTH_ENABLE=true ^
-e NACOS_AUTH_TOKEN=SyyMx74CiQ3ciBQaTBpo33oVr0xY0t9ab1jL44aaaeXceNhr ^
-e NACOS_AUTH_IDENTITY_KEY=nacos ^
-e NACOS_AUTH_IDENTITY_VALUE=nacos ^
-p 8080:8080 ^
-p 8848:8848 ^
-p 9848:9848 ^
-d nacos/nacos-server:v3.0.0
```

然后初始化密码![img.png](img.png)  
这里需要注意的是，8080是前端端口，8848是api的接口

### 该死的
1. 一定要开放上面的这三个端口，分别是web页面，http 以及grpc。不然排查错误整死人
2. 该死的，读取配置信息的之后，一定要自己创建命名空间，然后再进行配置后读取