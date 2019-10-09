### 这是fragmentation 和 底部的导航栏 进行一些封装，效果就像是微信的页面，上面是碎片，下面是tab。使用了映射的关系，也就是键值对的意思，键对应tab ，键对应的fragmentation非常好看，使用起来非常方便，逻辑清楚，层次分明。效果如图

![1556271611706](assets\1556271611706.png)



这个框架使用的fragment 不是原生的 fragment ，而是 fragmentation。不会的也可以去查一下。其实这里使用fragment也是可以的。只需要改一下继承的类就好了。

还有ButterKnife ，使用注解生成 R文件，不需要findViewById。不会使用也没关系，直接按原来的来就好，会的话就更好了。



下面看一下实现：

1，抽象的基类，实际上就是fragment的基类

```java
public abstract class BaseDelegate extends SwipeBackFragment {

    @SuppressWarnings("SpellCheckingInspection")
    private Unbinder mUnbinder = null;

    /**
     * @return 可以是一个View ，也可以是一个Layout的Id，代表一个视图
     */
    public abstract Object setLayout();


    public abstract void onBindView(@Nullable Bundle savedInstanceState, View rootView);

    @Nullable
    @Override
    public View onCreateView(@NonNull LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        final View rootView;
        //返回的是 Layout的 id
        if (setLayout() instanceof Integer) {
            rootView = inflater.inflate((Integer) setLayout(), container, false);
        } else if (setLayout() instanceof View) {
            rootView = (View) setLayout();
        } else {
            throw new ClassCastException("setLayout() type must be int or view");
        }
        //绑定 资源（ButterKnife）
        mUnbinder = ButterKnife.bind(this, rootView);
        //子类 实现，传入 Bundle 和 碎片视图
        onBindView(savedInstanceState, rootView);

        return rootView;
    }

    /**
     * @return 返回一个 Activity
     */
    public final ProxyActivity getProxyActivity(){
        return (ProxyActivity) _mActivity;
    }

    @Override
    public void onDestroyView() {
        super.onDestroyView();
        if (mUnbinder != null) {
            mUnbinder.unbind();
        }
    }
}
```

```java
public abstract class LatteDelegate extends PermissionCheckerDelegate{
	//这里没用，但是可以在这里进行一些权限的操作
}
```

上面进行了一些简单的封装。

2,fragment 界面的基类，如果要显示某个页面，就需要继承自这个类

```java
/**
 * Copyright (C)
 *
 * @file: BottomItemDelegate
 * @author: 345
 * @Time: 2019/4/25 19:26
 * @description: 导航栏对应的 页面
 */
public abstract class BottomItemDelegate extends LatteDelegate implements View.OnKeyListener {
    private long mExitTime = 0;
    private static final int EXIT_TIME = 2000;

    @Override
    public void onResume() {
        super.onResume();
        final View rootView = getView();
        if (rootView != null) {
            rootView.setFocusableInTouchMode(true);
            rootView.requestFocus();
            rootView.setOnKeyListener(this);
        }
    }

    @Override
    public boolean onKey(View v, int keyCode, KeyEvent event) {
        if (keyCode == KeyEvent.KEYCODE_BACK && event.getAction() == KeyEvent.ACTION_DOWN) {
            if ((System.currentTimeMillis() - mExitTime) > EXIT_TIME) {
                Toast.makeText(_mActivity, "双击退出" + getString(com.wang.avi.R.string.app_name), Toast.LENGTH_SHORT).show();
                mExitTime = System.currentTimeMillis();
            } else {
                _mActivity.finish();
                if (mExitTime != 0) {
                    mExitTime = 0;
                }
            }
            return true;
        }
        return false;
    }
}
```

