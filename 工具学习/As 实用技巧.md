- 查看 app 启动耗时

  第一种，在 Logcat 中，过滤 tag 为 Displayed，勾选 No Filters 即可

  第二种，实用 adb 命令即可，adb shell am start -S -W -R 10 com.example.test/.MainActivity

  ​	-S ：关闭 activity 所属的进程在启动 activity

  ​	-W：等待启动完成

  ​	-R ：重复次数