#### ThreadLocal 是一个线程内部的数据存储类，通过他可以指定线程中存储的数据，在读取的时候，只有指定的线程才可以读取到数据，对于其他线程来说是无法拿到数据的。

---

##### 我们可以通过一个简单的例子了解一下：

```
public class MainActivity extends AppCompatActivity {

    private static final String TAG = "MainActivity：";
    private ThreadLocal<Boolean> threadLocal;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        threadlocal();
    }
    private void threadlocal() {
        threadLocal = new ThreadLocal<>();
        threadLocal.set(true);
        Log.d(TAG, "main--ThreadLocal: " + threadLocal.get());

        new Thread("thread1") {
            @Override
            public void run() {
                threadLocal.set(false);
                Log.d(TAG, "thread1--ThreadLocal: " + threadLocal.get());
            }
        }.start();

        new Thread("thread2") {
            @Override
            public void run() {
                Log.d(TAG, "thread2--ThreadLocal: " + threadLocal.get());
            }
        }.start();
    }
}

```
在上面代码中主线程设置为true，线程1为false，线程2没有设置，
打印信息如下：

```
 D/MainActivity：: main--ThreadLocal: true
 D/MainActivity：: thread1--ThreadLocal: false
 D/MainActivity：: thread2--ThreadLocal: null
```
可以看出，虽然在不同线程访问的是同一个ThreadLocal对象，但是他们每个线程获取到的值是不一样的。这就是ThreadLocal奇妙之处。

---
#### 下面我们分析一下他set()方法的源码

```
public class ThreadLocal<T> {

    public void set(T value) {
        //拿到当前线程对象
        Thread t = Thread.currentThread();
        //通过getMap拿到ThreadLocalMap对象的实例，调用getMap()时传入当前线程的实例
        ThreadLocalMap map = getMap(t);
        //map不为空则存入数据，否则通过createMap创建一个对象
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
    //通过传入的参数，拿到Thread中的threadLocals，
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
    //这句话是Thread中的，写在这里是为了看起来方便
    //每创建一个Thread，都会创建一个ThradLocal.ThreadLocalMap 的引用，以便上面的getMap使用。
    ThreadLocal.ThreadLocalMap threadLocals = null;
    
    //当通过当前线程获取的ThreadLocalMap为空时，就会创建一个他的对象，这个方法是从set中调用的。
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
    //这个就是 ThreadLocalMap的构造函数了。
    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            //private Entry[] table;
            //private static final int INITIAL_CAPACITY = 16;
            //table 是Entry类型的数组
            table = new Entry[INITIAL_CAPACITY];
            //通过算法 得到对应的下标
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            //将数据存进数组
            table[i] = new Entry(firstKey, firstValue);
            //数组长度加 1
            size = 1;
            //设置阈值
            setThreshold(INITIAL_CAPACITY);
        }
}
```
简单的 总结一下ThreadLocal的set方法，进入方法后，他首先会获取当前线程的实例，然后通过getMap()方法去拿线程实例中的ThreadLocal.ThreadLocalMap threadLocals = null，这个东西是写在Thread类中的。判断 如果为空，调用createMap 创建对象然后保存值，如果不为空，则直接保存值。当再次在这个线程中保存值得时候getMap()的值就不会为空了，则会直接保存。

---
#### 然后 分析一下 get 

```
public class ThreadLocal<T> {
    public T get() {
        //
        Thread t = Thread.currentThread();
        //通过getMap拿到ThreadLocalMap对象的实例，调用getMap()时传入当前线程的引用
        ThreadLocalMap map = getMap(t);
        //不为null
        if (map != null) {
            //拿到保存的Entry对象
            ThreadLocalMap.Entry e = map.getEntry(this);
            //如果不为空 则将拿到的值返回
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
    //拿到报错的Entry对象，在get方法中会调用此方法。
    private Entry getEntry(ThreadLocal<?> key) {
            //拿到下标
            int i = key.threadLocalHashCode & (table.length - 1);
            //通过下标 拿到对象
            Entry e = table[i];
            //判断 满足条件则返回值，否则返回null
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
    }
    //如果get方法中的getMap方法返回null，或者拿到的值为null，则对当前线程进行初始化.
    //这个和set()方法基本一样，只不过他的Value一直是null。
    private T setInitialValue() {
        T value = initialValue();//始终返回null
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
}
```
简单的总结一下，其实和set非常相似，都是先 拿到当前线程对象，然后在去通过getMap()方法拿到当前线程中的ThreadLocal.ThreadLocalMap 的引用，如果为空，则表示没有保存过数据，直接调用setInitiaValue()对向前线程进行初始化，然后返回null。如果不为空，则通过getEntry()方法拿到保存的对象，判断这个对象不为空 就拿到保存的Value然后返回，否则 就调用setInitiaValue()进行初始化，然后返回null。

---
从ThreadLocal的set 和 get 方法可以看出，==他们操作的都是根据当前线程中的ThreadLocal.ThreadLocalMap threadLocals = null 来判断当前线程有没有保存数据，如果保存了，就会在当前线程中产生一个ThreadLocalMap 的对象 。数据就会保存在这个对象 里面，如果 没有保存过数据，那么当前线程中的ThreadLocalMap 就会为空==。他们对ThreadLocal 所做的读/写操作仅限于线程的内部。因此在不同线程中访问同一个ThreadLocal的 set 和 get 方法 所得到的值 也是不一样的。


> 参考自Android开发艺术探索，如有错误，还请指出，谢谢！