### 1.Redisson简介
Redis 是最流行的 NoSQL 数据库解决方案之一，而 Java 是世界上最流行（注意，我没有说“最好”）的编程语言之一。虽然两者看起来很自然地在一起“工作”，但是要知道，Redis 其实并没有对 Java 提供原生支持。

相反，作为 Java 开发人员，我们若想在程序中集成 Redis，必须使用 Redis 的第三方库。而 Redisson 就是用于在 Java 程序中操作 Redis 的库，它使得我们可以在程序中轻松地使用 Redis。Redisson 在 java.util 中常用接口的基础上，为我们提供了一系列具有分布式特性的工具类。
>Redisson底层采用的是Netty 框架。支持Redis 2.8以上版本，支持Java1.6+以上版本。

### 2.Redisson实现分布式锁的步骤
#### 2.1.引入依赖
引入重要的两个依赖，一个是spring-boot-starter-data-redis，一个是redisson：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>

<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.7.5</version>
</dependency>
```
#### 2.2.application.properties

```properties
# Redis服务器地址（默认session使用）
spring.redis.host=127.0.0.1
# Redis服务器连接密码（默认为空）
# spring.redis.password=
# Redis服务器连接端口
spring.redis.port=6379
```
#### 2.3.定义Loker
接口编程的思想还是要保持的。我们定义一个Loker接口，用于分布式锁的一些操作：

```java
package com.bruceliu.lock;

/**
 * @BelongsProject: RedissonLock
 * @BelongsPackage: com.bruceliu.lock
 * @Author: bruceliu
 * @QQ:1241488705
 * @CreateTime: 2020-05-08 10:20
 * @Description: 锁接口
 */
import java.util.concurrent.TimeUnit;

public interface Locker {

    /**
     * 获取锁，如果锁不可用，则当前线程处于休眠状态，直到获得锁为止。
     *
     * @param lockKey
     */
    void lock(String lockKey);

    /**
     * 释放锁
     *
     * @param lockKey
     */
    void unlock(String lockKey);

    /**
     * 获取锁,如果锁不可用，则当前线程处于休眠状态，直到获得锁为止。如果获取到锁后，执行结束后解锁或达到超时时间后会自动释放锁
     *
     * @param lockKey
     * @param timeout
     */
    void lock(String lockKey, int timeout);

    /**
     * 获取锁,如果锁不可用，则当前线程处于休眠状态，直到获得锁为止。如果获取到锁后，执行结束后解锁或达到超时时间后会自动释放锁
     *
     * @param lockKey
     * @param unit
     * @param timeout
     */
    void lock(String lockKey, TimeUnit unit, int timeout);

    /**
     * 尝试获取锁，获取到立即返回true,未获取到立即返回false
     *
     * @param lockKey
     * @return
     */
    boolean tryLock(String lockKey);

    /**
     * 尝试获取锁，在等待时间内获取到锁则返回true,否则返回false,如果获取到锁，则要么执行完后程序释放锁，
     * 要么在给定的超时时间leaseTime后释放锁
     *
     * @param lockKey
     * @param waitTime
     * @param leaseTime
     * @param unit
     * @return
     */
    boolean tryLock(String lockKey, long waitTime, long leaseTime, TimeUnit unit)
            throws InterruptedException;

    /**
     * 锁是否被任意一个线程锁持有
     *
     * @param lockKey
     * @return
     */
    boolean isLocked(String lockKey);
}

```
有了Locker接口，我们再添加一个基于Redisson的实现类RedissonLocker，实现Locker中的方法：

```java
package com.bruceliu.lock;

/**
 * @BelongsProject: RedissonLock
 * @BelongsPackage: com.bruceliu.lock
 * @Author: bruceliu
 * @QQ:1241488705
 * @CreateTime: 2020-05-08 10:20
 * @Description: 基于Redisson的分布式锁
 */
import java.util.concurrent.TimeUnit;

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;

public class RedissonLocker implements Locker {

