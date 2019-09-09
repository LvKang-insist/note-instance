
使用okhttp请求网络数据一般可以使用两种方式，
-  1是使用okhttp自带的接口，他的内部已经帮我们开好子线程了，你只需要实现这个接口就可以在主线程中拿到数据，每调用一次，他就会开启一个线程
- 2，自己开一个线程，使用okhttp进行发送请求，注意，他拿回来的数据是在子线程中，需要使用handle发送到主线程。

## 一，使用okhttp自带的接口拿到数据：

##### 1，设置 post 请求的编码形式：

```
MediaType JSON = MediaType.pare("application/json;charset=utf-8");
```
##### 2,创建Okhttp 对象

```
OkhttpClient client = new OkHttpClient();
```
##### ##### 3,调用 RequestBody的create方法，返回一个RequestBody的对象。

create方法接收两个参数
          参数1：post 请求的编码形式， 参数2：要发送的json串

```
RequestBody body = RequestBody.create(JSON,josn);
```
##### 4,创建 一个Request 对象，用来发送Http请求

```
Request request = new Request.Builder()
                .url(url)
                .post(body)
                .build();
```
##### 5，使用接口回调，将请求到的数据 传递到主线程。

```
 client.newCall(request).enqueue(callback);
```
##### 6，在主线程获取数据，然后解析。（这里直接打印，没有解析）

```
new OkHttpRequestJson().requestJson(url, json, new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.d("MainActivity","解析异常");
            }
            @Override
            public void onResponse(Call call, Response response) throws IOException {
                System.out.println(response.body().string());
            }
        });
```
全部代码如下

```
public class OkHttpRequestJson {

    public void requestJson( String url, String Json,okhttp3.Callback callback) {

        final MediaType jJSON = MediaType.parse("application/json; charset=utf-8");
        String json = Json;
        OkHttpClient client = new OkHttpClient();
        RequestBody body = RequestBody.create(jJSON, json);
        Request request = new Request.Builder()
                .url(url)
                .post(body)
                .build();
            client.newCall(request).enqueue(callback);
    }
}
```

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        String url = "http://192.168.1.100:8080/transportservice/type/jason/action/SetCarMove.do";
        String json = "{\"CarId\":1, \"CarAction\":\"Stop\"}" ;
        new OkHttpRequestJson().requestJson(url, json, new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.d("MainActivity","解析异常");
            }
            @Override
            public void onResponse(Call call, Response response) throws IOException {
                System.out.println(response.body().string());
            }
        });
    }
}
```

在请求数据的方法中，有三个参数，分别是，url，要post的数据，还有okhttp接口的 引用，你只需要调用这个方法，设置url和post，然后实现接口，就可以直接拿到数据

---

## 二，自己开线程发送请求。

```
public interface Requst{
        void oncuess(String s);
    }
    OkHttpClient client = new OkHttpClient();
    Handler handler = new Handler();
    MediaType type = MediaType.parse("application/json;charset=utf-8");
    public void anay(final String url, final String post, final Requst listener){
        new Thread(new Runnable() {
            @Override
            public void run() {
                RequestBody body = RequestBody.create(type,post);
                Request request = new Request.Builder()
                        .url(url)
                        .post(body)
                        .build();

                try {
                    final String str= client.newCall(request).execute().body().string();
                    handler.post(new Runnable() {
                        @Override
                        public void run() {
                            listener.oncuess(str);
                        }
                    });
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }).start();
    }
}
```

这里就多了一步，拿到数据后，先赋值给 一个字符串，然后使用handler发送到主线程，这里我使用了一个接口回调。只有在外面实现这个接口就可以拿到数据。


---


区别：第一种是okhttp帮你开的线程，你每调用一次他就会开启一个线程，不需要你自己去new线程，第二种就是自己new线程了，但是在 你自己new的线程中可以循环发送请求，而第一种是发送一次就要开启一个线程。

[使用okhttp遇到的问题1](https://blog.csdn.net/baidu_40389775/article/details/86316907)

[使用okhttp遇到的问题2](https://blog.csdn.net/baidu_40389775/article/details/86421119)

> 如有错误，还请指出，谢谢