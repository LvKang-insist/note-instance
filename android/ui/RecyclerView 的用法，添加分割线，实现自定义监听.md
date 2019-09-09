RecyclerView 是一个非常强大的控件，他可以说是一个增强版的listView，不仅可以轻松实现listview的效果，而且还增加了很多的效果。通过设置不同的LayoutManager、ItemDecoration、ItemAnimator可以实现丰富的效果。

### 一、RecyclerView的简单使用


---
1，配置buid.gradle

```
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'com.android.support:appcompat-v7:28.0.0'
    implementation 'com.android.support.constraint:constraint-layout:1.1.3'
    implementation 'com.android.support:recyclerview-v7:28.0.0'
    testImplementation 'junit:junit:4.12'
    ......
}

```
2，在布局中加入RecyclerView控件，然后创建一个RecyclerView的布局文件item_recycler。

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <android.support.v7.widget.RecyclerView
        android:id="@+id/id_recyclerview"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

</LinearLayout>
```

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="vertical">

    <TextView
        android:id="@+id/tv_item"
        android:layout_width="match_parent"
        android:layout_height="50dp"
        android:gravity="center"
        android:text="moon"/>
</LinearLayout>
```


3,使用RecyclerView。

```
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        List<String> list = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            list.add(String.valueOf(i));
        }
        RecyclerView recyclerView = findViewById(R.id.id_recyclerview);
        //设置线性布局管理器
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        //设置item增加和删除时的动画
        recyclerView.setItemAnimator(new DefaultItemAnimator());

        HomeAdapter adapter = new HomeAdapter(list);
        recyclerView.setAdapter(adapter);

    }

    class HomeAdapter extends RecyclerView.Adapter<HomeAdapter.MyViewHolder> {

        List<String> list;

        public HomeAdapter(List<String> list) {
            this.list = list;
        }


        //移除元素
        public void removeData(int position){
            list.remove(position);
            notifyItemRemoved(position);
        }
    
        //加载条目布局
        @Override
        public HomeAdapter.MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
            MyViewHolder holder = new MyViewHolder(LayoutInflater.from(MainActivity.this).
                        inflate(R.layout.item_recycler,parent,false));
            return holder;
        }

       
       //将条目和数据进行绑定
        @Override
        public void onBindViewHolder(HomeAdapter.MyViewHolder holder, int position) {
            holder.tv.setText(list.get(position));
        }

        //条目的总数量
        @Override
        public int getItemCount() {
            return list.size();
        }
        //内部类，构造函数接收一个View参数，这个参数通常就是条目
        class MyViewHolder extends RecyclerView.ViewHolder {
            TextView tv;

            public MyViewHolder(View itemView) {
                super(itemView);
                tv = itemView.findViewById(R.id.tv_item);
            }
        }
    }
}

```
需要说的就是上面的布局管理器，我们这里使用的是线性布局的方式，当然也可以使用其他的方式，比如：

```
        //设置水平线性布局管理器
        LinearLayoutManager linearLayoutManager = new LinearLayoutManager(this);
        linearLayoutManager.setOrientation(LinearLayoutManager.HORIZONTAL);
        recyclerView.setLayoutManager(linearLayoutManager);
        
        //还有网格的方式
        StaggeredGridLayoutManager manager = new StaggeredGridLayoutManager(3,StaggeredGridLayoutManager.VERTICAL);
        recyclerView.setLayoutManager(manager);
        
```

---
### 二、设置分割线，直接上代码

