### 常用的图片处理：

​		Drawable ：通用的图形对象，用于装载常用格式的图像，可以是 png ，jpg ，可以理解为一个用来放画的画框

​		Bitmap ：我们可以把它看做一个画架，把图片放在上面，我们可以进行一些处理，例如获取图片的文件信息，旋转切割，放大缩小等。

​		Canvas ：画布，我们可以再上面进行绘制，可以使用 Paint 来画出各种形状，Path 来绘制路径

​		Matrix ：矩阵，用来做图像特效处理的，颜色矩阵，还以偶 Matrix 进行图像的平移，缩放，旋转等。

### Bitmap

首先看一下 Bitmap

```java
 /**
     * Private constructor that must received an already allocated native bitmap
     * int (pointer).
     */
    // called from JNI
    Bitmap(long nativeBitmap, int width, int height, int density,
            boolean isMutable, boolean requestPremultiplied,
            byte[] ninePatchChunk, NinePatch.InsetStruct ninePatchInsets) {
```

私有的构造方法，我们不可以直接创建 bitmap 的对象，但是可以通过 BitmapFactory 来创建 bitmap 

### BitmapFrctory

```java
public static Bitmap decodeFile(String pathName, Options opts)
public static Bitmap decodeResource(Resources res, int id, Options opts)
```

这种方法有很多，通过使用这些方法就可以创建出对应的 bitmap 对象，注意看这些方法的最后一个参数都是 Options，这是什么呢，接着我们来看一下：

### BitmapFactory.Options

这个类是 BitmapFractor 的静态内部类，这个类中定义了很多参数，我们可以对这些参数进行设置，例如防止内存溢出等。

中文文档 [Bitmap](https://www.cnblogs.com/over140/archive/2011/11/21/2256727.html)

### 使用方法

#### 	bitmap

-  public boolean **compress** (Bitmap.CompressFormat format, int quality, OutputStream stream) 将位图的压缩到指定的OutputStream，可以理解成将Bitmap保存到文件中！ **format**：格式，PNG，JPG等； **quality**：压缩质量，0-100，0表示最低画质压缩，100最大质量(PNG无损，会忽略品质设定) **stream**：输出流 返回值代表是否成功压缩到指定流！ 

-  void **recycle**()：回收位图占用的内存空间，把位图标记为Dead 
-  boolean **isRecycled**()：判断位图内存是否已释放 
-  int **getWidth**()：获取位图的宽度 
-  int **getHeight**()：获取位图的高度 
-  boolean **isMutable**()：图片是否可修改 
- int **getScaledWidth**(Canvas canvas)：获取指定密度转换后的图像的宽度
- int **getScaledHeight**(Canvas canvas)：获取指定密度转换后的图像的高度

静态方法

-  Bitmap **createBitmap**(Bitmap src)：以src为原图生成不可变得新图像 

-  Bitmap **createScaledBitmap**(Bitmap src, int dstWidth,int dstHeight, boolean filter)：以src为原图，创建新的图像，指定新图像的高宽以及是否变 
-  Bitmap **createBitmap**(int width, int height, Config config)：创建指定格式、大小的位图 
-  Bitmap **createBitmap**(Bitmap source, int x, int y, int width, int height)以source为原图，创建新的图片，指定起始坐标以及新图像的高宽。 

关于 BitmapFactory 和 Options 的使用直接查 api 即可

### Bitmap 的压缩

​	图片的压缩格式

​	

| Enum Value |                                                              |
| ---------- | ------------------------------------------------------------ |
| ALPHA_8    | 每个像素都存储为一个半透明(alpha)通道                        |
| ARGB_4444  | 此字段在 api13 中弃用，由于配置的质量比较差，建议使用 ARGB_888 |
| ARGB_8888  | 每个像素存储在 4 个字节                                      |
| RGB_565    | 每个像素存储在 2 个字节中                                    |

​	在做压缩处理的时候，可以通过改变 Bitmap 的图片格式，来达到压缩的效果，其实压缩最主要的就是改变宽高，要么就是通过减少单个像素占用的内存。

