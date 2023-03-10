### 背景

`WorkManager` 作为 JetPack 中的一员，并不是一个新鲜的玩意，但是用的却比较少，今天我们由浅入深来聊一聊关于 `WorkManager` 的那些事。首先看一看官方的解释 `使用 WorkManager API 可以轻松地调度那些必须可靠运行的可延期异步任务。通过这些 API，您可以创建任务并提交给 WorkManager，以便在满足工作约束条件时运行。` 



### 声明依赖项

```groovy
dependencies {
    def work_version = "2.7.1"
    // (Java only)
    implementation "androidx.work:work-runtime:$work_version"
    // Kotlin + coroutines
    implementation "androidx.work:work-runtime-ktx:$work_version"
}
```