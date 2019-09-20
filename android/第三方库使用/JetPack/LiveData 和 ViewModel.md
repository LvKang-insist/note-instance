LiveData ：是一种持有可被观察的类,和其他可被观察的类不同的是，LiveData 是就要生命周期感知能力的，这意味着他可以在 Activity ，fragment 或者 service 生命周期活跃状态时 更新这些组件。

ViewModel：它是以一种关注生命周期的方式存储和管理与 UI 相关的数据。当 屏幕旋转时，活动会重新创建，onCreate方法被重新调用，但是我们可以使用 Bundle 恢复数据，但是这种方法只适用于进行序列化的少量数据，不适合大量数据， 使用ViewModel 会自动保留 之前的数据并给行的 Activity 或者 Fragment 使用。知道 Activity 销毁时 ，Framework 会调用 ViewModel 的 onCleared 方法，我们可以在 onCleared 方法中 做一些 资源清理工作

使用方式

1，导入依赖

```
 implementation "android.arch.lifecycle:viewmodel:1.1.1" 
 implementation "android.arch.lifecycle:livedata:1.1.1"
```

2，这里我们以一种最简单的方式来演示它，新建 UserViewModel 类 继承自 ViewModel。

```java
public class UserViewModel extends ViewModel {
    
   MutableLiveData<String> mutableLiveData;

   public MutableLiveData<String> getStr(){
       if (mutableLiveData == null){
           mutableLiveData = new MutableLiveData<>();
       }
       new Thread(new Runnable() {
           @Override
           public void run() {
               for (int i = 0; i < 3; i++) {
                   try {
                       Thread.sleep(2000);
                   } catch (InterruptedException e) {
                       e.printStackTrace();
                   }
                   mutableLiveData.postValue("哈哈");
               }
           }
       }).start();
       return mutableLiveData;
   }
}
```

我们使用 MutableLiveData 来保存数据，泛型为数据的类型，在 getStr 方法中，创建了一个线程，然后 通过 MutableLiveData 将数据发送出去，子线程使用postValue，主线程使用 setValue,最后将 MutableLiveData 返回即可。

3，接下来在 Activity 中使用即可

```java
public class MainActivity extends AppCompatActivity {

    private UserViewModel model;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);


        model = new ViewModelProvider.NewInstanceFactory().create(UserViewModel.class);
        model.getStr().observe(this, new Observer<String>() {
            @Override
            public void onChanged(String s) {
                Log.e("哈哈哈", "onChanged: ");
            }
        });
     
    }
}

```

通过上面的方式 即可 拿到数据，

4，如果需要多次获取数据呢，例如界面需要刷新等...

​	如下所示：

```java
    findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                if (model != null) {
                    model.getStr();
                }
            }
        });
```

如果点击了按钮，则就会重新获取一次数据，对应的监听也会继续执行一次。