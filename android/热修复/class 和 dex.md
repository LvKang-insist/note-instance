热修复解决的问题：

- 刚发布的应用发现了比较严重的bug

-  有一些小功能想要及时推送给用户

插件和解决的问题

- 解决应用越来越大所带来的各种技术限制
- 解决应用越来越大带来的合作开发的问题

### class 文件结构深入解析

#### 	什么是 class 文件

​		能够被 JVM 识别，加载并执行的文件格式，他就类似于 mp3 的格式，只能运用于特定的地方，如 mp3 只能被播放，而 class 文件则是需要被 JVM 进行加载。

​		并不是只有 java 语言才能生成 class 文件，当然还有其他的一下语言:

​	![1573269124110](class%20%E5%92%8C%20dex.assets/1573269124110.png)



#### 	如何生成一个 class文件

​		通过 ide 自动生成 class 文件，通过 run 来执行 calss 文件

​		通过 javac 生成 class 文件 ，通过 java 命令去执行 class 文件

#### 	class 文件的作用

​		记录一个类文件中的所有信息，记住是所有的信息，包括类名称，方法，变量等，class 中记录的信息远远多于java源代码中的信息。例如：我们并没有在类中定义 this、super 这样的关键字，但是我们可以使用，这是因为 java 虚拟机在生成 class 文件时已经帮我们记录在里面了。

#### 	class 文件结构

​		一种8位字节的二进制文件 (8位即一个字节)

​		各个数据按顺序紧密的排列，没有间隙，不想有些文件，为了读取方便，则不会让数据紧密排列。而 class 文件这样的好处就是 体积小。

​		每个类或者接口都单独占据一个 class 文件

​		下面看一下具体的结构

​	

| 类型                   | 名称                | 数量                    | 作用                                                         |
| ---------------------- | ------------------- | ----------------------- | ------------------------------------------------------------ |
| u4(无符号四字节)       | magic               | 1                       | 加密段，当前calss 文件是否被串改过                           |
| u2                     | minor_version       | 1                       | 最小可以被那个版本的 jdk所加载                               |
| u2                     | major_version       | 1                       | 当前calss是别那个版本生成的                                  |
| u2                     | constant_pool_count | 1                       | 常量池的数量，通常只有一个                                   |
| cp_info(结构体类型)    | constant_pool       | constant_poll_count - 1 | 真正的常量池，它内部包含了很多内容，下面有详细解释           |
| u2                     | access_flags        | 1                       | 作用域标志，例如这个 class 文件是 public 还是 public final 类型的 等等。下面会放一张图 |
| u2                     | this_class          | 1                       | this，jvm 在生成 class 的时候帮我们补充了这个字段，这就是为什么我们没有定义它却能够使用他了 |
| u2                     | super_class         | 1                       | 和上面的 this 一样                                           |
| u2                     | interfaces_count    | 1                       | 数量                                                         |
| u2                     | interfaces          | interfaces_count        | 这两个表示当前类实现了多少个接口，注意：只算直接的，间接的不算，如不会计算父类实现的接口。 |
| u2                     | fields_count        | 1                       | 数量                                                         |
| field_info(结构体类型) | fields              | fields_count            | 当前class中的所有成员变量，他的类型为结构体，说明他里面还包含了其他的信息，如 类型，所属的类等 |
| u2                     | medthods_count      | 1                       | 数量                                                         |
| medthod_info(结构体)   | methods             | methods_sount           | 记录了class 中所有的方法。类型为结构体，包含了其他信息。     |
| u2                     | attribute_count     | 1                       | 数量                                                         |
| attribute_info         | attributes          | attributes_count        | 记录类属性相关的，上面没有包含的都会在这里面，如注解等。     |

我们看一下这个表格，class 文件中定义了很多字段，这些字段又包含了非常多的内容。通过这些字段，jvm 就可以找到我们类中所有的内容

access_flags ：作用域

![1573278691458](class%20%E5%92%8C%20dex.assets/1573278691458.png)

