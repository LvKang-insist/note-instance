### IPC 简介

IPC 的含义为进程间通信或者跨进程通信，是指两个进程之间数据交换的过程
	多进程的情况分为两种:
	1，一个应用因为某些原因自身需要采用多进程模式来实现
	2，当前应用需要向其他应用获取数据
	
Android 中的多进程模式
	正常情况下，在Android中多进程是指一个应用中存在多个进程的情况
	在android 中使用多进程的方法：
	1，给四大组件在AndroidMenifest中指定android:process属性（我们无法给一个线程或一个实体类指定其运行时所在的进程）
	2，非常规方法：通过JNI在native层去fork一个新的进程
	

第一种情况：默认进程的名子为程序包名。如果需要指定，有两种方式。
android:process=":remote"
android:process="com.ryg.chapter_2.remote"

两种区别：
第一种 “:”的含义是在当前进程名前附加上当前的包名，是一种简洁的写法
第二种 是完整的命名方式
第一种 是私有进程，其他应用组件不可以和它跑在同一个进程中
第二种 是全局进程，其他应用可以通过 ShareUID 的方式和他跑在同一个进程中。 

Android系统会为每一个应用分配一个 UID ，具有相同UID 的应用才能共享数据，两个应用通过ShareUID 跑在同一个进程中是有要求的。需要这两个应用具有相同的ShareUID 并且签名相同才可以，在这种情况下，他们可以互相访问对方的私有数据，比如data目录，组件信息等，不管他们是否跑在同一个进程中。如果过他们跑在同一进程中，还可以共享内存数据，他们看起来就像是一个应用的两个部分。

android 系统会为每个进程分配一个独立的虚拟机，不同地虚拟机在内存分配上有不同的地址空间，所以早期不同的虚拟机中访问同一个类的对象会产生多个副本。

使用多进程容易造成一下几个问题
1，静态成员和单例完全失效

2，线程同步机制完全失效：无论锁对象还是全局对象。

3，SharedPrefrences 可靠性下降

4，Application 会多次创建

![1577696338343](../android/%E6%9D%82/IPC.assets/1577696338343.png)

​	如上面图片，启动多个进程后，Application 会多次执行，而且 as 上面显示有三个进程，通过试验，两个进程之间的静态数据不会被干扰，相当于把原来进程中的数据重新拷贝了一份。修改当前进程，别的进程数据也不会改变，尽管他是静态属性。

------

### IPC 基础，主要包含三个方面:

#### 1，Serializable 接口

