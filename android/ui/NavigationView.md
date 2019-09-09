使用Navigation 实现侧滑菜单，

其实你可以在侧滑出来的布局上自定义任意的布局，不过谷歌给我们提供了一个更好的方法，使用NavigationView,NavigationView是Design Support库中的一个控件,他是严格按照MaterialDesign的要求来进行设计的，而且可以将滑动菜单页面的实现变得非常的简单.

下面我们看一下他的用法;

1,添加依赖：

```
    ///Design 库
    implementation 'com.android.support:design:28.0.0'
    ///一个开源项目: CircleImageView ：他可以轻松直线图片圆形化的功能
```

2,准备用来显示菜单项的Menu，和 一个头部布局

在raw文件夹下创建menu文件夹，创建nav_menu.xml文件,
```
<?xml version="1.0" encoding="utf-8"?>
<!--Navigation : 导航-->
<menu xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- group 表示一个组
         checkableBehavior= single 表示组中的所有菜单项只能单选-->
    <group android:checkableBehavior="single"/>

    <item
        android:id="@+id/nav_1"
        android:icon="@drawable/b"
        android:title="B"/>

    <item
        android:id="@+id/nav_2"
        android:icon="@drawable/c"
        android:title="C"/>

    <item
        android:id="@+id/nav_3"
        android:icon="@drawable/e"
        android:title="E"/>

    <item
        android:id="@+id/nav_4"
        android:icon="@drawable/f"
        android:title="F"/>

    <item
        android:id="@+id/nav_5"
        android:icon="@drawable/p"
        android:title="P"/>

</menu>
```
创建头部布局nav_header.xml

```
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="200dp"
    android:padding="10dp"
    android:background="?attr/colorPrimary">
    <!--图片 iamgeView-->
    <de.hdodenhof.circleimageview.CircleImageView
        android:layout_width="70dp"
        android:layout_height="70dp"
        android:src="@drawable/friends"
        android:layout_centerInParent="true"/>

    <TextView
        android:id="@+id/mail"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:text="lv_345@163.com"
        android:textColor="#FFF"
        android:textSize="14sp" />

    <TextView
        android:id="@+id/username"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_above="@id/mail"
        android:text="Lv_345"
        android:textColor="#FFF"
        android:textSize="14sp"/>
</RelativeLayout>
```

3，将NavigationView 设置为侧滑布局


```
<android.support.v4.widget.DrawerLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/draw_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">
    
    <!--主 界面-->
    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">
        <!--  高度设置为 ActionBar的高度 -->
        <!--  背景色设置为 colorPrimary-->
        <!--  因为已将主题设置为淡色主题，而Toolbar上面的各种元素就会使用深色系
              这是为了主题和颜色区别开，但是效果就非常差，主题是淡色，但是字体是深色，所以会显得不好看
              为了让Toolbar能单独使用深色主题，这是我们使用
              android:theme属性，指定一个深色的主题,-->
        <!--  如果Toolbar中有菜单按钮，那么弹出的菜单项也是深色主题，这样就再次变得非常难看
              这里使用app:popupTheme 将菜单项指定为淡色主题，-->
        <android.support.v7.widget.Toolbar
            android:id="@+id/tolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"
            app:popupTheme="@style/ThemeOverlay.AppCompat.Light">
        </android.support.v7.widget.Toolbar>
    </FrameLayout>

    <!--滑动菜单-->
    <!--android:layout_gravity="start" 指定滑动-的位置-->
    <android.support.design.widget.NavigationView
        android:id="@+id/nav_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_gravity="start"
        app:menu="@menu/nav_menu"
        app:headerLayout="@layout/nav_header"/>

</android.support.v4.widget.DrawerLayout>
```
上面使用了DrawLayout控件，他是一个可以滑动的布局，他包含两个布局，第一个布局是主屏幕,第二个是滑动菜单显示的内容.我们将滑动的布局改为NavigationView，并添加了android:layout_gravity="start"，表示他是从左边滑出来的。注意这个必须加，否则会无法滑动。然后指定 menu 和 头布局 ，这样NavigationView 就定义完成了

4，设置点击事件
    
```
public class MainActivity extends AppCompatActivity {

    private DrawerLayout drawerLayout;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = findViewById(R.id.tolbar);
        setSupportActionBar(toolbar);

        drawerLayout = findViewById(R.id.draw_layout);

        

        //得到ActionBar的实例
        ActionBar actionBar = getSupportActionBar();
        if (actionBar!= null){
            //让导航按钮显示出来
            //导航的按钮就叫做HomeAsUp按钮,默认图标是一个返回的箭头,含义是返回到上一个活动
            //这里对他的样式和作用都做了修改
            actionBar.setDisplayHomeAsUpEnabled(true);
            //设置一个导航按钮的图标
            actionBar.setHomeAsUpIndicator(R.drawable.arrow);
        }

        NavigationView navView = findViewById(R.id.nav_view);
        navView.setNavigationItemSelectedListener(new NavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(@NonNull MenuItem menuItem) {
                drawerLayout.closeDrawers();
                return true;
            }
        });
    }
    //加载Menu文件
    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        //getMenuInflater可以得到MenuInflater的对象
        // 在调用他的inflate方法就可以创建菜单了.
        //inflate接收两个参数，1，传入布局文件  2，将菜单项添加到那个Menu对象中，
        getMenuInflater().inflate(R.menu.toolbar,menu);
        //返回true就会让对象显示出来.
        return true;
    }

    //处理Menu的点击事件
    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        switch (item.getItemId()){
            case android.R.id.home:
                drawerLayout.openDrawer(GravityCompat.START);
                break;
            case R.id.backup:
                Toast.makeText(this, "返回", Toast.LENGTH_SHORT).show();
                break;
            case R.id.delete:
                Toast.makeText(this, "删除", Toast.LENGTH_SHORT).show();
                break;
            case R.id.settings:
                Toast.makeText(this, "设置", Toast.LENGTH_SHORT).show();
                break;
        }
        return true;
    }
}
```

主要就是166 到 173 行，首先加载布局，然后设置监听，就完了(其他的是都Toolbar的内容，和NavigationView没有关系).在点击事件中，只关闭了侧滑菜单。
    

到这里我们的滑动菜单页面就已经做完了，    