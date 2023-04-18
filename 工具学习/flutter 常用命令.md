## Flutter 常用命令

### flutter doctor

查看所需的依赖项是否安装/配置完成。

### flutter run

启动项目，启动完成后 R 键更新，Q 退出

### flutter run -D

用指定的设备运行项目，R 更新，Q 退出

### flutter build web 

打 web 包，生成的文件在 build 目录下

### flutter build app

打包 app

### flutter build ios

打包 ios

### flutter clean

清理缓存，删除 build/ 和 .dart_tool/目录

### flutter devices

查看当前可用虚拟机

### flutter -h

帮助



### 打包 web

```undefined
flutter build web --web-renderer html 打开速度最快,兼容性好(是指ie,chrome,safari等浏览器兼容)

flutter build web 打开速度一般,兼容性好

flutter build web --web-renderer canvaskit 打开速度最慢,兼容性好
```

flutter build web 可以使用 --web-renderer 指定渲染引擎， 可以是 html 或者 canvaskit，

而未指定的效果是：在移动端使用 html 渲染，在pc使用 canvaskit 渲染；
至于 canvaskit 慢，是因为有静态资源使用了国外资源；

### flutter切换环境

- 查看当前 flutter 环境分支

  ```
  flutter channel
  ```

- 切换分支

  ```
  flutter channel 分支名称
  //例如 flutter channel stable
  ```

  切换后，如果有更新，按照提示操作即可。

  最后可通过 flutter channel 查看分支是否切换成功

#### Flutter 发布包

发布前使用命令检测

```rust
flutter pub pub publish --dry-run
```

发布

```
flutter packages pub publish --server=https://pub.dartlang.org
```


