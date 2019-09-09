今天在一个项目中使用到了 ButterKnife 我使用的是当时的最新版 10.1.0 ，但是一直在报错，最后经过查找，发现是 Gradle升级到3.0以上 就会和 ButterKnife 冲突，最后得到了一个解决的办法

修改 ButterKnife 的版本为 8.4.0 ，然后Build 进行clean Project ，然后进行同步 即可解决问题

https://blog.csdn.net/Melect/article/details/80671080

```java
//ButterKnife 依赖
/*
 *  当Gradle 升级到3.0 以后会和 ButterKnife 产生冲突。
 *  Gradle 3.0以上支持的ButterKnife版本 为8.4.0，所以将版本改为8.4.0即可
 */
 
 	//noinspection GradleDependency
    implementation 'com.jakewharton:butterknife:8.4.0'
    //noinspection GradleDependency
    annotationProcessor 'com.jakewharton:butterknife-compiler:8.4.0'
```

