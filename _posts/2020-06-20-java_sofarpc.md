---
layout:     post
title:      项目中的sofaRpc报错
subtitle:
date:       2020-06-20
author:
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - sofaRpc
    - sofaTracer
    - 从设计模式看中看SofaRpc中的EventBus
---


#### 背景案例

 * 当服务器通过walle4 发热更新数值, 走天梯发布执行成功之后, 在操作GM平台, 任意一命令,比如: 查询玩家信息, 发送邮件, 导号等命令(使用RPC的操作), 都会报空指针异常
 * 报错如下:
 ![Image](/img/error.jpg)


### 排查:
 * 经过各种测试, 发现只要在走天梯流程时, 有一个流程是`更新filecache, 只要跳过此步骤, 就没问题`
 ![Image](/img/filecache.png)

 * 那这个步骤具体都干了什么事
   * 那简单来说,就是把一些数据重新刷到zk
    ![Image](/img/updateZk.png)


 * 在热更新的时候, 通过global 的system.log 日志观察到
   * 会多次调RPCCenter里面的afterPropertiesSet()方法
   ![Image](/img/error1.jpg)
   * 这个方法会多次将sofaRpc的 Provider export(), 初步怀疑是多次export的问题~
     * `改了一波代码, 让afterPropertiesSet() 只被调用一次, 又重试了一下, OK了`
     * ` 那究竟是不是多次export问题? 看下sofa源码`

### 继续看下sofa:
   * 第一张图里的空指针异常, 这个报错是: `Tracer 中的 ConsumerTracerFilter中 clientSpan为空导致的`
   ![Image](/img/sofaTracer1.jpg)
   * clientSpan 中是通过ThreadLocal 来临时存储的, 那说明没有走push方法, 而push方法是在SofaTracerSubscriber走事件订阅触发的, `说明没有触发事件订阅`,
   ![Image](/img/sofaTracer2.jpg)

   * 通过debug发现 客户端引用的时候会看到如下这段代码：

   ```java

     // 产生开始调用事件
     if (EventBus.isEnable(ClientStartInvokeEvent.class)) {
         EventBus.post(new ClientStartInvokeEvent(request));
     }

   ```

   * 查看了源代码:
     * sofaRpc 本身有很很多模块扩展, RPC上下文会自动加载, 比如:模块有 tracer, metric 都属于此类扩展
     ![Image](/img/sofaTracer3.png)
     * SofaTracerModule 类实现了Module 接口，并增加 @Extension(“sofaTracer”) 注解，方便SOFARPC在启动时将相关模块加载进来。
        SofaTracerModule 作为SOFA-PRC 链路追踪的入口，在SofaTracerModule模块被加载时完成一些事件的订阅。

     * SofaTracerSubscriber是它的事件订阅者, 这里会`订阅 9 种事件`， 通过监听SOFARPC的这 9 种事件，
       来完成埋点数据的获取和异步磁盘写入操作。SOFARPC通过事件总线(EventBus)设计来订阅这些事件，当这些事件发生时会创建事件传入到EventBus。
        EventBus中一旦发布新的事件就会通知所有的订阅者，SAFA-Tracer 统一采用 SofaTracerSubscriber 订阅和处理这9种事件，
         最终链路追踪数据的获取操作都交给了RpcSofaTracer处理。
   * 如图:
   ![Image](/img/sofaTracer4.png)

   * 那这些事件何时被注册的呢?
   ![Image](/img/sofaTracer5.png)

   * 通过上面的继承关系图可以看到，ConsumerConfig是AbstractIdConfig的子类，所以在初始化ConsumerConfig的时候AbstractIdConfig静态代码块也会被初始化。

     ```java

      public abstract class AbstractIdConfig<S extends AbstractIdConfig> implements Serializable {

          static {
               RpcRuntimeContext.now();
          }
      }

     ```
   * 在调用`RpcRuntimeContext#now方法`的时候，会调用到RpcRuntimeContext的静态代码块
   ![Image](/img/moduleInstall.jpg)
   * 各个模块会被装载
   ![Image](/img/moduleInstall1.jpg)
   * 9种事件在这里注册的, 有install 也是有uninstall
   ![Image](/img/moduleUninstall.jpg)
   * `而通过debug发现, 在热更的时候会调detroy方法, 那显然是所有的module被uninstall了, 并不是因为provider export多次的问题`
   ![Image](/img/destroy.jpg)
   * 在我们的rest-api中, `RPCCenter类 重写了destroy(), 调用顺序: 先调destroy() -> afterPropertiesSet()`

     ```java

           @Override
           public void destroy() {
               RpcRuntimeContext.destroy();
           }

     ```
   ![Image](/img/destroy1.jpg)

   * `问题到这里已经找到了, 就是因为卸载了sofa各个模块, 所以再次调用sofa的时候, SofaTracer就报错了`
   * 应该不止sofaTracer, 涉及到destroy方法里的相关内容, 再次调用都会报错~~


### 结论
  * 不是provider 多次export 导致的, 是因为调了desotry(), 被uninstall了
  * [参考文档](https://www.sofastack.tech/blog/sofa-rpc-link-tracking/)
  * [参考文档](https://www.cnblogs.com/luozhiyun/p/11324181.html)