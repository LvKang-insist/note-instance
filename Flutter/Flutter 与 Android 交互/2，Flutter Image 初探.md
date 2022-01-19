### Image 内存初探

#### Android：

在 Android 中将内存分为 Java 虚拟内存和 Native 内存，各大厂商对 Java 虚拟机内存有一个上限，到达上限后就会触发 OOM 异常，而对 Native 内存使用没有太多的限制，现在的手机内存很大，一般都有较大的 Native 内存富余，那么Android 中的 ImageView 使用的是 Java 虚拟机内存还是 Native 内存呢？

测试：在一个界面上，没点击一次，就在上面堆加一张图片，每张图片都小于上一张的图片，这样就不会出现完全覆盖的情况，如下图所示：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20220117223451.png" alt="image-20220117223451052" style="zoom:50%;" />

测试结果如下：        

- Android 6.0 中：

  使用的是 Java 内存 ，共使用了 `104.4Mb` 。

- Android 7.0 中：

  使用的是 Java 内存，共使用了 `144Mb`。

- Android 8.0 中：

  使用的是 Native 内存，共使用了 `196.6Mb`。

- Android 10.0 中：

  使用的是 Native 内存，共使用了 `159Mb`

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20220117223356.png" alt="image-20220117223355995" style="zoom:33%;" />

- Android 12 中：

  使用的是 Native 内存，使用了 `147Mb`

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20220117223745.png" alt="image-20220117223745462" style="zoom:33%;" />

在测试中，Android 6.0 和 7.0 都是 Java 的部分内存增长，而 8.0 ，10.0  和 12 中 则是 Native 部分内存在增长。由此可得出后面的版本中使用的都是 Native 内存。

上面测试中，从 6.0 到 8.0 测试结果参考的是 `https://www.jianshu.com/p/a241b73ce244` ，后面的都是本人亲自测试。

#### Flutter ：

测试方式和上面一样，代码如下：

```dart
@override
Widget build(BuildContext context) {
  return MaterialApp(
    title: 'Material App',
    home: Scaffold(
      appBar: AppBar(
        title: const Text('测试Flutter图片占用内存情况'),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          setState(() {
            addImage();
          });
        },
        tooltip: 'Increment',
        child: const Icon(Icons.add),
      ),
      body: Container(
        child: SizedBox(
          width: 400,
          height: 600,
          child: Stack(
            children: images,
          ),
        ),
      ),
    ),
  );
}

addImage() {
  if (pos < list.length) {
    images.add(Positioned(
      child: Image.network(
        list[pos],
        fit: BoxFit.cover,
        width: 400 - (pos + 30),
        height: 600 - (pos + 60),
      ),
      left: 0,
      top: 0,

    ));
    pos++;
  }
}
```

效果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20220117231000.png" alt="image-20220117230959928" style="zoom: 33%;" />

测试的结果如下：

- Android 6.0 中：

  Graphics 使用  `604Mb`

- Android 8.0 中：

  Graphics 使用 `320Mb`

- Android 10.0 中：