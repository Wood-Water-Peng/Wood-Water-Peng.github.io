---
layout: post
title: "Android纵向通信概览"
excerpt: "自底向上的分析"
tags: 
- architecture
- binder
- Android
categories:
- Android
comments: true
share: true
---

<figure class="half">
	<img src="/images/longitudinal-overview/data_transmission_over_the_internet_through_tcp-ip.png">
	<figcaption>TCP/IP传输模型</figcaption>
</figure>

<p>　　TCP/IP协议的传输机制能给我们很大的启发，构造这个协议是自底向上的，底层提供接口，供上层调用。上层使用者只要给到符合接口的数据，便可跨层传输数据。当然，所有的层都是抽象出来的，数据最终的传递还是要靠物理层，但是呢，每一层都保存了对应的上下文信息，在层与层之间通信的时候，便不会出错。最后，所有的信息最后都是通过物理层出去的，以电信号的形式。</p>

	那么，这个理念搬移到Android的纵向通信中该怎么理解呢？    

<!-- more --> 

<figure class="half">
	<img src="/images/longitudinal-overview/Android_architecture.png">
	<figcaption>TCP/IP传输模型</figcaption>
</figure>
　

<p>　　且看这张官方的Android架构图</p>

　　现在心里想着这么一个需求，一段java代码要修改硬件值  

	java-->C++-->HAL-->Kernel-->硬件驱动程序
你在java层写了个setValue(String val)，他会一层层穿透，直达内核，内核调用硬件的驱动程序，执行最后的操作。

* java-->C++   　　　JNI实现  
* C++-->HAL　　　　　C++代码通过HAL定义的硬件访问接口打开硬件设备  
* HAL-->Kernel　　　根据HAL的规范，定义出相应模块，上层的C++代码可以访问，该层通过系统调用达到内核
* 在内核空间，内核直接调用驱动程序