3，tab 的封装，也就是导航栏，其实就是一个bean类，导航栏有几个tab，就new几个对象。(注意这个的导航栏使用了矢量图 第三方库ionicons),不了解的可以先看一下[这篇文章](https://blog.csdn.net/baidu_40389775/article/details/89564368)

```java
**
 * Copyright (C)
 *
 * @file: BottomTabBean
 * @author: 345
 * @Time: 2019/4/25 19:40
 * @description: 导航栏 的Tab
 */
public final class BottomTabBean {
    /**
     * 图标
     */
    private final CharSequence ICON;
    /**
     * 文字
     */
    private final CharSequence TITLE;

    public BottomTabBean(CharSequence ICON, CharSequence TITLE) {
        this.ICON = ICON;
        this.TITLE = TITLE;
    }

    public CharSequence getIcon() {
        return ICON;
    }

    public CharSequence getTitle() {
        return TITLE;
    }
}
```

4,创建一个 ItemBuilder 类，用来创造键值对，相当于一个工厂类。用来让tab 和 fragment 产生 映射关系。

```java
/**
 * Copyright (C)
 *
 * @file: ItemBuilder
 * @author: 345
 * @Time: 2019/4/25 19:42
 * @description: 工厂类，用来构建 导航栏 和 碎片
 */
public final class ItemBuilder {
    private final LinkedHashMap<BottomTabBean,BottomItemDelegate> ITEMS = new LinkedHashMap<>();

    static ItemBuilder builder(){
        return new ItemBuilder();
    }

    public final ItemBuilder addItem(BottomTabBean bean,BottomItemDelegate delegate){
        ITEMS.put(bean,delegate);
        return this;
    }

    public final ItemBuilder addItem(LinkedHashMap<BottomTabBean,BottomItemDelegate> items){
        ITEMS.putAll(items);
        return this;
    }

    public final LinkedHashMap<BottomTabBean,BottomItemDelegate> build(){
        return ITEMS;
    }
}
```

5,对键值对 进行一些封装和使用。通果上面的ItemBuilder 类拿到要显示的所有 fragment和 tab 的类。然后进行一些逻辑的处理。这是一个抽象类。还需要一个子类来实现一些方法。比如setItems()方法 添加需要的 map 。对tab进行监听。对fragment进行处理。他的子类就是用来创建 所有的键值对的。

```java
/**
 * Copyright (C)
 *
 * @file: BaseBottomDelegate
 * @author: 345
 * @Time: 2019/4/25 19:24
 * @description: 对所有的键值对进行管理，也就是 碎片和tab，这是一个抽象类。
 */
public abstract class BaseBottomDelegate extends LatteDelegate implements View.OnClickListener {

    /**
     * 存储所有的子 Fragment
     */
    private final ArrayList<BottomItemDelegate> ITEM_DELEGATES = new ArrayList<>();
    /**
     * 存储所有的子 TabBean
     */
    private final ArrayList<BottomTabBean> TAB_BEANS = new ArrayList<>();

    /**
     * 存储 Fragment和TabBean 的映射
     */
    private final LinkedHashMap<BottomTabBean, BottomItemDelegate> ITEMS = new LinkedHashMap<>();

    /**
     * 当前Fragment 的位置
     */
    private int mCurrentDelegate = 0;
    /**
     * 进入程序展示 的Fragment
     */
    private int mIndexDelegate = 0;
    /**
     * Tab 的颜色
     */
    private int mClickedColor = Color.RED;

    /**
     * 底部 tab
     */
    @BindView(R2.id.bottom_bar)
    LinearLayoutCompat mBottomBar = null;

    /**
     * @param builder 要添加的 映射
     * @return  返回值为 LinkedHashMap
     */
    public abstract LinkedHashMap<BottomTabBean, BottomItemDelegate> setItems(ItemBuilder builder);

    @Override
    public Object setLayout() {
        return R.layout.delegate_bottom;
    }

    public abstract int setIndexDelegate();
    /**
     * 该注解表示 这必须是一个颜色
     */
    @ColorInt
    public abstract int setClickedColor();

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        mIndexDelegate = setIndexDelegate();
        if (setClickedColor() != 0){
            mClickedColor = setClickedColor();
        }
        //拿到工厂类的实例
        final ItemBuilder builder = ItemBuilder.builder();
        //获取 添加完成的键值对
        final LinkedHashMap<BottomTabBean,BottomItemDelegate> items = setItems(builder);
        //将 键值对 保存在ITEMS 中
        ITEMS.putAll(items);
        //拿到键和值
        for (Map.Entry<BottomTabBean,BottomItemDelegate> item  :ITEMS.entrySet()){
            final BottomTabBean key = item.getKey();
            final BottomItemDelegate value = item.getValue();
            TAB_BEANS.add(key);
            ITEM_DELEGATES.add(value);
        }
    }

    @Override
    public void onBindView(@Nullable Bundle savedInstanceState, View rootView) {
        final int size = ITEMS.size();

        for (int i = 0; i < size; i++) {
            //第一个参数 布局，第二个参数 为给第一个参数加载的布局 设置一个父布局
            LayoutInflater.from(getContext()).inflate(R.layout.bottom_item_icon_text_layout,mBottomBar);
            //返回指定的视图
            final RelativeLayout item = (RelativeLayout) mBottomBar.getChildAt(i);
            //设置每个 item的点击事件 和标记
            item.setTag(i);
            item.setOnClickListener(this);
            //拿到 item 的第一个和 第二个子布局
            final IconTextView itemIcon = (IconTextView) item.getChildAt(0);
            final AppCompatTextView itemTitle  = (AppCompatTextView) item.getChildAt(1);

            //获取 集合中对应的 Tab
            final BottomTabBean bean = TAB_BEANS.get(i);

            //初始化 tab 数据
            itemIcon.setText(bean.getIcon());
            itemTitle.setText(bean.getTitle());
            //判断是否是 当前显示
            if (i == mIndexDelegate){
                itemIcon.setTextColor(mClickedColor);
                itemTitle.setTextColor(mClickedColor);
            }
        }
        //返回一个数组，里边是fragment的。注意fragment 是继承 supportFragment 的，所以这里的集合是这个类型
        //fragmentation 需要我们这样做
        final SupportFragment[] delegateArry = ITEM_DELEGATES.toArray(new SupportFragment[size]);
        //加载多个根fragment ，并显示其中一个，第二个参数为要显示的fragment
        loadMultipleRootFragment(R.id.bottom_bar_delegate_container,mIndexDelegate,delegateArry);
    }

    /**
     * 重置所有颜色
     */
    private void resetColor(){
        //拿到 底部tab的子布局的size
        final int count = mBottomBar.getChildCount();
        for (int i = 0; i < count; i++) {
            final RelativeLayout item = (RelativeLayout) mBottomBar.getChildAt(i);
            final IconTextView itemIcon = (IconTextView) item.getChildAt(0);
            itemIcon.setTextColor(Color.GRAY);
            final AppCompatTextView itemTitle = (AppCompatTextView) item.getChildAt(1);
            itemTitle.setTextColor(Color.GRAY);
        }
    }

    @Override
    public void onClick(View v) {
        final int tag = (int) v.getTag();
        resetColor();
        final RelativeLayout item = (RelativeLayout) v;
        final IconTextView itemIcon = (IconTextView) item.getChildAt(0);
        itemIcon.setTextColor(mClickedColor);
        final AppCompatTextView itemTitle = (AppCompatTextView) item.getChildAt(1);
        itemTitle.setTextColor(mClickedColor);

        //第一次参数 为要显示的，第二个则是要隐藏的
        showHideFragment(ITEM_DELEGATES.get(tag),ITEM_DELEGATES.get(mCurrentDelegate));
        // 记住当前显示 的下标，注意顺序
        mCurrentDelegate = tag;
    }
}
```

这里有两个布局，一个是整个界面的布局，还有一个是 tab的子布局

delegate_bottom.xml

```java
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <android.support.v7.widget.ContentFrameLayout
        android:id="@+id/bottom_bar_delegate_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_above="@id/bottom_bar"/>

    <android.support.v7.widget.LinearLayoutCompat
        android:id="@+id/bottom_bar"
        android:layout_width="match_parent"
        android:layout_height="60dp"
        android:layout_alignParentBottom="true"
        android:orientation="horizontal"/>
</RelativeLayout>
```

bottom_item_icon_text_layout.xml

```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_weight="1"
    android:paddingBottom="6dp"
    android:paddingTop="6dp"
    android:layout_height="match_parent">

    <com.joanzapata.iconify.widget.IconTextView
        android:id="@+id/item_bottom_item"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_centerHorizontal="true"
        android:layout_alignParentTop="true"
        android:textSize="25sp"
        android:gravity="center"/>

    <android.support.v7.widget.AppCompatTextView
        android:id="@+id/tv_bottom_item"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:layout_centerHorizontal="true"
        android:layout_alignParentBottom="true"/>

</RelativeLayout>
```



6,要显示的 fragment，这里就是用户可以看见的界面了，这些会和对应的tab 一起存进map中，然后被 上面的类进行管理

```java
public class IndexDelegate extends BottomItemDelegate {
    @Override
    public Object setLayout() {
        return R.layout.delegate_index;
    }

    @Override
    public void onBindView(@Nullable Bundle savedInstanceState, View rootView) {

    }
}
```

```java
public class SortDelegate extends BottomItemDelegate {
    @Override
    public Object setLayout() {
        return R.layout.delegate_sort;
    }

    @Override
    public void onBindView(@Nullable Bundle savedInstanceState, View rootView) {

    }
```

这里只加了 两个碎片，有需要的可以多加。

对应布局如下

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <android.support.v7.widget.AppCompatTextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="首页"/>

</LinearLayout>
```

```java
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <android.support.v7.widget.AppCompatTextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="分类" />
</LinearLayout>
```

7,继承自抽象BaseBottomDelegate ，在这个类中 我们可以创建多个 键值对，这些会返回到 上面的第5点的那个类中，进行处理并进行显示。

```java
/**
 * Copyright (C)
 *
 * @file: EcBottomDelegate
 * @author: 345
 * @Time: 2019/4/26 14:26
 * @description: ${DESCRIPTION}
 */
public class EcBottomDelegate extends BaseBottomDelegate {
    @Override
    public LinkedHashMap<BottomTabBean, BottomItemDelegate> setItems(ItemBuilder builder) {
        final LinkedHashMap<BottomTabBean, BottomItemDelegate> items = new LinkedHashMap<>();
        items.put(new BottomTabBean("{fa-home}","主页"),new IndexDelegate());
        items.put(new BottomTabBean("{fa-sort}","分类"),new SortDelegate());
        items.put(new BottomTabBean("{fa-compass}","发现"),new IndexDelegate());
        items.put(new BottomTabBean("{fa-shopping-cart}","购物车"),new IndexDelegate());
        items.put(new BottomTabBean("{fa-user}","我的"),new IndexDelegate());
        return builder.addItem(items).build();
    }

    @Override
    public int setIndexDelegate() {
        return 0;
    }

    @Override
    public int setClickedColor() {
        return Color.parseColor("#ffff8800");
    }
}
```

在这个类里面可以对全局的 碎片进行管理。如果导航栏要增加一个 tab ，只需要创建一个字体的bean类，和一个碎片类存入map里面可以显示了。



经过上面的这几个步骤，这个通用的 导航栏+fragment 就弄好了。如果以后我们有任何更改的话，就可以在 BaseBottomDelegate类里面进行更改。如果要添加tab 直接new 字体的bean 在创建一个 fragment就好。如果每个每个碎片中有相同的功能，还可以在页面的基类BottomItemDelegate 中进行更改。目前这个类中只加了 双击退出的功能

效果如下所示

![1556281108751](F:\笔记\android\Fragment\assets\1556281108751.png)



![1556281129970](F:\笔记\android\Fragment\assets\1556281129970.png)

![1556282286973](F:\笔记\android\Fragment\assets\1556282286973.png)