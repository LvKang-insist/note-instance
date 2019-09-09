在添加retrofit2 依赖 时报了如下错误

Unable to resolve dependency for ':example@debugAndroidTest/compileClasspath': Could not resolve  com.squareup.retrofit2:retrofit:2.1.0.

后来在github 上查了一下如下

![1555480050152](F:\笔记\android\坑\assets\1555480050152.png)

翻译如下

![1555480149385](F:\笔记\android\坑\assets\1555480149385.png)



修改依赖如下：

```
api ' com.squareup.retrofit2:retrofit:2.3.0'
```

即可

