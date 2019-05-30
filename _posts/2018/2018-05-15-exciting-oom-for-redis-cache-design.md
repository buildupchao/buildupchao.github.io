---
title: Redis缓存设计之令人兴奋的OOM
date: 2018-05-15 12:33:36
tags: ['Redis', '缓存设计', 'OutOfMemoryError', 'OOM']
categories: ['Redis']
---

先来个自我介绍，博主是Synnex WEBOP OE的Sales部门的一枚Java程序员。<br/>
再来介绍下具体情况，Sales部门有3个母系统，它们的年龄和博主相差不多。这半年来，一直在做通过微服务技术重构以及改写旧系统，而博主主要负责的就是大数据量订单查询以及审批，异常状态矫正的所有功能。<br/>
<!-- more -->
其中有这么一种情况，有个子模块需要能够实时查看到所有Open状态以及通过DashBoard进行划分种类的订单。但是鉴于每分钟Open的订单数据量在5W左右，如果每次查询检测都需要直逼DB进行查询，那么就可能会给数据库造成无形的压力（虽然已经采用了读写分离以及分库分表技术，但是每个订单所包含的Detail商品信息就可能上万，所以并不是看到5W个订单，就可以认为数据量很少。更何况，在微服务架构的基础上，我们不能直接进行跨库做表Join操作，而且只能通过网络传输数据）。
<br/>
# **1. 刚开始的设计**

每分钟Load一次Open态的订单数据，然后全部放入到Redis缓存中。<br/>

然而，结果是，因为数据量庞大，序列化和网络传输需要耗费大量的时间，所以产生了如下结果：<br/>

```
2018-05-1421:07:50.691 WARN [qtp683718244-2772]org.eclipse.jetty.server.HttpChannel.handleException:517-/sales-order-portal/

org.springframework.data.redis.serializer.SerializationException:Cannot serialize; nested exception isorg.springframework.core.serializer.support.SerializationFailedException:Failed to serialize object using DefaultSerializer; nested exception isjava.lang.OutOfMemoryError: Java heap space

       at org.springframework.data.redis.serializer.JdkSerializationRedisSerializer.serialize(JdkSerializationRedisSerializer.java:93)

       atorg.springframework.data.redis.core.AbstractOperations.rawHashValue(AbstractOperations.java:171)

       atorg.springframework.data.redis.core.DefaultHashOperations.putAll(DefaultHashOperations.java:129)

       atorg.springframework.data.redis.core.DefaultBoundHashOperations.putAll(DefaultBoundHashOperations.java:86)

       at org.springframework.session.data.redis.RedisOperationsSessionRepository$RedisSession.saveDelta(RedisOperationsSessionRepository.java:778)

       atorg.springframework.session.data.redis.RedisOperationsSessionRepository$RedisSession.access$000(RedisOperationsSessionRepository.java:670)

       atorg.springframework.session.data.redis.RedisOperationsSessionRepository.save(RedisOperationsSessionRepository.java:388)

       atorg.springframework.session.data.redis.RedisOperationsSessionRepository.save(RedisOperationsSessionRepository.java:245)
```

# **2. 缓存的重新设计**

考虑到查询条件有这么几种情况：

- 订单header级别的条件

- 订单line级别的条件

- 配置信息类的条件

而我们拿到的Open态的订单就属于header级别的数据，所以我们可以直接根据已知的三个条件分别分组，根据分组编号放到三个不同的Cache中，这样不仅避免了每次数据存放的传输以及序列化时间，同时也已经在无形之中减免了一部分查询条件。

## **2.1 进一步优化查询**

后来发现查询条件中还有一个根据订单号以及上述header级别信息进行组合过滤订单的条件，所以又针对此处加了一个特殊逻辑，即，如果有订单号这个查询条件，就先根据订单号进行查询，然后根据header级别信息进行匹配该订单，这样就又有效减免了Redis缓存数据的访问。

好了，项目也已经在UAT环境进行多轮测试以及集成、回归测试，期待上PROD环境的那天！
