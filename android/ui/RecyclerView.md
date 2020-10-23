## RecyclerView 核心知识点

### 1，RecyclerView是什么

- 为有限的屏幕显示大量的数据且灵活的View，如下图

<img src="D:\android\note-instance\android\ui\RecyclerView.assets\image-20201023224410224.png" alt="image-20201023224410224" style="zoom:50%;" />

- 相比较 ListView

  ListView：

  - 只有纵向列表一种布局
  - 没有支持动画的 API 
  - 接口设计和系统不一致，如 setOnItemClickListener
  - 没有强制实现 ViewHolder
  - 性能不如 RecyclerView

  RecyclerView：

  - 默认支持 Linear，Grid ，Staggered Grid 布局
  - 友好的 ItemAnimator 动画 Api。在刷新的时候调用对应的刷新 api 即可看到动画
  - 强制实现 ViewHolder
  -  RecyclerView 的源码是非常解耦的，且性能非常好

2，RecyclerView 中重要的组件

- RecyclerView：一个特殊的 ViewGroup，他本身不会做太多的工作。重要的工作都会交给下面的三个组件来完成
  - LayoutManager：负责布局和摆放 item 
  - ItemAnimator：负责动画
  - Adapter：适配器模式，对数据进行适配，把数据列表转化成 RecyclerView 需要的 ItemViewAdapter

3，简单的使用

​	[Demo](https://blog.csdn.net/baidu_40389775/article/details/82933053)

4，ViewHoder 究竟是什么

​	ViewHolder 和 item 是一一对应的关系，在创建一个item的时候就会创建一个 ViewHolder，这样当 Item 进行复用的时候就可以直接拿到 ViewHolder，从而防止重复进行 findViewById 。

​	所以说就算你没有使用 ViewHolder，你的 item 还是会被复用，不同的是他会重新进行 findViewById 的操作。

​	ViewHolder 的实践：一般情况下我们是在 onBindViewHolder 方法中绑定数据，但是如果是多个条目，那么这种写法就会非常臃肿，这种情况下就可以吧绑定数据的代码写在 ViewHolder 中。

5，RecyclerView 的缓存机制

- 1，Scrap

  屏幕内部的 itemView

- 2，Cache

- 3，ViewCacheExtension

  用户自定义的换乘策略

- 4，RecyclerViewPool

  缓存池

