		EventBus 是一个开源库，他利用 发布/订阅 者模式来对项目进行解耦，他可以利用很少的代码，来实现多组件的通信。android 的多组件通信可以通过 Handler，广播等，但使用他们进行通信 代码量非常多，组件之间容易产生耦合。关于 EventBus 的工作模式如官方的图所示：

### 1，基本使用

- 添加依赖：

  ```java
  compile 'org.greenrobot:eventbus:3.1.1'
  ```

- 定义事件，可以是普通的对象例如：

  ```java
  public class EventMessage {
      private int type;
      private String message ;
  	
  	......
      public EventMessage(int type, String message) {
          this.type = type;
          this.message = message;
      }
  
      @Override
      public String toString() {
  
          return "type="+type+"--message= "+message;
      }
  }
  ```

- 注册订阅之 和 注销订阅者，只有订阅者注册了，他们才会收到时间。在Android 中，可以根据 Activity 或者 Fragment 的生命周期来注册和注销。如：

  ```java
      @Override
      protected void onStart() {
          super.onStart();
          // 注册订阅者
          EventBus.getDefault().register(this);
      }
      
      @Override
      protected void onDestroy() {
          super.onDestroy();
          //注销订阅者
          EventBus.getDefault().unregister(this);
      }
  ```

- 然后就可以发布事件了，在需要的地方发布事件，所有订阅了该事件类型并进行注册的订阅者将会接收到该事件。如：

  ```java
   // 发布事件
   EventMessage msg = new EventMessage(1,"main2Activity");
   EventBus.getDefault().post(msg);
  ```

- 最后还需要一个接受事件的地方，也就是订阅事件，订阅者需要定义事件的处理方法(也称之为 订阅之方法)，当发布对应事件类型时，该方法被调用。 EventBus3 采用 @Subscribe 注解来定义订阅者方法。方法可以是任意的合法方法名，参数类型为订阅事件的类型。如：

  ```java
    /**
       * 订阅事件
       * @param message 
       */
      @Subscribe(threadMode = ThreadMode.MAIN)
      public void onReceiveMsg(EventMessage message){
          Log.e("-------", "onReceiveMsg: "+message );
      }
  ```

- 下面看一个例子

  ```java
  public class MainActivity extends AppCompatActivity {
  
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_main);
      }
  
      @Override
      protected void onStart() {
          super.onStart();
          // 注册订阅者
          EventBus.getDefault().register(this);
      }
  
      @Override
      protected void onResume() {
          super.onResume();
          new Thread(new Runnable() {
              @Override
              public void run() {
                  try {
                      Thread.sleep(3000);
                      runOnUiThread(new Runnable() {
                          @Override
                          public void run() {
                              startActivity(new Intent(MainActivity.this,Main2Activity.class));
                          }
                      });
                  } catch (InterruptedException e) {
                      e.printStackTrace();
                  }
              }
          }).start();
      }
  
      @Override
      protected void onDestroy() {
          super.onDestroy();
          //注销订阅者
          EventBus.getDefault().unregister(this);
      }
  
      /**
       * 订阅者方法
       * @param message 参数
       */
      @Subscribe(threadMode = ThreadMode.MAIN)
      public void onReceiveMsg(EventMessage message){
          Log.e("-------", "onReceiveMsg: "+message );
          Toast.makeText(getApplicationContext(), "哈哈", Toast.LENGTH_SHORT).show();
      }
  }
  ```

  ```java
  public class Main2Activity extends AppCompatActivity {
  
      @Override
      protected void onCreate(Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_main2);
  
          findViewById(R.id.main2_btn).setOnClickListener(new View.OnClickListener() {
              @Override
              public void onClick(View view) {
                  // 发布事件
                  EventMessage msg = new EventMessage(1,"main2Activity");
                  EventBus.getDefault().post(msg);
              }
          });
      }
  }
  
  ```

   当 MainActivity 启动后的三秒 ， 然后启动Main2Activity，点击按钮发送事件，然后 MainActivity 中的订阅事件的方法就会得到执行。

  结果如下：

  ```java
  -------: onReceiveMsg: type=1--message= main2Activity
  ```

