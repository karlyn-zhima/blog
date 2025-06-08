---
title: 轻量云服务器结合RabbitMQ实现调用本地AI接口
tags:
  - Java
  - 消息队列
categories:
  - 项目
mathjax: true
description: 如果有一个部署在在公网上的服务端，但是算力不支持部署DeepSeek，但是本地有一个机器可以部署，可以尝试用RabbitMQ来提供消息的中转，充当RPC调用的服务。
abbrlink: deepseek
sticky: 1
swiper_index: 1
published: 2025-03-28 20:50:42
---

# 背景

我有一个仿微信的IM聊天项目，但是想做出一点新的东西，AI这个风口上所以还是想看看能不能做点AI接口的调用。但是直接调用官方接口意义不大，其实本质上来说，本地用Ollama部署完然后本地调用本地接口其实也不算很有难度的事情。

但是恰好我的IM项目就有这么一点——需要支持跨服务器之间的消息发送，所以设计了一个发布订阅的中间件来进行服务器上某个节点的消息扩散到其他服务器上，这样就天然支持了一件事情，就是A服务器的任务，如果被B服务器处理了，依然可以发送回到A服务器。

## 简单展示

客户端发送消息给服务端，我的服务端是京东云的一个轻量级服务器，2G内存的那种。

现在我在客户端给服务端发一条消息，这时候我没关我本地的服务器啊，也就是说，我本地有个能处理AI图片的服务器挂在那里消费消息队列，这时候有正常的响应（别介意，不是32B，只是我当时觉得能跑，但是确实太慢了，后面我直接换1.5b了）

![客户端发消息](/picture/ai_message.png)

这时候我们看看我京东云的日志，可以看到，有一条日志说消息队列消息增加，这是我在消息进入交换机的时候打的一个log，证明这条消息在不同服务器上扩散了，这时候就要去自己服务器上把这条消息发给对应的人，如果自己维护的WebSocketChannel里没有，就直接不管了。

```java
@RabbitListener(queues = "${rabbit.queue.message}")
private void Listener(String msg, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
    MessageSendDto sendDto = JsonUtils.convertJson2Obj(msg, MessageSendDto.class);
    log.info("消息队列收到消息");
    log.info(msg);
    channelContextUtils.sendMessage(sendDto);
    channel.basicAck(tag, false);
}
```

这条消息直接被丢弃了，但是因为是机器人，所以我们同时把它加到了机器人对应的消息队列里，但是我们京东云的日志并没有后续的处理。日志如下：

![京东云日志](/picture/jd_log.png)

但是我们本地处理了，这是我本地运行jar包的输出

![本地输出](/picture/local_log.png)

这是消息队列的消费情况。

![消息队列](/picture/mq1.png)

这时候如果我们停掉本地的服务器，消息就会堆积在消息队列里等待有能力处理的服务器上线。

![消息队列](/picture/mq2.png)

自然也不会给用户消息反馈。

![客户端发消息](/picture/ai_message2.png)

# 技术选型

为了后续扩展方便，以及为了跨服务器聊天就是用了RabbitMQ的fanout扇出交换机来进行服务器之间的消息扩散，所以这次AI接口的技术扩展我们继续使用RabbitMQ，但是与之前选择使用发布订阅模型不同，我们这次选择的模式是生产者-消费者模型，也就是工作队列模式。

## 这里简单介绍一下RabbitMQ的五种模式吧

### 简单模式

包含一个生产者，一个消费者，一个队列。生产者发送消息，消费者监听并消费消息。

这种模式的作用为：解耦，削峰填谷

![简单模式](/picture/simple.png)

其实邮件、聊天都是这种场景的受众，只不过我们的服务器充当了一个消息队列的功能

### 工作队列模式

这种模式就是一个生产者，一个队列，多个消费者，生产者源源不断往队列里放任务，消费者监听并处理任务。这样的模式也被称为能者多劳模式，能力越强的消费者消费的消息更多。

![工作队列](/picture/work.png)

但是需要说的是，Springboot集成RabbitMQ，默认限制限制消费者一次从队列里获取250条消息，也就是说，消费者会一次预支250条消息，能力差的消费者这250条消息可能会处理很久。这点在我们的任务中表现的极为明显。我有一个7650GRE的卡和一张4090的卡，同时用这两个本地机器消费消息，显然7650GRE的能力比不上4090，250条消息要处理很久才能结束，但是我们的项目是一个实时聊天项目，可能4090处理完250条消息之后又会接着获取新的消息，导致后来的消息比前来的消息更快被模型处理和响应。