```
public class DividerItemDecoration extends RecyclerView.ItemDecoration {
    private static final int[] ATTRS = new int[]{
        android.R.attr.listDivider
    };

    public static final int HORIZONTAL_LIST = LinearLayoutManager.HORIZONTAL;
    public static final int VERTICAL_LIST = LinearLayoutManager.VERTICAL;

    private Drawable mDivider;
    private int mOrientation;

    public DividerItemDecoration(Context context ,int orientation){
        final TypedArray a = context.obtainStyledAttributes(ATTRS);
        mDivider = a.getDrawable(0);
        setOrientation(orientation);
    }

    private void setOrientation(int orientation) {
        if (orientation != HORIZONTAL_LIST && orientation != VERTICAL_LIST){
            throw new IllegalArgumentException("invaild orientation");
        }
        mOrientation = orientation;
    }

    @Override
    public void onDraw(@NonNull Canvas c, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        if (mOrientation == VERTICAL_LIST){
            drawVertical(c,parent);
        }else {
            drawHorizontal(c,parent);
        }
    }

    private void drawHorizontal(Canvas c, RecyclerView parent) {
        final int top = parent.getPaddingTop();
        final  int bottom = parent.getHeight()-parent.getPaddingBottom();
        final  int childCount = parent.getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View child = parent.getChildAt(i);
            final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child.getLayoutParams();
            final int left = child.getRight()+params.rightMargin;
            final int right = left + mDivider.getIntrinsicWidth();
            mDivider.setBounds(left ,top,right ,bottom);
            mDivider.draw(c);
        }
    }

    private void drawVertical(Canvas c, RecyclerView parent) {
        final int left = parent.getPaddingLeft();
        int right = parent.getWidth()-parent.getPaddingRight();
        final int childCound = parent.getChildCount();
        for (int i = 0; i < childCound; i++) {
            final View child = parent.getChildAt(i);
            final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams)child.getLayoutParams();
            final int top= child.getBottom() + params.bottomMargin;
            final int bottom = top + mDivider.getIntrinsicHeight();
            mDivider.setBounds(left,top,right,bottom);
            mDivider.draw(c);
        }
    }


    @Override
    public void getItemOffsets(@NonNull Rect outRect, @NonNull View view, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        if (mOrientation == VERTICAL_LIST){
            outRect.set(0,0,0,mDivider.getIntrinsicHeight());
        }else {
            outRect.set(0,0,mDivider.getIntrinsicWidth(),0);
        }
    }
}

```
这里的核心方法是onDraw方法，他根据传进来的orientation来判断是绘制横向的item和分割线还是纵向的分割线。其中，drawHorizontal用于绘制横向的item的分割线，drawVerical用于绘制纵向的item的分割线，getItemOffsets方法则用于设置item的padding属性，虽然没有默认的分割线，但是好处也发现了，我们可以更灵活的自定义分割线，实现自定义的分割线。

我们只需要在setAdapter之前加入入下代码便可以加入分割线。

```
        recyclerView.addItemDecoration(new DividerItemDecoration(this,DividerItemDecoration.VERTICAL_LIST));

```

---
### 三，自定义点击事件

在适配器中定义接口并提供回调。这里我们定义了条目的点击事件和长按事件。

1，首先定义接口。

```
public class MainActivity extends AppCompatActivity {

    public interface OnItemClickListener {
        void onItemClick(View view, int position);
        void onItemLogClick(View view, int position);
    }
    .......

```
2,在适配器类中创建该接口的引用，并创建回调方法。

```
        private OnItemClickListener onItemClickListener;
        //回调方法
        public void setOnItemClickListener(OnItemClickListener onItemClickListener) {
            this.onItemClickListener = onItemClickListener;
        }
```
3，在适配器类中对每个item进行监听，并且将事件回调给我们的自定义监听。

```
class HomeAdapter extends RecyclerView.Adapter<HomeAdapter.MyViewHolder> implements View.OnClickListener, View.OnLongClickListener {

        private OnItemClickListener onItemClickListener;
        //回调方法
        public void setOnItemClickListener(OnItemClickListener onItemClickListener) {
            this.onItemClickListener = onItemClickListener;
        }
        List<String> list;

        public HomeAdapter(List<String> list) {
            this.list = list;
        }

        //移除元素
        public void removeData(int position) {
            list.remove(position);
            notifyItemRemoved(position);
        }
        @Override
        public HomeAdapter.MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
            View view = LayoutInflater.from(MainActivity.this).
                    inflate(R.layout.item_recycler, parent, false);
            MyViewHolder holder = new MyViewHolder(view);
            //实现监听
            view.setOnClickListener(this);
            view.setOnLongClickListener(this);
            return holder;
        }

        @Override
        public void onBindViewHolder(HomeAdapter.MyViewHolder holder, int position) {
            //对每一个条目设置标记
            holder.itemView.setTag(position);
            holder.tv.setText(list.get(position));
        }

        @Override
        public int getItemCount() {
            return list.size();
        }
        
        @Override
        public void onClick(View v) {
            //如果不为空则回调监听
            if (onItemClickListener != null) {
                onItemClickListener.onItemClick(v, (int) v.getTag());
            }
        }
        //如果回调使用长时间单击，则为true，否则为false。
        @Override
        public boolean onLongClick(View v) {
            //如果不为空则回调监听
            if (onItemClickListener != null) {
                onItemClickListener.onItemLogClick(v, (int) v.getTag());
                return true;
            }
            return false;
        }


        //内部类，构造函数接收一个View参数，这个参数通常就是条目
        class MyViewHolder extends RecyclerView.ViewHolder {
            TextView tv;
            public MyViewHolder(View itemView) {
                super(itemView);
                tv = itemView.findViewById(R.id.tv_item);
            }
        }
    }
```
4，最后我们在activity中实现我们的自定义监听

