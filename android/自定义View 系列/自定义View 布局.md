**布局的过程，就是程序运行时 利用布局文件的代码来计算出实际尺寸的过程**

### 布局的两个阶段：测量阶段 和 布局阶段

​	1,测量阶段，measure 方法被父View 调用，在 measure 中做一些优化工作后，调用 onMeasure 方法来进行实际的自我测量，在 View 和 ViewGroup 中 onMeasure 做的事不太一样。

​	View：view 在 onMeasure 中计算出自己的尺寸然后保存

​	ViewGroup ： ViewGroup 在onMeasure 中调用 所有的子View 的measure 方法进行自我测量，并根据子View 计算出来他们的实际尺寸和位置(实际上 99.99% 的父View 都会使用子View 给出的期望尺寸来作为实际尺寸)然后保存。同时它也会根据子View 的尺寸和位置来计算自己的尺寸然后保存

​	2,布局阶段，layout 方法被父View 调用，在layout 方法中会保存父View 传进来的位置和 尺寸，并且调用 onLayout 来进行实际的内部布局，在View 和 ViewGroup 中，layout 做的事也不一样。

​	View ： 由于没有子View ，所以View 的onLayout 什么也不做

​	ViewGroup ：ViewGroup 在onLayout 中会调用自己的所有子View 的layout 方法，把自己的尺寸和位置传给他们，让他们他们完成自我的内部布局

布局过程 自定义的方式：

三类 ：

1，重写 onMeasure 来修改已有的View 的尺寸

2，重写 onMeasure 来全新的定制 自定义View 的尺寸

3，重写 onMeasure 和 onLayout 来全新的定制 自定义ViewGroup 的内部布局。



## View 和 ViewGroup

View 是所有控件的 基类，我们平常用 所用的 TextView 和 ImageView 等，都是继承自View 。

ViewGroup 可以理解为 View 的组合，他里面可以包含多个 View 以及ViewGroup，而他包含的 ViewGroup 又可以包含View 和 ViewGroup，以此类推，形成一个树

![1559636481091](F:\笔记\android\自定义View 系列\assets\1559636481091.png)

需要注意的是 ViewGroup 继承自View ，ViewGroup 作为 View 或者ViewGroup 这些组件的容器，派生了 多种布局控件的子类，比如 LinearLayout 等，如图所示

![1559636656390](F:\笔记\android\自定义View 系列\assets\1559636656390.png)



​	