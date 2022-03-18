### 简介

ViewRootImpl 是 View 的最高层级，是所有 View 的根。ViewRootImpl 实现了 View 和 WindowManager 之间所需要的协议。ViewRootImpl 的创建过程是从 `WindowManagerImpl` 中开始的。View 的测量，布局，绘制以及上屏，都是从 ViewRootImpl 中开始的。

我们通过一张图来认识一下它：

![image-20220317223445177](https://cdn.jsdelivr.net/gh/LvKang-insist/PicGo/202203172234346.png)



- Window

  我们知道界面中所有的元素都是由 View 构成的，View 是依附于 Window 上面的。Window 只是一个抽象概念，把界面抽象成一个 窗口，也可以抽象成一个 View。

- ViewManange

  一个接口，内部定义了三个方法，用来对 VIew 的添加，更新和删除。

  ```java
  public interface ViewManager
  {
      public void addView(View view, ViewGroup.LayoutParams params);
      public void updateViewLayout(View view, ViewGroup.LayoutParams params);
      public void removeView(View view);
  }
  ```

- WindowManager

  也是一个接口，继承自 ViewManager，在应用程序中，通过 WindowManager 来管理 Window，将 View 附加到 Window 上。他有一个实现类 `WindowManagerImpl`。

- WindowManagerImpl

  `WindowManager` 的实现类，`WindowManagerImpl` 中的内部方法实现都是通过代理类 `WindowManagerGlobal` 来完成。

- WindowManagerGlobal

  `WindowManagerGlobal` 是一个单例，也就是说一个进程中只有一个 `WindowManagerGlobal`对象，他服务与所有页面的 View。

- ViewParent

  一个接口，定义了将成为 View 父级类的职责。

- ViewRootImpl

  视图层次结构的顶部。一个 Window 对应着一个 ViewRootImpl 和 一个 VIew。这个 View 就是被 `ViewRootImpl` 操作的。

> 一个小栗子，我们都只 Actiivty 中 会创建一个 Window 对象。 `setContentView` 方法中的 View 最终也会被添加到 Window 对象中的 `DecorView` 中，也就是说一个 Window 中对应着一个 View。这个 View 是被  `RootViewImpl` 操作的。
>
>  WindowManager 就是入口。通过 WindowManager 的 addView 添加一个 Window(也可以理解为 View)，然后就会创建一个 ViewRootImpl，来对 view 进行操作，最后将 View 渲染到屏幕的窗口上。
>
> 例如 Activity 中，在 onresume 执行完成后，就会获取 Window 中的 DecorView，然后通过 WindowManager 把 DecorView 添加到窗口上，这个过程中是由 RootViewImpl 来完成测量，绘制，等操作的。

> 如果对 Window，WindowManager 不太熟悉可以先看一下这篇文章

