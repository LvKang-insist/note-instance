在上篇文章中我们使用 AIDL 完成了进程之间的通信，下面我们分析一下 AIDL 文件的实现类

aidl 文件如下：

```java
/**
 * 连接服务
 */
interface IConnectionService {

   oneway void connect();

    void disconnect();

    boolean isConnected();
}
```

实现类在 gen/aidl.... 目录下面，如下：

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 */
package com.www.mk_ipc_demo;

/**
 * 杩炴帴鏈嶅姟
 */
public interface IConnectionService extends android.os.IInterface {
    /**
     * Default implementation for IConnectionService.
     */
    public static class Default implements com.www.mk_ipc_demo.IConnectionService {
        @Override
        public void connect() throws android.os.RemoteException {
        }

        @Override
        public void disconnect() throws android.os.RemoteException {
        }

        @Override
        public boolean isConnected() throws android.os.RemoteException {
            return false;
        }

        @Override
        public android.os.IBinder asBinder() {
            return null;
        }
    }

    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.www.mk_ipc_demo.IConnectionService {
        private static final java.lang.String DESCRIPTOR = "com.www.mk_ipc_demo.IConnectionService";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.www.mk_ipc_demo.IConnectionService interface,
         * generating a proxy if needed.
         */
        public static com.www.mk_ipc_demo.IConnectionService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            //判断当前进程和服务进程是否为一个，如果是一个，则不需要进行 IPC 通信
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.www.mk_ipc_demo.IConnectionService))) {
                return ((com.www.mk_ipc_demo.IConnectionService) iin);
            }
            //生成 IConnectionService 服务的代理
            return new com.www.mk_ipc_demo.IConnectionService.Stub.Proxy(obj);
        }

        //返回 IBinder 对象
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
             //根据传递过来的 code 来判断要触发那个方法
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_connect: {
                    //校验是不是为同一个 DESCRIPTOR
                    data.enforceInterface(descriptor);
                     //触发具体实现
                    this.connect();
                    return true;
                }
                case TRANSACTION_disconnect: {
                    data.enforceInterface(descriptor);
                    this.disconnect();
                     //disconnect 方法没有被 oneway 标记，所以需要一个返回值
                    reply.writeNoException();
                    return true;
                }
                case TRANSACTION_isConnected: {
                    data.enforceInterface(descriptor);
                    boolean _result = this.isConnected();
                    reply.writeNoException();
                    reply.writeInt(((_result) ? (1) : (0)));
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements com.www.mk_ipc_demo.IConnectionService {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            //返回 IBinder 对象
            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }
			//返回标记
            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            //代理实现的 connect ，主进程调用时会触发的方法
            @Override
            public void connect() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                try {
                    //写入标记，用于在子进程接收到这个 binder 之后能知道这个 Parcel 是来自哪个接口
                    _data.writeInterfaceToken(DESCRIPTOR);
                      //1，标识请求的是哪个方法，2,发送的数据，3,不需要返回数据，不需要等待子进程的回复
     			    //4,是一个标志位 FLAG_ONEWAY，因为我们定义的时候使用了 oneway 关键字
                    boolean _status = mRemote.transact(Stub.TRANSACTION_connect, _data, null, android.os.IBinder.FLAG_ONEWAY);
                    if (!_status && getDefaultImpl() != null) {
                        getDefaultImpl().connect();
                        return;
                    }
                } finally {
                    _data.recycle();
                }
            }

            @Override
            public void disconnect() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    //3，因为没有标记 oneway，所以需要返回数据，所以传入 reply
                    boolean _status = mRemote.transact(Stub.TRANSACTION_disconnect, _data, _reply, 0);
                    if (!_status && getDefaultImpl() != null) {
                        getDefaultImpl().disconnect();
                        return;
                    }
                     // 因为没有标记 oneway，所以需要去读取 IPC 调用的结果，读取过程中可能会出现异常
                    _reply.readException();
                } finally {
                    //回收
                    _reply.recycle();
                    _data.recycle();
                }
            }

            @Override
            public boolean isConnected() throws android.os.RemoteException {
                //发送过去的数据是通过 _data
                android.os.Parcel _data = android.os.Parcel.obtain();
                 //返回回来的数据是通过 _reply
                android.os.Parcel _reply = android.os.Parcel.obtain();
                boolean _result;
                try {
                    //写入标记
                    _data.writeInterfaceToken(DESCRIPTOR);
                    boolean _status = mRemote.transact(Stub.TRANSACTION_isConnected, _data, _reply, 0);
                    if (!_status && getDefaultImpl() != null) {
                        return getDefaultImpl().isConnected();
                    }
                    _reply.readException();
                    //获取返回的数据
                    _result = (0 != _reply.readInt());
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            public static com.www.mk_ipc_demo.IConnectionService sDefaultImpl;
        }

        static final int TRANSACTION_connect = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_disconnect = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
        static final int TRANSACTION_isConnected = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);

        public static boolean setDefaultImpl(com.www.mk_ipc_demo.IConnectionService impl) {
            if (Stub.Proxy.sDefaultImpl == null && impl != null) {
                Stub.Proxy.sDefaultImpl = impl;
                return true;
            }
            return false;
        }

        public static com.www.mk_ipc_demo.IConnectionService getDefaultImpl() {
            return Stub.Proxy.sDefaultImpl;
        }
    }

    public void connect() throws android.os.RemoteException;

    public void disconnect() throws android.os.RemoteException;

    public boolean isConnected() throws android.os.RemoteException;
}
```

​	可以看到他内部有 三个方法，分别对应着 AIDL 中定义的三个方法。还有一个默认的实现类，实现了这三个方法。这个不是重点，重点是 抽象类Stub

​	Stub 就是一个 Binder 类，实现了IConnectionService，当客户端和服务端位于同一个进程时，方法调用不会走跨进程的 transact 方法，当两者唯一不同进程时，方法调用需要走 transact 过程，这个逻辑有 Proxy来完成，在 asInterface 中就会判断是否为同一个进程，如果不是则就会返回 内部的代理类 Proxy

​	下面介绍一下这两个类的方法和含义

**DESCRIPTOR**

​	Binder 的唯一标识，一般用当前类名表示

**asInterface(android.os.IBinder obj)**

​	由客户端调用，用户将服务端的 Binder 对象转为客户端所需要的 AIDL 接口类型的对象。在内部判断了是否Wie同一个进程，如果是同一个进程，返回的就是服务端 Stub 对象本身，否则返回的是系统封装后的 Stub.proxy 对象，这个对象中实现了跨进程的调用

**asBinder**

​	返回当前 Binder 对象

**onTransact**

​	这个方法运行在服务端的 Binder 线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交给此方法处理。该方法的原型为： public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)，服务端通过 code 确定客户端请求的方法是啥，从 data 中取出目标方法所需要的参数(如果目标方法有参数)，然后执行目标方法。目标方法执行完后就向 reply 中写入返回值(如果有返回值)。

​	调用目标方法，这个目标方法其实就是我们在服务端实现的 Stub 的方法

​	onTransact 的执行过程就是这个样子。要注意的是如果此方法返回 false，那么客户端的请求会失败。

**Proxy#isConnected**

​	这个方法执行在客户端，当客户端调用此方法时，内部实现是这样的：首先创建输入型 Parcel 对象_data，输出型 Parcel 对象_  _replay 和 返回值 boolean _result。然后把该方法的参数写入_data 中(如果有参数的话)，接着调用 transact 方法来发起RPC(远程过程调用)请求，同时将当前线程挂起，然后服务端的 onTransact(就是上面的方法) 方法被调用，直到 RPC 过程结束返回后当前线程继续执行，并从 _replay 中取出 RPC 过程的返回结果。最后返回 ——reply 中的数据

**Proxy#connect**

​	这个方法执行在客户端，执行过程和 isConnected 是一样的。只不过 connect 没有返回值，所以不用从_reply 中取出返回值。

**需要注意的地方：**

- 发起远程请求时，当前线程会被挂起，直至服务端进程返回数据，所以这个方法时很耗时的，尽量不要在 UI 线程中发起远程请求。当然可以使用 oneway 关键字 ，使用后当前线程不会进行挂起，而是继续往下执行，但是使用 这个关键字的方法不能有返回值
- 由于服务的的 Binder 方法运行在 Binder 的线程池中，所以 Binder 方法不管是否耗时都应该采用同步的方式去实现。

**执行流程**

​	客户端 通过  asInterface 拿到 AIDL 的接口类型的对象，并且内部判断是否为同一个进程。接着使用 接口类型的对象调用 AIDL的方法，如果是不同的进程就会调用的代理 Proxy ，然后创建 发送数据和接收数据(根据方法的需求)，然后调用 transact 方法发起远程调用请求，当前线程挂起，等待远程带调用结果。拿到结果后最终返回给客户端

![image-20200408170342272](IPC%20%E4%B9%8B%20AIDL%20%E5%8E%9F%E7%90%86.assets/image-20200408170342272.png)

