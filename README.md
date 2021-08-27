local blacklist = KEYS[1]; --黑名单
local ip = ARGV[1];--请求ip
local time = ARGV[2]; --过期时间
local max_count = tonumber(ARGV[3]); --最大请求数
local count = redis.call('incrby',ip,1); --请求计数器
if count == 1 then
    redis.call('expire',ip,time) --首次访问设置过期时间
elseif count > max_count then
    redis.call('sadd',blacklist,ip) --如果大于最大请求数加入到黑名单
end    




package com.cloud.kk.consumer2.rabbitmq;

import org.springframework.amqp.core.Binding;
import org.springframework.amqp.core.BindingBuilder;
import org.springframework.amqp.core.DirectExchange;
import org.springframework.amqp.core.Queue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.HashMap;
import java.util.Map;

@Configuration
public class DelayQueueConfig {

    @Bean
    public Queue delayQueue(){
        Map<String,Object> map = new HashMap<>();
        map.put("x-message-ttl",60000);//设置消息过期时间
        map.put("x-dead-letter-exchange","dead.exchange");//绑定死信交换机
        map.put("x-dead-letter-routing-key","dead.order");//绑定路由key
        return new Queue("delay.queue",true,false,false,map);
    }

    @Bean
    public DirectExchange delayExchange(){
        return new DirectExchange("delay.exchange");
    }

    @Bean
    public Binding delayBinding(){
        return BindingBuilder.bind(delayQueue()).to(delayExchange()).with("delay.order");
    }

    @Bean
    public Queue deadQueue(){
        return new Queue("dead.queue",true,false,false);
    }

    @Bean
    public DirectExchange deadExchange(){
        return new DirectExchange("dead.exchange");
    }

    @Bean
    public Binding deadBinding(){
        return BindingBuilder.bind(deadQueue()).to(deadExchange()).with("dead.order");
    }
}
