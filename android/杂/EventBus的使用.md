​		EventBus 是一个开源库，他利用 发布/订阅 者模式来对项目进行解耦，他可以利用很少的代码，来实现多组件的通信。android 的多组件通信可以通过 Handler，广播等，但使用他们进行通信 代码量非常多，组件之间容易产生耦合。关于 EventBus 的工作模式如官方的图所示：

### 1，基本使用：

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

### 2,线程模式

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

