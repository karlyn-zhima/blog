---
title: 通过Canal订阅binlog自动更新Redis
tags:
  - Java
  - Canal
  - 消息队列
  - binlog
categories:
  - 项目
mathjax: true
sticky: 1
swiper_index: 1
abbrlink: canal-binlog
published: 2025-04-01 00:08:49
description: 使用Canal订阅binlog实现联系人修改后的自动缓存刷新
---

# 写在前面
这是我扩充我的项目的一个点，有点摸着石头过河的意思，可能很多思路也不够企业化，然后技术选型什么的也不够正确。

# 背景介绍

因为我的项目是一个IM聊天项目，所以前端发来消息带有Uid和联系人Id，这时候后端需要进行权限验证，判断是不是好友，不是好友禁止发消息，这种验证是频繁的，所以用户的联系人要放在缓存里。但是用户可能会频繁的添加删除好友，这时候就需要维护缓存和数据库的一致性。

我一开始是采用手动删除，延迟双删的思路，用户发第一条消息的时候就会出现明显的卡顿。所以其实应该在更新完联系人之后就把新的联系人信息放到缓存里。但是手工操作容易出错忽略在哪里没有删。

刚好看八股看到了这种思路，就是通过订阅binlog，根据消息队列的消息里找出哪个用户的联系人信息被修改了，就来更新对应用户的联系人信息的缓存。

所以就有了这篇博客。

# MySQL开启binlog并且设定为RAW模式
```mysql
mysql> show variables like'%log_bin%';
+---------------------------------+-----------------------------+
| Variable_name                   | Value                       |
+---------------------------------+-----------------------------+
| log_bin                         | ON                          |
| log_bin_basename                | /var/lib/mysql/binlog       |
| log_bin_index                   | /var/lib/mysql/binlog.index |
| log_bin_trust_function_creators | OFF                         |
| log_bin_use_v1_row_events       | OFF                         |
| sql_log_bin                     | ON                          |
+---------------------------------+-----------------------------+
6 rows in set (0.03 sec)
```
这里我简单看了一下我的库，不知道为什么是自己开启的，但是还是准备去配置文件看一眼是不是配置了server_id

这里进去
```
/etc/mysql# vi my.cnf
```
加一下配置开启binlog就行了
令人烦躁时的我50块的京东云服务器的2G内存快要顶不住压力了：）

```
[mysqld]
# 开启 binlog
log-bin = /var/log/mysql/mysql-bin.log

# 设置 server-id（每个 MySQL 实例必须唯一）
server-id = 1

# 可选：设置 binlog 格式（ROW 是推荐的格式）
binlog_format = ROW

# 可选：设置 binlog 过期时间（单位为天）
expire_logs_days = 7

# 可选：限制 binlog 文件大小（单位为字节，默认值为 1GB）
max_binlog_size = 100M
```

# Canal下载并配置

然后是下载Canal，技术选型方面，其实我能选择的不太多，主要就是Canal 和 Debezium。

我选择Canal的原因大抵如下：

轻量级：Canal专注于 MySQL 数据库的 CDC，架构相对简单，更加轻量化。

独立于 Kafka：不像 Debezium一样，最初专为Kafka设计。

易于部署：Canal 的部署相对简单，尤其是对单一数据库的监听、

下载然后一键tar-zxf之后进行一下简单的配置，在提供的样例里修改。

## 修改conf/example/instance.properties
## mysql serverId
```
canal.instance.mysql.slaveId = 100
## 这个要和数据库的server-id不相同
canal.instance.master.address = 127.0.0.1:3306 
canal.instance.dbUsername = zhima  
canal.instance.dbPassword = ********
```
配置完之后可以自己先启动一下，就很简单，bash /bin/startup.sh

然后稍微修改一下数据库中的一行，之后看一下example输出的日志

