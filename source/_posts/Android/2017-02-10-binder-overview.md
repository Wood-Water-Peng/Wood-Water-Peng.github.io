---
layout: post
title: "Binder架构概览"
excerpt: "从宏观上了解Binder架构"
tags: 
- binder
- Android
categories:
- Android
comments: true
share: true
---

在分析Binder架构的时候,我一直想着---"抽象,了解当前代码所处的层级"  
现在来分析一下Binder架构中中的几个概念，真正把这几个概念搞懂了，绝对是事半功倍的。  
以Service的IPC流程为例:  

* Client　　　　　　　　　　　客户端进程
* Server　　　　　　　　　　　服务端进程
* Service Manager　　　　　　ServiceManager进程

这三个是进程的概念,是横向的比较。

<!-- more --> 

<figure class="half">
	<img src="/images/binder-overview/process_model.png">
	<figcaption>进程简单模型</figcaption>
</figure>

<p>　　当你打包出的apk被加载到内存中时，内存中构造出这样一个进程模型，站在我们角度，可以简单的认为代码是这样按层次分布的。所有的IPC通信，肯定是建立在和内核通信的基础之上的。那么，这里的Binder机制到底体现在哪里呢？</p>
<p>　　Java层中的Binder对象，C++层中的Binder对象，内核中的Binder驱动程序---这三者就是Android中的Binder机制，他们的目的就是来干IPC通信的，注意，这又是纵向层面的分析，他们在同一个进程中，自上而下(类似于TCP/IP协议通信)。</p>

**怎样来理解Binder驱动程序呢**

<p>　　我们知道有所谓的硬件驱动程序，他们在内核中注册，由内核调用，达到操作硬件的目的。那么Binder驱动程序，可以简单的理解为他操作的对象是一个Binder设备文件，所有的进程要想操作这个文件就必须通过Binder驱动程序。</p>

<figure class="half">
	<img src="/images/binder-overview/binder.png">
	<figcaption>Binder通信机制</figcaption>
</figure>

---
**Service Manager的启动**
  
ServiceManager由系统实现,在应用程序启动之前，它已经运行在系统中了。

<p>　　我们可以认为ServiceManager由C/C++代码实现,他一启动立刻执行main()函数，然后他在C/C++层创造了一个Binder对象，进而穿透到内核层，创建了一个Binder实体，并且让内核知道，这个ServiceManager在内核中的Binder实体是整个Binder机制的管理者，所有对其他Binder的实体请求必须经过该实体。</p>
那么，该进程运行起来后，就存在这么几个东西

1. C/C++层的Binder对象
2. Binder驱动中的Binder实体(句柄为0)  

**Service的启动**

　　Server进程,即Service运行的进程,系统实现了一部分，用户也可以自己实现。  
<p>　　以系统的MediaPlayerService为例，同样是C/C++代码实现,启动后执行main()函数，在C/C++层创建一个Binder对象，然后穿透到内核层创建一个Binder实体,然后告诉内核把自己的Binder实体注册到ServiceManager的Binder实体中，那么后续对MediaPlayerService的Binder实体的请求都会经过ServiceManager的Binder实体。</p>

那么，该MediaPlayerService进程运行起来后，就存在这么几个东西

1. C/C++的MediaPlayerService的Binder对象
2. Binder驱动中的Binder实体(已经注册到ServiceManager中)

**客户端的请求**  

　　客户端进程也就是我们的App,比如我们在MainActivity的onCreate()中写了  
`ServiceManager. getIServiceManager()`  
<p>　　那么，实际上返回的是ServiceManagerProxy对象，该对象包含一个BinderProxy对象，这个BinderProxy是用来向C++层穿透，进行通信的，下面来看下这个BinderProxy对象是怎么获得的。</p>
`BinderInternal.getContextObject()`  
<p>　　然后进行JNI调用，穿透到C++层，再到内核，因为之前Binder驱动中已经有了ServiceManager的Binder实体信息，他会将实体信息返回到C++层，C++层根据驱动层的Binder实体信息创建出一个Binder对象，然后再向上返回到调用该函数的Java层，同样，Java层根据C++层返回的信息创建一个BinderProxy对象(与C++的Binder对象，驱动中的Binder实体对应)。</p>  
<p>　　最后，在Java层中，就有了与底层ServiceManager的Binder实体进行交互的BinderProxy对象。（注意，这一切并没有涉及到IPC，因为ServiceManager在Binder驱动的Binder实体的句柄为0，这是默认的）。</p>
　　然后ServiceManagerProxy执行获取CalculateService的代码
`getService("CalculateService")`。  
<p>　　一句话概括，返回的是一个可以与CalculateService进行交互的在Java层的Binder对象。
(Binder体系的Binder对象都是对应的，Java中一个，C++中必然对应一个，Binder驱动中也有一个)。</p>
  
具体流程如下：

1.`ServiceManagerProxy.getService(String serviceName)`  
通过拿到的ServiceManager在java层的BinderProxy对象，执行
`mRemote.transact(GET_SERVICE_TRANSACTION, data, reply, 0);`  
该函数执行JNI调用，调用到C++层的BinderProxy对象，再到Binder驱动中的Binder实体(ServiceManager的)，Binder驱动根据传入的参数找到CalculateService的Binder实体，将信息返回到C++层,C++层再将信息返回到Java层，Java层根据返回的信息创建出一个BinderProxy对象(CalculateService的)。  
2.`IBinder binder = reply.readStrongBinder();`   
后，拿到的是CalculateService在Java层的BinderProxy对象。
  
**打住,既然这里涉及到IPC，那么一直让人恐惧的AIDL又在哪里体现出作用呢？**  
具体的又涉及到这样一个问题,怎样自己实现一个Service,比如，CalculateService。












