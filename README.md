package com.cloud.kk.consumer2.service;

import com.cloud.kk.consumer2.controller.entity.Killer;
import com.cloud.kk.consumer2.controller.entity.Order;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Service;

import java.text.SimpleDateFormat;
import java.util.Arrays;
import java.util.Date;

@Service
@Slf4j
public class SkillService {
    @Autowired
    RedisTemplate<String,Object> objectRedisTemplate;
    ThreadLocal<SimpleDateFormat> threadLocal = new ThreadLocal<SimpleDateFormat>(){
        @Override
        protected SimpleDateFormat initialValue(){
            return new SimpleDateFormat("yyyy-MM-dd hh:mm:ss");
        }
    };

    final static private String blackList = "skill_blackList";
    final static private String order = "skill_order";
    final static private String storage = "skill_storage";
    final static private String goods = "skill_goods";
    final static private String users = "skill_users";
    final static private String unpaid_order = "skill_unpaid_order";

    public String  kill(Killer killer){
        //判断用户是否登录
        if(killer == null)
            return "您的账号尚未登录";
        //判断活动是否开始或结束
        SimpleDateFormat format = threadLocal.get();
        String now = format.format(new Date());
        //判断是否在黑名单
        /*if(objectRedisTemplate.opsForSet().isMember(blackList,killer.getUser_id()))
            return "您已被加入黑名单";
        //判断是否已经下单成功
        if(objectRedisTemplate.opsForHash().hasKey(order,killer.getUser_id()))
            return "您已下单成功，每人限购一单";*/
        //开始下单
        Order killerOrder = new Order();
        killerOrder.setOrder_id(100001L);
        killerOrder.setUser_id(1111L);
        killerOrder.setUser_name("zwj");
        killerOrder.setGood_id(1L);
        killerOrder.setGood_count(1L);
        killerOrder.setPrice(58.0);
        String killScript = "if redis.call('sismember',KEYS[1],ARGV[1]) == 1 "+
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
                            "         redis.call('hset',KEYS[5],ARGV[1],ARGV[3]) "  +
                            "         return redis.call('hset',KEYS[6],ARGV[1],ARGV[4]) " + //创建订单
                            "         else return 0 " + //0表示库存不足，创建订单失败
                            "         end " +
                            "    end " +
                            "end";
        DefaultRedisScript<Long> script = new DefaultRedisScript<>(killScript, Long.class);
        Long back = objectRedisTemplate.execute(script, Arrays.asList(blackList,order,storage,goods,users,unpaid_order),killer.getUser_id(),killer.getGood_id(),killer,killerOrder);
        log.info("===================back info: "+back);
        return "11";
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

    public Killer(){}

    public Killer(Long user_id,String user_name,Long good_id){
        this.user_id = user_id;
        this.user_name = user_name;
        this.good_id = good_id;
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
    private Double price;

    public Order(){};
}
    
