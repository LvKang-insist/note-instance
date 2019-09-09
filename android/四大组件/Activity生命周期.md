![img](F:\笔记\android\Activity\assets\682504-1405607172778d9b.gif) 

生命周期：

onCreate()  ——> onStart() ——>onResume() ——> onPause() ——> onStop() ——> onDestroy()



1，启动 activity：系统首先调用 onCreate() 然后调用onStart ，最后调用onResume，最后activity 进入了运行状态

2，activity 被其他 activity 覆盖，dialog 或者锁屏，系统就会调用 onPause 方法，暂停活动。

3，activity 由覆盖的状态 回到前台，或者解锁，系统则会调用 onResume 方法，再次进入运行状态。

4，当前activity 转到其他的activity 界面或者按 home 返回主屏，则系统会调用 onPause ，进入停滞状态。

5，用户回退到此 activity 系统会先调用 onRestart 方法，然后调用 onstart 方法，最后调用 onResume 方法 进入运行状态。

6，当activity 被覆盖 或者  处于后台 ，即 第 2，4 步，系统内存不足，杀死当前 activity ，而后用户退回 当前 activity ，再次调用 onstart ，onCreate，onResume。进入运行状态。



- onRestart ：表示当前 activity 正在重新启动，一般情况下，当前activity 从不可见转为可见时，onRestart 会被调用，这种情况一般是 由用户行为导致的。如  按下 home 到桌面，然后重新打开 app 或者按 back 键。
- onStart ： activity 可见了， 但是还没有出现在 前台，无法和用户进行交互。
- onPause ：表示当前 activity 正在 停止，此时可以做一些存储的工作，停止动画等。注意不能太耗时，因为这将影响到新 activity 的显示，onPause 必须先执行完，新的 activity 的onResume 才会执行。



**从 activity 是否可见来说， onstart 和 onstop 是配对的， 从 activity 是否在前台来说 onResume 和 onPause 是配对的。**

**旧的 activity 先 onPause ，新的 activity 在启动。**

**注意：当 activity 弹出 dialog 的时候 ，activity不会 回调 onPause 。然而当 activity 启动 dialog 风格的 activity 的时候，此时 activity 会调用 onPause。**

dialog风格的 activity ：[dialog 风格的 activity](https://blog.csdn.net/cswhale/article/details/86596943 )



异常情况下的 生命周期：比如系统资源配置 发生改变 以及 系统内存不足时， activity 有可能被杀死。

- 情况1，资源相关的 系统配置 发生改变调至 activity 被杀死并重新 创建。比如当前 activity 处理竖屏 状态，如果突然旋转屏幕，由于系统配置发生了改变。在默认情况下，activity 就会 被销毁并且重新创建。当然我们也可以 组织系统重新创建 activity。

  ​	

  系统 配置发生改变后，activity 会销毁， 其中 onPause ，onstop onDestory 均会被 调用，由于 activity 是在异常情况下终止的 ，系统 会调用 onSaveInstanceStart 来保存当前 activity 的状态，这个方法的调用时机 是在 onStop 之前。

  当activity 重新创建 后 ，系统会调用 onRestoreInstanceState ，并 把 activity销毁 时 onSaveInstanceStart 方法保存的 Bundle 对象作为 参数 传递给 onRestoreInstance 方法 和 onCreate 方法。

  

  恢复 数据 [activity 被回收了怎么办](https://blog.csdn.net/baidu_40389775/article/details/86707585)

-  情况2，资源内存导致的优先级低的 activity 被杀死。

  这里的情况 和 前面的情况1 数据的存储和恢复是完全一致的。activity 按照优先级 高低 可以分为 如下 三种：

   1，前台 activity ----正在和用户交互的 activity，优先级 最高。

   2，可见 但是 非前台的 activity -----比如activity中 弹出了 一个 对话框。导致 activity可见，但是无法进行交互。

   3，后台 activity -----已经被暂停的 activity，比如执行了 onStop ，优先级 最低。



Activity 和 Fragment生命周期的关系：

创建：

```java
2019-05-24 15:38:56.119 9111-9111/com.admin.design E/Frag:: onAttach: 
2019-05-24 15:38:56.119 9111-9111/com.admin.design E/Frag:: onCreate: 
2019-05-24 15:38:56.120 9111-9111/com.admin.design E/Frag:: onCreateView: 
2019-05-24 15:38:56.132 9111-9111/com.admin.design E/MainActivity:: onCreate: 
2019-05-24 15:38:56.133 9111-9111/com.admin.design E/Frag:: onActivityCreated: 
2019-05-24 15:38:56.134 9111-9111/com.admin.design E/Frag:: onStart: 
2019-05-24 15:38:56.134 9111-9111/com.admin.design E/MainActivity:: onStart: 
2019-05-24 15:38:56.138 9111-9111/com.admin.design E/MainActivity:: onResume: 
2019-05-24 15:38:56.138 9111-9111/com.admin.design E/Frag:: onResume: 
```



销毁：

```java
2019-05-24 15:42:08.803 9242-9242/com.admin.design E/Frag:: onStop: 
2019-05-24 15:42:08.803 9242-9242/com.admin.design E/MainActivity:: onStop: 
2019-05-24 15:42:08.804 9242-9242/com.admin.design E/Frag:: onDestroyView: 
2019-05-24 15:42:08.804 9242-9242/com.admin.design E/Frag:: onDestroy: 
2019-05-24 15:42:08.804 9242-9242/com.admin.design E/Frag:: onDetach: 
2019-05-24 15:42:08.804 9242-9242/com.admin.design E/MainActivity:: onDestroy: 
```



