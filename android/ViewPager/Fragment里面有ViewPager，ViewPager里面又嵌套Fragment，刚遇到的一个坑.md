在碎片嵌套碎片遇到的坑
---
1,在父Fragment中的onCreateView方法里面调用方法去对ViewPager进行初始化，并且new 了四个碎片的对象添加到viewpager里面

```
 @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {
        view = inflater.inflate(R.layout.fragment_layout06, container, false);
        init();
        //绘制线性图
        mp_LineChart();
        //初始化ViwPager
        init_ViewPager();
        System.out.println("onCreate");
        return view;
    }
```

```
private void init_ViewPager() {
        List<Fragment> list = new ArrayList<>();
        barChart = new Mp_BarChart();
        lineChar_1 = new Mp_LineChar_1();
        lineChar_2 = new Mp_LineChar_2();
        lineChar_3 = new Mp_LineChar_3();
        list.add(barChart);
        list.add(lineChar_1);
        list.add(lineChar_2);
        list.add(lineChar_3);
        viewPager =view.findViewById(R.id.frag6_viewpager);
        viewPager.setAdapter(new Frag6_ViewPager_Adapter(getChildFragmentManager(),list));
        viewPager.setOffscreenPageLimit(3);
        viewPager.addOnPageChangeListener(new ViewPager.OnPageChangeListener() {
        ......
```

---
2，因为我四个子碎片都是要做网络请求的，而且请求的接口都一样，所以我想在父碎中直接进行网络请求，然后通过在初始化ViewPager时候的碎片对象调用一个set方法，直接把网络请求的值设置进去。

```
 @Override
    public void onResume() {
        System.out.println("onResume");
        Random random = new Random();
        barChart.setData(random.nextInt(130-90+1)+90);
        super.onResume();
    }
```

---
3，就在父碎片 中用子碎片的对象 调用他的setData方法，报了空指针，经过一系列的研究和请教 才发现在父碎片的生命周期的onResume方法执行完之后才会开始执行子碎片的生命周期，也就是说不能再onResume方法中调用自碎片的方法，因为在这个时候子碎片的生命周期还没用执行。所以子碎片没有被初始化，会报错，

---
4，解决的办法
- ，在onResume方法里面加一个定时器，当经过多少秒之后才调用子碎片的方法，这样就不会报错，但是一定要控制倒计时的时间，我测了一下0.5秒以上一般是没有问题的。如果想更安全就让倒计时的时间长一点就好。
```
  @Override
    public void onResume() {
        System.out.println("onResume");
        CountDownTimer startTime = new CountDownTimer(500,500) {
            @Override
            public void onTick(long millisUntilFinished) {
            }
            @Override
            public void onFinish() {
                Random random = new Random();
                barChart.setData(random.nextInt(130-90+1)+90);
            }
        }.start();
        super.onResume();
    }
```
- 或者是做一个接口回调，当子碎片加载完了回调接口去拿到数据就可以了。

---
总结一下就是 ：==**当一个碎片里面有嵌套了一个碎片或者是碎片里面有个ViewPager里面嵌套了好多个碎片。在在这样的情况下他先执行的是父碎片的生命周期，当父碎片的onResume方法执行完成之后才会执行子碎片的生命周期。**==

> 如果有错误，还请指正，谢谢！