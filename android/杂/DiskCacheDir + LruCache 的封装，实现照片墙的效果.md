​	上次对 DiskLruCache 进行了封装，但是他只能每次加载一张图片，不能放在ListView 等控件中使用。下面进行一次二次封装。

### 	首先是网络请求

```java
public class NetRequest {
    private OkHttpClient client = new OkHttpClient();
    public byte[] request(final String url) {
        Request request = new Request.Builder()
                .url(url)
                .get()
                .build();
        try {
            final ResponseBody body = client.newCall(request).execute().body();
            if (body != null) {
                return body.bytes();
            }
        } catch (IOException e) {
            e.printStackTrace();
            return null;
        }
        return null;
    }
```

### 	然后是封装的 DiskLruCache

```java
 * @author Lv
 * Created at 2019/6/14
 *
 * 图片缓存
 */
@SuppressWarnings({"ConstantConditions", "ResultOfMethodCallIgnored"})
public class DiskBitmapCache {

    private DiskLruCache mDiskLruCache;
    private Handler mHandler = new Handler(Looper.myLooper());

    public DiskBitmapCache(Context context, String uniqueName){
        open(context,uniqueName);
    }
    public DiskBitmapCache(){}

    /**
     * 用于返回 硬盘缓存后的数据
     */
    public interface OnCacheDataListener {
        /**
         * 返回数据
         *
         * @param bytes 缓存的数据
         */
        void onData(byte[] bytes);
    }

    /**
     * 打开磁盘缓存
     *
     * @param context    Context
     * @param uniqueName 缓存文件夹的名字
     * @return 返回一个 boolean 类型，true 表示 创建成果
     */
    public boolean open(Context context, String uniqueName) {
        File cacheDir = getCacheDir(context, uniqueName);
        //路径是否存在，不存在则创建
        if (!cacheDir.exists()) {
            cacheDir.mkdirs();
        }
        try {
            // 1，数据的缓存地址，2，指定当前应用程序的版本号
            // 3，指定同一个 key 可以对应多少个缓存文件，基本都是1
            // 4，指定据图可以缓存多少字节的数据
            mDiskLruCache = DiskLruCache.open(cacheDir, getVersion(context),
                    1, 10 * 1024 * 1024);
            return mDiskLruCache != null;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * 传入对应的 url ，进行缓存
     *
     * @param url      需要缓存的图片 ，缓存的文件名字为 url
     * @param listener 回调，将数据进行缓存后，返回一份，如果不需要传入 null 即可。
     */
    public void writeData(String url, final OnCacheDataListener listener) {
        downloadData(url, listener);
    }

    /**
     * 传入指定的 key 和 bitmap ，对图片进行缓存
     *
     * @param key    这个key 为缓存文件的名字
     * @param bitmap 要缓存的图片
     * @return 返回缓存的结果
     */
    public boolean writeData(String key, Bitmap bitmap) {
        key = hashKeyForDisk(key);
        DiskLruCache.Editor edit = null;
        try {
            edit = mDiskLruCache.edit(key);
            if (edit != null) {
                OutputStream outputStream = edit.newOutputStream(0);
                boolean compress = bitmap.compress(Bitmap.CompressFormat.PNG, 100, outputStream);
                edit.commit();
                return compress;
            } else {
                return false;
            }
        } catch (IOException e) {
            e.printStackTrace();
            if (edit != null) {
                try {
                    edit.abort();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
            }
            return false;
        }
    }

    /**
     * 根据 传入的 key 来查找缓存
     *
     * @param key 用来读取缓存的 key
     * @return 返回缓存的图片字节，没有则为 null
     */
    public byte[] readCache(String key) {
        try {
            List<Byte> data = new ArrayList<>();
            key = hashKeyForDisk(key);
            DiskLruCache.Snapshot snapShot = mDiskLruCache.get(key);
            if (snapShot != null) {
                InputStream is = snapShot.getInputStream(0);
                byte[] bytes = new byte[2048];
                int len;
                while ((len = is.read(bytes)) != -1) {
                    for (int i = 0; i < len; i++) {
                        data.add(bytes[i]);
                    }
                }
                bytes = new byte[data.size()];
                for (int i = 0; i < bytes.length; i++) {
                    bytes[i] = data.get(i);
                }
                return bytes;
            } else {
                return null;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 根据 key 删除指定的 缓存
     *
     * @param key 缓存的 key
     * @return 成功则返回 true
     */
    public boolean removeCache(String key) {
        String md5Key = hashKeyForDisk(key);
        boolean remove = false;
        try {
            remove = mDiskLruCache.remove(md5Key);
            return remove;
        } catch (IOException e) {
            e.printStackTrace();
        }
        return remove;
    }

    /**
     *  判断 该key 是否由缓存的图片
     * @param key 缓存文件对应的 key
     * @return 返回true 表示 有缓存，可以直接读取
     */
    public boolean isCache(String key) {
        byte[] bytes = readCache(key);
        return bytes != null;
    }

    /**
     * @return 返回 DiskLruCache 的实例
     */
    public DiskLruCache getInstance() {
        return mDiskLruCache;
    }

    /**
     * @return 返回缓存的大小，以字节为单位
     */
    public long size() {
        return mDiskLruCache.size();
    }

    /**
     * 将内存中的操作记录同步到日志文件，这个方法非常重要
     * 频繁地调用这个这个方法不会有任何好处，标准的做法是在 onPause 中调用一次就可以了
     */
    public void flush() {
        if (mDiskLruCache != null) {
            try {
                mDiskLruCache.flush();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 这个方法用于将DiskLruCache关闭掉，是和open()方法对应的一个方法。
     * 关闭掉了之后就不能再调用DiskLruCache中任何操作缓存数据的方法，
     * 通常只应该在Activity的onDestroy()方法中去调用close()方法
     */
    public void close() {
        if (mDiskLruCache != null) {
            try {
                mDiskLruCache.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    /**
     * 这个方法用于将所有的缓存数据全部删除
     */
    public void delete() {
        try {
            mDiskLruCache.delete();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * @return 获取 缓存的路径
     */
    private File getCacheDir(Context context, String uniqueName) {
        String cachePath;
        // 判断 SD 卡是否可用
        if (Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())
                || !Environment.isExternalStorageRemovable()) {
            //获取 有sd 卡时的路径
            cachePath = context.getExternalCacheDir().getPath();
        } else {
            // 获取 无sd 卡时的路径
            cachePath = context.getCacheDir().getPath();
        }
        //File.separator 分隔符 /
        return new File(cachePath + File.separator + uniqueName);
    }

    private int getVersion(Context context) {
        try {
            PackageInfo info = context.getPackageManager().getPackageInfo(context.getPackageName(), 0);
            return info.versionCode;
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        return 1;
    }

    private String hashKeyForDisk(String key) {
        String cacheKey;
        try {
            final MessageDigest mDigest = MessageDigest.getInstance("MD5");
            mDigest.update(key.getBytes());
            cacheKey = bytesToHexString(mDigest.digest());
        } catch (NoSuchAlgorithmException e) {
            cacheKey = String.valueOf(key.hashCode());
        }
        return cacheKey;
    }

    private String bytesToHexString(byte[] digest) {
        StringBuilder sb = new StringBuilder();
        for (byte b : digest) {
            String hex = Integer.toHexString(0xFF & b);
            if (hex.length() == 1) {
                sb.append('0');
            }
            sb.append(hex);
        }
        return sb.toString();
    }
    private void downloadData(final String url, final OnCacheDataListener listener) {
        try {
            String key = hashKeyForDisk(url);
            final DiskLruCache.Editor editor = mDiskLruCache.edit(key);
            final OutputStream ops = editor.newOutputStream(0);
            if (ops != null) {
                new Thread(new Runnable() {
                    @Override
                    public void run() {
                        NetRequest okhttp = new NetRequest();
                        try {
                            final byte[] result = okhttp.request(url);
                            ops.write(result);
                            editor.commit();
                            mHandler.post(new Runnable() {
                                @Override
                                public void run() {
                                    listener.onData(result);
                                }
                            });
                        } catch (IOException e) {
                            e.printStackTrace();
                            try {
                                if (editor != null) {
                                    editor.abort();
                                }
                            } catch (IOException e1) {
                                e1.printStackTrace();
                            }
                        }
                    }
                }).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

 上面两个 和上一篇博客内容是一样的，下面就看一下行加了什么

### LruCache 的封装

```java
/**
 * @author Lv
 * Created at 2019/6/16
 */
