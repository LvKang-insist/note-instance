Service 和Activity 一样都是android的四大组件之一，他们都有各自的生命周期，想要理解和使用Service就必须熟悉他的生命周期，知道在是啥时候会回调什么方法。在Service中，有两种方式启动Service。分别是startService()和bindService()这两种方式。

直接看一下bindService()启动服务。  


1,bingService()是和活动绑定在一起的，通过bindService开启的服务都可以在活动中控制，


```
public class MyService extends Service {
    
    private DownloadBinder downloadBinder = new DownloadBinder();

    class DownloadBinder extends Binder{
        public void start(){
            Log.d("TAG", "start: ");
        }
        public void stop(){
            Log.d("TAG", "stop: stop");
        }

    }
    @Override
    public IBinder onBind(Intent intent) {
        return downloadBinder;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Log.e("MyService", "onCreate: " );
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.e("MyService", "onStartCommand: " );
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public void onDestroy() {
        Log.e("MyService", "onDestroy: " );
        super.onDestroy();
    }
}

```


2，在活动中启动服务：

```
public class MainActivity extends AppCompatActivity {

    private MyService.DownloadBinder downloadBinder;


    //重写的两个方法会在活动与服务绑定及服务连接断开的时候调用
    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            downloadBinder = (MyService.DownloadBinder) service;
            downloadBinder.start();
            downloadBinder.stop();
        }
        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);


        findViewById(R.id.start_service).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //将服务和当前Activity关联起来
                Intent intent = new Intent(MainActivity.this, MyService.class);
                bindService(intent, connection, BIND_AUTO_CREATE);
            }
        });

        findViewById(R.id.stop_service).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //解绑服务
                unbindService(connection);
            }
        });
    }
}

```
通过bindService启动服务后，通过重写ServiceConnection类的方法就可以拿到服务里面类的对象，有了这个对象，活动和服务之间的关系就非常紧密了。我们可以在活动中使用服务中的类的对象，即实现了指挥服务干什么服务就会干什么的功能，


服务的生命周期：

一旦在任何地方调用了Context的startService()方法，则相应的服务就会启动起来，并回调onStartCommand()方法，如果这个服务没有被创建过，onCreate()方法就会先于onStartCommand()方法执行，服务启动了之后会一直爆出运行状态，直到stopService()或者stopSelf()方法被调用，



通过Context的bindService()来获取一个服务的持久连接，这时就会回调服务中的onBind()方法。 如果这个服务之前没有被创建过，onCreate()方法会先于onBind()方法执行，之后，调用方可以获取到onBind()方法返回的Ibinder对象的实例。这样就可以自由的和服务进行通信了，只要调用方和服务之间的连接没有被断开，服务就会一直保持运行状态。