### 2，线程模式

​		EventBus 支持订阅者方法在不同的发布事件所在线程中被调用，你可以在订阅事件的时候在注解中指定这个方法所执行的线程！

- ThreadMode.POSTING 

  订阅者发布的事件将会在所在的线程调用，这是默认的模式。时间的传递是同步的，一旦发布事件，所有对该模式 的订阅事件都会执行。这种线程模式意味着最少的性能开销，因为它避免了线程的切换。因此，对于不要求是主线程并且耗时很短的简单任务推荐使用该模式。使用该模式的订阅者方法应该快速返回，以避免阻塞发布事件的线程，这可能是主线程。

- ThreadMode.MAIN

  订阅方法在主线程（UI）调用 ，因此可以直接更新 UI ，如果发布事件的线程是主线程，那么该模式的订阅者方法将被直接调用。使用该模式的订阅者方法必须快速返回，以避免阻塞主线程。

- ThreadMode.MAIN_ORDERED

  订阅方法在主线程（UI）调用，因此可以直接更新UI，事件将先进入队列然后才发送给订阅者，所以发布事件的调用将立即返回。这使得事件的处理保持严格的串行顺序。使用该模式的订阅者方法必须快速返回，以避免阻塞主线程。

- ThreadMode.BACKGROUND 

  订阅者方法 将在 后台线程中被调用，如果发布事件的线程不是主线程，那么订阅者方法将会直接在该线程被调用。如果是主线程，那么将会使用一个单独的后台线程，该线程将按顺序发送所有的事件。使用该模式的订阅者方法应该快速返回，以避免阻塞后台线程。

- ThreadMode.ASYNC 

  订阅者方法将在一个单独的线程中被调用。因此，发布事件的调用者将立即返回。。如果订阅者方法的执行需要一些时间，例如网络访问，那么就应该使用该模式。避免触发大量的长时间运行的订阅者方法，以限制并发线程的数量。EventBus使用了一个线程池来有效地重用已经完成调用订阅者方法的线程。

```java
 E/MainActivity: onMessageMain(), current thread is main
 E/MainActivity: onMessagePosting(), current thread is main
 E/MainActivity: onMessageBackground(), current thread is pool-1-thread-2
 E/MainActivity: onMessageAsync(), current thread is pool-1-thread-1
 E/MainActivity: onMessageMainOrdered(), current thread is main
```

​	在主线程发布事件，调用如上所示

3，粘性事件

​		如果先发布了事件，然后有订阅者订阅了事件，那么除非再次发布该事件，否则订阅者将永远接收不到该事件。此时可以使用粘性事件。发布一个粘性事件后，EventBus 将在内存中缓存该粘性事件。当有订阅者订阅了该粘性事件，订阅者将接收到该事件

​		下面看一个例子：

```java
public class MainActivity extends AppCompatActivity {
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        findViewById(R.id.tv).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                // 发布事件
                EventMessage msg = new EventMessage(1,"mainActivity");
                EventBus.getDefault().postSticky(msg);
                Toast.makeText(MainActivity.this, "发布粘性事件", Toast.LENGTH_SHORT).show();

                startActivity(new Intent(MainActivity.this,Main2Activity.class));
            }
        });
    }
}
```

```java
public class Main2Activity extends AppCompatActivity {

    private static final String TAG = "Main2Activity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main2);

        // 注册订阅者
        EventBus.getDefault().register(this);
    }

    @Subscribe(threadMode = ThreadMode.POSTING ,sticky =true)
    public void onMessage(EventMessage event) {
        Toast.makeText(this, "收到粘性事件", Toast.LENGTH_SHORT).show();
        Log.e(TAG, "onMessagePosting(), current thread is " + Thread.currentThread().getName());
        //移除粘性事件
        EventBus.getDefault().removeStickyEvent(event);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        //注销订阅者
        EventBus.getDefault().unregister(this);
    }
}
```

