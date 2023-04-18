### 前言

PackageManagerService (PMS) 是 Android 提供的包管理系统服务，用来管理所有的包信息，以及安装，卸载，更新，解析 AndroidManifest 文件，将解析出来的数据生成 javaBean ，保存在 List 中。