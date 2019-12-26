​	 **ViewPager2取代了ViewPager，解决了它的前身的大部分难点，包括从右到左的布局支持、垂直方向、可修改的片段集合等。** 

​	  上个月的 20 号(19/11-20) ViewPager2 正式发布，用来取代 ViewPager，并且解决了很多的问题，因为原来还没有正式发布，也没有学习，今天正式的学一下

​	首先，Viewpager2 是在 AndroidX 的库中，没有被内置到系统源码中，如果没有迁移 AndroidX 的赶紧迁移。下面看一下使用方式

#### 1，添加依赖

```java
implementation "androidx.viewpager2:viewpager2:1.0.0"
```

#### 2，布局

```xml
<androidx.viewpager2.widget.ViewPager2
    android:id="@+id/page2"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

​		item 布局

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/page2_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

</LinearLayout>
```

​		没错，他就是个空的。只是用来展示一下效果而已。

#### 3，适配器

​		ViewPage2 内部是基于 RecyclerView 实现的，意味着 Rv 的所有优点都将被继承，ViewPager 的适配器也和 Rv 是一样的。

```java
public class Page2Adapter extends RecyclerView.Adapter<Page2Adapter.ViewHolder> {

    public List<String> colors;

    public Page2Adapter(List<String> colors) {
        this.colors = colors;
    }

    @NonNull
    @Override
    public ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext()).inflate(R.layout.page2_layout, parent, false);
        return new ViewHolder(view);
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        holder.layout.setBackgroundColor(Color.parseColor(colors.get(position)));
    }

    @Override
    public int getItemCount() {
        return colors.size();
    }

    class ViewHolder extends RecyclerView.ViewHolder {
        LinearLayout layout;

        public ViewHolder(@NonNull View itemView) {
            super(itemView);
            layout = itemView.findViewById(R.id.page2_layout);
        }
    }

}

```

​		这段代码就不用解释了。

#### 4，使用

```java
List<String> colors = new ArrayList<>();
colors.add("#FFFF00");
colors.add("#FF0000");
colors.add("#AAFF00");
colors.add("#FF44FF");
colors.add("#EEFFEE");
colors.add("#EEE000");

ViewPager2 pager2 = findViewById(R.id.page2);
Page2Adapter adapter = new Page2Adapter(colors);
pager2.setAdapter(adapter);
```

​		效果如下：

#### 5，常用的属性：

- 切换为上下滑动

  ```java
  pager2.setOrientation(ViewPager2.ORIENTATION_VERTICAL);
  ```

- 添加事件

  ```java
   pager2.registerOnPageChangeCallback(new ViewPager2.OnPageChangeCallback() {
              @Override
              public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
                  super.onPageScrolled(position, positionOffset, positionOffsetPixels);
                  //当滚动当前页面时，将调用此方法
              }
  
              @Override
              public void onPageSelected(int position) {
                  super.onPageSelected(position);
                  //当选择新页面时，将调用此方法。动画不是一定完成。
              }
  
              @Override
              public void onPageScrollStateChanged(int state) {
                  super.onPageScrollStateChanged(state);
                  //当滚动状态改变时调用。
              }
          });
  ```

- 禁止滑动

  ```java
  pager2.setUserInputEnabled(false);
  ```

- 模拟拖拽

  ```java
  findViewById(R.id.scroll_btn).setOnClickListener(new View.OnClickListener() {
              @Override
              public void onClick(View v) {
                  pager2.beginFakeDrag();
                  if (pager2.fakeDragBy(-800)) {
                      pager2.endFakeDrag();
                  }
              }
          });
  ```

  ​		其中 beginFakeDrag() 方法时开启拖拽，fakeDragBy() 返回一个 boolean 类型，执行了拖动返回 true，否则返回 false。他还有一个参数，代表的是偏移量，负数表示向下一个页面滑动，反正则是上一个页面。endFakeDarg 是结束虚拟的拖拽。

- 设置显示的 item

  ```java
  pager2.setCurrentItem(1);
  ```

- 获取适配器

  ```java
  pager2.getAdapter();
  ```

- 设置缓存

  ```java
  pager2.setOffscreenPageLimit(1);
  ```



------



### PageTransformer

​	 无论何时滚动可见/附加页面，都会调用PageTransformer。这为应用程序提供了使用动画属性将自定义转换应用到页面视图的机会 

