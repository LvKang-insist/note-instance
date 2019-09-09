


默认的主题
    
```
parent="Theme.AppCompat.Light.DarkActionBar"
```

去掉默认的主题


```
Theme.AppCompat.Light.NoActionBar"
```
Theme.AppCompat.Light.NoActionBar"表示淡色主题，他会将界面的主体颜色设置为淡色，陪衬的颜色设置为深色。会取消标题栏


```
"Theme.AppCompat.NoActionBar"
```
他会将界面的主题颜色设成声色，陪衬颜色设成淡色，界面颜色比较深.会去掉标题栏.



Toolbar的使用

1，首先将主题改为淡色主题.

```
 <!-- Base application theme. -->
    <style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
        <!-- Customize your theme here. -->
        <item name="colorPrimary">@color/colorPrimary</item>
        <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
        <item name="colorAccent">@color/colorAccent</item>
    </style>
```

2，添加Toolbar控件

```
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
```
3,编写Menu，在raw目录下新建menu文件夹，然后新建一个toolbar.xml的文件

```
<menu xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto">
    <!-- always 表示永远显示在Toolbar中，屏幕空间不够则，不显示
         ifRoom表示屏幕空间空载的情况下显示在Toolbar中，不够就显示在菜单中，
         never 表示永远显示在菜单中-->

    <item
        android:id="@+id/backup"
        android:title="返回"
        app:showAsAction="always"/>
    <item
        android:id="@+id/delete"
        android:title="删除"
        app:showAsAction="ifRoom"/>
    <item
        android:id="@+id/settings"
        android:title="设置"
        app:showAsAction="never"/>

</menu>
```
4，在活动中使用Toolbar，并且设置menu的点击事件(Toolbar是v7包下的)


```
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.support.v7.widget.Toolbar;
import android.view.Menu;
import android.view.MenuItem;
import android.widget.Toast;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        Toolbar toolbar = findViewById(R.id.tolbar);
        setSupportActionBar(toolbar);
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
