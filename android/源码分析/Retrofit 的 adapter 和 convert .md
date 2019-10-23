##  不解释，直接撸

自定义 Adapter 和 Converter

#### 1，接口：

```java
@GET
LiveData<CustomResponse> get(@Url String url, @QueryMap Map<String, Object> params);

@GET
LiveData<String> get(@Url String url, @FieldMap Map<String, Object> params);
```

 两个 方法，一个返回的是自定义对象，还有一个是 String

```java
public class CustomResponse {
    private Response value;
    private String string;
    public CustomResponse(String string) {
        this.string = string;
    }
 	......
}
```

#### 2，接着看一下自定义 Adapter

```java
/**
 * 自定义的响应类型
 */
public class LiveDataCallAdapterFactory extends CallAdapter.Factory {
    @Override
    public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
	   //returnType 为 接口方法的返回值类型
        //当接口返回类型不是 LiveData时，返回null
        if (getRawType(returnType) != LiveData.class) {
            return null;
        }
        //泛型 type
        Type bodyType = getParameterUpperBound(0, (ParameterizedType) returnType);
        //如果 泛型为 CustomRsponse 
        if (bodyType == CustomResponse.class) {
            return new LiveDataCustomAdatper<>();
        }
        return new LiveDataCallAdapter<>(bodyType);
    }

    //第二个泛型就是我们 定义接口方法中的返回值，我们需要将请求的结果转为他
    class LiveDataCustomAdatper<R> implements CallAdapter<R, LiveData<CustomResponse>> {
        @Override
        public Type responseType() {
            return CustomResponse.class;
        }

        @Override
        public LiveData<CustomResponse> adapt(final Call<R> call) {
            final AtomicBoolean started = new AtomicBoolean(false);
            //返回 LiveData 对象，并且重写了一个方法，这个可以不关注
            return new LiveData<CustomResponse>() {
                @Override
                protected void onActive() {
                    super.onActive();
                    if (started.compareAndSet(false, true)) {
                        // enqueue 方法在最下面会讲
                        call.enqueue(new Callback<R>() {
                            @Override
                            public void onResponse(@NonNull Call<R> call, Response<R> response) {
                                CustomResponse r = (CustomResponse) response.body();
                                if (r != null) {
                                    r.setValue(response);
                                }
                                postValue(r);
                            }

                            @Override
                            public void onFailure(@NonNull Call<R> call, Throwable t) {
                                postValue(null);
                            }
                        });
                    }
                }
            };
        }
    }

    class LiveDataCallAdapter<R> implements CallAdapter<R, LiveData<R>> {
        private final Type responseType;

        LiveDataCallAdapter(Type responseType) {
            this.responseType = responseType;
        }

        @Override
        public Type responseType() {
            return responseType;
        }

        @Override
        public LiveData<R> adapt(final Call<R> call) {
			//返回 LiveData<R>
            return new LiveData<R>() {
                //原子更新的boolean
                AtomicBoolean started = new AtomicBoolean(false);

                @Override
                protected void onActive() {
                    super.onActive();
                    if (started.compareAndSet(false, true)) {
                        call.enqueue(new Callback<R>() {
                            @Override
                            public void onResponse(Call<R> call, Response<R> response) {
                                R body = response.body();
                                String path = call.request().url().encodedPath();
                                postValue(body);
                            }
                            @Override
                            public void onFailure(Call<R> call, Throwable throwable) {
                            }
                        });
                    }
                }
            };
        }

    }
}

```

在这里 **LiveDataCallAdapter** 中返回的是 LiveData<R> ，并且在构造方法中传入了type,为啥这样写呢？别急，后面我们继续说！  注意 **responseType** 方法的返回值

#### 3，自定义 Converter

```java
public class AddConverterFactory extends Converter.Factory {
    @Nullable
    @Override
    public Converter<?, RequestBody> requestBodyConverter(Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations, Retrofit retrofit) {
        return super.requestBodyConverter(type, parameterAnnotations, methodAnnotations, retrofit);
    }

    @Override
    public Converter<ResponseBody, ?> responseBodyConverter(Type type, Annotation[] annotations, Retrofit retrofit) {
        //注意这里的 type 类型
        if (type == CustomResponse.class) {
            return new B();
        } else {
            return new A();
        }
    }

    public class A implements Converter<ResponseBody, String> {

        @Override
        public String convert(@NonNull ResponseBody value) throws IOException {
            return value.string();
        }
    }

    public class B implements Converter<ResponseBody, CustomResponse> {
        @Override
        public CustomResponse convert(@NonNull ResponseBody value) throws IOException {
            return new CustomResponse(value.string());
        }
    }
}
```

