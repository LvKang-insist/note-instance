使用第三方的 RecyclerView 适配器

1,导入 第三方库、

```java
api 'com.github.CymChad:BaseRecyclerViewAdapterHelper:+'
```

2,创建 holder

```java
public class MultipleViewHolder extends BaseViewHolder {

    public MultipleViewHolder(View view) {
        super(view);
    }

    public static MultipleViewHolder create(View view){
        return new MultipleViewHolder(view);
    }
}
```

2,创建数据，

```java
public class MultipleItemEntity implements MultiItemEntity {

    private final LinkedHashMap<Object, Object> MULTIPLE_FIELDS = new LinkedHashMap<>();
    private final ReferenceQueue<LinkedHashMap<Object, Object>> ITEM_QUENE = new ReferenceQueue<>();


    /**
     * 软引用：SoftReference
     *       如果内存空间足够，垃圾回收器绝不会回收他，如果内存不足，则会回收这些对象的内存。
     */
    private final SoftReference<LinkedHashMap<Object, Object>> FIELDS_RETERENCE
            = new SoftReference<>(MULTIPLE_FIELDS, ITEM_QUENE);

    MultipleItemEntity(LinkedHashMap<Object,Object> fields) {
        FIELDS_RETERENCE.get().putAll(fields);
    }

    /**
     * 使用 建造者来构建数据
     */
    public static MultipleltemEntityBuilder builder(){
        return new MultipleltemEntityBuilder();
    }

    /**
     * 控制RecyclerView 中每一个item的样式和他的表现特征,强制实现
     * 这个type 用于区分当前数据模型需要展示在哪一种 item 之上
     *
     *   之后使用addItemType(int type ,int layoutResld)方法中输入对应的参数，
     *   你有几个布局，就写几个addItemType ，如果你的List中有10中不同的数据类型，
     *   但是你只写了8种，一定会报错的。
     *
     */
    @Override
    public int getItemType() {
        //返回每条数据的类型  ITEM_TYPE 为键
        return (int) FIELDS_RETERENCE.get().get(MultipleFields.ITEM_TYPE);
    }

    /**
     * @param key 要查询的键
     * @param <T> 返回类型
     * @return 返回根据键查询到的结果
     */
    public final <T> T getField(Object key){
        return (T)FIELDS_RETERENCE.get().get(key);
    }

    /**
     * @return 返回数据集合
     */
    public final LinkedHashMap<?,?> getFields(){
        return FIELDS_RETERENCE.get();
    }

    /**
     * 添加 数据
     * @param key 键
     * @param value 值
     * @return 返回当前对象
     */
    public final MultipleItemEntity setField(Object key, Object value){
        FIELDS_RETERENCE.get().put(key,value);
        return this;
    }
}
```

上面这个类 相当于一个数据集，RecyclerView 要显示的数据都在这里。数据在软引用中的linkedHashMap中保存

3，创建适配器类

