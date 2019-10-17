  获取 cookie 的拦截器

```
public class GetSessionIdInterceptor implements Interceptor {
    @NonNull
    @Override
    public Response intercept(@NonNull Chain chain) throws IOException {
        Request request = chain.request();

        Response response = chain.proceed(request);
        String cookie = response.header("set-cookie");
        // 取得 sessid
        if (cookie != null) {
            Utils.id = cookie.substring(0, cookie.indexOf(";"));
        }
        return response;
    }
}
```

添加 cookie

```
public class SetSessionIdInterceptor implements Interceptor {
    @NonNull
    @Override
    public Response intercept(@NonNull Chain chain) throws IOException {
        Request.Builder builder = chain.request().newBuilder();
        builder.addHeader("cookie", Utils.id);
        return chain.proceed(builder.build());
    }
}
```

