NDK 简介：

​	如百度地图，libs 文件下面有 .so 文件，这个 so 文件就是用 c 和  c++ 写的。

​	JNI 和 NDK 的区别：JNI 只是一种方式，如用 java 语言去调用 c 和 c++

​	NDK：是 Android 的一种开发工具包，快速开发 C ，C++ 的动态库，并自动将 so 和 应用打包成 apk，即可通过 NDK 在 Android 中使用 JNI 与本地代码(C,C++)进行交互

​	好处：防止反编译 ，JAVA 就是 C 和 C++ 写的，很多算法都写不了，只能用 C 和 C++ 写。速度比java运行快，运行效率高。比较方便，IOS ，Android ，Java 后台都可以调用。

