# RabbitMQ实现延迟队列

## 简介

延迟队列存储的对象是对应的延迟消息，所谓“延迟消息”是指当消息被发送以后，并不想让消费者立刻拿到消息，而是等待特定时间后，消费者才能拿到这个消息进行消费。

## 场景

1、用户下单后，几分钟内未付款提醒付款
2、提醒付款后几分钟内还未支付，就自动取消订单
3、用户支付完成后，几秒钟后下发音频
... ...

## 实现

### 方法一(使用rabbitmq官方提供的插件)

[安装插件参考文章](https://blog.csdn.net/m0_67402774/article/details/124169540)
[官方插件GitHub下载地址](https://github.com/rabbitmq/rabbitmq-delayed-message-exchange/releases)

#### 1）上传插件到服务器

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/3a69a96d5b7847eec90f05be9ac9df1d.png)

#### 2）拷贝到docker容器中并启用、重启

```powershell
# 拷贝插件
docker cp /root/docker-rabbitmq/rabbitmq_delayed_message_exchange-3.12.0.ez rabbitmq:/opt/rabbitmq/plugins/

# 进入容器内
docker exec -it rabbitmq bash

# 进入目录
cd plugins

#查看插件
ls |grep delay

#启用插件
rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 查看插件列表
# rabbitmq-plugins list 

# 开启插件支持 
# rabbitmq-plugins enable rabbitmq_delayed_message_exchange

# 退出容器
ctrl + p +q
#or
exit

# 重启容器
docker restart rabbitmq

```

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/a9d93764416a5cf4d610df416633fa9e.png)

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/bcc491b03a88453f072849893a8da0cf.png)

#### 3）代码实现

```java
#消费者 ===================================================》

package com.aurora.consumer;

import lombok.extern.slf4j.Slf4j;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Component;

import java.util.Date;

/**
 * 死信消费者
 *
 * @author Yelf
 * @date 2024/01/19
 */
@Slf4j
@Component
public class DeadLetterConsumer {

    public static final String DELAYED_QUEUE_NAME = "delayed.queue";

    /**
     * 接收延迟队列
     *
     * @param data 数据
     */
    @RabbitListener(queues = DELAYED_QUEUE_NAME)
    public void receiveDelayedQueue(byte[] data){
        String msg = new String(data);
        log.info("当前时间：{}，收到延时队列的消息：{}", new Date(), msg);
    }
}


#mq配置文件 ===================================================》

package com.aurora.config;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.CustomExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

/**
 * 延迟队列配置
 *
 * @author Yelf
 * @date 2024/01/19
 */
@Configuration
public class DelayedQueueConfig {

    public static final String DELAYED_QUEUE_NAME = "delayed.queue";
    public static final String DELAYED_EXCHANGE_NAME = "delayed.exchange";
    public static final String DELAYED_ROUTING_KEY = "delayed.routingKey";

    @Bean("delayedQueue")
    public Queue delayedQueue(){
        return new Queue(DELAYED_QUEUE_NAME);
    }

    /**
     * 自定义交换机 定义一个延迟交换机
     *  不需要死信交换机和死信队列，支持消息延迟投递，消息投递之后没有到达投递时间，是不会投递给队列
     *  而是存储在一个分布式表，当投递时间到达，才会投递到目标队列
     * @return
     */
    @Bean("delayedExchange")
    public CustomExchange delayedExchange(){
        Map<String, Object> args = new HashMap<>(1);
        // 自定义交换机的类型
        args.put("x-delayed-type", "direct");
        return new CustomExchange(DELAYED_EXCHANGE_NAME, "x-delayed-message", true, false, args);
    }

    @Bean
    public Binding bindingDelayedQueue(@Qualifier("delayedQueue") Queue delayedQueue,
                                       @Qualifier("delayedExchange") CustomExchange delayedExchange){
        return BindingBuilder.bind(delayedQueue).to(delayedExchange).with(DELAYED_ROUTING_KEY).noargs();
    }
}

# 生产者 ===================================================》

/**
     * 生产者测试
     *
     * @param message   消息
     * @param delayTime 延迟时间
     * @return {@link ResultVO}<{@link String}>
     */
    @GetMapping("/test/test01")
    public ResultVO<String> test01(@RequestParam("message") String message,@RequestParam("delayTime") Integer delayTime) {
        rabbitTemplate.convertAndSend(DELAYED_EXCHANGE_NAME, DELAYED_ROUTING_KEY, message.getBytes(StandardCharsets.UTF_8), messagePostProcessor ->{
            messagePostProcessor.getMessageProperties().setDelay(delayTime);
            return messagePostProcessor;
        });
        log.info("当前时间：{}，发送一条延迟{}毫秒的信息给队列delay.queue:{}", new Date(), delayTime, message);
        return ResultVO.ok();
    }


```

#### 4）结果

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/ed3b5c551aaddbc830a1cc9ef30bf1b3.png)

### 方法二 使用死信队列（无插件）

实际就是将队列或者消息设置过期时间，当过期后，消息就会进入死信队列，然后消费者消费的就是死信队列
