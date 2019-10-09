##### 不需要继承Dialog类，快速的实现一个dialog对话框。



### 一，dialog的快速实现

---
1，dialog的布局，这是必不可少的，

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="600dp"
    android:layout_height="wrap_content"
    android:background="#dfdfdf"
    android:orientation="vertical">

    <TextView
        android:layout_width="600dp"
        android:layout_height="50dp"

        android:text="小车账户充值"
        android:textSize="30sp"
        android:textStyle="bold"
        android:gravity="center"
        android:textColor="#000"
        android:background="#999"/>

    <LinearLayout
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:layout_marginTop="50dp"
        android:layout_gravity="center_horizontal">
        <TextView
            android:id="@+id/a43_d_car"
            android:layout_width="match_parent"
            android:layout_height="50dp"
            android:text="车牌号："
            android:textSize="30sp"
            android:textStyle="bold"
            android:textColor="#000" />

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="50dp"
            android:orientation="horizontal"
            android:layout_marginTop="10dp">
            <TextView
                android:layout_width="wrap_content"
                android:layout_height="50dp"
                android:text="充值金额："
                android:textSize="30sp"
                android:textStyle="bold"
                android:gravity="center"
                android:textColor="#000" />
            <EditText
                android:id="@+id/a43_d_edit"
                android:layout_width="200dp"
                android:layout_height="match_parent"
                android:inputType="number"
                android:maxLength="3"/>

        </LinearLayout>

        <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="50dp"
            android:layout_marginTop="50dp"
            android:gravity="center">
            <Button
                android:id="@+id/a43_d_chongzhi"
                android:layout_width="150dp"
                android:layout_height="match_parent"
                android:text="充值"
                android:textSize="25sp"
                android:textColor="#000"/>
            <Button
                android:id="@+id/a43_d_quxioa"
                android:layout_width="150dp"
                android:layout_height="match_parent"
                android:layout_marginLeft="50dp"
                android:text="取消"
                android:textSize="25sp"
                android:textColor="#000"/>

        </LinearLayout>
        <TextView
            android:layout_width="match_parent"
            android:layout_height="50dp" />
    </LinearLayout>
</LinearLayout>
```
这是一个dialog布局，最上面是一个title，然后下面就是一些控件，没什么可说的。



2，在某个事件中，弹出dialog，实现对应的功能。

```
 Button btn = convertView.findViewById(R.id.a43_chongzhi);
            btn.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    //加载dialog布局
                    View view = View.inflate(A43.this,R.layout.a43_dialog,null);
                   //创建dialog对象
                    dialog = new Dialog(A43.this);
                   
                   //去掉标题栏，使用我们自定义的
                    dialog.requestWindowFeature(Window.FEATURE_NO_TITLE);
                   
                   //加载视图
                    dialog.setContentView(view);
                    //弹出对话框
                    dialog.show();

                    //获取对话框中的控件，实现相应的功能，
                    final EditText edit = view.findViewById(R.id.a43_d_edit);
                    Button btn = view.findViewById(R.id.a43_d_chongzhi);
                    Button quxioa = view.findViewById(R.id.a43_d_quxioa);
                    TextView title = view.findViewById(R.id.a43_d_car);
                    title.setText("车牌号："+position+1);
                    quxioa.setOnClickListener(new View.OnClickListener() {
                        @Override
                        public void onClick(View v) {
                            dialog.dismiss();
                        }
                    });

                    btn.setOnClickListener(new View.OnClickListener() {
                        @Override
                        public void onClick(View v) {
                            int money = Integer.parseInt(edit.getText().toString().trim());
                            upData(position+1,money);
                        }
                    });
                }
            });
            
    //这里是一个网络请求，用来从服务器拿到数据。        
    private void upData(int i, int money) {
        String url = "http://192.168.1.104:8088/transportservice/action/SetCarAccountRecharge.do";
        String post = "{\"CarId\":"+i+",\"Money\":"+money+", \"UserName\":\"user1\"}";

        http.post(url, post, new Http.onReqeust() {
            @Override
            public void onReqsult(String str) {
                try {
                    JSONObject object = new JSONObject(str);
                   if (object.optString("RESULT").equals("S")){
                       Toast.makeText(A43.this, "充值成功", Toast.LENGTH_SHORT).show();

                       reqeust();
                       dialog.dismiss();
                   }
                } catch (JSONException e) {
                    e.printStackTrace();
                }
            }
        });
    }        
            
```
简单的说一下，就是加载视图，创建dialog对象，弹出dialog，最后根据加载的视图就可以拿到dialog中的控件，进行逻辑的处理。

---
#### 二，使用dialog快速播放视频.

直接上代码，这个连布局都不用写。

```
layout.setOnClickListener(new View.OnClickListener() {
                @Override
                public void onClick(View v) {
                    //创建VideoView对象，
                    VideoView videoView = new VideoView(A30.this);
                   
                   //设置视频路径
                    videoView.setVideoURI(Uri.parse("android.resource://"+getPackageName()+"/"+R.raw.movie));
                    //创建dialog对象
                    Dialog dialog = new Dialog(A30.this);
                   
                   //去掉标题栏
                    dialog.requestWindowFeature(Window.FEATURE_NO_TITLE);
                   
                   //将VideoView传入
                    dialog.setContentView(videoView);

                    //设置dialog的窗口位置和大小，这里没怎么设置，
                    Window window = dialog.getWindow();
                    WindowManager.LayoutParams params = window.getAttributes();
                    params.dimAmount = 0;
                    window.setAttributes(params);

                    //弹出dialog，
                    dialog.show();
                    //播放视频
                    videoView.start();

                }
            });
```

首先创建VideoView的对象，然后扔进dailog中，然后设置dialog的窗口位置和大小，最后弹出对话框，播放视频。

---


> 如有错误还请指出，谢谢！