这个图清楚地表示了 access_flags 字段的作用域。

constant_pool：常量池，下面是几种常用的类型

​		CONSTANT_Integer_info

​		CONSTANT_Long_info

​		CONSTANT_String_info

​		// 下面这几个里面存储的并不是整的内容，而是索引，这些索引最终指向的就是上面这几种类型，所以我们所有的信息都是存在常量池中的。

​		CONSTANT_Class_info			//类信息，例如名字等，引用到类的一些信息等

​		CONSTANT_Fieldref_info		//类中变量信息

​		CONSTANT_Methodref_info		//类中方法信息

我们可以通过工具来查看一下 class 文件内容，工具名字为 010 Editor。

![0](class%20%E5%92%8C%20dex.assets/0.png)

![1](class%20%E5%92%8C%20dex.assets/1.png)

如上面的 fields_count 为1，下面就有一个 struct field_info 的结构体，里面保存了字段的信息，methods_count 为 2，下面就有 methods[0] 和 methods[1] 两个结构体，来表示这两个方法。这也就对应了上面的表

#### class 文件弊端

​	内存占用大，不适合移动端

​	堆栈的加载模式，加载速度慢

​	文件 IO 操作多，类查找慢 。每次加载类的时候都要去寻找和加载

### dex文件结构深入解析

#### 什么是 dex 文件

​	能够被 DVM 所识别，加载并执行的文件格式，dex 文件可以 用 c 和c++ 进行生成

#### 如何生成一个 dex 文件

​	通过 IDE 自动帮我们 build 生成

​	手动通过 dx 命令生成 dex 文件

#### dex 文件的作用

​	记录整个工程中所有类文件的信息，是整个工程(class 则是记录当前类的信息)

#### dex 文件结构

​	一种 8 位字节的二进制流文件

​	各个数据按顺序紧密的排列，无间隙

​	整个应用中所有的 java 源文件都放在一个dex中，这里不考虑 multidex

![1573285094072](class%20%E5%92%8C%20dex.assets/1573285094072.png)

​	dex 文件头：

![1573285281488](class%20%E5%92%8C%20dex.assets/1573285281488.png)

字符串索引：

![1573285736701](class%20%E5%92%8C%20dex.assets/1573285736701.png)

​	如上面所示，类名字符串，还有最后一个我定义的 hellow android 字符串

类型索引：

![1573285833585](class%20%E5%92%8C%20dex.assets/1573285833585.png)

如 Hellow 类索引，Object 索引，String 索引的。这些都是我们所引用到的。

方法原型索引 ：

![1573285964548](class%20%E5%92%8C%20dex.assets/1573285964548.png)

字段索引：

![字段索引](class%20%E5%92%8C%20dex.assets/1573286058416.png)

方法索引：

![1573286081193](class%20%E5%92%8C%20dex.assets/1573286081193.png)

如上面的 main 方法 ，printStream 打印方法等。。他会记录当前类所引用的方法索引和继承的方法索引

类索引：

![1573286245923](class%20%E5%92%8C%20dex.assets/1573286245923.png)

 整个dex 文件中所有类索引

map 列表：

![1573286901541](class%20%E5%92%8C%20dex.assets/1573286901541.png)

他是对整个头文件的一个校验

数据区：每个索引对应的值就是数据了。

最后看一下整个 dex 未必会的格式

![1573286692820](class%20%E5%92%8C%20dex.assets/1573286692820.png)

### Class 和 Dex 的区别

每个 class 文件是一个结构体。这个结构体只记录了当前java的信息。

dex 将文件划分为了 三个区域，这三个区域存储了整个工程中所有的java 文件的信息，所以 dex 在类越来越多的时候优势就提现出来了。他只要一个dex文件，很多区域都是可以进行复用的，减少了dex 文件的大小。

本质上他们是一样的，dex 是从 class 文件演变而来的，但是 calss 中存在了许多沉余信息，dex 去掉了沉余信息，并进行了整合

![1573287480420](class%20%E5%92%8C%20dex.assets/1573287480420.png)

