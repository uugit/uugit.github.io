---
layout:     post
title:      SecureRandom生成随机数超慢,导致玩家登陆,或者握手也很慢
subtitle:
date:       2020-08-11
author:
header-img: img/post-bg-desk.jpeg
catalog: true
tags:
    - SecureRandom
---


#### 背景案例

 * 当每次发布完天梯流程, 测试登录, 在第一次登陆的时候, 时间超长, 需要等过 几秒才能成功登录
 * why ?


### 排查:
 * 压测的同事, 也遇到同样的问题 , 会有报错
 ![Image](/img/0A1.png)
 ![Image](/img/9D9.png)

 * client端有时候发不出去init 或者 client端有时候收不到init的返回也就是handshake的结果,
   所以在发送init消息后增加阻塞等待init返回能解决问题，也就是说消息在init发出去之后有耗时。


 * 具体分析
   * client发送init消息 到 收到服务器handshake消息 之间的间隔接近一分钟

    ![Image](/img/608.png)

   * 查看服务器端收到init消息的时间是14:54:53

    ![Image](/img/A27.png)

   * `那么在client->server的init消息都做了什么呢？`
     其实就是一个握手过程,在我们的网络层中叫做CONNECT过程：`其实主要做了加解密秘钥相关的工作`

     流程如下：
     client --> server : send init msg
     server: receive init msg
     server --> client : send handshake msg

     ![Image](/img/f3.png)

   * 在看看代码, 因为是静态代码块, 所以在初始化类的时候就已经执行了
     ![Image](/img/02C.png)

   * 具体看AesConfig：

     ![Image](/img/DA6.png)

   * 从逻辑看，耗时只能出现在下面初始化iv的地方：

    ```java

      this.iv = alg.generateIv();

      // 调用
      public byte[] generateIv() throws NoSuchAlgorithmException, NoSuchPaddingException {
          // 主要是这里
          SecureRandom ran = SecureRandom.getInstance("SHA1PRNG");
          Cipher c = Cipher.getInstance(AES_TYPE);
          byte[] iv = new byte[c.getBlockSize()];
          ran.nextBytes(iv);
          return iv;
      }

    ```

   * 通过查询资料了解SecureRandom：
    ![Image](/img/F84.png)

   * 意思就是说: `如果熵源是/dev/random, {@code generateSeed}, {@code reseed} and {@code nextBytes} 这三个方法会出现阻塞`

   * 然而: 对于底层选择entropy source的逻辑如下：所以我们的环境下默认的entropy source是/dev/random

    ```java

    		String PROP_EDG = "java.security.egd";
    		String PROP_RNDSOURCE = "securerandom.source";
    		String edg1 = System.getProperty(PROP_EDG, "");
    		String edg2 = Security.getProperty(PROP_RNDSOURCE);
    		Log.system.debug("sys edg1 : {}" + edg1);
    		Log.system.debug("sys edg2: {} " + edg2);

    ```

    ![Image](/img/C80.png)


### 解决方式:
   * 增加jvm配置参数：

    ```java
      -Djava.security.egd=file:/dev/./urandom
    ```
   * Default algorithm: DRBG
   * Provider: SecureRandom.DRBG algorithm from: SUN

   ![Image](/img/584.png)

### 参考
  * [参考文档](https://stackoverflow.com/questions/58991966/what-java-security-egd-option-is-for)
  * [参考文档](https://www.52jingya.com/aid4198.html)
  * [参考文档](https://hongjiang.info/jvm-random-and-entropy-source/)
