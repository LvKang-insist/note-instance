### 前言

`Paging` 库可以帮助你加载和显示本地的数据或者是网络中的数据。此库可以让你的应用更加高效的利用网络带宽和系统资源，`Paging` 库组件旨在契合推荐的Android应用架构，流畅的继承其他 `Jetpack` 组件，并提供了一流的 `Kotlin` 支持。

### 使用 Paging 库的优势

- 分页数据的内存中缓存，该功能可以确保在处理分页时高效的利用系统资源
- 内置的请求重复信息删除功能，可以确保应用高效利用网络带宽和系统资源
- 可配置的 RecyclerView 适配器，会在用户滚动到末尾时自动请求数据
- 对 Kotlin 协程和 `Flow` 以及 `LiveData` 和 `RxJava` 的一流支持
- 内置对错误的功能的支持，包括刷新和重试功能

### 架构分层

`Paging` 库直接集成到推荐的 Android 应用架构中，该库的组件在应用的三个层中运行

<img src="https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202206101716853.png" alt="image-20220610171642793" style="zoom:50%;" />

- Respository 层

  `Respository ` 中主要使用的组件是 `PagingSource`，其中定义了数据源，以及如何获取数据等，可以从本地或者网络加载数据。

  还可能使用的库是 `RemoteMediator`，该对象会出来来自分层的数据源，例如本地数据库的缓存和网络数据源的分页

- ViewModel 层

  `Pager` 是组件库提供的一个公共 API，基于 `PagingSource` 对象和 `PagingConfig` 配置对象来构造响应流中公开的 `PagingData` 实例 。

- 界面层

  界面层主要使用的是 `PagingDataAdapter` ，他是一种处理分页数据的 `RecyclerView` 适配器，另外还可以使用 `AsyncPagingDataDiffer` 组件来构建自定义的适配器
