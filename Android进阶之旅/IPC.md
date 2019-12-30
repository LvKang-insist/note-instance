IPC 简介

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
第二种 是全局进程，其他音乐可以通过 ShareUID 的方式和他跑在同一个进程中。 

Android系统会为每一个应用分配一个 UID ，具有相同UID 的应用才能共享数据，两个应用通过ShareUID 跑在同一个进程中是有要求的
需要这两个应用具有相同的ShareUID 并且签名相同才可以，在这种情况下，他们可以互相访问对方的私有数据，比如data目录，组件信息等，
不管他们是否跑在同一个进程中。如果过他们跑在同一进程中，还可以共享内存数据，他们看起来就像是一个应用的两个部分。

android 系统会为每个进程分配一个独立的虚拟机，不同地虚拟机在内存分配上有不同的地址空间，所以早期不同的虚拟机中访问同一个类的对象会产生多个副本。

使用多进程容易造成一下几个问题
1，静态成员和单例完全失效

2，线程同步机制完全失效：无论锁对象还是全局对象。

3，SharedPrefrences 可靠性下降

4，Application 会多次创建

![1577696338343](../android/%E6%9D%82/IPC.assets/1577696338343.png)

​	如上面图片，启动多个进程后，Application 会多次执行，而且 as 上面显示有三个进程，通过试验，两个进程之间的静态数据不会被干扰，相当于把原来进程中的数据重新拷贝了一份。修改当前进程，别的进程数据也不会改变，尽管他是静态属性。

------

IPC 基础，主要包含三个方面:

1，Serializable 接口

​		java 提供的序列化接口。可以为对象提供标准的 序列化和反序列化。使用也很简单：

2，Parcelable 接口

​		Android 提供过的序列化接口，只要实现这个接口，类对象就可以通过 Intent 和 Binder 传递。

3，Binder

​			直观上来说：Binder 是Android 中的一个类，实现了 IBinder 接口。

​			从 IPC 角度来说：Binder 是 Android 中一种跨进程通信的方式。

​			Binder 还可以理解为一种虚拟的物理设备，他的设备驱动是 /dev/binder。该通信方式在 Linux 上没有

​			从 AndroidFramework 角度来说：Binder 是 ServiceManager 连接各种 Manager(ActivityManager，WindowManager，等等) 和相应 ManagerService 的桥梁

​			从 Android 应用层来说：Binder 是客户端 和 服务端进行通信的媒介，当 binderService 的时候，服务端会返回一个包含了服务端业务调用的 Binder 对象，通过这个 Binder 对象，客户端就可以获取服务器提供的服务或者数据，这里的服务包括普通服务和基于 ALDL 的服务。



​	