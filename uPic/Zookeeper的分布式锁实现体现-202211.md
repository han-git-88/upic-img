# Zookeeper的分布式锁实现用户体现处理



## Zookeeper是什么？

本身是一款开源分布式服务协调中间件。是雅虎团队开发出来

## Zookeeper的应用场景 

- 统一配置管理 : 将每个子系统都需要配置的文件统一配置到zookeeper的znode节点上。类似于：nacos config server.
  - kafka集群
  - 配置中心
- 统一命名服务
  - 通过存放在Znode的资源进行统一命名，各个子系统可以通过名字获取节点上的相应的相信。典型就是：dubbo
- 分布式锁 ： 
  - 通过创建共享资源相关的：“顺序临时节点” 与 “动态watcher监听机制”。从而控制多线程对共享资源的并发访问。
- 集群状态监控
  - 通过动态删除，添加，从而保证集群下的相关节点数据是否和其他节点一致。同步节点信息。



## Zookeeper的节点类型

（1）持久化目录节点（PERSISTENT）

客户端与zookeeper断开连接后，该节点依旧存在

（2）持久化顺序编号目录节点（PERSISTENT_SEQUENTIAL）

客户端与zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号

（3）临时目录节点（EPHEMERAL）

客户端与zookeeper断开连接后，该节点被删除

（4）临时顺序编号目录节点（EPHEMERAL_SEQUENTIAL）

客户端与zookeeper断开连接后，该节点被删除，只是Zookeeper给该节点名称进行顺序编
## Zookeeper的分布式锁的原理

- 在zookeeper创建一个临时顺序节点来完成的。 采用watcher监听机制不断监听临时节点的增减
- 获取锁成功
- 当前线程获取锁，执行共享资源
- 操作完成
- 对当前线程释放共享资源的锁，即删除对应最小序号的临时节点，让其他的节点序号变成更小，从而对应的线程可用获取到锁。

> 为什么要采用临时顺序节点呢？

你就这样思考，我们并发请是很大的，肯定会产生很多的线程。比如线程A ，B ，C…N 。那么这么多线程，我们如何把他存下来，而且不让折现丢失。所以就必须维持一个序号，来一个线程我就产生一个序号，来一百个就产生一百，一次类推。





## Zookeeper的官网

- 官网：https://zookeeper.apache.org/doc/r3.8.0/zookeeperOver.html

- 封装zk的框架 ： https://curator.apache.org/getting-started.html
- 下载zookeeper: https://www.apache.org/dyn/closer.lua/zookeeper/zookeeper-3.7.1/apache-zookeeper-3.7.1-bin.tar.gz



## Zookeeper实现分布式锁如下

1: 安装zookeeper服务

2：依赖

```xml
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.6.3</version>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>4.2.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>4.2.0</version>
</dependency>
```

如果出现日志冲突，记得怕slf4j移除

3: 配置

```yaml
## 自定义 zookeeper配置
zk:
  host: 127.0.0.1:2181
  namespace: useramount_distributelock
```

4: 初始化client对象

```java
package com.xxb.user.config;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.RetryNTimes;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.env.Environment;

/**
 * @author 飞哥
 * @date 2022-09-25$ 23:03$
 */
@Configuration
public class CuatorFrameworkConfiguration {

    @Autowired
    private Environment environment;


    @Bean
    public CuratorFramework curatorFramework(){
        // 创建一个CuratorFramework第项
        CuratorFramework curatorFramework = CuratorFrameworkFactory
                .builder().connectString(environment.getProperty("zk.host"))
                .namespace(environment.getProperty("zk.namespace"))
                .retryPolicy(new RetryNTimes(5,1000)).build();

        // 启动服务
        curatorFramework.start();
        // 返回服务
        return curatorFramework;
    }

}

```



5: 使用zookeeper分布式锁解决用户提现问题

