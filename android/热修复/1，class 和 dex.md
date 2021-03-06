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

​	![1573269124110](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604154803.png)



#### 	如何生成一个 class文件

​		通过 ide 自动生成 class 文件，通过 run 来执行 calss 文件

​		通过 javac 生成 class 文件 ，通过 java 命令去执行 class 文件

#### 	class 文件的作用

​		记录一个类文件中的所有信息，记住是所有的信息，包括类名称，方法，变量等，class 中记录的信息远远多于java源代码中的信息。例如：我们并没有在类中定义 this、super 这样的关键字，但是我们可以使用，这是因为 java 虚拟机在生成 class 文件时已经帮我们记录在里面了。

#### 	class 文件结构

​		一种8位字节的二进制文件 

​		各个数据按顺序紧密的排列，没有间隙，不想有些文件，为了读取方便，则不会让数据紧密排列。而 class 文件这样的好处就是 体积小。

​		每个类或者接口都单独占据一个 class 文件		

​		class 的文件格式采用的是类似结构体的结构来存储数据，这种结构只有两种数据类型：无符号竖和表，其中无符号数属于基本的数据类型，u1，u2，u4，u8 来分别代表 一个字节 ，2,4,8个字节。无符号数可以用阿里描述数字，索引引用，数量值或者 utf-8构成的字符串值，而表是由多个无符号数或其他表构成的复合数据结构，所有的表都以 _info 结尾，表用于描述有层次关系的复合结构数据类型，其实整个 class 文件就是一张表。（但是我觉得更像结构体）

​		下面看一下具体的结构

​	

| 类型             | 名称                | 数量                    | 作用                                                         |
| ---------------- | ------------------- | ----------------------- | ------------------------------------------------------------ |
| u4(无符号四字节) | magic               | 1                       | 加密段，当前calss 文件是否被串改过                           |
| u2               | minor_version       | 1                       | 最小可以被那个版本的 jdk所加载                               |
| u2               | major_version       | 1                       | 当前calss是别那个版本生成的                                  |
| u2               | constant_pool_count | 1                       | 常量池的数量，通常只有一个                                   |
| cp_info          | constant_pool       | constant_poll_count - 1 | 真正的常量池，它内部包含了很多内容，下面有详细解释           |
| u2               | access_flags        | 1                       | 作用域标志，例如这个 class 文件是 public 还是 public final 类型的 等等。下面会放一张图 |
| u2               | this_class          | 1                       | this，jvm 在生成 class 的时候帮我们补充了这个字段，这就是为什么我们没有定义它却能够使用他了 |
| u2               | super_class         | 1                       | 和上面的 this 一样                                           |
| u2               | interfaces_count    | 1                       | 数量                                                         |
| u2               | interfaces          | interfaces_count        | 这两个表示当前类实现了多少个接口，注意：只算直接的，间接的不算，如不会计算父类实现的接口。 |
| u2               | fields_count        | 1                       | 数量                                                         |
| field_info       | fields              | fields_count            | 当前class中的所有成员变量，他里面还包含了其他的信息，如 类型，所属的类等 |
| u2               | medthods_count      | 1                       | 数量                                                         |
| medthod_info     | methods             | methods_sount           | 记录了class 中所有的方法，包含了其他信息。                   |
| u2               | attribute_count     | 1                       | 数量                                                         |
| attribute_info   | attributes          | attributes_count        | 记录类属性相关的，上面没有包含的都会在这里面，如注解等。     |

我们看一下这个表格，class 文件中定义了很多字段，这些字段又包含了非常多的内容。通过这些字段，jvm 就可以找到我们类中所有的内容

access_flags ：作用域

![1573278691458](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604154837.png)

这个图清楚地表示了 access_flags 字段的作用域。

constant_pool：常量池，下面是几种常用的类型

​		CONSTANT_Integer_info

​		CONSTANT_Long_info

​		CONSTANT_String_info

​		// 下面这几个里面存储的并不是真的内容，而是索引，这些索引最终指向的就是上面这几种类型，所以我们所有的信息都是存在常量池中的。

​		CONSTANT_Class_info			//类信息，例如名字等，引用到类的一些信息等

​		CONSTANT_Fieldref_info		//类中变量信息

​		CONSTANT_Methodref_info		//类中方法信息

我们可以通过工具来查看一下 class 文件内容，工具名字为 010 Editor。

![0](class%20%E5%92%8C%20dex.assets/0.png)

struct cp_info_constant_pool[0] 中的 u2，代表的是无符号数，u2 代表访问标志，如 u2 class_index 指向的就是这个方法所属的类。相当于索引吧，

注意看 u1 tag：

​	在常量池中有14种类型，这个常量都是一个表，每一个表都有各自组成的机构，这写常量都有一个特点，每个常量的开始都是一个用 u1 类型的无符号数表示的标志位，如下表所示：

 ![4179925-1d5f4a103a509c17](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604154851.png)

