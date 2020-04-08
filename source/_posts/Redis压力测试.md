---
title: Redis压力测试
date: 2019-09-05 11:40:22
tags:
- Redis
categories:
- Redis
---
# Redis压力测试

> 分布式情况下对Redis高并发测试

一个简单的Redis高并发测试，从Redis读取一个key值，并让值加一更新key值。<!--more-->
## 准备工作

### 测试代码

> RedissonController

```java
@Controller
public class RedissonController {
    @Autowired
    RedisUtil redisUtil;
    @Autowired
    RedissonClient redissonClient;

    @RequestMapping("testRedisson")
    @ResponseBody
    public String testRedisson(){
        Jedis jedis = redisUtil.getJedis();
        String v = jedis.get("k");
        if(StringUtils.isBlank(v)){
            v = "1";
        }
        System.out.println("--->"+v);
        jedis.set("k", (Integer.parseInt(v) + 1) + "");
        jedis.close();
        return "success";
    }
}
```

上面代码没有使用Redisson锁，先测试没有锁的结果。

---

### 多例模式

设置SpringBoot启动项为`Allow parallel run`，多例模式启动。

{% asset_img conf.png %}

### 整合Nginx负载均衡

nginx.conf

```
    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;
    upstream redisTest{
            	server	127.0.0.1:8081 weight=3;
	server	127.0.0.1:8082 weight=3;
	server	127.0.0.1:8083 weight=3;
    }
    server {
        listen       8084;
        server_name  localhost;

        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            #root   html;
            proxy_pass http://redisTest;
            index  index.html index.htm;
        }
```

让Nginx代理`8081,8082,8083`端口。

## 开始测试

测试软件使用的Apache测试工具，使用压力命令`ab -c xxx -n xxx http://xxx`即可模拟并发请求。ab是压力命令，c是并发数，n是总请求数。

开启Apache的httpd服务后，在bin目录下输入`ab -c 200 -n 1000 http://localhost:8084/testRedisson`,模拟200并发，1000个请求。

> 测试开始前Redis中k的值
{% asset_img redis.png %}
> 测试情况
{% asset_img result1.png %}
---
{% asset_img result2.png %}
---
{% asset_img result3.png %}
> 测试完成后Redis中k的值
{% asset_img result4.png %}

  从结果可以看出，k的值并没有到1000。并且还有一些重复的值出现。
---

使用Redisson锁测试

> RedissonController

```java
@Controller
public class RedissonController {
    @Autowired
    RedisUtil redisUtil;
    @Autowired
    RedissonClient redissonClient;

    @RequestMapping("testRedisson")
    @ResponseBody
    public String testRedisson(){
        Jedis jedis = redisUtil.getJedis();
        RLock lock = redissonClient.getLock("lock");
        lock.lock();
        try {
            String v = jedis.get("k");
            if (StringUtils.isBlank(v)) {
                v = "1";
            }
            System.out.println("->"+v);
            jedis.set("k", (Integer.parseInt(v) + 1) + "");

        }finally {
            jedis.close();
            lock.unlock();
        }
        return "success";
    }
}
```

重置Redis中k的值后，再次测试。
> 测试结果
{% asset_img result5.png %}

k的值为1000，控制台打印的值没有重复。