```log
2025-03-31 22:00:50.725 [destination = example , address = /127.0.0.1:3306 , EventParser] ERROR com.alibaba.otter.canal.common.alarm.LogAlarmHandler - destination:example[com.alibaba.otter.canal.parse.exception.CanalParseException: java.io.IOException: connect /127.0.0.1:3306 failure
Caused by: java.io.IOException: connect /127.0.0.1:3306 failure
    at com.alibaba.otter.canal.parse.driver.mysql.MysqlConnector.connect(MysqlConnector.java:85)
    at com.alibaba.otter.canal.parse.inbound.mysql.MysqlConnection.connect(MysqlConnection.java:104)
    at com.alibaba.otter.canal.parse.inbound.mysql.MysqlEventParser.preDump(MysqlEventParser.java:89)
    at com.alibaba.otter.canal.parse.inbound.AbstractEventParser$1.run(AbstractEventParser.java:171)
    at java.lang.Thread.run(Thread.java:750)
Caused by: java.io.IOException: Error When doing Client Authentication:ErrorPacket [errorNumber=1698, fieldCount=-1, message=Access denied for user 'root'@'localhost', sqlState=28000, sqlStateMarker=#]
    at com.alibaba.otter.canal.parse.driver.mysql.MysqlConnector.negotiate(MysqlConnector.java:325)
    at com.alibaba.otter.canal.parse.driver.mysql.MysqlConnector.connect(MysqlConnector.java:81)
    ... 4 more
]
```

如果像这样一般是配置有问题没连上，修改一下就行，之后成功连上之后就可以自己写一个客户端来进行调用啦，下面这段代码用来测试就很合适。

```java
package com.karlyn.dogie;

import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.protocol.CanalEntry.*;
import com.alibaba.otter.canal.protocol.Message;

import java.net.InetSocketAddress;
import java.util.List;


public class SimpleCanalClientExample {

    public static void run() {
        // 连接信息配置
        String hostname = "*.*.*.*";
        int port = 11111;
        String destination = "example";
        String username = "";
        String password = "";

        // 创建链接
        CanalConnector connector = CanalConnectors.newSingleConnector(
                new InetSocketAddress(hostname, port), destination, username, password
        );
        System.out.println("连接创立成功");
        int batchSize = 1000;

        try {
            connector.connect();
            connector.subscribe(".*\\..*");
            connector.rollback();

            while (true) {
                Message message = connector.getWithoutAck(batchSize); // 获取指定数量的数据
                long batchId = message.getId();
                int size = message.getEntries().size();

                // 没有拿到数据
                if (batchId == -1 || size == 0) {
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                    }
                } else {
                    System.out.printf("message[batchId=%s, size=%s] \n", batchId, size);
                    printEntry(message.getEntries());
                }

                connector.ack(batchId); // 提交确认
                // connector.rollback(batchId); // 处理失败, 回滚数据
            }
        } finally {
            connector.disconnect();
        }
    }

    private static void printEntry(List<Entry> entries) {
        for (Entry entry : entries) {
            if (entry.getEntryType() == EntryType.TRANSACTIONBEGIN || entry.getEntryType() == EntryType.TRANSACTIONEND) {
                continue;
            }

            RowChange rowChange = null;
            try {
                rowChange = RowChange.parseFrom(entry.getStoreValue());
            } catch (Exception e) {
                throw new RuntimeException("ERROR ## parser of eromanga-event has an error , data:" + entry.toString(),
                        e);
            }

            EventType eventType = rowChange.getEventType();
            System.out.println(String.format("binlog[%s:%s] , name[%s,%s] , eventType : %s",
                    entry.getHeader().getLogfileName(),
                    entry.getHeader().getLogfileOffset(),
                    entry.getHeader().getSchemaName(),
                    entry.getHeader().getTableName(),
                    eventType));

            // 数据变化
            for (RowData rowData : rowChange.getRowDatasList()) {
                if (eventType == EventType.DELETE) {
                    printColumn(rowData.getBeforeColumnsList());
                } else if (eventType == EventType.INSERT) {
                    printColumn(rowData.getAfterColumnsList());
                } else {
                    printColumn(rowData.getAfterColumnsList());
                }
            }
        }
    }

    private static void printColumn(List<Column> columns) {
        for (Column column : columns) {
            System.out.println(column.getName() + " : " + column.getValue());
        }
    }

    public static void main(String[] args) {
        run();
    }
}
```

之后开始和消息队列和Springboot整合啦

## 修改canal配置canal.properties
```
canal.serverMode = rabbitMQ

##################################################
#########           RabbitMQ         #############
##################################################
rabbitmq.host = 127.0.0.1
rabbitmq.virtual.host = /
rabbitmq.exchange = canal-exchange
rabbitmq.username = root
rabbitmq.password = 123456
```

