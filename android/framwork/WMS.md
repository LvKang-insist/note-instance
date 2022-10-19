### 前言

`WindowManagerService` 简称 WMS ，是系统的核心服务，主要分为四大部分，风别是 `窗口管理`，`窗口动画`,`输入系统中转站`,`Surface 管理` 。

### 了解 WMS

WMS 的职责很多，主要的就是下面这几点：

- 窗口管理：WMS是窗口的管理者，负责窗口的启动，添加和删除，另外窗口的大小也时有 WMS 管理的，管理窗口的核心成员有`DisplayContent`,`WindowToken` 和 `WindowState`
- 窗口动画：窗口间进行切换时，使用窗口动画可以更好看一些，窗口动画由 WMS 动画子系统来负责，动画的管理系统为 WindowAnimator
- 输入系统的中转站：通过对窗口触摸而产生的触摸事件，`InputManagerServer(IMS)` 会对触摸事件进行处理，他会寻找一个最合适的窗口来处理触摸反馈信息，WMS 是窗口的管理者，因此理所当然的就成为了输入系统的中转站。
- Surface 管理：窗口并不具备绘制的功能，因此每个窗口都需要有一个块 Sface 来供自己绘制，为每个窗口分配 Surface 是由 WMS 来完成的。

WMS 的职责可以总结为下图：

![2828107-9ddff2816fe1805e](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/img/202210191446431.webp)