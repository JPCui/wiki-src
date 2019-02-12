---
title: 【redis】RedisTemplate的一点疑问
date: 2018-11-05 15:11:52
tags:
  - redis
  - spring-data
---

最近老是看到 redis 从pool里获取/释放的频繁操作日志, 跟踪业务发现, 每调用一次 RestTemplate.execute(callback), 都会从连接池里获取jedis连接

频繁获取和释放连接, 会很浪费资源.
在获取资源时, 会经过下面几个操作:

- 获取空闲对象(takeObject/pollObject)
- 激活对象(activateObject)
- 检查对象是否可用(testOnBorrow/testOnCreate/validateObject)
- 修改统计状态(updateStatsBorrow)

释放资源时:

- 检查对象是否可用(testOnReturn/validateObject)
    - 如果不可用, 销毁(destroy), ensureIdle(1, false)
- 钝化对象(passivateObject), 意思就是刚对象刚被使用过, 让他休息一下
- 归还对象(addFirst/addLast)
- 修改统计状态(updateStatsBorrow)

当然你可以把业务放在callback里处理, 感觉又不太爽, 所以还是把jedis重新封装一下, 毕竟 [oasis-ct-core](/oasis-ct/oasis-ct-core) 里封装的 [RedisDao/RedisService](/oasis-ct/oasis-ct-core/src/master/src/main/java/cn/oasis/core/config/RedisConfiguration.java) 也很差, 序列化的方式使用的也比较乱.

这里来看下rap的一个issue, 值得借鉴: 
https://github.com/thx/RAP/pull/1008
解决:
https://github.com/thx/RAP/commit/ebd22f2b095b9cb6983a9945e5297a4ace9d439a
