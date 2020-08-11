---
layout:     post
title:      synchronized
subtitle:
date:       2020-06-22
author:
header-img: img/post-bg-rwd.jpeg
catalog: true
tags:
    - synchronized
---


#### synchronized
 * cas 最终的汇编指令: lock compare and exchange, 当一个cpu修改的时候, 其他cpu 不能打断
 * 工具JOL = java Object Layout (`描述下对象在内存中的布局`)
 ```java
   Object o = new Object();  // 在内存中占用多少字节?
   ClassLayout.parseInstance(o).toPrintable();

   压缩情况下, 占用16字节
 ```
   ![Image](/img/jol.jpg)