​	ViewPager2 可以通过 PageTransformer 来设置页面动画，还可以 设置间距及同时添加多个 PageTransformer。

​	PageTransformer 是一个接口，内部只有一个方法

```java
void transformPage(@NonNull View page, float position);
```

​	通过实现这个接口，即可自定义 ViewPager2 页面的切换动画

​	page ：将属性转换应用于给定页面

​	position ：这个参数非常重要。下面看一下具体的意思：

​	position 在 -1到 0 之间，代表向左滑动时，左侧滑出的 page 。或者向右滑动时 左侧滑入的page

​	position 在 0 到 1 之间，代码向左滑动时，右侧滑出的 page。或者向右滑动时，右侧滑出的page

​	position 小于 1 ，代表当前 page 最左侧 page。position 大于 1， 代表当前 page 最右侧的 page。(需要排除正在滑入和滑出的两个page)

​	下面进入正题，设置 动画和 间距

- 设置间距

  ```java
  pager2.setPageTransformer(new MarginPageTransformer(10));
  ```

- 设置动画

  设置动画也是上面的方法，那间距怎么设置呢，其实他们可以一起设置的，如下；

  ```java
  CompositePageTransformer compositePageTransformer = new CompositePageTransformer();
  compositePageTransformer.addTransformer(new MarginPageTransformer(10));
  compositePageTransformer.addTransformer(new TransFormer());
  pager2.setPageTransformer(compositePageTransformer);
  ```

  ```java
  class TransFormer implements ViewPager2.PageTransformer {
  
          @Override
          public void transformPage(@NonNull View page, float position) {
  
              if (position >= -1.0f && position <= 0.0f) {
                  //控制左侧滑入或者滑出的缩放比例
                  page.setScaleX(1 + position * 0.1f);
                  page.setScaleY(1 + position * 0.2f);
              } else if (position > 0.0f && position < 1.0f) {
                  //控制右侧滑入或者滑出的缩放比例
                  page.setScaleX(1 - position * 0.1f);
                  page.setScaleY(1 - position * 0.2f);
              } else {
                  //控制其他View缩放比例
                  page.setScaleX(0.9f);
                  page.setScaleY(0.8f);
              }
          }
      }
  ```

- 设置一屏多页的效果

  ```java
   pager2.setOffscreenPageLimit(1);
   RecyclerView recyclerView = (RecyclerView) pager2.getChildAt(0);
   int padding = getResources().getDimensionPixelOffset(R.dimen._15) + getResources().getDimensionPixelOffset(R.dimen._15);
   recyclerView.setPadding(padding, 0, padding, 0);
   recyclerView.setClipToPadding(false);
  ```

  

------



### ViewPager2 与 Fragment

​		在 ViewPager 2中新增了  FragmentStateAdapter  替代了 ViewPager 的 FragmentStatePagerAdapter 。，下面看一下简单的用法

```java
public class Page2FragmentAdapter extends FragmentStateAdapter {

    private List<Fragment> list;

    public static Page2FragmentAdapter start(FragmentActivity fragmentActivity, List<Fragment> fragments) {
        return new Page2FragmentAdapter(fragmentActivity, fragments);
    }

    public Page2FragmentAdapter(@NonNull FragmentActivity fragmentActivity, List<Fragment> fragments) {
        super(fragmentActivity);
        this.list = fragments;
    }

    @NonNull
    @Override
    public Fragment createFragment(int position) {
        return list.get(position);
    }

    @Override
    public int getItemCount() {
        return list.size();
    }
}

```

```java
List<Fragment> frags = new ArrayList<>();
frags.add(frag1);
frags.add(frag2);
frags.add(frag3);
final ViewPager2 pager2 = findViewById(R.id.page2);
pager2.setAdapter(Page2FragmentAdapter.start(this, frags));
pager2.setOffscreenPageLimit(1);
```

​		其实和原来的差不多

​		在打印日志后发现，如果不开启缓存，ViewPager2 不会默认缓存旁边的 Fragment。

------



### ViewPager2 的特性：

- 基于 RecyclerView
- 支持 竖屏滑动
- 可禁止用户滑动页面
- 可模拟用户滑动
- 可同时添加多个  PageTransformer 

当然新的特性不止这么几个，更多的需要自己去发现哈。。。