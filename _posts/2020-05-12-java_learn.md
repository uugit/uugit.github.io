---
layout:     post
title:      Self-Reflection
subtitle:
date:       2020-05-12
author:
header-img: img/post-bg-rwd.jpg
catalog: true
tags:
    - 面试题
    - 知识点
---


#### 如果一个对象 要作为hashmap的key需要做什么？

 * java中hashMap使用一个对象作为key时，对key进行唯一性表达`重写equals()方法`,
 * equals()比较的是内存地址是否相等。


#### Threadlocal类以及内存泄漏
 > 首先明确threadLocal 不是为了给线程共享变量, 而是给每个线程分配一个属于该线程的变量, 提供一个变量值的副本, 每个线程可以独立修改自己的副本

 * ThreadLocal只是操作Thread中的ThreadLocalMap，每个Thread都有一个map，ThreadLocalMap是线程内部属性，ThreadLocalMap生命周期是和Thread一样的，不依赖于ThreadMap。
 * set/get/remove/initialValue

 ![Image](/img/threadLocal.jpg)

 * 图中的虚线表示弱引用。 这样，当把threadlocal变量置为null以后，没有任何强引用指向threadlocal实例，所以threadlocal将会被gc回收。这样一来，
   ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程再迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链：
   Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value，而这块value永远不会被访问到了，所以存在着内存泄露。

 * 只有当前thread结束以后，current thread就不会存在栈中，强引用断开，Current Thread、Map value将全部被GC回收

```java

1.使用ThreadLocal，建议用static修饰 static ThreadLocal<HttpHeader> headerLocal = new ThreadLocal();
2.使用完ThreadLocal后，执行remove操作，避免出现内存溢出情况。调用remove()方法最佳时机是线程运行结束之前的finally代码块中调用


```
#### 线程同步方式，具体每一个怎么做的

 ![Image](/img/thread.jpg)

  * 保证每个线程访问资源的时候获取到的都是资源的最新值(可见性)
  * 当有线程 操作该资源的时候锁定该资源，禁止别的线程访问(锁)
  * 线程本地私有化一份本地变量，线程每次读写自己的变量(ThreadLocal)

#### EpollEventLoopGroup 与 NioEventLoopGroup 的使用区别以及出现场景

 * 项目代码中, 在读到NettyTcpNet的时候，发现有一个Epoll模式和nio模式, 去网上看了一下，大概对应了selector和epoll模式

 ```java

 发现epoll是linux特定内核，比如2.6以上的io方式。比如stackoverflow上有两者的比较，有人给出回复：

 If you are running on linux you can use EpollEventLoopGroup and so get better performance, less GC and have more advanced features that are only available on linux.

 就是在linux上使用EpollEventLoopGroup会有较少的gc有更高级的特性，只有在Linux上才可以使用。那么这句话几乎告诉我们epoll是最好的了，因为现在几乎所有的服务端程序都运行在Linux上。

 高性能的比如Minecraft中的游戏服务器，对这个的要求更是高

 ```

  > NioEventLoop循环调用 Selector中的注册的SelectionKey获取每个连接的数据。然后把获取到的字节数据传送给ChannelPipeline处理。
  > EpollEventLoop通过epollWait来得到数据事件，循环EpollEventArray的事件处理每个连接的数据。也是传给ChannelPipeline处理。

  * 所以epoll相对性能更高一些。

#### jvm类加载双亲委派模式，有没有能破坏这个模式的方法？类加载整个过程解释

  * 沿用双亲委派机制自定义类加载器很简单，只需继承ClassLoader类并重写findClass方法即可

  * 打破双亲委派机制则不仅要继承ClassLoader类，还要重写loadClass和findClass方法

 ![Image](/img/loadClass.jpg)

#### 垃圾回收算法，垃圾回收器有什么？
  * Mark-Sweep（标记-清除）算法
  * Copying（复制）算法
  * Mark-Compact（标记-整理）算法
  * Generational Collection（分代收集）算法

 > 垃圾回收器:
  * 新生代垃圾收集器有Serial、ParNew、Parallel Scavenge，G1
  * 属于老年代的垃圾收集器有CMS、Serial Old、Parallel Old和G1


 ![Image](/img/gc.jpg)