```java
/**
 * Copyright (C)
 *
 * @file: MultipleRecyclerAdapter
 * @author: 345
 * @Time: 2019/4/27 17:34
 * @description: RecyclerView 的适配器
 */

public class MultipleRecyclerAdapter extends
        BaseMultiItemQuickAdapter<MultipleItemEntity, MultipleViewHolder>
        implements BaseQuickAdapter.SpanSizeLookup, OnItemClickListener {

    /**
     * 确保初始化一次Banner，防止重复加载
     */
    private boolean mIsInitBanner = false;

    public MultipleRecyclerAdapter(List<MultipleItemEntity> data) {
        super(data);
        //初始化布局
        init();
    }

    public static MultipleRecyclerAdapter create(List<MultipleItemEntity> data) {
        return new MultipleRecyclerAdapter(data);
    }
    public static MultipleRecyclerAdapter create(DataConverter converter) {
        return new MultipleRecyclerAdapter(converter.convert());
    }

    private void init() {
        /*
         * 设置不同的item 布局
         * addItemType 中的type 类型，必须和接收到的类型一模一样。
         * 种类：有几种type ，就需要写几个addItemType，少些或者错写都会直接保存》
         * 报错类型(javax.net.ssl.SSLHandshakeException: SSL handshake aborted: ......)
         */
        addItemType(ItemType.TEXT, R.layout.item_multiple_text);
        addItemType(ItemType.IMAGE, R.layout.item_multiple_image);
        addItemType(ItemType.TEXT_IMAGE, R.layout.item_multiple_image_text);
        addItemType(ItemType.BANNER, R.layout.item_multiple_banner);
        //设置 宽度监听
        setSpanSizeLookup(this);
        //加载时打开动画
        openLoadAnimation();
        //多次执行动画
        isFirstOnly(false);
    }

    /**
     * 创建ViewHolder
     * @param view
     * @return
     */
    @Override
    protected MultipleViewHolder createBaseViewHolder(View view) {
        return MultipleViewHolder.create(view);
    }

    /**
     * 现此方法，并使用helper将视图调整为给定项
     */
    @Override
    protected void convert(MultipleViewHolder holder, MultipleItemEntity entity) {
        final String text;
        final String imageUrl;
        final ArrayList<String> bannerImages;
        //根据不同的 type 设置不同的数据
        switch (holder.getItemViewType()) {
            case ItemType.TEXT:
                text = entity.getField(MultipleFields.TEXT);
                holder.setText(R.id.text_single, text);
                break;
            case ItemType.IMAGE:
                imageUrl = entity.getField(MultipleFields.IMAGE_URL);
                Glide.with(mContext)
                        .load(imageUrl)
                        /*
                         * 图片的缓存：
                         * DiskCacheStrategy.NONE 什么都不缓存
                         * DiskCacheStrategy.SOURCE 只缓存全尺寸图
                         * DiskCacheStrategy.RESULT 只缓存最终的加载图
                         * DiskCacheStrategy.ALL 缓存所有版本图（默认行为）
                         */
                        .diskCacheStrategy(DiskCacheStrategy.ALL)
                        .dontAnimate()
                        //将他图片按比例缩放到足以填充imageView 的尺寸,但是图片可能显示不完整
                        .centerCrop()
                        //将图片缩放到小于等于imageView的尺寸，这样图片会完整显示但是 imageView 就可能填不满了
                        .fitCenter()
                        .into((ImageView) holder.getView(R.id.img_single));
                break;
            case ItemType.TEXT_IMAGE:
                text = entity.getField(MultipleFields.TEXT);
                imageUrl = entity.getField(MultipleFields.IMAGE_URL);
                Glide.with(mContext)
                        .load(imageUrl)
                        .diskCacheStrategy(DiskCacheStrategy.ALL)
                        .dontAnimate()
                        .centerCrop()
                        .into((ImageView) holder.getView(R.id.img_single));
                holder.setText(R.id.tv_multiple,text);
                break;
            case ItemType.BANNER:
                if (!mIsInitBanner){
                    bannerImages = entity.getField(MultipleFields.BANNERS);
                    final ConvenientBanner<String> convenientBanner = holder.getView(R.id.banner_recycler_item);
                    BannerCreator.setDefault(convenientBanner,bannerImages,this);
                    mIsInitBanner = true;
                }
                break;
            default:
                break;
        }
    }
    /**
     * 设置宽度
     */
    @Override
    public int getSpanSize(GridLayoutManager gridLayoutManager, int position) {
        return getData().get(position).getField(MultipleFields.SPAN_SIZE);
    }
    @Override
    public void onItemClick(int position) {

    }
}
```

```java
/**
 * Copyright (C)
 *
 * @file: ItemType
 * @author: 345
 * @Time: 2019/4/27 15:46
 * @description: 数据类型
 */
public class ItemType {
    public static final int TEXT = 1;
    public static final int IMAGE =2;
    public static final int TEXT_IMAGE=3;
    /**
     * 轮播图
     */
    public static final int BANNER =4;
}
```

4,解析数据

首先看一下数据类型

```java
{
    "code": 0,
    "message": "OK",
    "total": 100,
    "page_size": 6,
    "data": [
    	{
    		"goodsId": 0,
    		"spanSize": 4,
  			"banners": [
    			"http://47.106.101.44:8080/data/imagedata/imagerotation/1.png",
"http://47.106.101.44:8080/data/imagedata/imagerotation/2.png"
   			]
		},
		{
            "goodsId": 1,
            "text": "现货热卖",
            "spanSize": 4
		},
		{
            "goodsId": 2,
            "imageUrl" :"http://47.106.101.44:8080/data/imagedata/homepage/1four.png",
            "spanSize": 4
		},
		{
            "goodsId": 3,
            "imageUrl" :"http://47.106.101.44:8080/data/imagedata/homepage/5two.png",
            "spanSize": 2
		},
		{
            "goodsId": 4,
            "imageUrl" :"http://47.106.101.44:8080/data/imagedata/homepage/3two.png",
            "spanSize": 2
		},
		............
```

其中 goodsId  表示 id，imageurl 是图片的地址。spanSize 表示所占的位置，这个非常重要。

解析

```java
public abstract class DataConverter {

    /**
     *  convert 方法解析完数据后 会将结果存进这个集合中
     */
    protected final ArrayList<MultipleItemEntity> ENTITLES = new ArrayList<>();
    private String mJsonData = null;

    /**
     *解析数据
     */
    public abstract ArrayList<MultipleItemEntity> convert();

    public DataConverter setJsonData(String json){
        this.mJsonData = json;
        return this;
    }

    protected String getJsonData(){
        if (mJsonData == null || mJsonData.isEmpty()){
            throw new NullPointerException("DATA IS NULL");
        }
        return mJsonData;
    }

}
```

