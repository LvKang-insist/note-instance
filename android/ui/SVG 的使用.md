 **SVG(Scalable Vector Graphics，可伸缩矢量图形 )是W3C推出的一种开放标准的文本式矢量图形描述语言，它是基于XML、专门为网络而设计的图像格式，SVG是一种采用XML来描述二维图形语言，所以它可以直接打开xml文件来修改和编辑** 

在 Android 中如何使用 svg 呢，下面我们来看一下：

### 自定义 svg 图片

#### 1，创建 svg 格式的图片

​		制作地址：https://svg.wxeditor.com/

​		在上面这个网站中可以在线创建 svg 图片，最后 ctrl + s 进行保存即可。

![1571127572498](SVG%20%E7%9A%84%E4%BD%BF%E7%94%A8.assets/1571127572498.png)



#### 2，将 svg 转换为 vector Drawable 

​		工具连接：https://pan.baidu.com/s/1bp9HANH

​		使用转换工具，将上面创建好的图片导入到转换工具，然后进行格式转换。

​		![1571128146243](SVG%20%E7%9A%84%E4%BD%BF%E7%94%A8.assets/1571128146243.png)

​		点击圈起来的地方进行导入 svg 格式文件，然后就会自动生成代码如下：

​		![1571128251574](SVG%20%E7%9A%84%E4%BD%BF%E7%94%A8.assets/1571128251574.png)

#### 3，使用

然后复制这些代码，打开 as，在 drawable 文件夹下创建一个 xml 文件，并粘贴这些代码，即可使用，如下：

![1571128458190](SVG%20%E7%9A%84%E4%BD%BF%E7%94%A8.assets/1571128458190.png)

### 使用 as 自带的 svg 图片

​	 在drawable文件夹上右键->new->Vector Asset  

​	 选择要使用的图片，可以设置大小 和 透明度

​	 创建成功后就可以再 drawable 文件夹中看到了。