​		java 提供的序列化接口。可以为对象提供标准的 序列化和反序列化。[使用也很简单](https://blog.csdn.net/baidu_40389775/article/details/89096527)

#### 2，Parcelable 接口

​		Android 提供过的序列化接口，只要实现这个接口，类对象就可以通过 Intent 和 Binder 传递。

```kotlin
class Book(userId: Int, bookName: String) : Parcelable {

    var id: Int = userId

    var bookName: String? = bookName


    /**
     * 将当前对象写入序列化结构中，通过一系列的 write 方法完成
     * 其中 flags 有两种值：0 或 1，
     * <p> 为 1 时标识当前对象需要作为返回值返回，不能立即释放资源
     * 几乎所有情况都返回 0
     */
    override fun writeToParcel(parcel: Parcel, flags: Int) {
        parcel.writeInt(id)
        parcel.writeString(bookName)
    }

    /**
     * 返回当前对象的描述，如果含有文件扫描符，返回1，否则返回0
     * 几乎所有情况都返回 0
     */
    override fun describeContents(): Int {
        return 0
    }

    /**
     * 反序列化
     */
    companion object CREATOR : Parcelable.Creator<Book> {

        /**
         * 从序列化后对象中创建原始对象
         */
        override fun createFromParcel(source: Parcel): Book {
            return Book(source.readInt(),source.readString()!!)
        }
        override fun newArray(size: Int): Array<Book?> {
            /**
             * 创建指定长度的原始对象数组
             */
            return arrayOfNulls(size)
        }
    }
}
```

#### 3，Binder

​			直观上来说：Binder 是Android 中的一个类，实现了 IBinder 接口。

​			从 IPC 角度来说：Binder 是 Android 中一种跨进程通信的方式。

​			Binder 还可以理解为一种虚拟的物理设备，他的设备驱动是 /dev/binder。该通信方式在 Linux 上没有

​			从 AndroidFramework 角度来说：Binder 是 ServiceManager 连接各种 Manager(ActivityManager，WindowManager，等等) 和相应 ManagerService 的桥梁

​			从 Android 应用层来说：Binder 是客户端 和 服务端进行通信的媒介，当 binderService 的时候，服务端会返回一个包含了服务端业务调用的 Binder 对象，通过这个 Binder 对象，客户端就可以获取服务器提供的服务或者数据，这里的服务包括普通服务和基于 ALDL 的服务。

Binder 架构

​	<img src="IPC.assets/image-20200407133628641.png" alt="image-20200407133628641" style="zoom: 80%;" />

首先是 AIDL ，在 android 中通过编写 AIDL 去基于 Binder 去提供**对外的接口和实现**，然后通过 AIDL 文件调用到 对应的java 层，然后通过 java 的 JNI 调用到 Native 层，最后是调用到 内核

#### 4，AIDL

- AIDL 是定义 IPC 过程中接口的一种描述语言
- AIDL 文件在编译过程中生成接口的实现类，用于 IPC 通信
- 支持基本数据类型，实现了 Parcelable 接口的对象，List，Map

#### 5，Messenger

- 基于 Handler ，Message 实现
- 串行实时通信
- 传输 Bundler 支持的数据类型

### 实战

解决问题：

- AIDL 如何实现 IPC
- in ，out，inout 关键字的作用
- oneway 关键字的作用
- AIDL 如果实现 callback
- 如果自己编码实现 AIDL 的核心功能

项目场景

在子进程中有一个链接服务 和 消息服务

- 连接服务：connect (建连)，disconnect(断连)，isConnected(链接状态)

- 消息服务：sendMessage(发送消息)，registerMessaageReceiverListener(在主进程中监听子进程的消息)，unRegisterMessageReceiverListener(注销消息的监听)

#### 1，创建子进程的 Service

```kotlin
/**
 * 管理和提供子进程的连接和消息服务
 */
class RemoteService : Service() {
    override fun onBind(intent: Intent?): IBinder? {
        return null
    }
}
```

在清单文件中进行注册，并指定在私有的子线程

```xml
<!--  子进程的Service  -->
<service
    android:name=".RemoteService"
    android:process=":remote" />
```

接着在 MainActivity 中启动这个服务

```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        //启动子进程的 Service
        bindService(Intent(this, RemoteService::class.java), object : ServiceConnection {
            override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            }

            override fun onServiceDisconnected(name: ComponentName?) {
            }

        }, Context.BIND_AUTO_CREATE)
    }
}
```

运行程序后就会发现子进程已经启动了

#### 2，完成连接服务

​	右击 module ，选择 new 中的 AIDL ，创建一个名字为 IConnectionService.aidl 的文件，并定义三个方法，也就是上面我们所说的三个方法，如下所示：

```kotlin
/**
 * 连接服务
 */
interface IConnectionService {

    void connect();

    void disconnect();

    boolean isConnected();
}

```

有几点需要注意一下： 

​		文件创建完成后，你就会发现他会自动在 main 文件夹下创建 aidl 文件夹，并且里面和 java 下的包名是一样的。

​		这个接口中**不能有任何注释**，否则会编译不通过，切记

​		点一下锤子，进行编译，看一下生成的实现类和目录结构：

<img src="IPC.assets/m.png" alt="m" style="zoom:50%;" />

接着我们来完成一下 处于子进程中的 RemoteService 

```kotlin
/**
 * 管理和提供子进程的连接和消息服务
 */
class RemoteService : Service() {

    /**
     * 连接状态
     */
    private var isConnected: Boolean = false
    
    /**
     * Stub 是实现类中的抽象类
     * 这三个方法就是 aidl 文件中的三个方法，在这里进行实现
     */
    val connectionService =
        object : IConnectionService.Stub() {
            //当主进程调用 connect 时，这里个 connect 就会执行
            override fun connect() {
                //模拟连接耗时
                Thread.sleep(5000)
                this@RemoteService.isConnected = true
                Log.e(
                    "connect",
                    "${android.os.Process.myPid()}  --- ${Thread.currentThread().name}"
                )
                GlobalScope.launch(Dispatchers.Main) {
                    Toast.makeText(this@RemoteService, "connect", Toast.LENGTH_LONG).show()
                }
            }

            //当主进程调用 disconnect 时，这里个 disconnect 就会执行
            override fun disconnect() {
                this@RemoteService.isConnected = false
                Log.e(
                    "connect",
                    "${android.os.Process.myPid()}  --- ${Thread.currentThread().name}"
                )
                //切换到主线程
                GlobalScope.launch(Dispatchers.Main) {
                    Toast.makeText(this@RemoteService, "disconnect", Toast.LENGTH_LONG).show()
                }
            }

            override fun isConnected(): Boolean {
                return this@RemoteService.isConnected
            }

        }

    override fun onBind(intent: Intent?): IBinder? {
        return connectionService.asBinder()
    }
}
```

​		我们实现了 AIDL 实现类中的抽象类，并且实现了三个方法，这三个方法就是我们在接口中定义的方法。

​		接着在 onBind 中 将他转为了 Binder 进行返回，这样在主进程中就可以进行调用了。如下

修改 MainActivity ：

```kotlin
class MainActivity : AppCompatActivity() {

    private var connectionServiceProxy: IConnectionService? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        //启动子进程的 Service
        initService()
        
        //三个 button 
        connect.setOnClickListener {
            connectionServiceProxy?.connect()
        }
        dis_connect.setOnClickListener {
            connectionServiceProxy?.disconnect()
        }
        is_connect.setOnClickListener {
            val isconnected = connectionServiceProxy?.isConnected
            Toast.makeText(this, "$isconnected", Toast.LENGTH_LONG).show()
        }
    }

    private fun initService() {
        bindService(Intent(this, RemoteService::class.java), object : ServiceConnection {
            override fun onServiceConnected(name: ComponentName?, service: IBinder) {
                //拿到 子进程中 onBind 返回的值
                connectionServiceProxy = IConnectionService.Stub.asInterface(service)
            }

            override fun onServiceDisconnected(name: ComponentName?) {
            }

        }, Context.BIND_AUTO_CREATE)
    }
}
```

​		首先是启动了 RemoteService，然后拿到了 onBinde 中返回的值。

​		接着在 点击事件中进行了调用，观察打印的日志，如下：

```
 3741-3764/com.www.mk_ipc_demo:remote E/connect: 3741  --- Binder:3741_2
 3741-3764/com.www.mk_ipc_demo:remote E/connect: 3741  --- Binder:3741_2
```

​		对比进程 id，可以看出是执行在 子进程中，并且还是执行在子线程中，正因为如此，所以上面在 Toast 的时候转到了主线程

​		不知道你有没有注意到，当你点击连接的按钮后，按钮的状态在 5秒后才会恢复，如果没注意到赶紧去看一下。这是因为 connect 方法时阻塞的，只有等到子进程将这个方法执行完成后 主进程中的主线程才会继续往下执行。

​		那这种情况有没有什么办法解决呢？如下

- 直接在子线程中调用 connect 方法即可。

- 使用 **oneway** 关键字，如下：

  ```java
  interface IConnectionService {
     oneway void connect();
     .......
  }
  ```

  然后重写跑一些代码，就会发现按钮的状态直接恢复，不会在进行阻塞了。

  有一点需要注意：使用 oneway 后，该方法不能有任何的返回值，否则会报错

**通过上面的代码，我们就实现了通过 AIDL 实现了连接服务，在主进程和子进程之间的 IPC 调用。**