    private RedissonClient redissonClient;

    public RedissonLocker(RedissonClient redissonClient) {
        super();
        this.redissonClient = redissonClient;
    }

    @Override
    public void lock(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        lock.lock();
    }

    @Override
    public void unlock(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        lock.unlock();
    }

    @Override
    public void lock(String lockKey, int leaseTime) {
        RLock lock = redissonClient.getLock(lockKey);
        lock.lock(leaseTime, TimeUnit.SECONDS);
    }

    @Override
    public void lock(String lockKey, TimeUnit unit, int timeout) {
        RLock lock = redissonClient.getLock(lockKey);
        lock.lock(timeout, unit);
    }

    public void setRedissonClient(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    @Override
    public boolean tryLock(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        return lock.tryLock();
    }

    @Override
    public boolean tryLock(String lockKey, long waitTime, long leaseTime,
                           TimeUnit unit) throws InterruptedException{
        RLock lock = redissonClient.getLock(lockKey);
        return lock.tryLock(waitTime, leaseTime, unit);
    }

    @Override
    public boolean isLocked(String lockKey) {
        RLock lock = redissonClient.getLock(lockKey);
        return lock.isLocked();
    }

}

```
#### 2.4.定义工具类
有了Locker和实现类RedissonLocker，我们总不能一直去创建RedissonLocker对象或者不断的在每个要使用到分布式锁的地方都注入RedissonLocker的对象，所以我们定义一个工具类LockUtil，到时候想哪里使用就直接使用工具类的静态方法就行了：
```java
package com.bruceliu.utils;

/**
 * @BelongsProject: RedissonLock
 * @BelongsPackage: com.bruceliu.utils
 * @Author: bruceliu
 * @QQ:1241488705
 * @CreateTime: 2020-05-08 10:21
 * @Description: TODO
 */
import com.bruceliu.lock.Locker;

import java.util.concurrent.TimeUnit;

/**
 * redis分布式锁工具类
 *
 */
public final class LockUtil {

    private static Locker locker;

    /**
     * 设置工具类使用的locker
     * @param locker
     */
    public static void setLocker(Locker locker) {
        LockUtil.locker = locker;
    }

    /**
     * 获取锁
     * @param lockKey
     */
    public static void lock(String lockKey) {
        locker.lock(lockKey);
    }

    /**
     * 释放锁
     * @param lockKey
     */
    public static void unlock(String lockKey) {
        locker.unlock(lockKey);
    }

    /**
     * 获取锁，超时释放
     * @param lockKey
     * @param timeout
     */
    public static void lock(String lockKey, int timeout) {
        locker.lock(lockKey, timeout);
    }

    /**
     * 获取锁，超时释放，指定时间单位
     * @param lockKey
     * @param unit
     * @param timeout
     */
    public static void lock(String lockKey, TimeUnit unit, int timeout) {
        locker.lock(lockKey, unit, timeout);
    }

    /**
     * 尝试获取锁，获取到立即返回true,获取失败立即返回false
     * @param lockKey
     * @return
     */
    public static boolean tryLock(String lockKey) {
        return locker.tryLock(lockKey);
    }

    /**
     * 尝试获取锁，在给定的waitTime时间内尝试，获取到返回true,获取失败返回false,获取到后再给定的leaseTime时间超时释放
     * @param lockKey
     * @param waitTime
     * @param leaseTime
     * @param unit
     * @return
     * @throws InterruptedException
     */
    public static boolean tryLock(String lockKey, long waitTime, long leaseTime,
                                  TimeUnit unit) throws InterruptedException {
        return locker.tryLock(lockKey, waitTime, leaseTime, unit);
    }

