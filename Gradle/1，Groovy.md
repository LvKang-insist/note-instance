### Grade

​		是一种基于 Apache Ant 和 Maven 概念的项目自动化构建工具。它使用一种基于 Groovy的特定领域语言来声明项目设置，而不是传统的 XML 。Gradle 就是工程的管理，帮我们做了依赖，打包，部署，发布，各种渠道的差异管理工作

#### 优势

​	1，一种最新的，功能更强大的构建工具，用它逼格更高

​	2，使用程序替代 Xml ，项目构建更灵活

​	3，丰富的第三方插件，随性所欲的使用

​	4，Maven、Ant 能做的 ，Gradle 都能做，但是 Gradle 能做的他们却不一定能做



### DSL 介绍

​	全称 domain specific language ，特定领域语言

​	常见的 DSL 语言：XML , HTML 

​	

​	DSL 和 通用语言的区别：求专不求全，解决特定问题

### groovy 

#### 特性

​		是一种基于 JVM 的敏捷开发语言，groovy 编写的语言可以编译成字节码文件让 JVM 执行。groovy 可以直接将 groovy 源文件解释执行。

​		结合了 Python 、Ruby 和 Smalltalk 的许多强大的特性

​		groovy 可以与 java 完全结合，而且可以使用 java 所有的库

​		语法上支持动态类型，闭包等新一代语言特性。	

​		无缝集成所有已经存在的 java 类库

​		即支持面对对象，也支持面向过程

#### 	优势

​		一种更加敏捷的编程语言。他在语法上做了很多改变，许多在 java 上的代码 在 groovy 上只需要一点点。

​		入门非常容易，但是功能非常强大。

​		即可以作为 编程语言，也可以作为脚本语言

​		熟练掌握 java 的话会非常容易掌握 groovy



### 简单的使用

​		一般情况下使用 java 打印一句话需要如下：

```java
class HelloGroovy {

    public static void main(String[] args) {
        System.out.println("hello groovy");
    }
}
```

但是使用 Groovy 之后呢？

```java
class HelloGroovy {

     static void main(aggs) {
        System.out.println("hello groovy")
    }
}
```

是不是感觉没有多大区别，只是少写了 pulibc ， ； 等一些无关紧要的东西。接着看：

```groovy
println "hello groovy"
```

是不是感觉很舒服，只有一句话即可完成打印。