```java
package com.xxb.user.service.useramount;

import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import com.xxb.common.result.R;
import com.xxb.user.dto.UserAmountDto;
import com.xxb.user.mapper.UserAmountRecordMapper;
import com.xxb.user.mapper.UserMapper;
import com.xxb.user.pojo.User;
import com.xxb.user.pojo.UserAmountRecord;
import lombok.extern.slf4j.Slf4j;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.checkerframework.checker.units.qual.A;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.ValueOperations;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import javax.annotation.Resource;
import java.math.BigDecimal;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

/**
 * @author 飞哥
 * @date 2022-09-22$ 23:21$
 */
@Service
@Slf4j
public class ZookeperUserAmountRecordServiceImpl extends ServiceImpl<UserAmountRecordMapper, UserAmountRecord>
        implements IUserAmountRecordService {


    @Resource
    private UserMapper userMapper;
    @Autowired
    private CuratorFramework client;


    /**
     * 提现方法---无锁
     *
     * @param userAmountDto
     * @return
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    // @GlobalTransactional 分布式事务会影响分布式锁
    public R takeZookeeperMoney(UserAmountDto userAmountDto) {
        // 1 : 查询用户账户的实体记录
        // 提现的用户
        Long userId = userAmountDto.getUserId();
        // 提现的金额
        Double money = userAmountDto.getMoney();
        // 所有并发线程产生的序号存放的目录位置
        String lockPath = "/useramount/zklock/" + userId + "-lock";
        //A ---并发的线程---/useramount/zklock/1-clock 0001 0002
        //B ---并发的线程---/useramount/zklock/2-clock 0001 0002 0003
        // 抢购和秒杀
        //String lockPath = "/useramount/zklock/1";
        //A， B ---并发的线程---/useramount/zklock/1 0001 0002 0003 0004 0005
        // 抢购和秒杀
        //String lockPath = "/useramount/zklock/";
        //A， B ---并发的线程---/useramount/zklock 0001 0002 0003 0004 0005
        InterProcessMutex lock = new InterProcessMutex(client, lockPath);        // 调用setnx进行获取锁,如果获取返回true, 否则返回false.
        try {
            // 给锁设置过期时间 -- 防止出现死锁
            Boolean isLock = lock.acquire(20L, TimeUnit.SECONDS);
            if (isLock) {
                User user = userMapper.selectById(userId);
                // 2: 判断用户提现的余额是否充足，如果充足就提现，
                if (user != null && user.getAmount().doubleValue() - money > 0) {
                    // 3 : 如果余额充足，就开始扣减余额
                    User updateUser = new User();
                    updateUser.setId(userId);
                    updateUser.setAmount(new BigDecimal(user.getAmount().doubleValue() - money));
                    // 这行代码只是为了同步一些余额，避免再次查询数据库
                    user.setAmount(updateUser.getAmount());
                    // 更新用户余额
                    userMapper.updateById(updateUser);
                    // 同时插入提现记录信息保存到用户提现表中
                    UserAmountRecord userAmountRecord = new UserAmountRecord();
                    userAmountRecord.setUserId(userId);
                    userAmountRecord.setUsername(user.getUsername());
                    userAmountRecord.setAvatar(user.getAvatar());
                    userAmountRecord.setMoney(new BigDecimal(money));
                    // 保存用户提现记录
                    this.saveOrUpdate(userAmountRecord);
                    // 输出日志
                    log.info("当前提现的金额是：{}，用户的余额是：{}", money, user.getAmount());
                } else {
                    // 3 : 否则就返回账户余额不足
                    return R.error("账户不存在，或者余额不足!!!");
                }
                return R.ok();
            }else{
                log.info("没有拿到，离开了.......");
                return R.error("别着急，请慢一点....");
            }
        } catch (Exception ex) {
            return R.error("执行失败....");
        } finally {
            try {
                // 无论发送任何情况。都需要将锁释放掉。
                lock.release();
            }catch (Exception ex){
                ex.printStackTrace();
            }
        }
    }

}
```





## 布置作业

1： 了解redssion是如何使用布隆过来器的，可以缓存穿透的问题

2： 了解redssion是分布锁的实现原理和机制 (lua + 看门狗)

3 :   实战开放 - 用户注册

- 重新搭建一个纯粹的工程去联系。

- 没有任何的锁版本。直接使用 jemeter并发，是否会重复注册用户

- 先用本地锁解决，看看是否能解决重复提交，然后在集群暴露本地锁不能解决重复提交问题

- 使用分布式锁解决重复提交问题

  - redis 

    注册是没有用户id，第一种：使用注册的用户名或者手机作为key  (一定是用那个唯一的值取拼接key，否则就不唯一锁，) 第二种方式：在进入注册页面的时候，生成一个uuid，然后在提交的时候把这个uuid携带到服务端做key

  - zk

    注册是没有用户id，第一种：使用注册的用户名或者手机作为key  (一定是用那个唯一的值取拼接key，否则就不唯一锁，) 第二种方式：在进入注册页面的时候，生成一个uuid，然后在提交的时候把这个uuid携带到服务端做key

  - redssion

    注册是没有用户id，第一种：使用注册的用户名或者手机作为key  (一定是用那个唯一的值取拼接key，否则就不唯一锁，) 第二种方式：在进入注册页面的时候，生成一个uuid，然后在提交的时候把这个uuid携带到服务端做key