所以这里我们限制了每个消费者一次只能获取一条消息，处理完之后才能继续获取。

```propreties
spring.rabbitmq.listener.simple.prefetch=1
```

这种模型最典型的应用场景就是抢红包，但是可能会出现红包余额被错误修改的情况，这种时候需要对红包余额加锁或者CAS操作。

### 发布订阅模式

发布订阅模式与工作队列模式不同在于，一条消息可以被多个消费者消费，这种在RabbitMQ中的实现就是fanout交换机，fanout交换机将获得的消息扇出到bind到它上面的每个消息队列中，每个消息队列被一个消费者消费，这样即可构成发布订阅模式。

![发布订阅模式](/picture/sub.png)

这种模式最为经典的类比就是广播消息。但是这种模式和后面的路由模式的差别就在于无法过滤消息，也就是说要扇出就会扇出到全部绑定的队列。

### 路由模式

![路由模式](/picture/router.png)

路由模式根据生产者提供的路由key将消息发送到绑定到交换机上且路由key符合的消息队列。

### Topic模式

主题模式，是由路由模式衍生出来的一种模式，路由模式并不支持模糊匹配，路由Key必须完全对应才会发送到对应的消息队列，但是主题模式不同，可以使用通配符匹配

![Topic模式](/picture/topic.png)

1. 星号 和 井号代表通配符

2. 星号匹配1个词, #匹配一个或多个词（* 匹配一级任意多个字符，# 匹配多级任意多个字符）

​       例如：routingKey为"user.#"，表示可以匹配"user.add"和"user.add.log"。

​       routingKey为"user.*"，表示可以匹配"user.add"，对于"user.add.log"则无法匹配。

3. 路由功能添加模糊匹配

4. 消息产生者产生消息,把消息交给交换机

5. 交换机根据key的规则模糊匹配到对应的队列,由队列的监听消费者接收消息消费 

## 具体实现

到这里我们复习完了五种模式，我们选择了工作队列模式，同时限定消费者只能消费一个队列。

所以我们在具体实现中，判断用户发来的消息是否是发给指定的机器人ID的，在这里我们设定为URobot，如果是这个ID，那么我们就将用户的UID以及用户发送的信息先简单打个信息表，打完信息表之后直接将这个信息封装为一个消息Dto，序列化之后加入到消息队列中去。

```java
if (Constants.ROBOT_UID.equals(contactId)) {
    SysSettingDto sysSettingDto = redisComponet.getSysSetting();
    TokenUserInfoDto robot = new TokenUserInfoDto();
    robot.setUserId(sysSettingDto.getRobotUid());
    robot.setNickName(sysSettingDto.getRobotNickName());
    ChatMessage robotChatMessage = new ChatMessage();
    robotChatMessage.setContactId(sendUserId);
    //封装消息装到作为AI返回发送地址以及prompt准备加入到消息队列中去
    robotChatMessage.setMessageType(MessageTypeEnum.CHAT.getType());
    AIRabbitDto aIRabbitDto = new AIRabbitDto();
    aIRabbitDto.setChatMessage(robotChatMessage);
    aIRabbitDto.setMessage(chatMessage.getMessageContent());
    aIRabbitDto.setTokenUserInfoDto(robot);
  	//将对应的消息投递到对应的消息队列中去
    rabbitTemplate.convertAndSend("dogie.direct","chat",JsonUtils.convertObj2Json(aIRabbitDto));
} else {
  	messageHandler.sendMessage(messageSend);
}
```

之后我们就需要设定对应的Listener了，这里我们存留了一点私心，就是我不希望用分离的方式来实现这个Listener，我最佳的愿望肯定是我有多个服务器，然后有的服务器有能力处理消息传递和AI功能，没有AI功能的服务器只负责对应的消息传递。所以我们给Listener的Bean使用了@ConditionalOnProperty注解，当服务器有能力处理AI对话的时候，就把配置文件中对应的字段设置为true，对应的服务器就会注册这个Listener的Bean，就会进一步调用listener方法，如果服务器没有能力，就不注册这个bean，自然也不会从消息队列里去取这个任务。

关于AI的调用，我看了挺多博客的，他们都说SpringAI可以继承了ollama的调用，我好像没找着，所以还是手搓了一个发消息的方法，就是使用OkHttpClient去调这个api，为了等待异步消息结束，我加了个CountDownLatch