常量池中第一个 struct cp_info constant_poll[0] ，他的后面是 Methodref ，标志是 10 ，我们和上面的表对比一下，找到第 10 个，也是 Methodref ，通过这个表，我们可以查看 u1 tag 到底表示的是那种类型。



![1](class%20%E5%92%8C%20dex.assets/1.png)



如上面的 fields_count 为1，下面就有一个 struct field_info 的表，里面保存了字段的信息，methods_count 为 2，下面就有 methods[0] 和 methods[1]表，来表示这两个方法。这也就对应了上面的表

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

![1573285094072](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604154907.png)

##### 	dex 文件头

![1573285281488](class%20%E5%92%8C%20dex.assets/1573285281488.png)



![1573440355834](class%20%E5%92%8C%20dex.assets/1573440355834.png)

1.   magic ：一般称为魔数，他可以判断当前的 dex 文件是否有效，可以看到他的 大小就是 9 h，也就是他用了 8个1字节的无符号数表示。

2. checksum ： dex 文件的校验和，通过它可以判断 dex 文件是否被损坏

3. signature[] ：  用于校验 dex 文件，其实就是把整个 dex 文件用 SHA-1 签名得到的一个值，

4. file_size ：  表示真个文件的大小，占用了4个字节

5. header_size ：  表示 dex 头部分的大小，占用4字节

6. endian_tga ：字节序标记，用于指定 dex 运行环境的 cpu，预设值为0x123456789 

7. link_size 和 link_off ：分别指定了链接段的大小和文件的偏移，通常都为0

8. map_off ：指定了 DexMapList 的文件偏移

9.  string_ids_size 和 string_ids_offf：这两个字段表示了 dex 文件中所有用到的字符串的个数和位置偏移。通过转换 size 为16，偏移为112。

   ​	下面我们可以看一下 string_ids_size 的16进制和off 的16进制，分别为10 00 00 00 和 70 00 00 00，前者转换10进制后为 16，说明 size 有 16 个字符串，off 则代表偏移 70h ， 最后一个空字符“0”表示的是结尾 。然后我们找到70h 看一下

   ​	  ![12343145](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155039.png)

   下面我们看一下70h 

   ![1573450765210](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155106.png)

   从70 开始，到 AOh 结束，所选中的都是 字符串的偏移量，通过这些偏移量我们才能找字符串，看一下上面第二个框，是 字符串的索引，可以看到他也是从70h 开始，大小是40h，也就是到A0h。在这段区域内保存的才是真正的字符串索引。我们可以看一下70h 第一个 92 01 00 00 ，他代表的偏移地址就是 0192h，接着我们找一下 0192h 

   ![1573451393434](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155148.png)

   可以看到我一共选中了 8个字节，在这8个字节中我们可以用到的有6个，最开始的 06 则表示我们用到的个数，最后的 0 表示的是字符串结尾，下面我们把他们进行转换一下：

   | 十六进制 |  3C  |  69  |  6E  |  69  |  74  |  3E  |
   | :------: | :--: | :--: | :--: | :--: | :--: | :--: |
   |  十进制  |  60  | 105  | 110  | 105  | 116  |  62  |
   |  ASCII   |  <   |  i   |  n   |  i   |  t   |  >   |

   [在线进制转换表](http://tool.oschina.net/hexconvert/)

   [在线 ASCII 对照表](http://ascii.911cha.com/)

   **通过上面这种方式我们就可以找到字符串的16进制，并转换为对应的字符串。**

   其他的都一样，我们也没有必要像上面遮这样找，其实我们可以直接在索引区找到对应的值。例如上面这个例子：

   ![1573452803684](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155257.png)

   这里的数据是从 163开始的，是因为他并没有及时前面的 06 ，这个06至少代表后面要用到 6 个而已。

从 9 往下基本都是这个样子的。通过上面这种方法就可以查到具体的位置。



##### 索引区

类型索引：

![1573285833585](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155346.png)

如 Hellow 类索引，Object 索引，String 索引的。这些都是我们所引用到的。

方法原型索引 ：

![1573285964548](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155415.png)

字段索引：

![1573286058416](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155507.png)

方法索引：

![1573286081193](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155526.png)

如上面的 main 方法 ，printStream 打印方法等。。他会记录当前类所引用的方法索引和继承的方法索引

类索引：

![1573286245923](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155600.png)

 整个dex 文件中所有类索引

map 列表：

![1573286901541](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155656.png)

他是对整个头文件的一个校验

##### 数据区

​	每个索引对应的值就是数据了。

最后看一下整个 dex 未必会的格式

![1573286692820](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155743.png)

### Class 和 Dex 的区别

每个 class 文件是一个表。这个文件只记录了当前java的信息。

dex 将文件划分为了 三个区域，这三个区域存储了整个工程中所有的java 文件的信息，所以 dex 在类越来越多的时候优势就提现出来了。他只要一个dex文件，很多区域都是可以进行复用的，减少了dex 文件的大小。

本质上他们是一样的，dex 是从 class 文件演变而来的，但是 calss 中存在了许多沉余信息，dex 去掉了沉余信息，并进行了整合

![1573287480420](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210604155801.png)

