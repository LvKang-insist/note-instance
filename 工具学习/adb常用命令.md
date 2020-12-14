- 远程调试

  adb tcpip 端口

  adb connect ip:端口

- 计算app启动耗时

  adb shell am start -S -W 包名/启动类的全限定名

  ```java
  macmini@MacdeMac-mini ~ % adb shell am start -S -W com.up72.sandan/com.up72.sandan.ui.SplashActivity
  Stopping: com.up72.sandan
  Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.up72.sandan/.ui.SplashActivity }
  Status: ok
  LaunchState: COLD
  Activity: com.up72.sandan/.ui.main.NewMainActivity
  TotalTime: 6509
  WaitTime: 6520
  Complete
  macmini@MacdeMac-mini ~ % 
  ```

  - TotalTime：启动一连串的 Activity 总耗时(有几个 activity 就统计几个)
  - WaitTime：应用程序创建的过程 —— TotalTime