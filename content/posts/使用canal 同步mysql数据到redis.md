---
title: 使用canal 同步mysql数据到redis
date: 2024-04-24
draft: "false"
tags:
  - MySQL
  - 数据库
  - canal
---

## 0. 流程说明

①更新 mysql 数据

②canal 获取 binlog，java 的canal客户端感知到。

③将更新的数据 进行 封装（操作类型，数据库名，数据库表名，操作行的属性。。）

④将封装好的数据存入到 MQ 中 作为缓冲。

⑤MQ 处理 数据：将数据更新到redis 中

##  1. 安装canal

根据 github 文档，将canal安装到 linux。



## 2. 客户端代码

```java
@Component
@Slf4j
public class CanalRedisClient {

    @Resource
    @Lazy
    private CanalRedisProxy canalRedisProxy;
    
    @Value("${canal.address}")
    private String canalAddress;

    @Value("${canal.port}")
    private int canalPort;
    
    @Value("${canal.destination}")
    private String canalDestination;

    @PostConstruct
    public void canalExample() {
        // 创建链接
        CompletableFuture.runAsync(() -> {
            CanalConnector connector = CanalConnectors
                    .newSingleConnector(new InetSocketAddress(canalAddress,
                            canalPort), canalDestination, "", "");

            log.info("连接 canal 成功！");
            int batchSize = 1000;
            int emptyCount = 0;

            try {
                connector.connect();
                connector.subscribe(".*\\..*");
                connector.rollback();
                int totalEmptyCount = 120;
                while (emptyCount < totalEmptyCount) {
                    Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据
                    long batchId = message.getId();
                    int size = message.getEntries().size();
                    if (batchId == -1 || size == 0) {
                        emptyCount++;
                        log.info("empty count : {}", emptyCount);
                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                        }
                    } else {
                        emptyCount = 0;
                        //把修改消息进行打包
                        toJsonString(message.getEntries());
                    }

                    connector.ack(batchId); // 提交确认
                }

                log.warn("empty too many times, exit");
            } finally {
                connector.disconnect();
            }
        });

    }

    /**
     * 一行数据封装为一个 Canal 类
     * @param entries
     */
    private void toJsonString(List<Entry> entries){
        for (Entry entry : entries) {
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND) {
                continue;
            }

            RowChange rowChage = null;
            try {
                rowChage = RowChange.parseFrom(entry.getStoreValue());
            } catch (Exception e) {
                throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(),
                        e);
            }

            CanalModel canalModel = new CanalModel();

            canalModel.setDbName(entry.getHeader().getSchemaName());
            canalModel.setTableName(entry.getHeader().getTableName());
            canalModel.setEventType(rowChage.getEventType().toString());

            for (RowData rowData : rowChage.getRowDatasList()) {
                String dataId = getDataId(rowData.getBeforeColumnsList());
                if (StringUtils.hasText(dataId)){
                    canalModel.setDataId(dataId);
                }else {
                    canalModel.setDataId(getDataId(rowData.getAfterColumnsList()));
                }

                canalModel.setBefore(columnToJsonString(rowData.getBeforeColumnsList()));
                canalModel.setAfter(columnToJsonString(rowData.getAfterColumnsList()));

                //把jsonString 放入rabbitmq中
                canalRedisProxy.putMQ(canalModel);
            }
        }
    }

    /**
     * 获取数据的id，用于 redis 的 存储key
     * @param columns
     * @return
     */
    private String getDataId(List<Column> columns){
        for (Column column : columns){
            if ("id".equals(column.getName())){
                return column.getValue();
            }
        }

        return "";
    }

    /**
     * 把表中的行转换为json格式，用于存储到redis
     * @param columns
     * @return
     */
    private String columnToJsonString(List<Column> columns){
        HashMap<String, String> map = new HashMap<>();
        for (Column column : columns) {
            String columnName = column.getName();
            String resultStr = CaseFormat.LOWER_UNDERSCORE.to(CaseFormat.LOWER_CAMEL, columnName);

            map.put(resultStr, column.getValue());
        }

        String jsonString = JSONObject.toJSONString(map);

        return jsonString;
    }
}
```



## 3.处理同步的数据

1. 放入 MQ 中做缓冲。

```java
@Component
@Slf4j
public class CanalRedisProxy {

    @Resource
    private RabbitTemplate rabbitTemplate;

    public void putMQ(CanalModel canalModel) {
        String eventType = canalModel.getEventType();

        if (!StringUtils.hasText(eventType)){
            throw new NullPointerException("内容为空！");
        }

        String routingKey = "";
        if ("DELETE".equals(eventType)){
            routingKey = "canal.redis.delete";
        }else if ("UPDATE".equals(eventType)){
            routingKey = "canal.redis.update";
        } else if ("INSERT".equals(eventType)) {
            routingKey = "canal.redis.insert";
        }

        rabbitTemplate.convertAndSend("canal-redis-exchange", routingKey, canalModel);
    }
}
```





2. 更新redis数据

```java
package com.vanky.chat.mq.service;

import com.rabbitmq.client.Channel;
import com.vanky.chat.common.bo.CanalModel;
import com.vanky.chat.common.utils.RedisUtil;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.core.Message;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.io.IOException;

/**
 * @author vanky
 * @create 2024/4/21 20:35
 */
@Component
@Slf4j
public class CanalRedisService {

    /**
     * 插入数据
     */
    @RabbitListener(queues = {"canal.redis.insert.queue"})
    public void CanalRedisInsertListener(Message message, Channel channel, CanalModel canalModel){

        String key = "im:data:" + canalModel.getTableName() + ":" + canalModel.getDataId();

        log.info("redis 【插入】 数据：{}", key);
        RedisUtil.put(key, canalModel.getAfter());

        try {
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 更新数据
     */
    @RabbitListener(queues = {"canal.redis.update.queue"})
    public void CanalRedisUpdateListener(Message message, Channel channel, CanalModel canalModel){

        String key = "im:data:" + canalModel.getTableName() + ":" + canalModel.getDataId();

        log.info("redis 【更新】 数据：{}", key);
        RedisUtil.put(key, canalModel.getAfter());

        try {
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }

    /**
     * 删除数据
     */
    @RabbitListener(queues = {"canal.redis.delete.queue"})
    public void CanalRedisDeleteListener(Message message, Channel channel, CanalModel canalModel){
        String key = "im:data:" + canalModel.getTableName() + ":" + canalModel.getDataId();

        log.info("redis 【删除】 数据：{}", key);
        RedisUtil.del(key);

        try {
            channel.basicAck(message.getMessageProperties().getDeliveryTag(), false);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

