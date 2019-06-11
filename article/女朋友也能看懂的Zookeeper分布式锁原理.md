#### 前言
分布式锁，在互联网行业常用，比如电商的库存扣减，秒杀活动，集群定时任务执行等需要进程互斥的场景。
常见的就是redis跟zookeeper，
zookeeper实现的分布式锁。

这篇文章主要借用Curator框架对zk分布式锁的实现思路，大家理解了以后完全可以自己手动实现一遍，但是在工作中还是建议使用成熟的开源框架，很多坑别人已经帮我们踩好了，除非万不得已，需要高度定制符合自己项目的需求的时候，才开始自行封装吧。

#### ------

既然是基于zookeeper的分布式锁，首先肯定要对这个zookeeper有一定了解，这里就不过多的进行讲解，只对其跟分布式锁有关联的特性做一个简单的介绍，更多详细的功能特性大家可以参阅官方文档。

zookeeper维护着类似文件系统的数据结构，它总共有四种类型的节点

- PERSISTENT：持久化的节点。一旦创建后，即使客户端与zk断开了连接，该节点依然存在。
- PERSISTENT_SEQUENTIAL：节点按照顺序编号。
- EPHEMERAL：临时节点。   （）当客户端与zk断开连接之后，该节点就被删除。
- EPHEMERAL_SEQUENTIAL：（分布式锁实现使用该节点类型）


“/curator_lock”锁节点用来存放相关客户端创建的临时顺序节点。  

假设两个客户端ClientA跟ClientB同时去争夺一个锁，此时ClientA先行一步获得了锁，那么它将会在我们的zk上创建一个“/curator_lock/xxxxx-0000000000”的临时顺序节点。



![](https://user-gold-cdn.xitu.io/2019/4/9/16a0179744c07b57?w=703&h=273&f=png&s=17226)

接着它会拿到“/curator_lock/”锁节点下的所有子节点，
这时候会判断它所创建的节点是否排在第一位（也就是序号最小），由于ClientA是第一个创建节点的的客户端，必然是排在第一位

```
[zk: localhost:2182(CONNECTED) 4] ls /curator_lock
[_c_f3f38067-8bff-47ef-9628-e638cfaad77e-lock-0000000000]
```

这个时候ClientB也来了，按照同样的步骤，先是在“/curator_lock/”下创建一个临时顺序节点“/curator_lock/xxxxx-0000000001”，接着也是获得该节点下的所有子节点信息，并比对自己生成的节点序号是否最小，由于此时序号最小的节点是ClientA创建的，并且还没释放掉，所以ClientB自己就拿不到锁。


![](https://user-gold-cdn.xitu.io/2019/4/9/16a0180f94aab52f?w=704&h=325&f=png&s=20632)

```
[zk: localhost:2182(CONNECTED) 4] ls /curator_lock
[_c_2a8198e4-2039-4a3c-8606-39c65790d637-lock-0000000001,
_c_f3f38067-8bff-47ef-9628-e638cfaad77e-lock-0000000000]

```

既然ClientB拿不到锁，也不会放弃，有锁就拿




客户端宕机释放锁

下面代码展示了Curator的基本使用方法，仅作为参考实例，请勿在生产环境使用的这么随意。

```java
CuratorFramework client = CuratorFrameworkFactory.newClient("127.0.0.1:2182",
                5000,10000,
                new ExponentialBackoffRetry(1000, 3));
        client.start();
        InterProcessMutex interProcessMutex = new InterProcessMutex(client, "/curator_lock");
        //加锁
        interProcessMutex.acquire();
        
        //业务逻辑
        
        //释放锁
        interProcessMutex.release();
        client.close();
    
```


#### 总结

我们在搞懂了原理之后，就可以抛弃Curator,自己动手实现一个分布式锁了，相信大家实现基本的功能都是没问题的，但是要做到生产级别，可能还是要在细节上下功夫，比如说一些异常处理，性能优化等因素。


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>

