本地服务（LocalService）

调用者 和 Service 在同一个进程里，所以运行在主进程的 mian 线程中，所以不能进行耗时操作。可以采用在service 里面创建 一个 Thread 来执行 任务，service 影响的是 进程 的 生命周期。

任何 Activity 都可以控制 同一 Service。而系统 只会创建 一个 对应的Service 的实例。

两种启动方式

​	1，通过start 方式 开启服务。

​		使用步骤：定义一个类继承Service，

​				  manifest 中配置 service 

​				  在 使用 context 的 startService( Intent ) 启动Service

​				  不在 使用时，调用 stopService( Intent ) 停止服务

​		 生命周期：

​				  onCreate --> onStartCommand --> onDestory

​			注意：如果服务已经开启，不会重复调用 onCreate 方法，如果再次调用 startService方法。service 而是会  			

​			调用onStart 或者 onStartCommand() 方法，停止服务时 会回调 onDestory 方法。

 		 特点：一旦开启服务就根调用者 没有任何关系了，开启者挂了，服务不会挂。开启者不能调用服务里面的方法

​	2，通过 bind 方式 开启服务

​		使用步骤：定义一个类继承 Service

​				  在 manifest 中 注册 service

​				  使用 context 的bindService （Intent , ServiceConnection ,int ） 方法启动服务

​				  不在使用，调用 unbindService （ServiceConnection）方法停止服务

​		生命周期：

​				  onCreate --> onBind -- onUnbind --> onDestory 

 		注意：绑定服务不会调用 onStart 或者 onStartCommand 方法

​		特点：bind 的方式开启服务，绑定服务，调用者挂了，服务也会挂，绑定着可以调用服务里面的方法。

​	示例：

​	创建Service

```java
public class MyService extends Service {
    @Override
    public void onCreate() {
        super.onCreate();
    }
    @Override
    public IBinder onBind(Intent intent) {
       return new BindManager();
    }

    @Override
    public boolean onUnbind(Intent intent) {
        return super.onUnbind(intent);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
    }
    private void MyMethod(){
        Log.e("Service", "MyMethod: " );
    }
    public class BindManager extends Binder {
        public void printServiceMy(){
            MyMethod();
        }
    }
}
```

绑定 MyService 并 调用 Service 中的方法

```java
 private MyConn conn;
 private MyService.BindManager bindManager;


class MyConn implements ServiceConnection {

    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        bindManager = (MyService.BindManager) service;
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {

    }
}
//绑定服务
 conn = new MyConn();
bindService(new Intent(MainActivity.this,MyService.class), conn,BIND_AUTO_CREATE);

//调用服务中的方法
 if (bindManager != null){
     bindManager.printServiceMy();
 }

```

