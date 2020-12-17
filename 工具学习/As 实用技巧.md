- ### 查看 app 启动耗时

  第一种，在 Logcat 中，过滤 tag 为 Displayed，勾选 No Filters 即可

  第二种，实用 adb 命令即可，adb shell am start -S -W -R 10 com.example.test/.MainActivity

  ​	-S ：关闭 activity 所属的进程在启动 activity

  ​	-W：等待启动完成

  ​	-R ：重复次数
  
- #### 删除无用的资源文件

  - 第一种

    app -> Refactor-Remove Unused Resources...

    然后选择 PreView 会预览要清除的资源文件，或者选择 Refactor 直接删除无用资源

  - 第二种

    从as中选择Analyze->Run inspection by Name...

    弹出的对话框中输入unused resources并点击下拉列表的对应项

    之后选择范围，然后点击 ok

    之后就会查找无用的资源，最终会出现一个界面，然后删除资源即可

