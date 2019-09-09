在实际的项目中我们应该根据特定的需求为每个活动指定恰当的启动模式。启动模式一共有四种，分别是standard、singleTop、singleTask、singleInstance。可以在AndroidManifest.xml中通过给<activity>标签指定android:launchMode属性来选择启动模式，下面我们逐个来进行学习。

---
1，standard

standard是互动默认的启动模式，在不进行显式指定的情况下，所有的活动都会自动使用这种启动模式。对于standard模式的活动，系统不会在乎这个活动是否已经在返回栈中存在，每次启动的时候都会创建一个该活动的实例。

```
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.first_layout);
        Log.d("FirstActivity：", "onCreate: "+this.toString());
        Button button = findViewById(R.id.button_1);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(FirstActivity.this,FirstActivity.class);
                startActivity(intent);
            }
        });
    }
```
在这个当前活动里边启动当前活动，从逻辑来看确实没有什么意义，但是重点是研究standard模式，因此不必在意这段代码的实际用途。我们在oncreate()方法中添加了一条打印语句，用于打印当前活动的实例。我们可以看到打印的信息如图所示：

从打印的信息我们可以看出，每点击一次按钮 就会创建出一个当前活动的实例，也就是说，每点击一次就会在返回栈中添加一次实例，点击Back的时候，添加了几次就要点击多少次Back。

---
2，singleTop

在有些情况下，standard模式不太合理，活动明明已经在栈顶了，为什么再次启动的时候还需要再次创建一个新的活动实例呢？ 这只是系统默认的一种启动模式而已，你完全可以更具自已的需求进行修改，比如说使用singleTop模式，当活动的模式指定为singleTop了，在启动活动的时候如果发现返回栈的栈顶已经是该活动，则认为可以直接使用它，不会再创建新的活动实例。


修改AndroidMainifest.xml中活动的启动模式，改为singleTop，如下所示
```
 <activity android:name=".FirstActivity"
            android:launchMode="singleTop">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```
然后重新运行程序，查看打印的日志，你会发现只创建了一个活动的实例，每当要启动一个活动时就会判断栈顶的元素是不是要启动的活动，如果是则直接使用栈顶的活动，否则创建新的活动。当要启动的活动未处于栈顶时，还是会创建新的实例的。

---
3，singleTask

使用singleTop模式可以很好的解决重复创建栈顶活动的问题，如但是如第二个所说的，如果该活动并没有处于栈顶的位置，还是可能会创建多个活动的实例。那么有没有什么办法让某个活动在应用程序中只有一个实例呢？这里就要用到了singleTask模式，当活动的模式指定为singleTask后，每次启动活动时先会在返回栈中检查是否存该互动的实例，如果有，则使用，并把在这个活动之上的所有互动统统出栈，如果没有则创建一个新的活动实例。

```
    <activity android:name=".FirstActivity"
            android:launchMode="singleTask">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```

---

4，singleInstance

singleInstance应该是四种启动模式中最为复杂的一个了，不同于以上三种启动模式，指定为singleInstance模式的活动会启用一个新的返回栈来管理活动， 想象以下场景，假设我们程序中有一个活动时允许其他程序调用的，如果我们想实现其他程序和我们的程序共享这个活动的实例，应该如何实现呢？使用前面三种肯定是做不到的，因为每个程序都会有自己的返回栈，同一个活动在不同的返回栈中入栈时必然创建了新的实例。而使用singleInstance模式就可以解决这个问题。在这种模式下会有一个单独的返回栈来管理这个活动，不管是哪一个程序来访问这个活动，都使用的是同一个返回栈。

```
     <activity android:name=".SecondActivity"
            android:launchMode="singleInstance">
          
        </activity>
```

修改SecondActivity的启动模式为singleInstance，然后修改FirstActivity中onCreate()方法中的代码

```
public class FirstActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.first_layout);
        Log.d("FirstActivity：", "onCreate: "+getTaskId());
        Button button = findViewById(R.id.button_1);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(FirstActivity.this,FirstActivity.class);
                startActivity(intent);
            }
        });
    }

```
在onCreate()方法中打印当前返回栈的Id，然后修改SecondActivity中onCreate()方法的代码：

```
public class SecondActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.second_layout);
        Log.d("SecondActivity：", "onCreate: "+getTaskId());
        Button button = findViewById(R.id.button_2);
        button.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                Intent intent = new Intent(SecondActivity.this,ThirdActivity.class);
                startActivity(intent);
            }
        });
    }

```
最后修改ThirdActivity的onCreate()中的代码，如下所示：

```
public class ThirdActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_third);
        Log.d("ThirdActivity：", "onCreate: "+getTaskId());
    }
}
```

同样在onCreate()方法中打印了当前返回栈的id，现在运行程序，在FirstActivity活动中点击按钮进入SecondActivity，在SecondActivity中点击按钮进入ThirdActivity中，查看打印的信息，如图所示：



当处于ThridActivity活动中时，点击Back将直接返回到了FirstActivity中，在按下返回键又会返回到SecondActivity中，然后点击Back才会退出程序，原理其实很简单，由于FirstActivity和Thirdactivity是放在同一个返回栈中的，所以第一次点击Back显示的是FirstActivity，当再次点击Back将这时当前返回栈已经空了，于是显示了另外一个返回栈的栈顶活动，即SecondActivity,最后按下Back，则会退出程序，因为所有返回栈都已经空了，所以就退出了程序。
