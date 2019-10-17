依赖：

```java
compile 'com.elvishew:xlog:1.6.1'
```

在 Application 中初始化

```
//log 工具
        XLog.init(new LogConfiguration.Builder()
                .t()//允许打印线程信息
                .tag("345")
                .build());
```

使用如下：

```
XLog.json(str);
```



更多配置：

```java
LogConfiguration config = new LogConfiguration.Builder()
    .logLevel(BuildConfig.DEBUG ? LogLevel.ALL             // 指定日志级别，低于该级别的日志将不会被打印，默认为 LogLevel.ALL
        : LogLevel.NONE)
    .tag("MY_TAG")                                         // 指定 TAG，默认为 "X-LOG"
    .t()                                                   // 允许打印线程信息，默认禁止
    .st(2)                                                 // 允许打印深度为2的调用栈信息，默认禁止
    .b()                                                   // 允许打印日志边框，默认禁止
    .jsonFormatter(new MyJsonFormatter())                  // 指定 JSON 格式化器，默认为 DefaultJsonFormatter
    .xmlFormatter(new MyXmlFormatter())                    // 指定 XML 格式化器，默认为 DefaultXmlFormatter
    .throwableFormatter(new MyThrowableFormatter())        // 指定可抛出异常格式化器，默认为 DefaultThrowableFormatter
    .threadFormatter(new MyThreadFormatter())              // 指定线程信息格式化器，默认为 DefaultThreadFormatter
    .stackTraceFormatter(new MyStackTraceFormatter())      // 指定调用栈信息格式化器，默认为 DefaultStackTraceFormatter
    .borderFormatter(new MyBoardFormatter())               // 指定边框格式化器，默认为 DefaultBorderFormatter
    .addObjectFormatter(AnyClass.class,                    // 为指定类添加格式化器
        new AnyClassObjectFormatter())                     // 默认使用 Object.toString()
    .addInterceptor(new BlacklistTagsFilterInterceptor(    // 添加黑名单 TAG 过滤器
        "blacklist1", "blacklist2", "blacklist3"))
    .addInterceptor(new MyInterceptor())                   // 添加一个日志拦截器
    .build();

Printer androidPrinter = new AndroidPrinter();             // 通过 android.util.Log 打印日志的打印器
Printer consolePrinter = new ConsolePrinter();             // 通过 System.out 打印日志到控制台的打印器
Printer filePrinter = new FilePrinter                      // 打印日志到文件的打印器
    .Builder("/sdcard/xlog/")                              // 指定保存日志文件的路径
    .fileNameGenerator(new DateFileNameGenerator())        // 指定日志文件名生成器，默认为 ChangelessFileNameGenerator("log")
    .backupStrategy(new NeverBackupStrategy()              // 指定日志文件备份策略，默认为 FileSizeBackupStrategy(1024 * 1024)
    .cleanStrategy(new FileLastModifiedCleanStrategy(MAX_TIME))     // 指定日志文件清除策略，默认为 NeverCleanStrategy()
    .flattener(new MyFlattener())                          // 指定日志平铺器，默认为 DefaultFlattener
    .build();

XLog.init(                                                 // 初始化 XLog
    config,                                                // 指定日志配置，如果不指定，会默认使用 new LogConfiguration.Builder().build()
    androidPrinter,                                        // 添加任意多的打印器。如果没有添加任何打印器，会默认使用 AndroidPrinter(Android)/ConsolePrinter(java)
    consolePrinter,
    filePrinter);
```