还记得 自定义 Adapter 中 responseType 方法的返回值吗？ 没错，他就是这里的 type！！

通过这个 type 我们可以自定义我们的转换器，在 Adpater 中我们返回了 CustomResponse.class ，还有一个默认的类型，那个类型就是 接口方法中返回值的泛型类型。我们给的是 String 和 CustomResponse。

注意看 responseBodyConverter 方法，这在这个方法中判断了 type ，如果为 CustomResponse，则看一下 B 类，实现了  Converter 接口，注意 两个泛型，第一个不可变，第二个为 你想要转换的类型。在 convert 方法中 返回了一个 CustomResponse 对象并传入了一个值，**注意：这个值就是我们请求的结果。A 类 也差不多一样。**

接着我们继续看一下 自定义 Adapter 中的一些东西：

```java
 @Override
        public LiveData<CustomResponse> adapt(final Call<R> call) {
            final AtomicBoolean started = new AtomicBoolean(false);
            return new LiveData<CustomResponse>() {
                @Override
                protected void onActive() {
                    super.onActive();
                    if (started.compareAndSet(false, true)) {
                        call.enqueue(new Callback<R>() {
                            @Override
                            public void onResponse(@NonNull Call<R> call, Response<R> response) {
                                CustomResponse r = (CustomResponse) response.body();
                                if (r != null) {
                                    r.setValue(response);
                                }
                                postValue(r);
                            }
                            @Override
                            public void onFailure(@NonNull Call<R> call, Throwable t) {
                                postValue(null);
                            }
                        });
                    }
                }
            };
        }
```

adapt 方法提供了一个 call 参数，在这个方法中返回了 我们需要的 LiveData<CustomResponse>，我们的重点是  call 参数，我们实现了他，并且有一个 response 对象，注意通过 response 的 body 方法 就可以拿到在自定义 Converter 中返回的 对象，如上面的 response.body() ，最后将 response 传入了 CustomResponse 中，并且进行发送。

------

经过上面的步奏 就可以拿到返回值为 LiveData<CustomResonse> 的对象了。然后只需要给他添加一个 观察者，上面的重写的 onActivie 方法就会执行，然后通过 postValue 将数据发送出去。

如果不会 LiveData 也没关系，学一下就好了，还是挺好用的。

------

### 源码分析：

```
 RETROFIT_CLIENT = new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .client(OK_HTTP_CLIENT)
                .addConverterFactory(new AddConverterFactory())
                .addCallAdapterFactory(new LiveDataCallAdapterFactory())
                .build();
        service = RETROFIT_CLIENT.create(RestService.class);
```

build()方法：

```java
public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }
	 .......
      //如果为空 则创建 OkHttpClient 对象
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }
      ......
      // 用来存放 adapter，也就是适配器
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // 用来存放 convertr，也就是 转换器
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);
	 //返回 Retrofit 对象
      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
```

接着看一下 生成接口实例的 create()

```java
public <T> T create(final Class<T> service) {
 	//动态代理
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            .......
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
}

//loadServiceMethod方法：通过这个方法返回 ServiceMethod 对象
ServiceMethod<?, ?> loadServiceMethod(Method method) {
    //从缓存中获取 method
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;
	//如果为空，则进行创建
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
 }

```

注意看 ServiceMethod 的 build() 方法

```java
 public ServiceMethod build() {
     //1，通过这个方法 可以拿到我们自定义的 adapter
      callAdapter = createCallAdapter();
     //还记得 我说的 responseType 方法的返回值吗？拿到返回值后 给了responseType，还记得他是什么吗？
     //他 应该是接口方法返回值的泛型
      responseType = callAdapter.responseType();
     //如果 为 Rresponse 或者 OKhttp3 的 response，直接抛异常
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }
      //2，通过这个方法 可以拿到我们自定义的 converter
      responseConverter = createResponseConverter();

      ......
      return new ServiceMethod<>(this);
}
// 先看一下 1.
private CallAdapter<T, R> createCallAdapter() {
     //拿到了当前方法的返回值
      Type returnType = method.getGenericReturnType();
      .......
      try {
        //noinspection unchecked
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
 }

 public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
 }
 public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
    checkNotNull(returnType, "returnType == null");
    checkNotNull(annotations, "annotations == null");
	//获取自定义的适配器
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      //这里调用的 get 方法就是我们重写的 get 方法
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      //如果 为 null 则继续执行，否则直接返回 adapter
      if (adapter != null) {
        return adapter;
      }
    }
	.......
    throw new IllegalArgumentException(builder.toString());

  }
```

