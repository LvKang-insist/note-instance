## DataBinding 的使用

**DataBinding 一般情况下很少使用，但是如果你要使用 MVVM 框架，那么 DataBinding就是必不可少的了。下面我们看一下他的用法：** 

准备工作

​		先在使用的 module 中添加如下

```java
android {
    ......
    dataBinding {
        enabled = true
    }
}
```

### 简单的使用：

#### 1，创建 Bean 类

```java
public class UserBean  {

    private String name;
    private int age;
    ......
}
```

#### 2，布局文件

```java
<layout xmlns:android="http://schemas.android.com/apk/res/android">

   <data>
       <variable
           name="User"
           type="www.testdemo.com.UserBean"/>
   </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:id="@+id/age"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{String.valueOf(User.age)}"
            android:textColor="@color/black"
            android:textSize="20sp" />

        <TextView
            android:id="@+id/name"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{User.name}"
            android:textColor="@color/black"
            android:textSize="20sp" />

    </LinearLayout>

</layout>

```

修改根视图为 layout，添加 data 标签，variable 标签就相当于一个数据类，name 为名字，type 为地址。

然后在 要使用的地方使用 定义的 name.属性即可，如上面的 TextView，注意 TextView 值必须是 String，所以对 age 进行了强转

#### 3，在代码中使用

```java
ActivityMainBinding activityMainBinding = DataBindingUtil.setContentView(MainActivity.this, R.layout.activity_main); 
UserBean bean = new UserBean("张三",20);
activityMainBinding.setUser(bean);
```

运行程序，就会发现数据已经自动显示到上面了。

***注意，因为 布局的名字是 activity_main ,所以这是使用的是 ActivityMainBinding。这个类是系统根据布局文件的名字自动生成的。***

#### 4，一处多用

​	如果 bean 类在布局的多个地方使用，且值不一样，该怎么办，如下所示：

​	修改布局如下：

```xml
<data>
        <import type="www.testdemo.com.UserBean" />

        <variable
            name="User1"
            type="UserBean" />

        <variable
            name="User2"
            type="UserBean" />

        <variable
            name="User3"
            type="UserBean" />
</data>
```

使用 import 导入Bean 类，然后在下面设置多个数据类即可。

​	使用：

```java
 activityMainBinding = DataBindingUtil.setContentView(MainActivity.this, R.layout.activity_main);
        bean = new UserBean("张三",23);
        bean1 = new UserBean("李四",24);
        bean2 = new UserBean("王五",25);
        activityMainBinding.setUser1(bean);
        activityMainBinding.setUser2(bean1);
        activityMainBinding.setUser3(bean2);
```

当布局修改为上面的之后，你就会发现可以设置多种 bean 类了。在布局中使用还是 name.xxx，例如 User3.age

问题：如果导入的路径不同，但是类名相同那该如何是好！如下所示：

```xml
<import type="www.testdemo.com.UserBean"/>
<import type="www.testdemo.com.bean.UserBean"/>
```

这个时候就可以使用 alias 属性了：

```xml
<data>
        <import type="www.testdemo.com.UserBean" />
        <import
            alias="BeanUser"
            type="www.testdemo.com.bean.UserBean" />

      <!--第一个 import-->
        <variable
            name="User1"
            type="UserBean" />

        <!--第二个 import-->
        <variable
            name="Bean1"
            type="BeanUser" />

        <!--第二个 import-->
        <variable
            name="Bean2"
            type="BeanUser" />
    </data>
```

------



### 默认值 default

```java
 <TextView
            android:id="@+id/name"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="@{User.name ,default= 哈哈哈}"
            android:textColor="@color/black"
            android:textSize="20sp" />
```

### 单项绑定( BaseObservable )

BaseObservable 有两个方法

```
//全局刷新
notifyChange();
//局部刷新
notifyPropertyChanged(BR.name);
```

 Bean 继承 BaseObservable

 UserBean 如下：

```java
public class UserBean extends BaseObservable {

    @Bindable
    public String name;
    private int age;

    public UserBean(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public int getAge() {
        return age;
    }

    public String getName() {
        return name;
    }

    public void setAge(int age) {
        this.age = age;
        //全局刷新
        notifyChange();
    }

    public void setName(String name) {
        this.name = name;
        //局部刷新
        //如果 BR中调不出来属性，就需要 Rebuild 一下
        notifyPropertyChanged(BR.name);
    }
}
```