```java
package com.karlyn.dogie.listener;

import com.karlyn.dogie.entity.dto.AIRabbitDto;
import com.karlyn.dogie.entity.dto.OllamaResult;
import com.karlyn.dogie.entity.dto.TokenUserInfoDto;
import com.karlyn.dogie.entity.po.ChatMessage;
import com.karlyn.dogie.service.ChatMessageService;
import com.karlyn.dogie.util.JsonUtils;
import com.rabbitmq.client.Channel;
import lombok.extern.slf4j.Slf4j;
import okhttp3.*;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

@Slf4j
@Component
@ConditionalOnProperty(name = "rabbit.listener.ai.enabled", havingValue = "true", matchIfMissing = true)
public class AIListener {
    private final ChatMessageService chatMessageService;
    @Value("${ai.timeout}")
    private Integer timeout;
    @Value("${ai.url}")
    private String URL_OLLAMA;
    @Value("${ai.model}")
    private String MODEL_DEEPSEEK;

    public AIListener(ChatMessageService chatMessageService) {
        this.chatMessageService = chatMessageService;
    }

    @RabbitListener(queues = "deepseek.queue")
    private void Listener(String msg, Channel channel, @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        AIRabbitDto aIRabbitDto = JsonUtils.convertJson2Obj(msg, AIRabbitDto.class);
        log.info("消费消息");
        log.info(aIRabbitDto.toString());
        try {
        		getAiResult4Deepseek(aIRabbitDto.getMessage(),
        										aIRabbitDto.getChatMessage(),
                            aIRabbitDto.getTokenUserInfoDto());
          	//如果消费成功了，就手动确认一下
         		channel.basicAck(tag, false);
        } catch (Exception e) {
            //如果消费失败了就得Nack一下，把对应的消息重新塞回消息队列
            channel.basicNack(tag, false,true);
            throw new RuntimeException(e);
        }
    }

    private String getAiResult4Deepseek(String message, ChatMessage robotChatMessage, TokenUserInfoDto robot) throws InterruptedException {
        // 设定头参数
        Map<String, Object> params = new HashMap<>();
        params.put("prompt", message);
        params.put("model", MODEL_DEEPSEEK);
        params.put("stream", false);
        params.put("temperature", 0.7);
        params.put("top_p", 0.9);
        params.put("max_tokens",400);

        String jsonParams = JsonUtils.convertObj2Json(params);

        //创建Http请求
        Request.Builder builder = new Request.Builder().url(URL_OLLAMA);
        RequestBody body = RequestBody.create(MediaType.parse("application/json; charset=utf-8"), jsonParams);
        Request request = builder.post(body).build();

        // 配置OkHttpClient
        OkHttpClient client = new OkHttpClient.Builder()
                .connectTimeout(timeout, TimeUnit.SECONDS)
                .writeTimeout(timeout, TimeUnit.SECONDS)
                .readTimeout(timeout, TimeUnit.SECONDS)
                .build();

        CountDownLatch eventLatch = new CountDownLatch(1);//定义一个只有1的计数器
        StringBuilder resultBuffer = new StringBuffer(); // 用来收集消息

        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                log.error("请求失败", e);
                eventLatch.countDown(); // 请求失败计数器也减一
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if (response.isSuccessful()) {
                    try (ResponseBody responseBody = response.body()) {
                        if (responseBody != null) {
                            // 读取响应内容
                            String fullResponse = responseBody.string();
                          	//这里我是定义了一个对应的参数来解析这个responseBody
                            OllamaResult aiResult = JsonUtils.convertJson2Obj(fullResponse, OllamaResult.class);
                            log.info(aiResult.getResponse());
                            resultBuffer.append(aiResult.getResponse()); //获取消息
                        }
                    }
                } else {
                    log.error("获取失败", response);
                }
                eventLatch.countDown(); // 请求成功计数器减一
            }
        });

        eventLatch.await(); //等待计数器为0，也就是要么失败要么成功
        String[] messages = resultBuilder.toString().split("</think>\n\n");
        if(messages.length<=1) messages[0]="服务器繁忙，请稍后再试";
        // 如果成功直接把消息封装到Message里去，然后就可以把它继续加到消息队列里面去，但是是用来跨服务器通信的消息队列
      	//后面这个bean就是做这件事情的，写表然后加消息队列
        robotChatMessage.setMessageContent(messages[messages.length - 1]);
        chatMessageService.saveMessage(robotChatMessage, robot);
        return resultBuilder.toString(); //这个返回其实没什么用，是我在测试的时候打印的
    }
}
```
