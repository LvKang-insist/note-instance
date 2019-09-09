今天 遇到了一个java.lang.OutOfMemoryError Coldnot not allocate JNI Env一直搞不好，内存泄露。

---
报错信息如下：



---


#### 先说一下 我的需求：
某一个页面需要 每隔三秒进行刷新，刷新的数据是从网络上面拿下来的。而我每次刷新的时候 要十五条数据，所以我没隔三秒就会去请求15 次数据。

#### 分析一下异常的原因：
- 原本 我以为是线程的问题，因为我用的是异步，每次请求的时候都会开启一个线程。所以导致 会有很多个线程。但是经过请教后 知道了，这个和线程的多少没有关系，因为每个线程执行完自己的任务后就会销毁。不会存在线程过多导致内存泄露的问题
。
- 经过一番请教后，内存泄露的最大原因就是某些东西一直在产生新的对象，还有就是在请求时候没有进行资源的释放，导致一直占用着内存。内存占用的越来越多。就会导致内存泄露。

---
#### 解决：
- 经过分析之后我感可能是 内存没有被释放，某些东西一直在产生新的对象，然后我再代码中反复的寻找，把所有需要new 的东西，全部放在了成员变量里面进行初始化，但是还是不行。
- 最后猛然间想起了我封装的okhttp3 的类，会不会是我的封装的类出了问题。然后打开封装的okhttp3，代码如下：

```
package com.mad.trafficclient.http;
import android.os.Handler;
import android.os.Looper;
import java.io.IOException;
import okhttp3.Call;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;

public class GetOkhttp {
    public interface onGetRequestListener{
        void success(String request);
        void error(String error);
    }
    
    
    Handler handler = new Handler(Looper.myLooper());
    private void anaysGet(String url,okhttp3.Callback callback){
        OkHttpClient client = new OkHttpClient();
        Request request = new Request.Builder()
                .url(url)
                .build();
        client.newCall(request).enqueue(callback);
    }
    public void anayGet(String url , final onGetRequestListener listener){
        anaysGet(url, new okhttp3.Callback() {
            @Override
            public void onFailure(Call call, final IOException e) {
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        listener.error("get请求异常："+e);
                    }
                });
            }
            @Override
            public void onResponse(Call call, final Response response){
                try {
                    final String result = response.body().string();
                    handler.post(new Runnable() {
                        @Override
                        public void run() {
                            listener.success(result);
                        }
                    });
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}

```
这段代码看起来没有问题，但是我是多次调用的呀。每次调用anayGet方法。在和这个方法里面有调用anaysGet方法，看到这方法我 感觉可能是这个方法的第一句话出了问题，也就是OkHttpClient client = new OkhttpClient（）；
- 然后经过测试，发现就是 这个原因，因为每次进行网络请求的时候都会产生一个OkHttpClient 的对象，因为我是每隔三秒就会调用一次，导致有好多个对象没有被释放掉。所以造成了内存泄露


---
#### 总结：
1. 在进行多次循环或者是死循环的时候。一定要保证资源的释放和不要 对一个引用进行多次的new对象操作，这样很容易造成内存泄露。
如果 
2. 不幸出现了内存泄露。首先检查你代码里面有没有死循环。然后就是 找一下没有被释放的资源，对一个引用进行了多次new对象的操作。
 

---
> 以上 就是我碰到的问题，如果有问题，还行指出，谢谢！