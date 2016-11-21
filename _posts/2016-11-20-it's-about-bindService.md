---
layout:     post
title:      "关于bindService的那些事"
subtitle:   ""
date:       2016-11-20
author:     "Bewater"
header-img: "img/post-bg-js-module.jpg"
tags:
    - Anrdroid
    - Service
---

>写在前面的话： 知识是永远学不完的，就算是世界上的天才，他也只能掌握茫茫知识海洋的一小部分，所以不管你在什么领域，什么位置，永远保持一颗学习的态度，这样我们才会明白自己的无知，才会激励自己不断成长、强大。

### 一、明确文章目标，通过该篇文章我们需要掌握什么？
- bindService的使用场景
- bindService的实现原理
  
  关于Service的使用方法官方文档、其他blog到处都有我就不在此赘述了

### 二、bindService的使用场景：  

bindService接口定义在Context中，所以有Context的地方我们都可以取bindService，通常是在Activity的生命周期方法中（onCreate、onStart）去bindService，另外bindService通常用在需要跨进程的场景，如果是同进程建议使用startService即可，这里提下startService和bindService的特性，startService后需要stopService来结束Service的生命周期，而bindService对应的是unbindservice来接收Service的生命周期，当所有bindService的客户端都unbindService时，Service就会被系统kill掉（没有结合startService启动的话），bind远程Service时，如果远程Service挂掉，client端也能通过onServiceDisconnected感知到（通过Binder的linkToDeath实现）

### 三、bindService的实现原理：
![Public License](\img\in-post\it's-about-bindservice\bindService.png)
以上时序图基本囊括bindService的主干流程，具体每一步我不多说了，结合源码去看没什么难度，其中onServiceConnected接口的回调是AMS通过scheduleBindService构建Server端Binder，再通过publishService调用IServiceConnection connected来完成的，另外需要注意的是这张图中涉及到3个进程的交互，发起调用的Client进程，提供Service服务的Server端进程，AMS所在的SystemServer进程。同时bindService时Service所在进程有3种状态，1.Service所在进程不在运行状态，2.Service所在进程为运行状态，但Service不在运行状态，3.Service为运行状态。上面的时序图是针对相对复杂的第一种情况画的。

两个问题：  
a.为什么说bindService是一个异步调用？  
b.可以在onServiceConnected中直接使用返回的IBinder吗？为什么？
      
先分析第一个问题，为什么说bindService是一个异步调用？也就是说调用bindservice后会理解返回，而我们知道一般情况下，通过Binder跨进程调用时会将当前进程挂起，等待Server端Binder完成函数操作后再将当前发起函数调用的线程恢复并返回结果，而bindService这个地方有什么特殊的玄机吗？先公布实现的方式就是Binder的oneway方式，也就是在AIDL接口中若声明oneway关键字，则在发起IPC调用时不会阻塞在client端跟驱动的交互中，我们知道bindService接口本身的声明为同步的，注意如下最后一个参数为0，IBinder.FLAG_ONEWAY为1。
      
        mRemote.transact(BIND_SERVICE_TRANSACTION, data, reply, 0)  

这说明在其他地方发起了异步调用，经过分析，从IServiceConnection.aidl中找到了答案:

        /** @hide */
        oneway interface IServiceConnection {
            void connected(in ComponentName name, IBinder service);
        }
        
该interface的实现类为：ServiceDispatcher.InnerConnection

当要bind的Service已经是运行状态时且有其他Client端bind过，bindService时AMS中会通过IServiceConnection通过Binder方式发起IPC回调，然后bindService方法直接返回，connected最终会回调Client端的onServiceConnted方法。当要bind的Service为非运行状态或之前未bind过，AMS会通过Binder方式想Server端进程发起scheduleBindService请求，该请求也为异步方式，scheduleBindService主要通过调用Server的onBind函数获取Binder对象，然后通过AMS将其publish，publish时也会回调Client端的onServiceConnected接口。

这里可能大家会有疑惑，善于专研的同学可能会发现无论你再哪个线程去发起bindService操作，onServiceConnected都会在主线程完成回调，原因直接上代码：

        public void connected(ComponentName name, IBinder service) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0));
            } else {
                doConnected(name, service);
            }
        }  

  mActivityThread实际为ActivityThread中的mH对象，也就是处理UI线程相关操作的Handler，到此我们就明白不管你再Activity中的哪个生命周期方法发起bindService操作，也不管需要bind的Service是否已经运行或被被bind过（Service的onBinnd已经回调，Binder已经构建完成），onServiceConneced总是无法在你bind后的再下一行代码就能获取到。   
  
  到此第一个问题基本分析结束，其实第二个问题第一个已经解释一部分，就是onServiceConned在主线程完成回调，如果通过IBinder发起的调用耗时的话就会严重影响用户体验，所以，如果Binder Server端提供的接口是耗时操作，在对应的Client端请使用线程池方式发起调用，同时如果定义的某个aidl接口存在多线程操作的话，需要做好Binder Server端接口的同步操作！

---
说明：这是本站的第一篇博客，所以肯定存在纰漏之处，后续针对每篇博客存在问题的 地方我也会不断更新，力求完美一点，大家如果有什么问题可以在下面留言（不是关于这篇博客的也可以），我看到后会尽快给大家回复。

欢迎转发，转发请注明作者:*bewater*, 以及原博客原文地址：[http://beawaters.com/2016/11/20/its-about-bindService/](http://beawaters.com/2016/11/20/it%27s-about-bindService/)