Retrofit 是Square公司计入 RESTful 风格推出的网络框架封装

Retrofit 与OKhttp 的关系

​	Retrofit 是基于OKHttp 的网络请求框架的二次封装，其本质仍是OKHttp

```java
/**
 * Created by caihan on 2017/1/11.
 *
 * @GET 表明这是get请求
 * @POST 表明这是post请求
 * @PUT 表明这是put请求
 * @DELETE 表明这是delete请求
 * @PATCH 表明这是一个patch请求，该请求是对put请求的补充，用于更新局部资源
 * @HEAD 表明这是一个head请求
 * @OPTIONS 表明这是一个option请求
 * @HTTP 通用注解, 可以替换以上所有的注解，其拥有三个属性：method，path，hasBody
 * @Headers 用于添加固定请求头，可以同时添加多个。通过该注解添加的请求头不会相互覆盖，而是共同存在
 * @Header 作为方法的参数传入，用于添加不固定值的Header，该注解会更新已有的请求头
 * @Body 多用于post请求发送非表单数据, 比如想要以post方式传递json格式数据
 * @Filed 多用于post请求中表单字段, Filed和FieldMap需要FormUrlEncoded结合使用
 * @FiledMap 和@Filed作用一致，用于不确定表单参数
 * @Part 用于表单字段, Part和PartMap与Multipart注解结合使用, 适合文件上传的情况
 * @PartMap 用于表单字段, 默认接受的类型是Map<String,REquestBody>，可用于实现多文件上传
 * <p>
 * Part标志上文的内容可以是富媒体形势，比如上传一张图片，上传一段音乐，即它多用于字节流传输。
 * 而Filed则相对简单些，通常是字符串键值对。
 * </p>
 * Part标志上文的内容可以是富媒体形势，比如上传一张图片，上传一段音乐，即它多用于字节流传输。
 * 而Filed则相对简单些，通常是字符串键值对。
 * @Path 用于url中的占位符,{占位符}和PATH只用在URL的path部分，url中的参数使用Query和QueryMap代替，保证接口定义的简洁
 * @Query 用于Get中指定参数
 * @QueryMap 和Query使用类似
 * @Url 指定请求路径
 */
```



## 简单的使用

### 1，导包

```java
	api 'com.squareup.okhttp3:okhttp:3.12.0'
    api 'com.squareup.retrofit2:retrofit:2.3.0'
    api 'com.squareup.retrofit2:converter-scalars:2.3.0'//转换器
    api 'com.squareup.retrofit2:converter-gson:2.3.0' //转换器
```

### 2，创建实例

```java
String url = "http://192.168.167.2:8090/Frame_Api/";
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(url)
                .addConverterFactory(ScalarsConverterFactory.create())
                .build();
```

创建 Retrofit 对象，添加 url ，现在我们使用的转换器为 converter 转换器。Retrofit 也支持其他的转换器 http://square.github.io/retrofit/

### 3，创建一个接口

```java
public interface MyService {
    @GET("index.php")
    Call<Student> getTop(@Query("name")String name,@Query("age")int age);
}
```

方法名字为 getTop，注解里面的字符串是拼接的地址(baseUrl + 尾址)，该请求对应两个参数 分别是 name 和 age。如果参数较多，可以使用@QueryMap。

### 4，进行请求

```java
	  	   MyService movieService = retrofit.create(MyService.class);
            Call<String> call = movieService.getTop("张三", 20);
            call.enqueue(new Callback<String>() {
                @Override
                public void onResponse(Call<String> call, Response<String> response) {
                    String student = response.body();
                    Log.e("-----", "onResponse: " + student.toString());
                }
                @Override
                public void onFailure(Call<String> call, Throwable t) {

                }
            });
```

首先通过 retrofit 获得 接口的实例，然后调用getTop 方法，获得到 call 。接着进行请求。

请求结果如下：

```java
E/-----: onResponse: {"code":"ok","value":{"name":"张三","age":"20"}}
```

