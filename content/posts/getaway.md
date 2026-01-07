+++
date = '2025-07-11T10:27:28+08:00'
draft = true
title = 'Getaway'
+++

# 服务网关

## 目的：

1. 动态路由
2. 服务发现
3. 负载均衡
4. 熔断限流降级
5. 流量控制
    1. 黑名单
    2. 反爬策略

## 本次使用的是 **kong**

## kong = nginx + lua

## docker-compose -- 社区版 + 界面（konga）

```docker-compse.yml
version: '3.9'

volumes:
  kong_db_data: { }

networks:
  kong-ee-net:
    driver: bridge

# 公共 Kong 配置（无 license）
x-kong-config: &kong-env
  KONG_DATABASE: postgres
  KONG_PG_HOST: kong-ee-database
  KONG_PG_DATABASE: kong
  KONG_PG_USER: kong
  KONG_PG_PASSWORD: kong

services:

  kong-ee-database:
    container_name: kong-ee-database
    image: postgres:15  # 推荐用非 latest，避免未来兼容性问题
    restart: on-failure
    volumes:
      - kong_db_data:/var/lib/postgresql/data
    networks:
      - kong-ee-net
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kong
    healthcheck:
      test: [ "CMD", "pg_isready", "-U", "kong" ]
      interval: 5s
      timeout: 10s
      retries: 10
    ports:
      - "5432:5432"

  kong-bootstrap:
    image: kong:3.6  # 使用社区版 Kong 镜像（推荐指定版本）
    container_name: kong-bootstrap
    networks:
      - kong-ee-net
    depends_on:
      kong-ee-database:
        condition: service_healthy
    restart: "no"  # 只运行一次进行迁移
    environment:
      <<: *kong-env
    command: kong migrations bootstrap

  kong-cp:
    image: kong:3.6
    container_name: kong-cp
    restart: on-failure
    networks:
      - kong-ee-net
    depends_on:
      kong-bootstrap:
        condition: service_completed_successfully
    environment:
      <<: *kong-env
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: 0.0.0.0:8001, 0.0.0.0:8444 ssl
    ports:
      - "8000:8000"  # Proxy HTTP
      - "8443:8443"  # Proxy HTTPS
      - "8001:8001"  # Admin API HTTP
      - "8444:8444"  # Admin API HTTPS

  konga:
    image: pantsel/konga:latest
    container_name: konga
    restart: unless-stopped
    ports:
      - "1337:1337"
    networks:
      - kong-ee-net
    environment:
      NODE_ENV: production
```

## 启动  docker-compose up -d

## routes - server - upstream

### routes

1. 一个路由可以有多个服务

### server

1. 服务，和目标服务和 upstream 一一对应

### upstream

1. 类似与 **nginx** 的配置多个服务，然后可以进行负载均衡

### 待会儿玩一下就知道了
### 需要注意的是，现在是通过固定的ip和端口访问，所以，各个服务之间需要区分是哪个，比如 /v1/order/   /v1/goods/ ...
## 硬编码配置，即指定服务的 **protocol://ip:port**
1. 比如我的商品服务是 http://192.168.3.5:8082/v1/goods
2. 在server里面配置就是配置这个， 然后在path配置 **/**。
3. 则访问的时候就是 http://192.168.3.5:8000/v1/goods 
4. 如果server配置的 **/v1** ，则通过 http://192.168.3.5:8000/goods  访问