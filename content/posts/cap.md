+++
date = '2025-06-27T15:07:48+08:00'
draft = false
title = 'Cap'
+++

## CAP 
分布式系统理论基石
### consistency 一致性
"all nodes see same data in same time"

### availability 可用性
"read and write always success"

### partition tolerance 分区容错性
 
### CP 一致性 + 分区容错
1. 先保证数据一致性
2. consul，redis

### AP 


### CA
mysql

## base理论
对一致性和可用性的权衡。即使无法做到强一致性，但每个应用根据自身业务特点，用合适的方式达到最终一致性
### 基本可用
1. 响应时间上的损失
2. 引导降级页面：系统繁忙等 。 熔断/降级

### 软状态

### 最终一致性