```java
public class IndexDataConverter extends DataConverter {
    @Override
    public ArrayList<MultipleItemEntity> convert() {
        final JSONArray dataArray = JSON.parseObject(getJsonData()).getJSONArray("data");
        int size = dataArray.size();
        for (int i = 0; i < size; i++) {

            final JSONObject data = dataArray.getJSONObject(i);

            final String imageUrl = data.getString("imageUrl");
            final String text = data.getString("text");
            final int spanSize = data.getInteger("spanSize");
            final int id = data.getInteger("goodsId");
            final JSONArray banners = data.getJSONArray("banners");

            final ArrayList<String> bannerImages = new ArrayList<>();
            int type = 0;
            //text类型 ：共有4种
            if (imageUrl == null && text != null){
                type = ItemType.TEXT;
                //image类型
            }else if (imageUrl != null && text == null){
                type = ItemType.IMAGE;
                //imageText类型
            }else if (imageUrl != null){
                type = ItemType.TEXT_IMAGE;
                //banners类型
            }else if (banners != null){
                type = ItemType.BANNER;
                // Banner 的初始化
                final int bannerSize = banners.size();
                for (int j = 0; j < bannerSize; j++) {
                    final String banner = banners.getString(j);
                    bannerImages.add(banner);
                }
            }
            final MultipleItemEntity entity = MultipleItemEntity.builder()
                    .setField(MultipleFields.ITEM_TYPE,type)
                    .setField(MultipleFields.SPAN_SIZE,spanSize)
                    .setField(MultipleFields.ID,id)
                    .setField(MultipleFields.TEXT,text)
                    .setField(MultipleFields.IMAGE_URL,imageUrl)
                    .setField(MultipleFields.BANNERS,bannerImages)
                    .build();
            ENTITLES.add(entity);
        }
        return ENTITLES;
    }
}
```

```java
/**
 * Copyright (C)
 *
 * @file: MultipleFields
 * @author: 345
 * @Time: 2019/4/27 15:01
 * @description: 解析出来的数据 对应的类型，这里作为键存在
 */
public enum MultipleFields {
    /**
     * 键
     */
    ITEM_TYPE,
    //文本
    TEXT,
    //图片
    IMAGE_URL,
    //轮播图
    BANNERS,
    //长度
    SPAN_SIZE,
    //id
    ID,
    //姓名
    NAME,
    //标记
    TAG
}
```

上面解析用的 是阿里的JSON 库，依赖如下

```java
//JSON 依赖
api 'com.alibaba:fastjson:1.2.31'

//JSON 依赖 这个是 Android优化的
api 'com.alibaba:fastjson:1.2.31.android'
```

5,使用如下

```java
public class IndexDelegate extends BottomItemDelegate {

    @BindView(R2.id.rv_index)
    RecyclerView mRecyclerView = null;
    @BindView(R2.id.srl_index)
    SwipeRefreshLayout mRefreshLayout = null;
    @BindView(R2.id.tb_index)
    Toolbar mToobar = null;
    @BindView(R2.id.icon_index_scan)
    IconTextView mIconScan = null;
    @BindView(R2.id.et_search_view)
    AppCompatEditText mSearchView = null;

    private RefreshHander mRefreshHandler = null;

    @Override
    public void onBindView(@Nullable Bundle savedInstanceState, View rootView) {

        mRefreshHandler = RefreshHander.creawte(mRefreshLayout,mRecyclerView,new IndexDataConverter());
    }

    private void initRefreshLayout() {
        mRefreshLayout.setColorSchemeColors(
                Color.BLUE, Color.RED, Color.GRAY);
        //第一个参数为true，表示下拉的过程中按钮由小变大，回弹过程有大变小
        // 第二个参数为起始高度，第三个为终止高度
        mRefreshLayout.setProgressViewOffset(true, 120, 300);
    }

    private void initRecyclerView(){
        GridLayoutManager manager = new GridLayoutManager(getContext(),4);
        mRecyclerView.setLayoutManager(manager);
        //设置分割线
        mRecyclerView.addItemDecoration(BaseDecoration.create(ContextCompat.getColor(getContext(), R.color.app_background), 5));
        final EcBottomDelegate ecBottomDelegate  = getParentDelegate();
        //监听事件
        mRecyclerView.addOnItemTouchListener(IndexItemClickListener.create(ecBottomDelegate));
    }

    /**
     * 懒加载时 回调的方法，当前界面显示是，会回调该方法
     * @param savedInstanceState
     */
    @Override
    public void onLazyInitView(@Nullable Bundle savedInstanceState) {
        super.onLazyInitView(savedInstanceState);
        initRefreshLayout();
        initRecyclerView();
        mRefreshHandler.firstPage("index.json");
    }

    @Override
    public Object setLayout() {
        return R.layout.delegate_index;
    }
}
```