    /**
     * 锁释放被任意一个线程持有
     * @param lockKey
     * @return
     */
    public static boolean isLocked(String lockKey) {
        return locker.isLocked(lockKey);
    }
}
```
#### 2.5.Redisson的配置类
现在我们开始配置吧，创建一个redisson的配置类RedissonConfig，内容如下：

```java
package com.bruceliu.config;

import java.io.IOException;

import com.bruceliu.lock.RedissonLocker;
import com.bruceliu.utils.LockUtil;
import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.config.Config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @BelongsProject: RedissonLock
 * @BelongsPackage: com.bruceliu.config
 * @Author: bruceliu
 * @QQ:1241488705
 * @CreateTime: 2020-05-08 10:22
 * @Description: TODO
 */
@Configuration
public class RedissonConfig {

    @Value("${spring.redis.host}")
    private String host;

    @Value("${spring.redis.port}")
    private String port;

    //@Value("${spring.redis.password}")
    //private String password;

    /**
     * RedissonClient,单机模式
     * @return
     * @throws IOException
     */
    @Bean(destroyMethod = "shutdown")
    public RedissonClient redisson() throws IOException {
        Config config = new Config();
        //config.useSingleServer().setAddress("redis://" + host + ":" + port).setPassword(password);
        config.useSingleServer().setAddress("redis://" + host + ":" + port);
        return Redisson.create(config);
    }

    @Bean
    public RedissonLocker redissonLocker(RedissonClient redissonClient){
        RedissonLocker locker = new RedissonLocker(redissonClient);
        //设置LockUtil的锁处理对象
        LockUtil.setLocker(locker);
        return locker;
    }
}

```
#### 2.6.Redisson分布式锁业务类
```java
package com.bruceliu.service;

import com.bruceliu.utils.LockUtil;

import java.util.concurrent.TimeUnit;

/**
 * @BelongsProject: RedissonLock
 * @BelongsPackage: com.bruceliu.service
 * @Author: bruceliu
 * @QQ:1241488705
 * @CreateTime: 2020-05-08 10:26
 * @Description: TODO
 */
public class SkillService {

    int n = 500;

    public void seckill() {
        //加锁
        LockUtil.lock("resource", TimeUnit.SECONDS,5000);
        try {
            System.out.println(Thread.currentThread().getName() + "获得了锁");
            Thread.sleep(3000);
            System.out.println(--n);
        } catch (Exception e) {
            //异常处理
        }finally{
            //释放锁
            LockUtil.unlock("resource");
            System.out.println(Thread.currentThread().getName() + "释放了锁");
        }
    }
}

```
#### 2.7.Redisson分布式锁测试

```java
package com.bruceliu.test;

import com.bruceliu.App;
import com.bruceliu.service.SkillService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * @BelongsProject: RedissonLock
 * @BelongsPackage: com.bruceliu.test
 * @Author: bruceliu
 * @QQ:1241488705
 * @CreateTime: 2020-05-08 10:29
 * @Description: TODO
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = App.class)
public class TestLock {

    @Test
    public void testLock() throws Exception{
        SkillService service = new SkillService();
        for (int i = 0; i < 10; i++) {
            ThreadA threadA = new ThreadA(service);
            threadA.setName("ThreadNameA->"+i);
            threadA.start();
        }
        Thread.sleep(50000);
    }

}

class ThreadA extends Thread {

    private SkillService skillService;

    public ThreadA(SkillService skillService) {
        this.skillService = skillService;
    }

    @Override
    public void run() {
        skillService.seckill();
    }
}


```
**运行结果:**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200508115126598.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1MjI2Nzk=,size_16,color_FFFFFF,t_70)
注释文中加锁代码：

```java
public void seckill() {
    //加锁
    //LockUtil.lock("resource", TimeUnit.SECONDS,5000);
    try {
        System.out.println(Thread.currentThread().getName() + "获得了锁");
        Thread.sleep(3000);
        System.out.println(--n);
    } catch (Exception e) {
        //异常处理
    }finally{
        //释放锁
        //LockUtil.unlock("resource");
        System.out.println(Thread.currentThread().getName() + "释放了锁");
    }
}
```
运行结果：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200508115435254.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE1MjI2Nzk=,size_16,color_FFFFFF,t_70)
可以看到存在并发的问题！



