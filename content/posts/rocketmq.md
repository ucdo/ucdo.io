+++
date = '2025-07-01T15:31:15+08:00'
draft = true
title = 'Rocketmq'
+++

## 今天在安装使用rocket的时候

### docker-compose 命令：
```docker
version: '3.8'
services:
  namesrv:
    image: apache/rocketmq:5.3.2
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    networks:
      - rocketmq
    command: sh mqnamesrv
  broker:
    image: apache/rocketmq:5.3.2
    container_name: rmqbroker
    ports:
      - 10909:10909
      - 10911:10911
      - 10912:10912
    environment:
      - NAMESRV_ADDR=rmqnamesrv:9876
    depends_on:
      - namesrv
    networks:
      - rocketmq
    command: sh mqbroker
  proxy:
    image: apache/rocketmq:5.3.2
    container_name: rmqproxy
    networks:
      - rocketmq
    depends_on:
      - broker
      - namesrv
    ports:
      - 8083:8080
      - 8081:8081
    restart: on-failure
    environment:
      - NAMESRV_ADDR=rmqnamesrv:9876
    command: sh mqproxy
  console:
    image: apacherocketmq/rocketmq-dashboard:latest
    container_name: rmqconsole
    ports:
      # 将宿主机的 8090 端口映射到Console容器的 8080 端口，用于访问Web UI
      # 这样避免了宿主机 8080 端口可能存在的冲突
      - 8085:8080
    environment:
      - JAVA_OPTS=-Drocketmq.namesrv.addr=rmqnamesrv:9876
      # 可选：启用Console的认证功能
      # - rocketmq.console.login.username=admin
      # - rocketmq.console.login.password=admin
    depends_on:
      - namesrv
    networks:
      - rocketmq
    restart: on-failure
networks:
  rocketmq:
    driver: bridge
```

### 在使用创建生产者时，发现连接不上grpc
1. 网上一搜，说是docker里面的**端口要和宿主机的端口一致**，至少grpc的要一致，他这个grpc占用的是**8081**端口
2. 我尝试修改了docker-compose.yml中的 proxy并用 docker-compose -p rokcermq up -d proxy 对proxy这一个容器进行重启。
3. 然后发现又行了
4. 我再改回去，重复 2 的步骤，又不行了

### 5.x版本没有服务器推送到客户端来的。只能自己simple地轮询去拉
1. 当然，要是push过来的话，推太多了消费不过来也是个问题

### 事务消息
1. 事务消息是可靠的
2. 创建事务消息是不会立马创建，而是创建一个半事务消息并持久化，等待本地事务提交/回滚这条消息。此时，该条消息对于下游是不可见的
3. 如果等待了一定的时间之后，发现没有提交/回滚，就会检查发送者，并根据返回进行提交/回滚
4. 如果提交，则重新放在普通消息中，并等待消费
5. 消费过程中，如果迟迟没有收到回应，就会按策略进行消息重试，即mq重新投递消息
6. 消费确认，提交该消息成功消费或者消费失败。默认会保留，只是标记为消费状态，未删除之前可以进行回溯
7. 消息删除：滚动机制删除最早的消息