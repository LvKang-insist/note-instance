### 介绍	

​	DiskLruCache 是硬盘缓存，他并没有显示 数据的缓存位置，可以自由的设置，但是通常情况下 程序员都会将缓存的位置选为  /sdcard/Android/data/<application package>/cache   这个路径。在这个路径有两个好处，第一：这是存储在 SD 卡上的，因此不会对内存有什么影响，第二：这个路径被Android 认定为 应用程序的缓存路径，当程序卸载时，这里的我数据也会被 清除掉，这样就不会出现程序卸载后还有残留数据的问题。

​	举个例子，如果应用程序的包名是 com.netease.newsreader.activity ，那么他的缓存地址就是 /sdcard/Android/data/com.netease.newsreader.activity/cache 。这个目录下的文件待会在看，

对DiskLruCache 有了大概了解后，下面就学一下他的用法

首先 添加一下依赖

```java
api 'com.jakewharton:disklrucache:2.0.2'
```



### 打开缓存

​	DiskLruCache 是不能创建实例的，如果需要创建实例，则需要调用他的 open 方法，如下所示：

```java
 // 1，数据的缓存地址，2，指定当前应用程序的版本号
 // 3，指定同一个 key 可以对应多少个缓存文件，基本都是1
 // 4，指定据图可以穿出多少字节的数据
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)
```

​	自定义缓存地址，通常是 /sdcard/Android/data/<application package>/cache 这个路径下

```java
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
```

​	getExternalCacheDir()方法来获取缓存路径  为  /sdcard/Android/data/<application package>/cache 

​	getCacheDir() 方法获取的路径为 /data/data/<application package>/cache 

​	uniqueName 是为了对不同类型数据进行区分 而设置的一个唯一值，

​	获取版本号：

```java
 private int getVersion(Context context) {
        try {
            PackageInfo info = context.getPackageManager().getPackageInfo(context.getPackageName(), 0);
            return info.versionCode;
        } catch (PackageManager.NameNotFoundException e) {
            e.printStackTrace();
        }
        return 1;
    }
```

​	需要注意的是 当版本号发生改变后，缓存路径下的所有数据都会被清掉，所有的数据都要从网上获取

​	后面两个参数就没什么需要解释的了，第三个参数传1，第四个参数通常传入10M的大小就够了，这个可以根据自身的情况进行调节 

​	因此一个open 方法的写法就可以这样写

```java
 public void open(Context context, String uniqueName) {
        File cacheDir = getCacheDir(context,uniqueName);
        //路径是否存在，不存在则创建
        if (!cacheDir.exists()){
            cacheDir.mkdirs();
        }
        try {
            DiskLruCache.open(cacheDir, getVersion(context),
                    1, 10 * 1024 * 1024);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

​	

### 写入缓存：

比如现在有一个图片，为了将图片下载，可以这样写

```java
public class MyOkhttp {
    public interface onRequestListener {
        void onSuccess(List<Object> data);
    }
    private Handler handler = new Handler(Looper.myLooper());
    private OkHttpClient client = new OkHttpClient();
    private ArrayList<Object> list;

