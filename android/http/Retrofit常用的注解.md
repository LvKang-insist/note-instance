@GET("表示一段地址");

```java
@GET("url")
Call<String> get(@QueryMap Map<String,Object> params);
```



@URL ：表示一段地址

```java
@GET()
Call<String> get(@Url String url ,@QueryMap Map<String,Object> params);
```

@Query Map

如上面的所示：表示附加到URL的查询参数键和值 

@Query("") 

```java
@GET("url")
Call<String> get(@Query("xxx") String url);
```



### 标记类注解‘

@FormUrlEncoded 表示请求体是一个 Form表单

```
@FormUrlEncoded
@POST
Call<String> post(@Url String url, @FieldMap Map<String, Object> params);
```

 @Streaming

```java
/**
 * download 默认是吧文件下载到内存中，最后统一写入文件里，这种方式存在一个问题：
 * 当文件过大是 会导致文件溢出，所以这里加入@Streaming ，代表一边下载一边写入到文件中
 * 这样就避免了 一次性在内存中写入过大的文件造成的问题
 */
@Streaming
@GET
Call<ResponseBody> download(@Url String url, @QueryMap Map<String, Object> params);
```

@Multipart

```java
/**
 * @Multipart : 表示请求体是一个支持文件上传的From 表单
 */
@Multipart
@POST
Call<String> upload(@Url String url, @Part MultipartBody.Part file);
```