同时继续修改instance.properties
```
# mq config
# canal.mq.topic=example
canal.mq.topic=canal-routing-key
##为了过滤指定的表，我还加了如下限定
canal.instance.defaultDatabaseName=dogie

# table regex
canal.instance.filter.regex=dogie\\.user_contact
```

然后重启canal服务

# Springboot集成

消息订阅
```java
package com.karlyn.dogie.Canal;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.amqp.core.Queue;

@Configuration
public class CanalProvider {
    /**
     * 队列
     */
    @Bean
    public Queue canalQueue() {
        /**
         * durable:是否持久化，默认false，持久化队列：会被存储在磁盘上，当消息代理重启时仍然存在；暂存队列：当前连接有效
         * exclusive:默认为false，只能被当前创建的连接使用，而且当连接关闭后队列即被删除。此参考优先级高于durable
         * autoDelete:是否自动删除，当没有生产者或者消费者使用此队列，该队列会自动删除
         */
        return new Queue(RabbitConstant.CanalQueue, true);
    }

    /**
     * 交换机，这里使用直连交换机
     */
    @Bean
    DirectExchange canalExchange() {
        return new DirectExchange(RabbitConstant.CanalExchange, true, false);
    }

    /**
     * 绑定交换机和队列，并设置匹配键
     */
    @Bean
    Binding bindingCanal() {
        return BindingBuilder.bind(canalQueue()).to(canalExchange()).with(RabbitConstant.CanalRouting);
    }
}
```
消息消费，我这里写的比较简单，如果消息消费失败之后我会把它重新放回消息队列，但是这时候消息队列会一直把这个消息发给消费者，所以这块还需要优化一下。

```java

package com.karlyn.dogie.Canal;

import com.alibaba.fastjson.JSON;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.core.type.TypeReference;
import com.karlyn.dogie.entity.enums.UserContactStatusEnum;
import com.karlyn.dogie.entity.po.UserContact;
import com.karlyn.dogie.entity.query.UserContactQuery;
import com.karlyn.dogie.mappers.UserContactMapper;
import com.karlyn.dogie.redis.RedisComponent;
import com.karlyn.dogie.util.JsonUtils;
import com.rabbitmq.client.Channel;
import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitHandler;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.amqp.support.converter.Jackson2JsonMessageConverter;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * Canal消息消费者
 */
@Component
@Slf4j
@RabbitListener(queues = RabbitConstant.CanalQueue)
public class CanalConsumer {
    private final UserContactMapper userContactMapper;
    private final RedisComponent redisComponent;

    public CanalConsumer(UserContactMapper userContactMapper, RedisComponent redisComponent) {
        this.userContactMapper = userContactMapper;
        this.redisComponent = redisComponent;
    }

    @RabbitHandler
    public void Listener(String message, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException, InterruptedException {
        System.out.println("收到canal消息：" + message);
        ObjectMapper objectMapper = new ObjectMapper();
        Map<String, Object> msg = objectMapper.readValue(message,new TypeReference<Map<String, Object>>() {});
        boolean isDdl = (boolean) msg.get("isDdl");
        // 不处理DDL事件
        if (isDdl) {
            return;
        }
        String database = (String) msg.get("database");
        String table = (String) msg.get("table");
        String type = (String) msg.get("type");
        List<LinkedHashMap> data = (List<LinkedHashMap>) msg.get("data");

        if(database.equals("dogie")&&table.equals("user_contact")){
            try{
                for (LinkedHashMap s : data) {
                    String UserId = (String) s.get("user_id");
                    log.info("更新{}的联系人缓存",UserId);
                    UserContactQuery userContactQuery = new UserContactQuery();
                    userContactQuery.setUserId(UserId);
                    userContactQuery.setStatus(UserContactStatusEnum.FRIEND.getStatus());
                    List<UserContact> userContactList = this.userContactMapper.selectList(userContactQuery);
                    List<String> contactIds = userContactList.stream().map(item->item.getContactId()).collect(Collectors.toList());
                    this.redisComponent.cleanUserContact(UserId);
                    if(!contactIds.isEmpty()){
                        this.redisComponent.addUserContactBatch(UserId, contactIds);
                    }
                }
                channel.basicAck(tag,false);
            }catch (Exception e){
                System.out.println(e.getMessage());
                channel.basicNack(tag,false,true);
            }
        }
    }
}
```
