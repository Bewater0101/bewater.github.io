---
layout:     post
title:      "关于bindService的那些事"
subtitle:   ""
date:       2016-11-17
author:     "Bewater"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Anrdroid
    - Service
---

写在前面的话： 知识是永远学不完的，就算是世界上的天才，他也只能掌握茫茫知识海洋的一小部分，所以不管你在什么领域，什么位置，永远保持一颗学习的态度，这样我们才会明白自己的无知，才会激励自己不断成长、强大。

>1、明确文章目标，通过该篇文章我们要掌握什么？
- bindService的使用场景
- bindService的实现原理
  
  关于Service的使用方法官方文档、其他blog到处都有我就不在此赘述了

> 2、 bindService的使用场景：
 
 bindService接口定义在Context中，所以有Context的地方我们都可以取bindService，通常是在Activity的生命周期方法中（onCreate、onStart）去bindService，另外bindService通常用在需要跨进程的场景，如果是同进程建议使用startService即可，我们理解完startService和bindService的特性后结合使用，startService后需要stopService来结束Service的生命周期，而bindService对应的是unbindservice来，当所有bindService的客户端都unbindService时，该Server端Service就会被系统kill掉（没有结合startService启动的话），bind远程Service时，如果远程Service挂掉，client端也能通过onServiceDisconnected感知到（通过Binder的linkToDeath实现）

>3、bindService的实现原理：

![Public License](https://www.processon.com/chart_image/582ede27e4b05594f5090a7c.png)

说明：以上文章并未写完整，博主先写一个大概测试一下，我会尽快更新完整。