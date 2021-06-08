### 概述

在几乎所有的 `widget` 中，都有一个参数 `key` ，那么这个 key 的作用是什么，在什么时候才需要使用到 key ？



### 没有 key 会出现什么问题？

我们直接看一个计数器的例子：

```dart
class Box extends StatefulWidget {
  final Color color;

  Box(this.color);

  @override
  _BoxState createState() => _BoxState();
}

class _BoxState extends State<Box> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      child: Container(
          width: 100,
          height: 100,
          color: widget.color,
          alignment: Alignment.center,
          child: Text(_count.toString(), style: TextStyle(fontSize: 30))),
      onTap: () => setState(() => ++_count),
    );
  }
}
```

```dart
Row(
  mainAxisAlignment: MainAxisAlignment.center,
  children: <Widget>[
    Box(Colors.blue),
    Box(Colors.red),
  ],
)
```

运行效果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210608151911.png" alt="image-20210608151911182" style="zoom:33%;" />

可以看到上图中蓝色的数字时三，而红色的是 5，接着修改代码，将蓝色和红色的位置互换，然后热更新一下，如下：

```dart
 Row(
  mainAxisAlignment: MainAxisAlignment.center,
  children: <Widget>[
    Box(Colors.red),
    Box(Colors.blue),
  ],
),
```

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210608154106.png" alt="image-20210608154106755" style="zoom:33%;" />

接着就会发现，颜色已经互换了，但是数字并没有发生改变，

这时，我们在后面新添加一个红色，如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210608154249.png" alt="image-20210608154249651" style="zoom:50%;" />

接着在删除第一个带有数字 3 的红色，按道理来说应该就会剩下 5,0，结果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210608154407.png" alt="image-20210608154407087" style="zoom:50%;" />

但是你会发现结果依旧是 3,5。

在这个示例中 flutter 不能通过 Container 的颜色来设置标识，所以就没办法确定那个到底是哪个，所以我们需要一个类似于 id 的东西，给每个 widget 一个标识，而 key 就是这个标识。

接着我们修改一下上面的示例：

```dart
class Box extends StatefulWidget {
  final Color color;

  Box(this.color, {Key key}) : super(key: key);

  @override
  _BoxState createState() => _BoxState();
}
```

```dart
 Row(
  mainAxisAlignment: MainAxisAlignment.center,
  children: <Widget>[
    Box(Colors.blue, key: ValueKey(1)),
    Box(Colors.red, key: ValueKey(2)),
  ],
)
```

在代码中添加了 key，然后就会发现已经没有上面的问题了。但是如果我们给 Box 在包裹一层 Container，然后在次热更新的时候，数字都变成了 0，在去掉 Container 后数字也会变成 0，具体的原因我们在后面说；



### Widget 和 Element 的关系

#### 

widget 的定义就是 `对一个 Element 配置的描述`，也就是说，widget 只是一个配置的描述，并不是真正的渲染对象，就相当于是 Android 里面的 xml，只是描述了一下属性，但他并不是真正的 View。并且通过查看源码可知 widget 中有一个 createElement 方法，用来创建 Element。

而 Element 则就是 Widget 树 中特定位置对应的实例，如下图所示：

![image-20210608183249759](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210608183249.png)



**上图刚好对应上面的例子：**

**在没有 key 的情况下，**如果替换掉 第一个和第二个 box 置换，那么第二个就会使用第一个 box 的 Element，所以他的状态不会发生改变，但是因为颜色信息是在 widget 上的，所以颜色就会改变。**最终置换后结果就是颜色改变了，但是里面的值没有发生变化。**

又或者删除了第一个 box，第二个box 就会使用第一个 boxElement 的状态，所以说也会有上面的问题。

**加上 key 的情况：**

加上 key 之后，widget 和 element 会有对应关系，如果 key 没有对应就会重新在同层级下寻找，如果没有最终这个 widget 或者 Element 就会被删除

**解释一下上面遗留的问题**

在 Box 外部嵌套 Container 之后状态就没有了。这是因为 **判断 key 之前首先会判断类型是否一致，然后在判断 key 是否相同。**

正因为类型不一致，所以之前的 State 状态都无法使用，所以就会重新创建一个新的。