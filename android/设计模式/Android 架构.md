### MVC

- M：表示 Bean
- V：表示 View已经子类
- C：Activity 和 Fragment 已经子类

### MVP

- M：数据层（数据库，文件操作，网络等）
- V：View 和 Activity ，Fragment
- P：中介（Presenter）

### MVP的目的：

​	将 UI 和 数据进行解耦和（分离）。

#### MVVM

- M：实体模型，数据库，网络数据获取的处理，**最大的用处是获取数据的职责**，在 Model 中，数据的获取，存储，数据的状态变化都是 Model 层的业务，Model 层包括实体类型，获取网络接口，本地存储，数据的变化监听等，Model 提供数据接口供 ViewModel 调用，经数据转换和操作并最终显示在 VIew 上面
- V：对应 Activity 和 Fragment，负责 View 的绘制以及与用户的交互，View 只负责 UI 相关的工作，不做业务相关的事，数据的获取更新都是在 VIewModel 中完成(需要使用 DataBinding)，Activity 要做的就是初始化一些控件。View 可以提供一些更新 UI 的接口，但是在 MVVM 中一般都通过数据驱动 UI，但是也可以通过 LiveData 来实现这些。简单的说：View 不做任何业务逻辑，不涉及操作数据，不处理数据，UI 和数据严格分开。
- VM：负责完成 View 月 Model 之间的交互，ViewModel 只做 业务逻辑和数据相关的事，不做任何 UI 相关的事情，一般情况下在 MVVM 中会通过 DataBinding 的双向绑定以数据驱动 UI，也有一部分人是通过 LiveData 来实现的。

