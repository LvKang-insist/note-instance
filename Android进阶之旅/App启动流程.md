### Zygote

翻译过来叫受精卵，`Zygote` 进程的创建是由 Linux 系统中的 init 进程创建的。Android 中的所有进程都是直接或者间接由 init 进程 fork 出来的。

`Zygote` 进程负责其他进程的创建和启动，例如 `SystemServer` 进程，当需要启动一个新的 Android 应用程序的时候，ActivityManagerService 就会通过 Socket 通知 Zytote 进程为这个应用创建一个进程。

### Launcher

也就是手机桌面，我们的手机桌面也是一个 App，他的名字就是 `Launcher`，每一个手机应用都是在 Launcher 上显示，而 Launcher 的加载是在手机启动的时候加载 `Zygote`。

然后 Zygote 启动 SystemServer。SystemServer 会启动各种 ManagerServer，包括 ActivityManagerService，并且将这些 ManagerServer 注册到 ServiceManager 容器中。然后 ActivityManagerService 就会启动 Home 应用程序 Launcher。

### ActivityManagerService

ActivityManagerService 简称 AMS，它是用来管理四大组件的。四大组件的跨进程通信都需要和他合作

