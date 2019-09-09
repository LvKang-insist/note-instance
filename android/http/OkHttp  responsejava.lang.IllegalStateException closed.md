
    java.lang.IllegalStateException: closed
        at okio.RealBufferedSource.rangeEquals(RealBufferedSource.java:408)
        
---
##### 非法 语句异常。造成这个异常的原因就是在使用response.body().string() 的时候进行了多次调用。
### 如下所示：
    
```
 postanays(url, post, new okhttp3.Callback() {
            private String str;
            @Override
            public void onResponse(Call call, final Response response) {
                try {
                    System.out.println(Thread.currentThread().getName() + "" + response.body().string());
                } catch (IOException e) {
                    e.printStackTrace();
                }
               
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        if (listener != null) {
                            System.out.println(Thread.currentThread().getName());
                            try {
                                listener.success(response.body().string());
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                });
            }
            @Override
            public void onFailure(Call call, final IOException e) {
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        listener.error(e + "");
                    }
                });
            }
        });
```
##### 因为在获取值得时候 在前面已经掉用过了response.body().string()方法进行打印，然后在后面又调用了一次。所以造成了这个异常。

```
   */
  public final String string() throws IOException {
    BufferedSource source = source();
    try {
      Charset charset = Util.bomAwareCharset(source, charset());
      return source.readString(charset);
    } finally {
      Util.closeQuietly(source);
    }
  }

```
==上面这段代码就是.string()的源码，可以看到他的return之后还在finally里面关闭了资源。所以了response.body().string()方法不能被调用第二次。==


---
##### 当我 改完后，重新运行，发现还是报错。但是报的错不一样了。代码和错误如下所示：

```
 postanays(url, post, new okhttp3.Callback() {
            private String str;
            @Override
            public void onResponse(Call call, final Response response) {
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        if (listener != null) {
                            try {
                                listener.success(response.body().string());
                            } catch (IOException e) {
                                e.printStackTrace();
                            }
                        }
                    }
                });
            }
            @Override
            .....
```
错误为  android.os.NetworkOnMainThreadException 主线程访问网络异常。

经过一系列的研究，response.body()的时候在body这个方法里面 会有网络耗时的操作。所以才会造成这个错误。

修改如下：

```
public void postanay(String url, String post, final OnReqeustListener listener) {
        postanays(url, post, new okhttp3.Callback() {
            private String str;
            @Override
            public void onResponse(Call call, final Response response) {
                try {
                    str = response.body().string();
                } catch (IOException e) {
                    e.printStackTrace();
                }
                handler.post(new Runnable() {
                    @Override
                    public void run() {
                        if (listener != null) {
                            listener.success(str);
                        }
                    }
                });
            }
```
  在handler 将数据发送到 主线程之前 将response.body().string()的结果赋给一个变量。然后用handler直接发送到的主线程就可以了。
  
>   以上就是 我遇到的bug。是不是特别坑。  如果以上文章存在错误，还请指出，谢谢！