Android Studio 插件开发

### 1，简介

基于上篇文章实现的 IOC 注解框架，进行一个插件的开发，用于自动生成 id 的 事件的绑定

### 2，插件工具的安装

开发 Android Studio 插件需要使用 intellij 来进行开发。

### 3，项目简介

在 创建项目的时候选择 Intellij Platform Plugin ，然后右边选择 jdk 版本，最后 next 即可。

项目创建成功后默认会打开一个 xml 文件。如下：

```xml
<idea-plugin>
  <id>com.qs.helloWord</id>
  <name>HelloWord</name>
  <version>1.0</version>
  <vendor email="1831712732@qq.com" url="http://www.baidu.com">no</vendor>

  <description><![CDATA[
      Enter short description for your plugin here.<br>
      <em>most HTML tags may be used</em>
    ]]></description>

  <change-notes><![CDATA[
      Add change notes here.<br>
      <em>most HTML tags may be used</em>
    ]]>
  </change-notes>

  <!-- please see http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/build_number_ranges.html for description -->
  <idea-version since-build="173.0"/>

  <!-- please see http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/plugin_compatibility.html
       on how to target different products -->
  <depends>com.intellij.modules.platform</depends>

  <extensions defaultExtensionNs="com.intellij">
    <!-- Add your extensions here -->
  </extensions>

  <actions>
    <!-- Add your actions here -->
  </actions>

</idea-plugin>
```

文件为 plugin.xml，其中

id：插件的 id

name：插件的名称

version：插件的版本

action：执行的动作，例如在 as 中进行右击，然后选择 generate ，就可以看到插件的名字，如果点击某个插件，则执行的就是 一个action

description：描述

change-notes：版本改变之后的描述

### 4，Action

​		action 是执行的动作，我们需要的效果如下：在 as 中 右击选择 generate，可以找到 HelloWord，点击后弹出一个窗口即可。

- 创建 action

  <img src="AndroidStudio%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91.assets/QQ%E6%88%AA%E5%9B%BE20200505110207.png" alt="QQ截图20200505110207" style="zoom:50%;" />

  点击 action 会出现一个 窗口，其中

  Action ID ：Action 的id

  ClassName ：Action 的 name

  Name：代表 generate 中显示的名字

  Descriptions：描述

  groups：显示的菜单地方，例如：

  <img src="AndroidStudio%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91.assets/QQ%E6%88%AA%E5%9B%BE20200505110915.png" alt="QQ截图20200505110915" style="zoom:50%;" />

  

   选择 codeMenu ，那就会显示在 as 控制栏的 code 中。最右边的 anchor 代表显示的位置

  最下面两个是快捷键，可直接设置

- action 的逻辑

  ```java
  public class HelloWorld extends AnAction {
  
      @Override
      public void actionPerformed(AnActionEvent e) {
          //在光标的位置弹出一个窗口，显示 HelloWorld
  
          Editor editor = e.getData(PlatformDataKeys.EDITOR);
  
          if (editor == null) {
              return;
          }
  
          //显示一个 窗口
          showPopupBalloon(editor, "HelloWord", 5);
      }
  
      /**
       * 显示一个窗口5
       *
       * @param editor 光标
       * @param result 内容
       * @param time   世界，单位 秒
       */
      public static void showPopupBalloon(final Editor editor, final String result, final int time) {
  
          ApplicationManager.getApplication().invokeLater(new Runnable() {
              @Override
              public void run() {
                  JBPopupFactory factory = JBPopupFactory.getInstance();
                  factory.createHtmlTextBalloonBuilder(result, null, new JBColor(new Color(116, 214, 238), new Color(76, 112, 117)), null)
                          .setFadeoutTime(time * 1000)
                          .createBalloon()
                          .show(factory.guessBestPopupLocation(editor), Balloon.Position.below);
  
              }
          });
      }
  }
  ```

- 运行，生成插件

  上面的写完之后就可以运行了。运行的时候会重启，然后随便打开一个文件，点击 code，就可以看到 HelloWorld 了。

  ![QQ截图20200505115702](AndroidStudio%E6%8F%92%E4%BB%B6%E5%BC%80%E5%8F%91.assets/QQ%E6%88%AA%E5%9B%BE20200505115702.png)

   点击如上，就会生成一个 jar 包，jar 包位置在项目的目录下

  然后打开 as ，找到设置中的 plugins ，然后从本地安装 插件，找到刚才的 jar包，然后倒入，最后重启即可。

5，IOC 注解生成插件

https://github.com/HCDarren/DarrenIOC