上面是异步请求，不需要自己开子线程。当然也有同步，如下所示:

```java
 	ThreadFactory factory = new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    return new Thread(r);
                }
            };
            ThreadPoolExecutor executor = new ThreadPoolExecutor(1,1,0,
                    TimeUnit.SECONDS,new LinkedBlockingDeque<>(),factory);
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        MyService movieService = retrofit.create(MyService.class);
                        Call<String> call = movieService.getTop("张三", 20);
                        String body = call.execute().body();
                        System.out.println(body+Thread.currentThread().getName());
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
```

### 5，转换器

上面使用的转换器是 ScalarsConverterFactory，下面看一下Gson 的转换器如何使用

首先看一下返回的结果，根据返回的结果生成一个 Bean 类。

```java
  public class Student {
    /**
     * code : ok
     * value : {"name":"张三","age":"20"}
     */

    private String code;
    private ValueBean value;

    public String getCode() {
        return code;
    }

    public void setCode(String code) {
        this.code = code;
    }

    public ValueBean getValue() {
        return value;
    }

    public void setValue(ValueBean value) {
        this.value = value;
    }
    
    public static class ValueBean {
        /**
         * name : 张三
         * age : 20
         */
        private String name;
        private String age;

        public String getName() {
            return name;
        }

        public void setName(String name) {
            this.name = name;
        }

        public String getAge() {
            return age;
        }

        public void setAge(String age) {
            this.age = age;
        }
    }
}

```

然后修改转换器，修改接口

```java
   	    String url = "http://192.168.167.2:8090/Frame_Api/";
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl(url)
                .addConverterFactory(GsonConverterFactory.create())
                .build();	
```

```java
public interface MyService {
    @GET("index.php")
    Call<Student> getTop(@Query("name")String name,@Query("age")int age);
}
```

请求如下:

```java
     executor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        MyService movieService = retrofit.create(MyService.class);
                        Call<Student> call = movieService.getTop("张三", 20);
                        Student student = call.execute().body();
                        Log.e("---------", "run: " + student.getCode());
                        Log.e("---------", "run: " + student.getValue().getName());
                        Log.e("---------", "run: " + student.getValue().getAge());
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
```

结果如下：

```java
2019-06-28 10:23:22.810 10100-10144/com.admin.frame E/---------: run: ok
2019-06-28 10:23:22.810 10100-10144/com.admin.frame E/---------: run: 张三
2019-06-28 10:23:22.810 10100-10144/com.admin.frame E/---------: run: 20
```

结果返回后 转换器会将结果转成一个 对象，方便使用。

## POST 请求

### 	1，以 json 串的形式进行请求

​	修改接口如下：

```java
public interface MyService {
    @GET("index.php")
    Call<Student> getTop(@Query("name") String name, @Query("age") int age);

 	@POST("index.php")
    Call<String> post(@Body RequestBody body);
}
```

请求如下：

```java
	executor.execute(new Runnable() {
                @Override
                public void run() {
                    try {
                        MyService movieService = retrofit.create(MyService.class);
                        Map<Object, Object> map = new HashMap<>();
                        map.put("name", "李四");
                        String json = JSON.toJSONString(map);

                        MediaType type = MediaType.parse("application/json;charset=utf-8");
                        RequestBody body = RequestBody.create(type,json);
                        Call<String> call = movieService.post(body);
                        String string = call.execute().body();
                        Log.e("---------", "run: " + string);
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                }
            });
```

结果如下：

```java
2019-06-28 10:50:27.856 10630-10707/com.admin.frame E/---------: run: {"code":"ok","value":{"name":"李四"}}
```

### 2，以表单的方式请求

接口如下：

```java
	/**
     *  @FormUrlEncoded 表示请求体是一个Form 表单，
     */
    @FormUrlEncoded
    @POST
    Call<String> post(@Url String url, @FieldMap Map<String, Object> params);
```

使用的时候将参数存入 map ，最后调用post 方法 传入 拼接的url 和 map 即可。