​	在上面 MainActivity 中，发布了一个粘性事件，注意发送的时候订阅者没有订阅，也没有订阅者方法，发布后直接跳转到 Main2Activity 中，然后在进行注册，定义订阅者方法。最后这个订阅者方法就会得到执行。

​	总的来看，粘性事件就是在订阅者未注册的时候发布事件，然后在订阅之后还可以接收到的事件。

### 3，事件优先级

​	EventBus 支持在订阅者方法中指定事件传递的优先级，默认情况下，订阅者方法的时间传递优先级为 0，数值越大，优先级越高。在相同的线程模式下，更高的优先级订阅者方法将优先接收到事件。**注意：优先级只有在相同的线程模式下才有效。**

​	如下订阅者方法：

```java
 @Subscribe(threadMode = ThreadMode.POSTING ,sticky =true,priority = 2)
    public void onMessage(EventMessage event) {
        Log.e(TAG, "onMessagePosting(), current thread is " + Thread.currentThread().getName());
        EventBus.getDefault().removeStickyEvent(event);
}
```

​	在上面这个订阅者方法中，制定了优先级为 2。

​	你也可以在优先级高的订阅者方法接收到时间后取消事件的传递。此时，低优先级的订阅者方法将不会收到该事件。**注意：订阅者方法只有在线程模式为 ThreadMode.POSTING 时，才可以取消一个事件的传递。如下**	

```java
@Subscribe(threadMode = ThreadMode.POSTING ,sticky =true,priority = 2)
    public void onMessage(EventMessage event) {
        //取消事件传递
        EventBus.getDefault().cancelEventDelivery(event);
    }
```

### 4，订阅者索引

​	默认情况下，EventBus 查找订阅者方法时采用的是反射，订阅者索引可以加速订阅者的注册，是一个可选的优化。其原理是：使用 EventBus 的注解处理器在应用构建期间创建订阅者索引类，该类包含了订阅者和订阅者方法的相关信息。EventBus 官方推荐在 Android 中使用订阅者索引以获得最佳的性能。

​	要开启订阅者索引的生成，你需要在构建脚本中使用annotationProcessor属性将EventBus的注解处理器添加到应用的构建中，还要设置一个eventBusIndex参数来指定要生成的订阅者索引的完全限定类名。

```java
 //订阅者索引，在 android defaultConfig 中，加入如下：
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [eventBusIndex: 'com.testdemo.www.eventbus.MainActivityIndex']
            }
        }
        
        //最后加入
        annotationProcessor 'org.greenrobot:eventbus-annotation-processor:3.1.1'
```

​	然后 Rebuild 会生成一个类

```java
public class MainActivityIndex implements SubscriberInfoIndex {
    private static final Map<Class<?>, SubscriberInfo> SUBSCRIBER_INDEX;

    static {
        SUBSCRIBER_INDEX = new HashMap<Class<?>, SubscriberInfo>();

        putIndex(new SimpleSubscriberInfo(MainActivity.class, true, new SubscriberMethodInfo[] {
            new SubscriberMethodInfo("onMessage", EventMessage.class, ThreadMode.POSTING, 3, false),
            new SubscriberMethodInfo("onMessageOne", EventMessage.class, ThreadMode.POSTING, 4, false),
        }));

    }

    private static void putIndex(SubscriberInfo info) {
        SUBSCRIBER_INDEX.put(info.getSubscriberClass(), info);
    }

    @Override
    public SubscriberInfo getSubscriberInfo(Class<?> subscriberClass) {
        SubscriberInfo info = SUBSCRIBER_INDEX.get(subscriberClass);
        if (info != null) {
            return info;
        } else {
            return null;
        }
    }
}

```

在自定义的 Application 中

```java
public class MyApplication extends Application {

    @Override
    public void onCreate() {
        super.onCreate();
        //配置EventBus
        EventBus.builder().addIndex(new MainActivityIndex()).installDefaultEventBus();
    }
}
```

然后正常使用即可