​		如果在设置某个属性的时候需要全局刷新，在 set 方法中调用notifyChange() 即可。	如果需要对某个属性进行刷新，则在set方法里面进行局部刷新即可。

​		上面给 age 全局刷新，给 name 局部刷新。

​		注意：如果需要局部刷新，那么他所刷新的字段必须为 public 类型，且必须使用  @Bindable 注解。

使用：

```java

        ActivityMainBinding activityMainBinding = DataBindingUtil.setContentView(MainActivity.this, R.layout.activity_main);
        bean = new UserBean("张三",20);
        activityMainBinding.setUser(bean);

        findViewById(R.id.main_b).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                //直接给类设置值，ui会自动改变
                bean.setAge(1000);
                bean.setName("王五");
            }
        });

```

 	事件回调中，设置了值，猜一下结果如果

​	 结果：两个值都被刷新了，虽然 name 是局部刷新，但是 age 是全局。所以值都刷新了，你可以试一下取消全局刷新，看一下效果。

------



### 点击事件

​	@{()-> listener.changeName()}

​	添加事件类

```java
  public class OnListener{
        public void changeAge(){
            Toast.makeText(MainActivity.this, "haha", Toast.LENGTH_SHORT).show();
            bean.setAge(1000);
        }
        public void changeName(){
            bean.setAge(30);
            bean.setName("王五");
        }
    }
```

​	修改布局

```xml
  <data>
        <variable
            name="user"
            type="www.testdemo.com.UserBean"/>

        <!-- 点击事件 -->
        <import type="www.testdemo.com.MainActivity.OnListener"/>
        <variable
            name="listener"
            type="OnListener"/>
    </data>

<-- 增加两个 Button -->
    
      <Button
            android:id="@+id/main_b"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:onClick="@{()-> listener.changeAge()}"
            android:background="#fff000" />

        <Button
            android:id="@+id/main_c"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:background="#fff000"
            android:onClick="@{()-> listener.changeName()}" />

```

 	进行绑定：

```
ActivityMainBinding activityMainBinding = DataBindingUtil.setContentView(MainActivity.this, R.layout.activity_main);
bean = new UserBean("张三",20);
activityMainBinding.setUser(bean);
//绑定事件
activityMainBinding.setListener(new OnListener());
```

​	然后点击 Button ，那两个方法就会得到执行

------



### 单项绑定（ObservableField）

​	使用单项绑定刷新 UI 的有三种

​		1，BaseObservable

​		2，ObservableField

​		3，ObservableCollection

​		这次我们 看一下第二种，记得第一种方法用于来特别麻烦，又要注解，还要 set 方法。

​		第二种就简单多了，只需要将 字段用 public final 进行修饰，然后重写 get 方法即可，如下：

```java
public class UserBean extends BaseObservable {

   public final ObservableField<Integer> age;
   public final ObservableField<String> name;

    public UserBean( ObservableField<String> name,  ObservableField<Integer> age) {
        this.name = name;
        this.age = age;
    }

    public ObservableField<Integer> getAge() {
        return age;
    }

    public ObservableField<String> getName() {
        return name;
    }
}
```

布局不需要修改

使用如下：

```java
public class MainActivity extends AppCompatActivity {
    private UserBean bean;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        ActivityMainBinding activityMainBinding = DataBindingUtil.setContentView(MainActivity.this, R.layout.activity_main);
        bean = new UserBean(new ObservableField<String>("张三"), new ObservableField<Integer>(20));
        activityMainBinding.setUser(bean);
        activityMainBinding.setListener(new OnListener());
    }
    
    //点击事件
    public class OnListener {
        public void changeAge() {
            bean.getAge().set(200);
        }

        public void changeName() {
            bean.getAge().set(30);
            bean.getName().set("王五");
        }
    }
}
```

是不是非常简单呢！

------

### 单项绑定（ObservableCollection）

​	单项绑定还提供了集合，**ObservableMap** 和 **ObservableList**，下面看一下使用方法

​	看一下布局

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">

    <data>
     <!-- map 集合 -->
        <import type="android.databinding.ObservableMap" />

        <variable
            name="user"
            type="ObservableMap&lt;String,Object&gt;"/>

    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <TextView
            android:id="@+id/name"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text='@{String.valueOf(user["马六"])}'
            android:textColor="@color/black"
            android:textSize="20sp" />

        <TextView
            android:id="@+id/name1"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text='@{String.valueOf(user["王五"])}'
            android:textColor="@color/black"
            android:textSize="20sp" />

    </LinearLayout>
