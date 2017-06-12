---
layout: post
title: "这样来理解AIDL"
excerpt: "明白AIDL在IPC中的作用"
categories: articles
tags: [android,binder，service，aidl]
comments: true
share: true
---

### 之前的学习方法
在面试之前，找几个AIDL的实例学习，比如CSDN上的博客，认真的话还会将实例敲一遍，不然就是直接死记硬背，整一个Service，然后再用Client请求一遍，完事。可是,`几个星期之后，你又忘记了！！！`

### 个人的一点小看法
要完全的理解AIDL的存在，以及其中各个类的作用，是需要前提知识，我只能说是窥得其中皮毛。  

1. Binder机制的理解
2. 我一直强调的层级的理解
3. 一点点Java基础

AIDL接口文件

```
//HelloService.aidl
interface HelloService{
	  void   setVal(String val);
	string   getVal();
}

```

HelloService代码

```
public class HelloService extends Service {
    //由AIDL文件生成的HelloService
    private final HelloService.Stub mHelloService = new HelloService.Stub() {
        @Override
        public String getVal() throws RemoteException {
           //
        }
        @Override
        public void setVal(String val) throw RemoteException{
        	//
        }
    }
    @Override 
    public IBinder onBind(Intent intent){
    	return mHelloService;
    }
        

```

Client代码

```
bindService(intent,mServiceConnection,Context.BIND_AUTO_CREATE)
//
private ServiceConnection mServiceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
                    mHelloService = HelloService.Stub.asInterface(service);
        }
}

```
Client只需要调用这简单的代码就可以获得和Service端在Java层相同的接口，当然，客户端的Java类是在客户端创建的，只不过，他这个类中的信息是从Service端传过来的。在IPC的时候，Client和Service在Java层的接口必须要统一，Client在Java层中调用了某一个方法，必然与Service端的那个方法对应。

AIDL生成的Java类的分析

```
package com.example.helloservice;

public interface IHelloService extends android.os.IInterface
{
/** Local-side IPC implementation stub class. */
public static abstract class Stub extends android.os.Binder implements com.example.helloservice.IHelloService
{
private static final java.lang.String DESCRIPTOR = "com.example.helloservice.IHelloService";
/** Construct the stub at attach it to the interface. */
public Stub()
{
this.attachInterface(this, DESCRIPTOR);
}
/**
 * Cast an IBinder object into an com.example.helloservice.IHelloService interface,
 * generating a proxy if needed.
 */
public static com.example.helloservice.IHelloService asInterface(android.os.IBinder obj)
{
if ((obj==null)) {
return null;
}
android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
if (((iin!=null)&&(iin instanceof com.example.helloservice.IHelloService))) {
return ((com.example.helloservice.IHelloService)iin);
}
return new com.example.helloservice.IHelloService.Stub.Proxy(obj);
}
@Override public android.os.IBinder asBinder()
{
return this;
}
@Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
{
switch (code)
{
case INTERFACE_TRANSACTION:
{
reply.writeString(DESCRIPTOR);
return true;
}
case TRANSACTION_setVal:
{
data.enforceInterface(DESCRIPTOR);
java.lang.String _arg0;
_arg0 = data.readString();
this.setVal(_arg0);
reply.writeNoException();
return true;
}
case TRANSACTION_getVal:
{
data.enforceInterface(DESCRIPTOR);
java.lang.String _result = this.getVal();
reply.writeNoException();
reply.writeString(_result);
return true;
}
}
return super.onTransact(code, data, reply, flags);
}
private static class Proxy implements com.example.helloservice.IHelloService
{
private android.os.IBinder mRemote;
Proxy(android.os.IBinder remote)
{
mRemote = remote;
}
@Override public android.os.IBinder asBinder()
{
return mRemote;
}
public java.lang.String getInterfaceDescriptor()
{
return DESCRIPTOR;
}
/**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
@Override public void setVal(java.lang.String val) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
try {
_data.writeInterfaceToken(DESCRIPTOR);
_data.writeString(val);
mRemote.transact(Stub.TRANSACTION_setVal, _data, _reply, 0);
_reply.readException();
}
finally {
_reply.recycle();
_data.recycle();
}
}
@Override public java.lang.String getVal() throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
java.lang.String _result;
try {
_data.writeInterfaceToken(DESCRIPTOR);
mRemote.transact(Stub.TRANSACTION_getVal, _data, _reply, 0);
_reply.readException();
_result = _reply.readString();
}
finally {
_reply.recycle();
_data.recycle();
}
return _result;
}
}
static final int TRANSACTION_setVal = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
static final int TRANSACTION_getVal = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
}
/**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
public void setVal(java.lang.String val) throws android.os.RemoteException;
public java.lang.String getVal() throws android.os.RemoteException;
}

```