@SuppressWarnings("WeakerAccess")
public class LruCachePhoto {
    /**
     * 图片 缓存技术的核心类，用于缓存下载好的所有图片，
     * 在程序内存达到设定值后会将最少最近使用的图片移除掉
     */
    private LruCache<String, Bitmap> mMenoryCache;

    public LruCachePhoto() {
        //获取应用最大可用内存
        int maxMemory = (int) Runtime.getRuntime().maxMemory();
        //设置 缓存文件大小为 程序最大可用内存的 1/8
        int cacheSize = maxMemory / 8;

        mMenoryCache = new LruCache<String, Bitmap>(cacheSize) {
            @Override
            protected int sizeOf(String key, Bitmap value) {
                return value.getByteCount();
            }
        };
    }
    /**
     * 从 LruCache 中获取一张图片，如果不存在 就返回 null
     * @param key LurCache 的键，这里是 图片的地址
     * @return 返回对应的 Bitmap对象，找不到则为 null
     */
    public Bitmap getBitmapFromMemoryCache(String key){
        return mMenoryCache.get(key);
    }

    /**
     *  添加一张图片
     * @param key key
     * @param bitmap bitmap
     */
    public void addBitmapToCache(String key,Bitmap bitmap){
            mMenoryCache.put(key,bitmap);
    }
}
```

### 	AsynTask 的封装

```java
package com.admin.utill.net.cache;

