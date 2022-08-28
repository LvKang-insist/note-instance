### MVC

- M：表示 Bean
- V：表示 View
- C：Activity 和 Fragment 已经子类

### MVP

- M：数据层（数据库，文件操作，网络等）
- V：View 和 Activity ，Fragment
- P：中介（Presenter）

### MVP的目的：

​	将 UI 和 数据进行解耦和（分离）。使用接口将 页面和数据分离，通过接口去向 P 层获取数据，然后 P层通过接口将数据回调给 V

坏处就是 P 层会有 V 的引用，并且无法感知 V 的生命周期，又可以 V 已经销毁了，P 还持有 V，就会内存泄露。

当页面特别复杂，接口就会变得很庞大

#### MVVM

- M：实体模型，数据库，网络数据获取的处理，**最大的用处是获取数据的职责**，在 Model 中，数据的获取，存储，数据的状态变化都是 Model 层的业务，Model 层包括实体类型，获取网络接口，本地存储，数据的变化监听等，Model 提供数据接口供 ViewModel 调用，经数据转换和操作并最终显示在 VIew 上面
- V：对应 Activity 和 Fragment，负责 View 的绘制以及与用户的交互，View 只负责 UI 相关的工作，不做业务相关的事，数据的获取更新都是在 VIewModel 中完成(需要使用 DataBinding)，Activity 要做的就是初始化一些控件。View 可以提供一些更新 UI 的接口，但是在 MVVM 中一般都通过数据驱动 UI，但是也可以通过 LiveData 来实现这些。简单的说：View 不做任何业务逻辑，不涉及操作数据，不处理数据，UI 和数据严格分开。
- VM：负责完成 View 月 Model 之间的交互，ViewModel 只做 业务逻辑和数据相关的事，不做任何 UI 相关的事情，一般情况下在 MVVM 中会通过 DataBinding 的双向绑定以数据驱动 UI，也有一部分人是通过 LiveData 来实现的。

相比于 MVP ，好处非常明显了，取消了接口，采用了 ViewModel，ViewModel 本身还有恢复机制，页面重建的时候 ViewModel 不会重建，并且还会保留数据，不会持有 V 的引用，配合 LiveData 监听页面的生命周期等。

不足之处，需要定义 livedata 的模版代码，一个可见，一个不可见，如果数据较多，就会显得特别多，还有就是交互比较分散。



MVI

和 MVVM 很相似，只不过更加强调数据的单向流动和唯一数据源

![image-20220803173354837](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202208031733937.png)

主要分为以下几个部分：

- Model：MVI 中的 Model 主要指的是 UI 状态，例如页面加载状态，控件位置等都是一种状态。
- View：是一个 Activity 或者任意一个 Ui 承载单元，MVI 中的 View 通过订阅 Model 的变化实现界面的刷新
- Intent：用户的任何操作都可以被包装成 Intent 后发送给 Model 层进行请求。

整个流程如下：

1. 用户以 Intent 的形式通知 Model
2. Model 基于 Intent 更新 State
3. View 接收到 State 变化刷新 UI 

数据永远是单向的，不能反向流动：

![](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202208031739149.png)