```
 adapter.setOnItemClickListener(new OnItemClickListener() {
            @Override
            public void onItemClick(View view, int position) {
                Toast.makeText(MainActivity.this, "点击了"+position+"条", Toast.LENGTH_SHORT).show();

            }
            @Override
            public void onItemLogClick(View view, final int position) {
                new AlertDialog.Builder(MainActivity.this)
                        .setTitle("确定删除吗?")
                        .setNeutralButton("取消",null)
                        .setPositiveButton("确定", new DialogInterface.OnClickListener() {
                            @Override
                            public void onClick(DialogInterface dialog, int which) {
                                adapter.removeData(position);
                            }
                        })
                        .show();
            }
        });
```
长按会弹出对话框，删除时会有消失的动画。如图所示：




---
### 最后贴出全部的代码。布局就不贴了哈。分割线的代码在上面有！

```
public class MainActivity extends AppCompatActivity {

    public interface OnItemClickListener {
        void onItemClick(View view, int position);

        void onItemLogClick(View view, int position);
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        List<String> list = new ArrayList<>();
        for (int i = 0; i < 20; i++) {
            list.add(String.valueOf(i));
        }
        RecyclerView recyclerView = findViewById(R.id.id_recyclerview);
        //设置线性布局管理器
        recyclerView.setLayoutManager(new LinearLayoutManager(this));
        //设置item增加和删除时的动画
        recyclerView.setItemAnimator(new DefaultItemAnimator());

        //添加分割线
        recyclerView.addItemDecoration(new DividerItemDecoration(this, DividerItemDecoration.VERTICAL_LIST));

        final HomeAdapter adapter = new HomeAdapter(list);
        recyclerView.setAdapter(adapter);


        adapter.setOnItemClickListener(new OnItemClickListener() {
            @Override
            public void onItemClick(View view, int position) {
                Toast.makeText(MainActivity.this, "点击了" + position + "条", Toast.LENGTH_SHORT).show();

            }

            @Override
            public void onItemLogClick(View view, final int position) {
                new AlertDialog.Builder(MainActivity.this)
                        .setTitle("确定删除吗?")
                        .setNeutralButton("取消", null)
                        .setPositiveButton("确定", new DialogInterface.OnClickListener() {
                            @Override
                            public void onClick(DialogInterface dialog, int which) {
                                adapter.removeData(position);
                            }
                        })
                        .show();
            }
        });

    }

    class HomeAdapter extends RecyclerView.Adapter<HomeAdapter.MyViewHolder> implements View.OnClickListener, View.OnLongClickListener {

        private OnItemClickListener onItemClickListener;

        //回调方法
        public void setOnItemClickListener(OnItemClickListener onItemClickListener) {
            this.onItemClickListener = onItemClickListener;
        }

        List<String> list;

        public HomeAdapter(List<String> list) {
            this.list = list;
        }

        //移除元素
        public void removeData(int position) {
            list.remove(position);
            notifyItemRemoved(position);
        }

        @Override
        public HomeAdapter.MyViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
            View view = LayoutInflater.from(MainActivity.this).
                    inflate(R.layout.item_recycler, parent, false);
            MyViewHolder holder = new MyViewHolder(view);
            //实现监听
            view.setOnClickListener(this);
            view.setOnLongClickListener(this);
            return holder;
        }

        @Override
        public void onBindViewHolder(HomeAdapter.MyViewHolder holder, int position) {
            //对每一个条目设置标记
            holder.itemView.setTag(position);
            holder.tv.setText(list.get(position));
        }

        @Override
        public int getItemCount() {
            return list.size();
        }

        @Override
        public void onClick(View v) {
            //如果不为空则回调监听
            if (onItemClickListener != null) {
                onItemClickListener.onItemClick(v, (int) v.getTag());
            }
        }

        //如果回调使用长时间单击，则为true，否则为false。
        @Override
        public boolean onLongClick(View v) {
            //如果不为空则回调监听
            if (onItemClickListener != null) {
                onItemClickListener.onItemLogClick(v, (int) v.getTag());
                return true;
            }
            return false;
        }


        //内部类，构造函数接收一个View参数，这个参数通常就是条目
        class MyViewHolder extends RecyclerView.ViewHolder {
            TextView tv;

            public MyViewHolder(View itemView) {
                super(itemView);
                tv = itemView.findViewById(R.id.tv_item);
            }
        }
    }
}

```

> 如有错误，还请指出，谢谢！