从1，我们可以看出 从集合中获取自定义的适配器(适配器在add 的时候已经被添加进集合了)。这里用了一个 for 循环。如果我们重写的 get 方法返回 null。那么他就会继续寻找下一个 适配器。如果没有找到 他会抛一个异常。如果找到则 直接 返回，结束循环。

下面看一下 2

```java
 private Converter<ResponseBody, T> createResponseConverter() {
      Annotation[] annotations = method.getAnnotations();
      try {
        //调用 responseBodyConverter 方法，返回 Converter 对象
        //responseType 是接口方法返回值的泛型。如果不知道请看一下 build 方法   
        return retrofit.responseBodyConverter(responseType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create converter for %s", responseType);
      }
 }
public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
}
public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    checkNotNull(type, "type == null");
    checkNotNull(annotations, "annotations == null");

    //这里的逻辑 和 1 基本一样
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }
```

​	注意 responseType 是 我们自定义 adapter 中 responseType 的返回值。其他的逻辑 和 1 基本一样，最终 返回一个 Converter 对象

​	最终 ServiceMethod 的 builder 返回了 一个 ServiceMethod 对象，此时 自定义的 adapter 和 converter 对象已经创完毕，注释已标明：1,2；

------

接着我们回到最开始的地方，Retrofit 的 create 方法

```java
public <T> T create(final Class<T> service) {
 	//动态代理
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            .......
            //在这个过程中创建好了自定义 adapter 和 converter    
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
              // 这里调用的是 自定义adapter。我们会在 adapt 方法中调用 okHttpCall的 enqueue方法 ，并实现Callback 接口，
              //在 enqueue 方法中 将 自定义的 converter 返回的对象 和 okhttp 的 response 对象封装进 Retrofit 的Response 中
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
}
```

可以看到最终调用了我们自定义的 adapt 方法。然后 传入创建好的 okHttpCall，就好了**。最终在我们在 adapt 方法中 使用 call 调用 enqueue 方法实现接口 。**

接着我们来看一下 okHttpCall 的 enqueue 方法：

```java
@Override public void enqueue(final Callback<T> callback) {
    checkNotNull(callback, "callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;
      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          //如果为空，最终调用的还是 Retrofit build 方法中创建的 OkHttpClient ，通过 newCall 方法返回 call
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

  ......
   //异步请求   
   call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        //rawResponse 就是请求的结果 response
        
        // 这个 Response 是 Retrofit 中的  
        Response<T> response;
        try {
          //1, 解析 rawResponse
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
          //2,将结果回调出去
        callSuccess(response);
      }

     ....
    });
  }
```

 下面我们看一下 1：

```java
 Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();
	
	.......
    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      // 注意这里  这里最终调用的是 我们自定义的 converter
      T body = serviceMethod.toResponse(catchingBody);
      // 返回一个 Retrofit 的 Response 对象
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }

```

```java
/** Builds a method return value from an HTTP response body. */
  R toResponse(ResponseBody body) throws IOException {
    //将请求完成后的 body 传给转换器进行返回
    return responseConverter.convert(body);
  }

```

```java
public static <T> Response<T> success(@Nullable T body, okhttp3.Response rawResponse) {
    checkNotNull(rawResponse, "rawResponse == null");
    if (!rawResponse.isSuccessful()) {
      throw new IllegalArgumentException("rawResponse must be successful response");
    }
    return new Response<>(rawResponse, body, null);
  }
  private Response(okhttp3.Response rawResponse, @Nullable T body,
      @Nullable ResponseBody errorBody) {
    this.rawResponse = rawResponse;
    this.body = body;
    this.errorBody = errorBody;
  }
```

 上面将 自定义中返回的 对象 和 rawResponse 对象装进了 Response 中

接着看一下 2：

```java
	private void callSuccess(Response<T> response) {
        try {
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
```

通过 callback 的回调方法，将 response 中的数据传递出去。这个 callback 就是在 自定义 adapter 的 adapt 中实现的！！

------

最后看一下 create 方法

```java
public <T> T create(final Class<T> service) {
 	//动态代理
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            .......
            //在这个过程中创建好了自定义 adapter 和 converter    
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
              // 这里调用的是 自定义adapter。我们会在 adapt 方法中调用 okHttpCall的 enqueue方法 ，并实现Callback 接口，
              //在 enqueue 方法中 将 自定义的 converter 返回的对象 和 okhttp 的 response 对象封装进 Retrofit 的Response 中
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
}
```

​	最后一句 adapt方法 返回的对象 则就是我们 自定义的对象了。

​	最后赞一哈，这 Retrofit 写的 真牛逼

​	以上就是 Retrofit 从创建到 请求完成 的流程了

>  如果错误，还请指出