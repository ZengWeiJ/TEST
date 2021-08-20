package com.cloud.kk.consumer2.service;

import com.cloud.kk.consumer2.controller.entity.Killer;
import com.cloud.kk.consumer2.controller.entity.Order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.core.io.ClassPathResource;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.scripting.support.ResourceScriptSource;
import org.springframework.stereotype.Service;

import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Date;
import java.util.Set;

@Service
@Slf4j
public class SkillService {
    @Autowired
    RedisTemplate<String,Object> objectRedisTemplate;
    @Autowired
    StringRedisTemplate stringRedisTemplate;

    ThreadLocal<SimpleDateFormat> threadLocal = new ThreadLocal<SimpleDateFormat>(){
        @Override
        protected SimpleDateFormat initialValue(){
            return new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
        }
    };

    final static private String status = "skill_status";
    final static private String blackList = "skill_blackList";
    final static private String order = "skill_order";
    final static private String storage = "skill_storage";
    final static private String goods = "skill_goods";
    final static private String order_expired_time = "skill_order_expiredTime";
    final static private String unpaid_order = "skill_unpaid_order";
    static private Long time = 30*60*1000L;

    public String  kill(Killer killer){
        //判断用户是否登录
        if(killer == null)
            return "您的账号尚未登录";
        //判断活动是否开始或结束
        SimpleDateFormat format = threadLocal.get();
        if(stringRedisTemplate.opsForValue().get(status).equals("100"))
            return "秒杀未开始";
        if (stringRedisTemplate.opsForValue().get(status).equals("300"))
            return "秒杀已结束";
        //判断是否在黑名单
        if(objectRedisTemplate.opsForSet().isMember(blackList,killer.getUser_id()))
            return "您已被加入黑名单";
        //判断是否已经下单成功
        if(objectRedisTemplate.opsForHash().hasKey(order,killer.getUser_id().toString()))
            return "您已下单成功，每人限购一单";
        //开始下单
        Order killerOrder = new Order();
        killerOrder.setOrder_id(System.currentTimeMillis());
        killerOrder.setUser_id(killer.getUser_id());
        killerOrder.setUser_name(killer.getUser_name());
        killerOrder.setGood_id(killer.getGood_id());
        killerOrder.setGood_count(killer.getGood_count());
        killerOrder.setGood_price(killer.getGood_price());
        Date now = new Date(); //创建订单时间
        Long expired_time = now.getTime() + time; //订单过期时间
        killerOrder.setCreate_time(format.format(now));
       /* String killScript = "if redis.call('sismember',KEYS[1],ARGV[1]) == 1 "+
                            "then return -1 " + //1表示存在黑名单
                            "else " +
                            "    if redis.call('hexists',KEYS[2],ARGV[4]) == 1 " +
                            "    then return 2 " + //2表示订单已存在
                            "    elseif redis.call('hexists',KEYS[6],ARGV[4]) == 1 " +
                            "    then return 3 " + //3表示存在未支付订单
                            "    else " +
                            "         if redis.call('hget',KEYS[3],ARGV[2]) > '0' " + //判断库存是否大于0
                            "         then redis.call('hincrby',KEYS[3],ARGV[2],-1) " + //扣减库存
                            "             if redis.call('hget',KEYS[3],ARGV[2]) == '0' " + //判断扣减库存后商品数是否为0，如果为0删除商品信息
                            "             then redis.call('hdel',KEYS[4],ARGV[2]) " +
                            "             end " +
                            "         redis.call('zadd',KEYS[5],ARGV[3],ARGV[1]) "  +
                            "         return redis.call('hset',KEYS[6],ARGV[1],ARGV[4]) " + //创建订单
                            "         else return 0 " + //0表示库存不足，创建订单失败
                            "         end " +
                            "    end " +
                            "end";
        DefaultRedisScript<Long> script = new DefaultRedisScript<>(killScript, Long.class);*/
        DefaultRedisScript<Long> script = new DefaultRedisScript<>();
        script.setScriptSource(new ResourceScriptSource(new ClassPathResource("/killScript.lua")));
        script.setResultType(Long.class);
        Long back = objectRedisTemplate.execute(script, Arrays.asList(blackList,order,storage,goods,order_expired_time,unpaid_order),
                killer.getUser_id().toString(),killer.getGood_id(),expired_time,killerOrder,killer.getGood_count());
        log.info("===================back info: "+back);
        return "11";
    }

    public void delExpiredOrder(){
        Long now = new Date().getTime();
        Set<String> set = stringRedisTemplate.opsForZSet().rangeByScore(order_expired_time,0,now);
        for(String user_id : set){
            Order killOrder = (Order)objectRedisTemplate.opsForHash().get(unpaid_order,user_id);
            if(killOrder == null)
                return;
            Long good_id = killOrder.getGood_id();
            Long count = killOrder.getGood_count();
            DefaultRedisScript<Long> script = new DefaultRedisScript<>();
            script.setScriptSource(new ResourceScriptSource(new ClassPathResource("/expiredScript.lua")));
            script.setResultType(Long.class);
            Long back = objectRedisTemplate.execute(script,Arrays.asList(unpaid_order,storage,order_expired_time),user_id,good_id,count);
            log.info("deal expired order==========="+back);
        }
    }
}




package com.cloud.kk.consumer2.controller.entity;

import lombok.Data;

import java.io.Serializable;

@Data
public class Killer implements Serializable {

    private Long user_id;
    private String user_name;
    private Long good_id;
    private Long good_count;
    private Double good_price;

    public Killer(){}

    public Killer(Long user_id,String user_name,Long good_id,Long good_count,Double good_price){
        this.user_id = user_id;
        this.user_name = user_name;
        this.good_id = good_id;
        this.good_count = good_count;
        this.good_price = good_price;
    }
}



package com.cloud.kk.consumer2.controller.entity;

import lombok.Data;

@Data
public class Order {
    private Long order_id;
    private Long user_id;
    private String user_name;
    private Long good_id;
    private Long good_count;
    private Double good_price;
    private String create_time;

    public Order(){};
}



if redis.call('hexists',KEYS[1],ARGV[1]) == 0 then //不存在过期订单表示已经删除
     return 0
else
     redis.call('hdel',KEYS[1],ARGV[1]) //删除过期订单
     redis.call('hincrby',KEYS[2],ARGV[2],ARGV[3]) //回退库存
     redis.call('zrem',KEYS[3],ARGV[1]) //删除过期记录
     return 1
end



local storage = tonumber(redis.call('hget',KEYS[3],ARGV[2]));
local count = ARGV[5];
if redis.call('sismember',KEYS[1],ARGV[1]) == 1 then
    return 1
else
    if redis.call('hexists',KEYS[2],ARGV[1]) == 1 then
        return 2
    elseif redis.call('hexists',KEYS[6],ARGV[1]) == 1 then
        return 3
    else
         if storage - count  >= 0 then
            redis.call('hincrby',KEYS[3],ARGV[2],-count)
            if storage - count == 0 then
                redis.call('hdel',KEYS[4],ARGV[2])
            end
            redis.call('zadd',KEYS[5],ARGV[3],ARGV[1])
            return redis.call('hset',KEYS[6],ARGV[1],ARGV[4])
         else
            return 0
         end
    end
end    
