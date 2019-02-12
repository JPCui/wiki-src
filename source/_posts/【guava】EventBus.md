---
title: 【guava】EventBus
date: 2018-11-05 15:11:52
tags:
  - guava
  - eventbus
---

# EventBus

消息总线, 望文生义, 发布订阅者模式中用来做事件调度的组件

下文基於`guava-18.0`

EventBus 比較值得学习的是:

- com.google.common.eventbus.EventBus#subscribersByType
  
  > SetMultimap<Class<?>, EventSubscriber> subscribersByType
  
  以 event object 作为 key, method 为方法
  
  这样在消费event的时候, 可以根据 event object 快速找出所有的订阅者方法(@Subscribe)
  
  注册消费者的方法可以借鉴
  -> `com.google.common.eventbus.EventBus#register`
  -> `com.google.common.collect.Multimap#putAll`

- AsyncEventBus

  异步的消息总线, 其实就是继承EventBus之后重写了3个方法:
  
  - enqueueEvent: event入队列
    
    - EventBus: 使用ThreadLocal隔离多线程, 每个线程自己处理自己的event
    - AsyncEventBus: 使用ConcurrentLinkedQueue解决并发, 使得任务调度更高效
     
  - dispatchQueuedEvents: 调度队列中的 event
  
    几乎没什么区别, 都是循环拿event进行调度
    - EventBus: 还要用isDispatching设置是否正在执行的开关, 感觉有点鸡肋
  
  - dispatch: 调度方法
  
    - EventBus: 反射调用真实的订阅者(`method.invoke(target, new Object[] { event })`)
    - AsyncEventBus: 使用 Executor 异步调度: super.dispatch
    
## 参考

- [[Google Guava] 11-事件总线](http://ifeve.com/google-guava-eventbus/)
  
  很好的解释了:
  - 为什么要用 EventBus
  - 为什么使用 @Subscribe 注解, 而不是接口
  - 为什么不使用泛型(spring event 中就用的泛型), 而是 Object

- [Guava：EventBus 源码分析](https://juejin.im/post/5b61c852e51d451956055476)