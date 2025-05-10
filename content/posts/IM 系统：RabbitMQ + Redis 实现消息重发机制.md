---
title: IM 系统：RabbitMQ + Redis 实现消息重发机制
date: 2024-08-30
draft: "false"
tags:
  - RabbitMQ
  - Redis
  - IM
---


![](https://raw.githubusercontent.com/vankykoo/image/main/cut/011.png)



**【消息未收到 ack 的情况】**

1. 发送一条消息后，将消息体存入到 redis 中，并将消息的【uniqueId 和 重发次数】放入 RabbitMQ 消息队列中。
2. 延迟队列中的 【uniqueId 和 重发次数】在规定客户端ack时间内， 会被放入到死信队列，用于检查是否 收到ack消息。
3. 死信队列收到 【uniqueId 和 重发次数】


1. 1. 到 redis 中检查这条消息体是否还存在，存在则说明还没有收到ack
   2. 再检查 重发次数，如果重发次数已达上限，报异常【重试发送多次消息失败】
   3. 如果还没到重发次数上限，就进入消息重发。



**【消息收到 ack 后的处理】**

（私信）

1. 删除 redis 中的等待ack 的消息体，这样死信队列在检查的时候没有查到这条消息体，说明消息已经ack
2. 修改消息状态 为【已送达】。

（群聊）

1. 修改用户的 last_ack_id
2. 删除消息体缓存