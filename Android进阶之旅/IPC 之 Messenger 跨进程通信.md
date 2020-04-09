​		使用 Messenger 进行 进程之间的通信，使用起来非常简单。Messenger 是一种轻量级的 IPC 方案，它是对 IPC 做了一层简单的封装，下面看一下使用，最后在分析一下源码

#### 1，首先看一下服务端的代码

```kotlin
class MessengerService : Service() {
    
    @SuppressLint("HandlerLeak")
    val handler = object : Handler() {
        override fun handleMessage(msg: Message) {
            val bundle = msg.data
            val string = bundle.getString("message")
            Toast.makeText(
                this@MessengerService,
                "${android.os.Process.myPid()} -- $string", Toast.LENGTH_LONG
            ).show()

            //三秒后返回消息给客户端
            //获取主进程发送的 messenger，并且给主进程发送消息
            val clientMessenger = msg.replyTo
            postDelayed({
                val message = Message.obtain()
                val clientBundle = Bundle()
                clientBundle.putString("clientMessage", "我是子进程返回的消息")
                message.data = clientBundle
                clientMessenger.send(message)
            }, 3000)
        }
    }

    //创建 messenger 对象，并传入 handler 
    private val messenger = Messenger(handler)

    override fun onBind(intent: Intent?): IBinder? {
        //返回 messenger 的binder
        return messenger.binder
    }

}
```

```xml
<!--  子进程的Service  -->
<service
    android:name=".MessengerService"
    android:process=":message" />
```

​	上面的代码非常简单，就是使用handler接收消息，接收完成的三秒后将数据发送出去。

​	在 onBind 中返回了 messenger 的对象

​	最后在 清单中注册，指定为私有进程

#### 2，客户端代码

```kotlin

override fun onCreate(savedInstanceState: Bundle?) {
	 initMessengerService()
}
private var messengerProxy: Messenger? = null

//服务端传递过来的消息
private val clientMessenger = Messenger(object : Handler(Looper.myLooper()) {
    override fun handleMessage(msg: android.os.Message) {
        val data = msg.data
        val string = data.getString("clientMessage")
        Toast.makeText(
            this@MainActivity,
            "${android.os.Process.myPid()} -- $string", Toast.LENGTH_LONG
        ).show()
    }
})

private fun initMessengerService() {
    //Button 点击事件
    messenger.setOnClickListener {
        val message = android.os.Message()
        val bundle = Bundle()
        bundle.putString("message", "我是主线程通过 Messenger 发送的消息")
        message.data = bundle
        //把客户端用于接收服务端数据的 clientMessenger 传递过去，
        //服务端通过 clientMessenger 给客户端传递消息
        message.replyTo = clientMessenger
        //给子进程发送消息
        messengerProxy?.send(message)
    }

    bindService(Intent(this, MessengerService::class.java), object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            messengerProxy = Messenger(service)
        }

        override fun onServiceDisconnected(name: ComponentName?) {
        }
    }, Context.BIND_AUTO_CREATE)
}
```

​	使用 bindService 的方式启动服务端，并且拿到了服务端的 messenger

​	接着使用 messengerProxy 发送消息到 子进程

​	在发送的时候通过 replyTo 发送了一个新的 Messenger 对象，用来从服务端返回数据

​	运行程序，点击 button，服务端就会弹出消息内容，并且显示当前进程 id，在三秒后返回消息到主进程

------

​	通过上面的程序可以看出，通过 Messenger 可以进行进程间的通信

​	如果是单项通信，只需要在服务端创建 Messenger 在 onBind 中返回给客户端即可

​	如果是双向，就需要在客户端发送消息的时候**附带一个 Messenger** 对象，用来从服务端返回消息到客户端

------

### 源码解析

​	其实 messenger 的底层实现就是 Handler + message + AIDL 来实现的，下面看一下 Messenger 的源码

```java
public final class Messenger implements Parcelable {
    private final IMessenger mTarget;
    
     public Messenger(Handler target) {
        mTarget = target.getIMessenger();
    }
    
     public Messenger(IBinder target) {
        mTarget = IMessenger.Stub.asInterface(target);
    }
    
    public void send(Message message) throws RemoteException {
        mTarget.send(message);
    }
    
    public IBinder getBinder() {
        return mTarget.asBinder();
    }
}
```

​	首先看一下构造方法，接收一个 handler ，并且通过 handle 获取到了一个 Imessager 的对象，这个 Imessenger 到底是啥呢？打开 handler 的源码，找到 getImessenger 看一哈：

```kotlin
final IMessenger getIMessenger() {
    synchronized (mQueue) {
        if (mMessenger != null) {
            return mMessenger;
        }
        //创建一个新的 MessengerImpl
        mMessenger = new MessengerImpl();
        return mMessenger;
    }
}

private final class MessengerImpl extends IMessenger.Stub {
    public void send(Message msg) {
        msg.sendingUid = Binder.getCallingUid();
        Handler.this.sendMessage(msg);
    }
}
```

​	注意看 MessengerImpl 继承的类，如果你使用过 AIDL 就会非常熟悉，这个类就是我们需要在服务端实现的呀。但是在 handler 中已经帮我们做了实现。

​	我们在服务端返回 binder 的时候调用的 getBinder，返回一个 Stub 实现类的 binder。**是不是和 AIDL 中的写法完全一致呢**？

​	在客户端拿到这个 binder 后 就会通过 Messenger 中的第二个构造方法将 服务端binder 转为 AIDL 的接口类型的对象。**是不是和AIDL 中的写法完全一致呢**？

​	最终就可以在客户端通过 Messenger 的send 方法发送消息了。这个消息就直接发送到服务端 handler 中 继承 Stub 的 MessengerImpl 的 send 方法中，也就是上面这个类。在 MessengerImpl 的 send 方法中就会 通过 handler 直接发送出去，最终就会被分发到 handleMessage 方法中。

​	