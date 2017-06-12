layout: post
title: "理解AIDL"
excerpt: "从细节上理解AIDL"
tags: 
- aidl
- Android
categories:
- Android
comments: true
share: true
---

	public static com.example.pj.testingexample.IHelloService asInterface(android.os.IBinder obj)
	{
		//obj为在客户端创建的Binder对象，可与Service端通信
		if ((obj==null)) {
			return null;
		}
		android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
		if (((iin!=null)&&(iin instanceof com.example.pj.testingexample.IHelloService))) {
			return ((com.example.pj.testingexample.IHelloService)iin);
		}
		//返回一个代理类，代理类包含一个Binder对象，可向Service发起通信
		return new com.example.pj.testingexample.IHelloService.Stub.Proxy(obj);
	}





