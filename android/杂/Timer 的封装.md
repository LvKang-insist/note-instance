1，创建接口

```java
public interface ITimeListener {
    void onTime();
}
```



2，首先 创建一个类，继承TimerTask ，也就是一个任务,并在构造器中 传入 接口的实例

```java
public class BaseTimeTask extends TimerTask {

    private ITimeListener mITimeListener = null;

    public BaseTimeTask(ITimeListener timeListener) {
        this.mITimeListener = timeListener;
    }

    @Override
    public void run() {
        if (mITimeListener != null){
            mITimeListener.onTime();
        }
    }
}
```

3，在需要的 执行Timer 的类中 实现ITimeListener 接口。并初始化Time

```java
public class LauncherDelegate extends LatteDelegate implements ITimeListener {

    @BindView(R2.id.tv_launcher_time)
    AppCompatTextView mTvTimer= null;

    private Timer mTimer = null;
    private int mCount = 5;

    @OnClick(R2.id.tv_launcher_time)
    void onClickTimerView(){
    }

    /**
     * 初始化 Timer
     */
    @SuppressWarnings("AlibabaAvoidUseTimer")
    private void initTimer(){
        mTimer = new Timer();
        final BaseTimeTask task = new BaseTimeTask(this);
        //第一个参数 要执行的任务，2，延迟的时间，3，每隔一秒执行一次
        mTimer.schedule(task,0,1000);
    }

    @Override
    public Object setLayout() {
        return R.layout.delegate_launcher;
    }

    @Override
    public void onBindView(@Nullable Bundle savedInstanceState, View rootView) {
        //初始化 timer 任务
        initTimer();
    }

    @Override
    public void onTime() {
        getProxyActivity().runOnUiThread(new Runnable() {
            @Override
            public void run() {
                if (mTvTimer != null){
                    mTvTimer.setText(MessageFormat.format("跳过\n{0}s", mCount));
                    mCount --;
                    if (mCount< 0){
                        if (mTimer!= null){
                            //倒计时 暂停
                            mTimer.cancel();
                            mTimer = null;
                        }
                    }
                }
            }
        });
    }
}
```

在 创建任务的时候 传入 this，因为这个类实现了接口，所以直接传入 this。启动任务之后就会回调onTime方法，然后可以执行倒计时了