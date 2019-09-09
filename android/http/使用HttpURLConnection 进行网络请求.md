
```java
package com.laohu.cn;

import android.os.Handler;

import java.io.BufferedReader;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.InputStreamReader;
import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.URI;
import java.net.URL;


public class Http {

    public interface onReqeust {
        void on(String request);
    }
    
    Handler handler = new Handler();
    private StringBuilder builder;
    public void get(final String add, final onReqeust reqeust) {
        new Thread(new Runnable() {
            @Override
            public void run() {
                HttpURLConnection connection = null;
                BufferedReader read = null;
                try {
                    URL url = new URL(add);
                    connection = (HttpURLConnection) url.openConnection();
                    connection.setRequestMethod("GET");
                    connection.setConnectTimeout(5000);
                    connection.setReadTimeout(5000);

                    InputStream in = connection.getInputStream();
                    read = new BufferedReader(new InputStreamReader(in));

                    builder = new StringBuilder();
                    String line;
                    while ((line = read.readLine()) != null) {
                        builder.append(line);
                    }
                    if (builder.length() > 0){
                        handler.post(new Runnable() {
                            @Override
                            public void run() {
                                reqeust.on(builder.toString());
                            }
                        });
                    }
                    reqeust.on(builder.toString());
                } catch (MalformedURLException e) {
                    e.printStackTrace();
                } catch (IOException e) {
                    e.printStackTrace();
                } finally {
                    try {
                        if (read != null) {
                            read.close();
                        }
                        if (connection != null) {
                            connection.disconnect();
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}


```
首先看一下get方法，首先开启一个子线程，在线程中创建一个HttpURLConnection对象，然后设置一些属性，最后使用BufferedReader对服务器返回的数据进行读取,读取完成之后判断一下，最后使用handler发送到主线程，下面我们看一下怎么调用。

```
 Http http = new Http();
        String url = "http://192.168.1.103:8080/work/json.txt";
        Log.e("-----------", "reqeust: 开始");
        http.get(url, new Http.onReqeust() {
            @Override
            public void on(String request) {
                Log.e("-----------", "reqeust: 结束"+request);
            }
        });
```
