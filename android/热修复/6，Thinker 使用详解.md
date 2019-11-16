### Tinker基本介绍

​		**Tinker 是微信官方的 Andriod 热补丁解决方案，它支持动态下发代码，so库以及资源，让应用在不需要安装的情况下实现更新，当然，你也可以使用 Thinker 来更新你的插件。**

#### 它主要包含以下几部分：

​		1，gradle 编译插件 tinker-patch-gradle-plugin：主要用于在 as 中直接完成 patch 文件的生成

​		2，核心 sdk 库 tinker-android-lib ：核心库，为应用层提供的 api 

​		3，非 gradle 编译用户的命令行版本，tinker-path-cli.jar ：为 eclipse 做的一个工具

#### 为什么使用 Tinker

​		当前市面上 热修复的解决方案很多，但是他们都有一些无法解决的问题，但是 Tinker 的功能是比较全面的。

|             | Tinker | QZone | AndFix | Robust |
| ----------- | :----: | :---: | :----: | :----: |
| 类替换      |  yes   |  yes  |   no   |   no   |
| So 替换     |  yes   |  no   |   no   |   no   |
| 资源替换    |  yes   |  yes  |   no   |   no   |
| 全平台支持  |  yes   |  yes  |  yes   |  yes   |
| 即时生效    |   no   |  no   |  yes   |  yes   |
| 性能损耗    |  较小  | 较大  |  较小  |  较小  |
| 补丁包大小  |  较小  | 较大  |  一般  |  一般  |
| 开发透明    |  yes   |  yes  |   no   |   no   |
| 复杂度      |  较低  | 较低  |  复杂  |  复杂  |
| gradle 支持 |  yes   |  no   |   no   |   no   |
| Rom 体积    |  较大  | 较小  |  较小  |  较小  |
| 成功率      |  较高  | 较高  |  一般  |  最高  |



### Tinker 执行原理及流程

​		基于 android 原生的 ClassLoader ，开发了自己的 ClassLoader，通过自定义的 ClassLoader去加载 patch 文件中的字节码

​		基于 android 原生的 aapt，开发了自己的 aapt ，加载 资源文件

​		基于 Dex 文件的格式，研发了 DexDiff 算法，通过 DexDiff 算法比较两个 apk 文件中的差异 

### 使用 Tinker 完成线上 bug 修复

​		集成

​		





为什么需要继承 ApplicationLike，而不直接在 Application 中初始化呢？

​		因为 ApplicationLike 需要对 Application 的生命周期进行监听，	所以他通过 ApplicationLike 进行代理。通过这个代理可以完成对 Application 的生命周期监听，然后在不同的生命周期做一些别的工作，因为Tinker 初始化非常复杂，所用用 ApplicationLike 进行了代理，这样使用就非常简单了！



#### patch 生成方式

- 使用命令行的方式生成 patch 

  ```java
  java -jar tinker-patch-cli.jar -old old.apk -new new.apk -config tinker_config.xml -out output_path
  ```

  ![1573799472963](6%EF%BC%8CThinker%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573799472963.png)

  ![1573799539754](6%EF%BC%8CThinker%20%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3.assets/1573799539754.png)

  我们用到的就是 patch_signned.apk 这个文件，下面的那个是未签名的。

### Tiner 源码

​		