import android.annotation.SuppressLint;
import android.graphics.Bitmap;
import android.graphics.BitmapFactory;
import android.os.AsyncTask;
import android.support.v7.widget.RecyclerView;
import android.util.Log;
import android.widget.GridView;
import android.widget.ImageView;
import android.widget.ListView;

import com.admin.utill.net.NetRequest;


/**
 * @author Lv
 * Created at 2019/6/15
 */
@SuppressLint("StaticFieldLeak")
public class BitmapWorkerTask extends AsyncTask<String, Void, Bitmap> {
    private LruCachePhoto mCachePhoto;
    private GridView mGridView;
    private DiskBitmapCache mDataCache;
    private ListView mListView;
    private ImageView mImageView;
    private Object mTag;

    private static final String TAG = "BitmapWorkerTask";

    /**
     * @param mCachePhoto 用于缓存下载好的图片，将图片缓存到内存
     * @param gridView    需要显示图片的 控件
     * @param Tag         每一个条目的 Tag
     */
    public BitmapWorkerTask(LruCachePhoto mCachePhoto, DiskBitmapCache dataCache, GridView gridView, Object Tag) {
        this.mGridView = gridView;
        init(mCachePhoto, dataCache, Tag);
    }

    public BitmapWorkerTask(LruCachePhoto mCachePhoto, DiskBitmapCache dataCache, ListView listView, Object tag) {
        this.mListView = listView;
        init(mCachePhoto, dataCache, tag);
    }

    public BitmapWorkerTask(LruCachePhoto mCachePhoto, DiskBitmapCache dataCache, ImageView imageView) {
        init(mCachePhoto, dataCache, null);
        this.mImageView = imageView;
    }

    private void init(LruCachePhoto mCachePhoto, DiskBitmapCache dataCache, Object tag) {
        this.mCachePhoto = mCachePhoto;
        this.mDataCache = dataCache;
        this.mTag = tag;
    }

    @Override
    protected Bitmap doInBackground(String... strings) {
        String imageUlr = strings[0];
        //获取内存的缓存
        Bitmap bitmap = mCachePhoto.getBitmapFromMemoryCache(imageUlr);
        if (bitmap!=null){
            return bitmap;
        }
        //判断本地是否有缓存
        if (!mDataCache.isCache(imageUlr)) {
            //没有缓存
            bitmap = downLoadBitmap(imageUlr);
        } else {
            byte[] bytes = mDataCache.readCache(imageUlr);
            bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
            mCachePhoto.addBitmapToCache(imageUlr, bitmap);
            return bitmap;
        }
        return bitmap;
    }

