当一个 活动进入到了停止状态，是有可能被系统回收的。想象以下场景：应用中有一个活动A，用户在A的基础上启动了B，活动A进入了停止状态。这个时候由于系统内存不足，将活动A回收了，然后用户按Back键返回活动，这个时候活动A会正常显示，只不过这个时候并不会执行onRestart()方法，而是执行活动A的onCreate()方法，因为活动A在这种情况下会被重新创建一次。

---
当我们的应用出现了这种情况，是非常影响用户体验的，所以必须解决这个问题，查阅文档可知 Activity中还提供了一个onSaveInstanceStart()回调方法，这个方法在活动回收前一定会调用，因此我们可以通过这个方法来解决活动被回收时 数据得不到保存的问题

---
##### 保存数据：onSaveInstanceStart()会携带一个Bundle类型的参数，    Bundle提供了一系列的方法用于保存数据，每个保存的方法都要传入两个参数，第一个是键，第二个是要保存的内容。
```
 @Override
    public void onSaveInstanceState(Bundle outState, PersistableBundle outPersistentState) {
        super.onSaveInstanceState(outState, outPersistentState);
        String tempData = "Something you just type";
        outState.putString("data",tempData);
    }
```
##### 恢复数据：数据以及保存下来了，然后我们要在onCreate()方法里面进行恢复，细心地你也许早就发现onCreate()方法也有一个参数，这个参数一般情况下是null，但是如果活动被系统回收之前使用onSaveInstanceStart()方法来保存过数据的话，这个参数就会携带我们之前保存的全部数据。

恢复数据也可以在 **onRestoreInstanceState**  方法中执行。这个方法 会在 onCreate 之后执行

```java
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if (savedInstanceState != null){
            String temp = savedInstanceState.getString("data");
            Toast.makeText(this, ""+temp, Toast.LENGTH_SHORT).show();
        }
        ......
}
```


```java
@Override
protected void onRestoreInstanceState(Bundle savedInstanceState) {
    super.onRestoreInstanceState(savedInstanceState);
    if (savedInstanceState != null){
        Log.e(TAG, "onRestoreInstanceState: "+savedInstanceState.getString("data") );
    }
}
```

这样就可以解决由于系统内存不足而导致的活动被回收。


> 如有错误，还请指出，谢谢！