</layout>
```

在 data 标签中使用了 map 数组 注意 type 的值 ，那两个&后面的代表是尖括号 &lt; 和 &gt;   ，然后在 TextView 中通过查找 键来获取值

**注意 TextView 的 text 后面是 '  '，并不是 “ ”**

在代码中使用：

```java
 ActivityMainBinding activityMainBinding = DataBindingUtil.setContentView(MainActivity.this, R.layout.activity_main);

        ObservableMap<String,Object> map = new ObservableArrayMap<>();
        map.put("王五",65);
        map.put("马六",80);
        activityMainBinding.setUser(map);
```

ObservableList 的使用

修改布局：

 

```java
<layout xmlns:android="http://schemas.android.com/apk/res/android">
<data>
    <import type="android.databinding.ObservableList" />
    <variable
        name="user"
        type="ObservableList&lt;String&gt;"/>

</data>

<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text='@{String.valueOf(user[0])}'
        android:textColor="@color/black"
        android:textSize="20sp" />

    <TextView
        android:id="@+id/name1"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text='@{String.valueOf(user[1])}'
        android:textColor="@color/black"
        android:textSize="20sp" />

</LinearLayout>
</layout>
```

使用如下：

```java
 ActivityMainBinding activityMainBinding = DataBindingUtil.setContentView(MainActivity.this, R.layout.activity_main);

        ObservableList<String> list = new ObservableArrayList<>();
        list.add("王五");
        list.add("马六");
        activityMainBinding.setUser(list);
```

------



### 双向绑定

​		简单解释一下：就是数据发生改变 ui 立刻刷新，ui 变化 数据也会改变。

​	看一下 Bean 类：

```java
public class UserBean extends BaseObservable {  
    public final ObservableField<String> string;
    public UserBean( ObservableField<String> string) {
        this.string = string;
    }

    public ObservableField<String> getString() {
        return string;
    }
}
```

布局

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
<data>
   <variable
       name="user"
       type="www.testdemo.com.UserBean"/>
</data>

<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text='@{user.string}'
        android:textColor="@color/black"
        android:textSize="20sp" />

    <EditText
        android:id="@+id/edit"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="@={user.string}"
        android:hint="请输入数据"/>

</LinearLayout>
</layout>
```

​	上面一个 一个 editText，一个 TextView，注意他们 text 的内容

```java
  ActivityMainBinding activityMainBinding = DataBindingUtil.setContentView(MainActivity.this, R.layout.activity_main);
        userBean = new UserBean(new ObservableField<>("345"));
        activityMainBinding.setUser(userBean);
```

​	运行，在 EditText 中输入内容，textView 内容也会跟着改变，反之也一样。

------

### 引用类方法

​	如下面这个类，有一个静态方法：

```java
public class Data {
    public static String setString(String s){
        return s+" ------";
    }
}
```

​	那么就可以在布局中直接调用他。如下：

```xml
<import type="www.testdemo.com.Data"/>

 <TextView
        android:id="@+id/name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text='@{Data.setString(user.string)}'
        android:textColor="@color/black"
        android:textSize="20sp" />
```

​	只贴出了有用的代码，在 text 属性中 调用了 Data 的 setString 方法，这个 TextView 显示的就是 setString 返回的字符串

​	**注意：只能够调用静态方法**

------

### 使用运算符

```java
<TextView
        android:id="@+id/name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text='@{user.age + user.name}'
        android:textColor="@color/black"
        android:textSize="20sp" />
            
<TextView
        android:id="@+id/name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text='@{String.valueOf(1+2)}'
        android:textColor="@color/black"
        android:textSize="20sp" />
           
<TextView
        android:id="@+id/name"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text='@{String.valueOf(1+2)}'
        android:textColor="@color/black"
        android:textSize="20sp"
        android:visibility="@{user.status ? View.VISIBLE : View.GONE}"/>    
```

### 在适配器中使用

​	简单的看一下就好：

```java
 @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        ItemNewsBinding itemNewsBinding = DataBindingUtil.inflate(LayoutInflater.from(mContext), R.layout.item_news,parent,false);
        return new ViewHolder(itemNewsBinding);
    }
```

这个是在 RecyclerView 适配器中使用，同理，其他的地方都差不多。



> 参考：https://blog.csdn.net/qq_40881680/article/details/102240892