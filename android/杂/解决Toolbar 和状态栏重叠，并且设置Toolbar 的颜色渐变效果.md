动态的解决 toolbar 和 状态栏 重叠,并且设置 toolbar 的渐变：

1,解决toolbar的 重叠，自动适配屏幕

只需要 获取一下状态栏的高度，然后让toobar 位于他的下方即可。

```java
/**
 * Copyright (C)
 *
 * @file: StatusBarHeight
 * @author: 345
 * @Time: 2019/4/30 11:39
 * @description: 获取 状态栏的高度 和转换
 */
public class StatusBarHeight {

    /**
     * @return 返回 状态栏的 高度，以像素为单位
     */
    public static int getStaticBarHeight(){
        int result = 0;
        int resourceId = Latte.getApplication().getResources().getIdentifier("status_bar_height",
                "dimen","android");
        if (resourceId > 0){
            result = Latte.getApplication().getResources().getDimensionPixelSize(resourceId);
        }
        return result;
    }

    /**
     *  根据手机的分辨率 从 px(像素)的单位 转换为 dp
     * @param pxValue
     * @return
     */
    public static int px2dip(float pxValue){
        final float scale = Latte.getApplication().getResources().getDisplayMetrics().density;
        return (int) (pxValue/scale+0.5f);
    }

    /**
     * 根据手机的分辨路 从 dp单位 转换为 px(像素)
     */
    private static int dip2px(float dipValue){
        final float scale = Latte.getApplication().getResources().getDisplayMetrics().density;
        return (int) (dipValue*scale+0.5f);
    }
}
```

```java
//获取 状态栏的高度，以像素为单位
int statusBar = StatusBarHeight.getStaticBarHeight();
//设置toolbar 位于 状态栏 的下方，并且距底部 20个像素
//注意：toolbar的布局中高度只能是 wrap_content
mToobar.setPadding(0,statusBar,0,20);
```

注意 布局中 toolbar 的高度 不能是死值，要设置为 wrap_content

2,在 RecyclerView 中设置toolbar 的渐变

```java
mRecyclerView.addOnScrollListener(new RecyclerView.OnScrollListener() {
    @Override
    public void onScrollStateChanged(@NonNull RecyclerView recyclerView, int newState) {
        super.onScrollStateChanged(recyclerView, newState);
    }

    @SuppressLint("ResourceAsColor")
    @Override
    public void onScrolled(@NonNull RecyclerView recyclerView, int dx, int dy) {
        super.onScrolled(recyclerView, dx, dy);
        //判断是 RecyclerView 是否位于顶部
        boolean flag = recyclerView.canScrollVertically(-1);
        int  toolbarHeight = mToobar.getHeight();
        mDy += dy;
        if (!flag){
            //位于顶部 ，则设置颜色 透明
            mToobar.setBackgroundColor(android.R.color.transparent);
        }
        else {
            //否则 设置渐变
            if (mDy >toolbarHeight){
                mToobar.setBackgroundColor(Color.rgb(255,124,2));
            }else if (mDy >0&& mDy <= toolbarHeight) {
                final float scale =(float) mDy / toolbarHeight;
                final float alpha = scale * 255;
                mToobar.setBackgroundColor(Color.argb((int)alpha,255,124,2));
            }
        }
    }
});
```

其中 mDy 为 全局int 变量，初始值 为0；

