title: "Handler机制详解"
excerpt: "彻底明白Handler机制"
date: 2016/5/25
tags: 
- handler
- looper
- Android
categories:
- Android
comments: true
share: true
---


当Activity创建好之后，UI线程便处于等待消息阶段，当用户做完了某件事情之后可以向UI线程发送消息，如更新控件，刷新界面。最常见的就是从网络加载完图片后将其显示出来，显示操作必须在UI线程执行。  
那么，由谁在工作线程中发送这个消息，又是谁在UI线程中执行消息呢？  

**当然是Handler了**  

<!-- more --> 

Handler由以下部分组成:    

* Handler
* Looper
* MessageQueue
* Message   

我们需要弄清楚的问题

1. 消息从哪里产生？
2. 谁来发送消息？
3. 消息发送到哪里去？
4. 谁来处理消息？

把上面的问题全部明白了，那么Handler机制也就完全理解了。  
消息一般是由工作线程产生的，因为一旦工作线程完成了某件事情，它会通知主线程，所以工作线程产生消息。  
**产生消息的方式：**  

		Message msg=Message.ontain();
		msg.what=IMAGE_LOAD_SUCCESS;  
		//这条消息用来说明，图片已经加载成功  


**发送消息：**

		hanlder.sendEmptyMessage(msg);

下面，问题来了，消息要发送到哪去呢？
实际情况是，当Handler创建的时候，他会与Looper进行绑定，每一个Looper创建的时候都对应了一个MessageQueue，那么handler也就绑定了相应的MessageQueue，handler会把消息发送到该MessageQueue中。    
且看Handler的构造函数：
		
 		<code>
		public Handler(Callback callback,boolean async){
				//code removed for simplity
				mLooper=Looper.myLooper();
				if(mLooper==null){
					throw new RuntimeException("can not create handler inside thread that has not call Looper.prepare()");
				}
				mQueue=mLooper.mQueue;
				mCallback=callback;
		}
		</code>  

**Attention:同一线程中的多个Handler分享同一个消息队列，因为他们共享的是同一个Looper对象**  

那么，消息最后是由谁来执行的呢？
MessageQueue中的message最后会被Looper取到，Looper会查看Message对象中的target变量，他就是处理该Message的对象，也就是我们发送消息的那个handler对象，Looper会调用handler的handleMessage()方法。  

**MessageQueue** 

<figure class="half">
	<a href="https://camo.githubusercontent.com/f60de4545ea6ca500e589179d268d0b88096df8a/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f313630302f312a6f6764576d585273356d642d4b6d69426e62363165672e706e67"><img src="/images/Handler-internal/handler_internal_01.png"></a>
	<figcaption>消息队列</figcaption>
</figure>

{% img /images/Handler-internal/handler_internal_01.png [Handler内部逻辑]%}



在发送消息的时候，我们可以发送延迟消息
  
		handler.sendMessageDelayed(Message msg,long delayMillis);  

每一个Message对象都带有when参数，如果这个参数小于消息队列中的dispatch barrier，那么Looper就可以取到这条消息，否则是取不到的  

Handler、MessageQueue、线程（生产线程、消费线程）之间的交互如下： 

<figure class="half">
	<a href="https://camo.githubusercontent.com/11a7614fcfceaea60697d4a9b7486f9f01d2a327/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f323030302f312a5f3270773535323872666f5470504245376c394e6d412e706e67"><img src="/images/Handler-internal/handler_internal_02.png"></a>
	<figcaption>消息队列、Handle、生产线程的交互</figcaption>
</figure>




**Looper**

Looper从消息队列中读取消息，然后分发给对应的Handler处理，一旦超过了阈值，那么Looper就会在下一轮读取过程中读到该Message。Looper在没有消息分发的时候会变成阻塞状态（执行线程被阻塞），当有消息时继续轮询（执行线程被唤醒）。  
每个线程只能关联一个Looper，问题来了，这是怎么实现的？这就要从Looper的构造方法出来，看看他的实例是怎么被创建出来的就明白了。

<figure class="half">
	<a href="https://camo.githubusercontent.com/586f61a1b8dafc1effcfb6ec23b8a206418b3777/68747470733a2f2f63646e2d696d616765732d312e6d656469756d2e636f6d2f6d61782f313630302f312a734e4a72672d336d566335346a5a66566576576f44672e706e67"><img src="/images/Handler-internal/handler_internal_03.png"></a>
	<figcaption>Handler与消息队列和Looper的交互</figcaption>
</figure>



我们通过Looper.myLooper()方法拿到一个Looper实例对象，且看该方法的说明

		/**
    	 * Return the Looper object associated with the current thread.  Returns
    	 * null if the calling thread is not associated with a Looper.
    	 * 返回与当前线程相关联的Looper对象，如果该线程还没有关联Looper对象，那么返回null
     	 */
    	public static @Nullable Looper myLooper() {
      	  return sThreadLocal.get();
    	}

那么，Looper对象到底在哪里被关联的呢？

		private static void prepare(boolean quitAllowed) {
        	if (sThreadLocal.get() != null) {
        	    throw new RuntimeException("Only one Looper may be created per thread");
        }
        	sThreadLocal.set(new Looper(quitAllowed));
    	}

我们且不管ThreadLocal的内部实现原理如何，他的作用就是将对象与线程关联，一个线程一定与一个Looper实例对象关联。

**更深层次的思考**

一般情况下（99.99%）,我们只会向UI线程的消息队列中发送消息，但要是你哪天兴致来了，自己new了一个Thread，而且你打算给这个线程发送消息，那么你需要做哪些工作呢？  
当你new Thread 得到一个线程对象时，它此时并没有和Looper关联，难道我还要自己去创建Looper?
放心，这么艰巨的任务还轮不到你来做，Android提供了一个简单的类来一步搞定---HandlerThread，他是Thread的子类，并提供对Looper创建的管理。

且看示例代码：

		private HandlerThread handlerThread;
		@Override
		protected void onCreate(@Nullable Bundle savedInstanceState) {
		    super.onCreate();
		    handlerThread = new HandlerThread("HandlerDemo");
		    handlerThread.start();
		    handler = new CustomHandler(handlerThread.getLooper()); //绑定handlerThread对应的Looper
		}
		@Override
		protected void onDestroy() {
		    super.onDestroy();
		    handlerThread.quit();  //在Activity被销毁的时候，终止这个线程，相应的Looper也会被终止
		}
		
		








