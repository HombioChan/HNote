## 前置知识
### 临界区
- 一次最多只能有一个进程进入临界区
- 不能让一个进程在临界区内无限循环
- 等待进入临界区的程序，不能无限等待
- 可以用锁机制实现户互斥

### 互斥锁
- 二元信号量
- 一个进程占有，其余必须等待（阻塞/自旋）
- 获取与释放为同个进程
- 获取和释放锁具有原子性
- 可重入


## 场景
100瓶营养快线特卖活动

### 业务逻辑
1. 获取锁
2. 判断是否有余量，有生成订单，否则返回失败。
3. 释放锁

### 分析
- 显而易见，业务逻辑2即为临届区
- 锁机制用来保证一次最多只能有一个进程进入临界区
- 为了避免一个进程在临界区内无限循环，除了要保证正常情况下临界区逻辑在有限的时间内执行完毕，对于异常情况（如发生异常，服务shutdown），需要有一种机制让进程退出临界区，即释放锁。
- 这样也就避免了等待进入临界区的程序无限等待

### 架构
为提高并发处理能力，后台服务采用集群部署，库存存放在redis，使用网关作负载均衡
- 因此必须采用分布式锁


## redis分布式锁设计
### 前提：redis高可用
1. 获取锁： setnx key value
2. 释放锁： del key
3. 可重入：进程局部变量（ThreadLocal）
4. 获取与释放为同个进程：进程局部变量（ThreadLocal）

#### 如何避免一个进程在临界区内无限循环, 即如何避免死锁/饥饿
- 某个占有锁的后台服务shutdown后，其他后台服务的进程陷入饥饿姿态： 设置key时指定过期时间(set key value [EX seconds|PX milliseconds|KEEPTTL] [NX|XX])， 保证其他服务进程可以获取到锁。同时，设置过期时间后要保证： 过期时间 > 业务逻辑处理时间， 避免锁被提前删除，导致多个进程进入临界区，可以开启单独线程续期。
- 正常情况下：有限时间内退出临界区，由业务逻辑的developer保证


## 实现代码
[来源](https://xie.infoq.cn/article/bf6ae60071d925ea42f76ce2)
```
public class RedisLockImpl implements RedisLock{
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    
    private ThreadLocal<String> threadLocal = new ThreadLocal<String>();
    
    private ThreadLocal<Integer> threadLocalInteger = new ThreadLocal<Integer>();
    
    @Override
    public boolean tryLock(String key, long timeout, TimeUnit unit){
        Boolean isLocked = false;
        if(threadLocal.get() == null){
            String uuid = UUID.randomUUID().toString();
        	threadLocal.set(uuid);
            isLocked = stringRedisTemplate.opsForValue().setIfAbsent(key, uuid, timeout, unit);
            //启动新线程来执行定时任务，更新锁过期时间
           new Thread(new UpdateLockTimeoutTask(uuid, stringRedisTemplate, key)).start();
        }else{
            isLocked = true;   
        }
        //加锁成功后将计数器加1
        if(isLocked){
            Integer count = threadLocalInteger.get() == null ? 0 : threadLocalInteger.get();
            threadLocalInteger.set(count++);
        }
        return isLocked;
    }
    @Override
    public void releaseLock(String key){
        //当前进程中绑定的uuid与Redis中的uuid相同时，再执行删除锁的操作
        if(threadLocal.get().equals(stringRedisTemplate.opsForValue().get(key))){
            Integer count = threadLocalInteger.get();
            //计数器减为0时释放锁
            if(count == null || --count <= 0){
                threadLocal.set(null);
             	stringRedisTemplate.delete(key);      
            }
        }
    }
}
```

### 结论
能不用分布式锁就不用，复杂且性能不高。比如本业务场景完全可以不用分布式锁
1. 通过广播方式通知集群服务均摊库存
2. 用户请求通过网关层作负载均衡


### 参考
`https://xie.infoq.cn/article/bf6ae60071d925ea42f76ce29`  
`https://mp.weixin.qq.com/s?__biz=MzAxODcyNjEzNQ==&mid=2247501091&idx=3&sn=8da5fed473700743aecb22d93446a4ce&chksm=9bd368bbaca4e1ad3c497eb10e437923ab9a14a208a76160e3098f10c758c70ee92d0e84423d&scene=27#wechat_redirect`