    public void request(final String url, final onRequestListener listener) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                Request request = new Request.Builder()
                        .url(url)
                        .get()
                        .build();
                InputStream is = null;
                try {
                    final ResponseBody body = client.newCall(request).execute().body();
                    if (body != null) {
                        list = new ArrayList<>();
                        byte[] buff = new byte[2048];
                        int len;
                        is = body.byteStream();
                        while ((len = is.read(buff)) != -1) {
                            for (int i = 0; i < len; i++) {
                                list.add(buff[i]);
                            }
                        }
                    }
                    handler.post(new Runnable() {
                        @Override
                        public void run() {
                            listener.onSuccess(list);
                        }
                    });
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    try {
                        if (is != null) {
                            is.close();
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();

    }
}

```

```java
public void writeData(DiskLruCache diskLruCache, String url, int index, final onCacheDataListener listener) {
        dowloadData(diskLruCache, url, index, listener);
    }

    private void dowloadData(final DiskLruCache diskLruCache, String url, final int index, final onCacheDataListener listener) {
        try {
            String key = hashKeyForDisk(url);
            final DiskLruCache.Editor editor = diskLruCache.edit(key);
            if (editor != null) {
                MyOkhttp myOkhttp = new MyOkhttp();
                myOkhttp.request(url, new MyOkhttp.onRequestListener() {
                    @Override
                    public void onSuccess(List<Object> data) {
                        //获取缓存文件的输入流,将缓存文件写入到本地
                        OutputStream ops = null;
                        if (data != null) {
                            byte[] buff = new byte[data.size()];
                            for (int i = 0; i < data.size(); i++) {
                                buff[i] = (byte) data.get(i);
                            }
                            try {
                                ops = editor.newOutputStream(index);
                                ops.write(buff);
                                editor.commit();
                                diskLruCache.flush();
                                if (listener != null) {
                                    listener.onData(buff);
                                }
                            } catch (IOException e) {
                                e.printStackTrace();
                                try {
                                    editor.abort();
                                } catch (IOException e1) {
                                    e1.printStackTrace();
                                }
                            } finally {
                                if (ops != null) {
                                    try {
                                        ops.close();
                                    } catch (IOException e) {
                                        e.printStackTrace();
                                    }
                                }
                            }
                        }
                    }
                });
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

在上面 通过okhttp 将文件缓存到集合，并且通过一个接口将下载的数据返回在调用者。注意缓存完了以后一定要调用 editor.commit(); 提交一下，否则缓存的文件无法加载和删除。在缓存到本地的时候 通过DiskLruCache.Editor 来进行写入，这个类同样不能 new ，需要调用 DiskLruCache 的 edit(key)来获取实例，这个key就是文件的名字，并且这个图片的 url 和 可key 必须对应，所以这里使用了 MD5 编码，编码后的字符串是 唯一的，并且只会包含 0-F 这些字符，完全符合命名规则，如下所示：

```java
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
    for (int i = 0; i < digest.length; i++) {
        String hex = Integer.toHexString(0xFF & digest[i]);
        if (hex.length() == 1) {
            sb.append('0');
        }
        sb.append(hex);
    }
    return sb.toString();
}
```

只需要调用一下 hashKeyForDisk() 将url 传入 就可以得到对应的 key 了。

因此一个完整的 写入如下所示：

```java
DataCache dataCache = new DataCache();
final DiskLruCache disk = dataCache.open(MainActivity.this, "image");
String url = "https://img-my.csdn.net/uploads/201309/01/1378037235_7476.jpg";

dataCache.writeData(disk, url, 0, new DataCache.onCacheDataListener() {
   @Override
	public void onData(byte[] bytes) {
  		 if (bytes != null) {
    		 Bitmap bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
     		 imageView.setImageBitmap(bitmap);
 		 }
 	}
});
```

调用一下 writeData 方法 就会自动缓存，并且将缓存的 数据以字节数组的形式回调过来。

进入到目录 下看一下

![1560591198835](F:\笔记\android\杂\assets\1560591198835.png)

文件名字很长的 那个文件就是缓存的文件。

### 读取缓存

读取缓存主要使用 DiskLruCache 的 get 方法来读取的。方法如下所示：

```java
public synchronized Snapshot get(String key) throws IOException
```

很明显 需要传入一个 key，这个可以 就是 我们在缓存时 使用的key了。

下面看一下使用：

```java
public byte[] readCache(DiskLruCache diskLruCache, String url) {
    try {
        List<Byte> data = new ArrayList<>();
        String key = hashKeyForDisk(url);
        DiskLruCache.Snapshot snapShot = diskLruCache.get(key);
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
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
    return null;
}
```

上面 就是 读取缓存，通过 getInputStream 获取到 输入流，就可以读取数据了，同样的 ，这里也需要传入一个参数，这里传入 0 就好了。

使用如下：

```java
DataCache dataCache = new DataCache();
final DiskLruCache disk = dataCache.open(MainActivity.this, "image");
String url = "https://img-my.csdn.net/uploads/201309/01/1378037235_7476.jpg";

byte[] bytes = dataCache.readCache(disk, url);
if (bytes != null) {
    Bitmap bitmap = BitmapFactory.decodeByteArray(bytes, 0, bytes.length);
    image.setImageBitmap(bitmap);
}else {
    Toast.makeText(MainActivity.this, "空", Toast.LENGTH_SHORT).show();
}
```

直接调用方法就 会返回一个字节数组，然后将数组转为 bitmap对象，最后 设置到 Imageview 就可以了。

### 移除缓存

移除非常简单，他是使用DiskLruCache的remove()方法实现的 ，使用如下

```java
public boolean removeCache(DiskLruCache diskLruCache, String key) {
    String md5Key = hashKeyForDisk(key);
    boolean remove = false;
    try {
        remove = diskLruCache.remove(md5Key);
        return remove;
    } catch (IOException e) {
        e.printStackTrace();
    }
    return remove;
}
```



```java
boolean b = dataCache.removeCache(disk, url);
if (b){
    Toast.makeText(MainActivity.this, "成功", Toast.LENGTH_SHORT).show();
}else {
    Toast.makeText(MainActivity.this, "失败", Toast.LENGTH_SHORT).show();
}
```



### 其他的方法

1，size()

​	这个方法会返回 缓存路径下所有缓存数据的总字节数，以 byte 为单位

2，flush()

​	这个方法用于将内存中操作记录同步到日志文件(也就是 journal 文件)，这个方法非常重要，DiskLruCache 能够正常工作的 前提条件就是依赖于 journal文件中的内容，在前面写入缓存的时候 我调用了一次这个方法，但是并不是写入缓存就要调这个方法，频繁调用这个方法没有任何好处，比较标准的做法就是在 Activity 的 onPause 方法中调用一次 flush 方法就可以了。

3，close()

​	这个方法用于将 DiskLruCache 关掉，和open 对应的一个方法，关了之后就不能进行任何的操作了，同常在 onDestory 中关闭即可。

4，delete()

​	这个方法会将 缓存的全部数据删除。

5，getDirectory()

​	获取换出数据的目录



### 解读 journal 

![1560592715306](F:\笔记\android\杂\assets\1560592715306.png)

journal 是 DiskLruCache 能够正常使用的前提，因此 理解 journal 就是非常重要的了，

第一行 是 libcore.io.DiskLruCache”  ，表示我们使用了 DiskLruCache 

第二行 是 DiskLruCache 的版本号，这个值是恒为 1 的，

第三行 是 应用程序的版本号。

第四行 是 valueCount ，这个值是open 方法中传入的，通常情况都是 1

第五行 是 空格，第五行 往上被称为 journal 的 文件头

第六行 是 DIRTY 开始的，后面是 图片的key，通常 DIRTY 代表着这是一条脏数据。每当我们调用一次 edit 方法后 ，都会在 journal 文件中写入一条 DIRTY 的记录，但是不知道结果如何，然后调用 commit()方法表示缓存成功，这是会添加一个 CLEAN 的记录，意味着 这条 “脏” 数据被洗干净了。调用abort 方法表示缓存失败，这是 会添加一条 REMOVE 的记录。也就是说 每一行 的 DIRTY 的key 后面都要 有 一行对应的 CLEAN 或者 REMOVE 的记录，否则这个数据就是 “脏”的，会被自动删除掉

CLEAN 的最后面 跟了一个 数子，这个数字的意识 就是 缓存数据的大小，以字节为单位。

还有一种你READ 开头的记录，每当 通过get 去读取 一掉缓存时，就会有 一条 READ 记录。

DiskLruCache 中使用了一个 redundantOpCount 的遍历 来记录用户的操作次数，每执行一次 写入，读取 或者 移除，这个变量就会 加1，当 变量值 达到 2000 的时候 就会 对 journal 进行重构，删除不必要的记录，保证 journal文件 的大小始终 保持在一个合理的范围内。