    @Override
    protected void onPostExecute(Bitmap bitmap) {
        super.onPostExecute(bitmap);
        ImageView imageView;
        if (mGridView != null) {
            imageView = mGridView.findViewWithTag(mTag);
        } else if (mListView != null) {
            imageView = mListView.findViewWithTag(mTag);
        }  else {
            imageView = this.mImageView;
        }
        if (imageView != null && bitmap!= null) {
            imageView.setImageBitmap(bitmap);
        }
    }
    private Bitmap downLoadBitmap(String imageUlr) {
        //如果没有缓存 就进行请求，然后进行缓存
        Bitmap bitmap = null;
        byte[] request = new NetRequest().request(imageUlr);
        if (request != null){
            bitmap = BitmapFactory.decodeByteArray(request, 0, request.length);
        }
        if (bitmap != null) {
            //将图片 缓存到内存
            mCachePhoto.addBitmapToCache(imageUlr, bitmap);
            //将图片 缓存到磁盘
            boolean b = mDataCache.writeData(imageUlr, bitmap);
            if (!b) {
                Log.e("PhotoCache", "磁盘缓存图片失败");
            }
            return bitmap;
        } else {
            return null;
        }
    }
}
```

加了一个 LruCache 和 一个异步任务，具体的注释都在上面了。

## 使用如下

在ListView 中使用

```java
public class MainActivity extends PermissionCheck {

    private DiskBitmapCache mDisk;
    private LruCachePhoto mCache;

    @RequiresApi(api = Build.VERSION_CODES.JELLY_BEAN)
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mCache = new LruCachePhoto();
        mDisk = new DiskBitmapCache(MainActivity.this, "456");

        final ListView listView = findViewById(R.id.listview);
        listView.setAdapter(new BaseAdapter() {
            @Override
            public int getCount() {
                return Images.imageThumbUrls.length;
            }

            @Override
            public Object getItem(int position) {
                return null;
            }

            @Override
            public long getItemId(int position) {
                return 0;
            }

            @Override
            public View getView(int position, View convertView, ViewGroup parent) {
                View view;
                if (convertView == null) {
                    view = LayoutInflater.from(MainActivity.this).inflate(R.layout.photo_layout, null);
                } else {
                    view = convertView;
                }
                String url = Images.imageThumbUrls[position];
                ImageView imageView = view.findViewById(R.id.photo);
                imageView.setImageResource(R.drawable.log);
                imageView.setTag(url);
                BitmapWorkerTask task = new BitmapWorkerTask(mCache, mDisk, listView, url);
                task.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, url);
                return view;
            }
        });

    }
}
```

​	在上面给 缓存的文件夹名字起名为 456，然后就是一个ListView 了，注意，在getView 里面给 ImageView 设置了一个Tag ，这个非常重要，如果没有这个 Tag，我们将无法找到显示 ImageView ，还有 DiskBitmapCache 和 LruCachePhoto必须是全局的。

效果如下

![ObjectAnimator](F:\笔记\android\杂\assets/ObjectAnimator.gif)

在GridView 中使用

```java
public class GridViewActvity extends AppCompatActivity {
    private DiskBitmapCache mDisk;
    private LruCachePhoto mCache;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_grid_view);
        mCache = new LruCachePhoto();
        mDisk = new DiskBitmapCache(this, "000");
        final GridView gridView= findViewById(R.id.gridView);
        gridView.setAdapter(new BaseAdapter() {
            @Override
            public int getCount() {
                return Images.imageThumbUrls.length;
            }

            @Override
            public Object getItem(int position) {
                return null;
            }

            @Override
            public long getItemId(int position) {
                return 0;
            }

            @Override
            public View getView(int position, View convertView, ViewGroup parent) {
                View view;
                if (convertView == null) {
                    view = LayoutInflater.from(GridViewActvity.this).inflate(R.layout.photo_layout, null);
                } else {
                    view = convertView;
                }
                String url = Images.imageThumbUrls[position];
                ImageView imageView = view.findViewById(R.id.photo);
                imageView.setImageResource(R.drawable.log);
                imageView.setTag(url);
                BitmapWorkerTask task = new BitmapWorkerTask(mCache, mDisk, gridView, url);
                task.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, url);
                return view;
            }
        });
    }
}
```

使用方式一样，注意别忘了 Tag 和 DiskBitmapCache 和 LruCachePhoto必须是全局的。

效果如下

![ObjectAnimator](F:\笔记\android\杂\assets/ObjectAnimator-1560950216376.gif)

下面看一下缓存目录

![ObjectAnimator](F:\笔记\android\杂\assets/ObjectAnimator-1560950294779.gif)



本文的 图片使用的是 郭神博客中的图片：<https://blog.csdn.net/guolin_blog/article/details/34093441> 