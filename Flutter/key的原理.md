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

可以看到上图中蓝色的数字时三，而红色的是 5，接着修改代码，将蓝色和红色的位置互换，然后热重载一下，如下：

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

在代码中添加了 key，然后就会发现已经没有上面的问题了。但是如果我们给 Box 在包裹一层 Container，然后在次热重载的时候，数字都变成了 0，在去掉 Container 后数字也会变成 0，具体的原因我们在后面说；



### Widget 和 Element 的对应关系



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

> 需要注意的是，继承自 StatelessWidget 的 Widget 是不需要使用 Key 的，因为它本身没有状态，不需要用到 Key。



___

键在具有相同父级的 [Element] 中必须是唯一的。相比之下，[GlobalKey] 在整个应用程序中必须是唯一的。另请参阅：[Widget.key]，其中讨论了小部件如何使用键。

### LocalKey 的三种类型

`LocalKey` 继承自 Key， 翻译过来就是局部键，`LocalKey` 在具有相同父级的 `Element` 中必须是惟一的。也就是说，LocalKey 在同一层级中必须要有唯一性。

`LocalKey` 有三种子类型，下面我们来看一下:

- #### `ValueKey` 

  ```dart
  class ValueKey<T> extends LocalKey {
    final T value;
    const ValueKey(this.value);
  
  
    @override
    bool operator ==(Object other) {
      if (other.runtimeType != runtimeType)
        return false;
      return other is ValueKey<T>
          && other.value == value;
    }
  }
  
  
  ```

  使用特定类型的值来标识自身的键，ValueKey 在最上面的例子中已经使用过了，他可以接收任何类型的一个对象来最为 key。

  通过源码我们可以看到它重写了 == 运算符，**在判断是否相等的时候首先判断了类型是否相等，然后再去判断 value 是否相等**；

- #### `ObjectKey`

  ```dart
  class ObjectKey extends LocalKey {
    const ObjectKey(this.value);
    final Object? value;
  
    @override
    bool operator ==(Object other) {
      if (other.runtimeType != runtimeType)
        return false;
      return other is ObjectKey
          && identical(other.value, value);
    }
  
    @override
    int get hashCode => hashValues(runtimeType, identityHashCode(value));
  }
  ```

  ObjectKey 和 ValueKey 最大的区别就是比较的算不一样，其中首先也是比较的类型，然后就调用 indentical 方法进行比较，其比较的就是内存地址，相当于 java 中直接使用 == 进行比较。而 LocalKey 则相当于 java 中的 equals 方法用来比较值的。

  > 需要注意的是使用 ValueKey 中使用 == 比较的时候，如果没有重写 hashCode 和 == ，那样即使 对象的值是相等的，但比较出来也是不相等的。所以说尽量重写吧！

- #### `UniqueKey`

  ```dart
  class UniqueKey extends LocalKey {
    UniqueKey();
  }
  ```

  很明显，从名字中可以看出来，这是一个独一无二的 key。

  每次重新 build 的时候，UniqueKey 都是独一无二的，所以就会导致无法找到对应的 Element，状态就会丢失。那么在什么时候需要用到这个 UniqueKey呢？我们可以自行思考一下。

  还有一种做法就是把 UniqueKey 定义在 build 的外面，这样就不会出现状态丢失的问题了。

___



### GlobalKey

`GlobalKey` 继承自 Key，相比与 LocalKey，他的作用域是全局的，而 LocalKey 只作用于当前层级。

在之前我们遇到一个问题，就是如果给一个 Widget 外面嵌套了一层，那么这个 Widget 的状态就会丢失，如下：

```dart
 children: <Widget>[
    Box(Colors.red),
    Box(Colors.blue),
  ],
  
 ///修改为如下，然后重新 build
  children: <Widget>[
    Box(Colors.red),
    Container(child:Box(Colors.blue)),
  ],
```

原因在之前我们也讲过，就是因为类型不同。只有在类型和 key 相同的时候才会保留状态 ，显然上面的类型是不相同的；

那么遇到这种问题要怎么办呢，这个时候就可以使用 GlobalKey 了。我们看下面的栗子：

```dart
class Counter extends StatefulWidget {
  Counter({Key key}) : super(key: key);

  @override
  _CounterState createState() => _CounterState();
}

class _CounterState extends State<Counter> {
  int _count = 0;

  @override
  Widget build(BuildContext context) {
    return RaisedButton(
      onPressed: () => setState(() => _count++),
      child: Text("$_count", style: TextStyle(fontSize: 70)),
    );
  }
}
```

```dart
final _globalKey = GlobalKey();

@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: Text(widget.title),
    ),
    body: Center(
      child: Row(
        mainAxisAlignment: MainAxisAlignment.center,
        children: <Widget>[
          Counter(),
          Counter(),
        ],
      ),
    ),
  );
}
```

上面代码中，我们定义了一个 Counter 组件，点击后 count 自增，和一个 GlobakKey 的对象。

接着我们点击 Counter 组件，自增之后，给 Counter 包裹一层 Container 之后进行热重载，就会发现之前自增的数字已经不见了。这个时候我们还没有使用 GlobalKey。

接着我们使用 GlobalKey，如下

```dart
 Row(
     mainAxisAlignment: MainAxisAlignment.center,
     children: <Widget>[
         Counter(),
         Counter(key: _globalKey),
     ],
   ),
 )
```

重新运行，并且点击自增，运行效果如下：

![image-20210610220722876](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210610220722.png)

接着我们来修改一下代码：

```dart
Column(
  mainAxisAlignment: MainAxisAlignment.center,
  children: <Widget>[
    Counter(),
    Container(child: Counter(key: _globalKey)),
  ],
),
```

我们将最外层的 Row 换成了 Column，并且给最后一个 Counter 包裹了一个 Container 组件，猜一下结果会如何？？，我们来看一下结果：

![image-20210610221029925](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210610221029.png)

结果就是 Column 已经生效了，使用了 GlobalKey 的 Counter 状态没有被清除，而上面这个没有使用的则没有了状态。

> 我们简单的分析一下，热重载的时候回重新  `build` 一下，执行到 Column 位置的时候发现之前的类型是 Row，然后之前 Row 的 Element 就会被扔掉，重新创建 Element。Row 的 Element 扔掉之后，其内部的所有状态也都会消失，但是到了最里面的 Counter 的时候，就会根据 Counter 的 globalkey 重新查找对应的状态，找到之后就会继续使用。



#### 栗子：

 在切换屏幕方向的时候改变布局排列方式，并且保证状态不会重置

```dart
Center(
  child: MediaQuery.of(context).orientation == Orientation.portrait
      ? Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Counter(),
            Container(child: Counter(key: _globalKey)),
          ],
        )
      : Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Counter(),
            Container(child: Counter(key: _globalKey)),
          ],
        ),
)
```

上面是最开始写的代码，我们来看一下结果：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210610223034.gif" alt="345" style="zoom:50%;" />

通过上面的动图就会发现，第二个 Container 的状态是正确的，第一个则不对，因为第一个没有使用 GlobalKey，所以需要给第一个也加上 GlobalKey，如下：

```dart
Center(
        child: MediaQuery.of(context).orientation == Orientation.portrait
            ? Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: <Widget>[
                  Counter(key: _globalKey1),
                  Counter(key: _globalKey2)
                ],
              )
            : Row(
                mainAxisAlignment: MainAxisAlignment.center,
                children: <Widget>[
                  Counter(key: _globalKey1),
                  Container(child: Counter(key: _globalKey2))
                ],
              ),
      )
```

但是这样的写法确实有些 low，并且这种需求我们其实不需要 GlobalKey 也可以实现，代码如下：

```dart
 Center(
  child: Flex(
    direction: MediaQuery.of(context).orientation == Orientation.portrait
        ? Axis.vertical
        : Axis.horizontal,
    mainAxisAlignment: MainAxisAlignment.center,
    children: <Widget>[Counter(), Counter()],
  ),
)
```

使用了 Flex 之后，在 build 的时候 Flex 没有发生改变，所以就会重新找到 Element，所以状态也就不会丢失了。

但是如果内部的 Container 在屏幕切换的过程中会重新嵌套，那还是需要使用 GlobalKey，原因就不需要多说了吧！

___



#### GlobalKey 的第二种用法



Flutter 属于声明式编程，如果页面中某个组件的需要更新，则会将更新的值提取到全局，在更新的时候修改全局的值，并进行 setState。这就是最推荐的做法。如果这个状态需要在两个 widget 中共同使用，就把状态向上提升，毫无疑问这也是正确的做法。

但是通过 `GlobalKey` 我们可以直接在别的地方进行更新，获取状态，widget中数据等操作。前提是我们需要拿到 GlobalKey 对象，其实就类似于 Android 中的 findViewById 拿到对应的控件，但是相比 GlobalKey，GlobalKey 可以获取到 State，Widget，RenderObject 等。

下面我们看一下栗子：

```dart
final _globalKey = GlobalKey();

@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: Text(widget.title),
    ),
    body: Center(
      child: Flex(
        direction: MediaQuery.of(context).orientation == Orientation.portrait
            ? Axis.vertical
            : Axis.horizontal,
        mainAxisAlignment: MainAxisAlignment.center,
        children: <Widget>[
          Counter(key: _globalKey),
        ],
      ),
    ),
    floatingActionButton: FloatingActionButton(
      onPressed: () {},
      tooltip: 'Increment',
      child: Icon(Icons.add),
    ),
  );
```

和之前的例子差不多，现在只剩了一个 Counter 了。现在我们需要做的就是在点击  FloatingActionButton 按钮的时候，使这个 Counter 中的计数自动增加，并且获取到他的一些属性，代码如下：

```dart
  floatingActionButton: FloatingActionButton(
    onPressed: () {
      final state = (_globalKey.currentState as _CounterState);
      state.setState(() => state._count++);
      final widget = (_globalKey.currentWidget as Counter);
      final context = _globalKey.currentContext;
      final render =
          (_globalKey.currentContext.findRenderObject() as RenderBox);
      ///宽高度  
      print(render.size);
      ///距离左上角的像素
      print(render.localToGlobal(Offset.zero));
    },
    child: Icon(Icons.add),
  ),
);
```

```
I/flutter (29222): Size(88.0, 82.0)
I/flutter (29222): Offset(152.4, 378.6)
```

可以看到上面代码中通过 _globakKey 获取到了 三个属性，分别是 state，widget 和 context。

其中使用了 state 对 _count 进行了自增。

而 widget 则就是 Counter 了。

但是 context 又是什么呢，我们点进去源码看一下：

```dart
Element? get _currentElement => _registry[this];

BuildContext? get currentContext => _currentElement;
```

通过上面两句代码就可以看出来 context 其实就是 Element 对象，通过查看继承关系可知道，Element 是继承自 BuildContext 的。

通过这个 context 的 `findRenderObject ` 方法可以获取到 `RenderObject` ，这个  `RenderObject` 就是最终显示到屏幕上的东西，通过 `RenderObject` 我们可以获取到一一些数据，例如 widget 的宽高度，距离屏幕左上角的位置等等。

> `RenderObject` 有很多种类型，例如 RenderBox 等，不同的 Widget 用到的可能并不相同，这里需要注意一点



### 实例

这个例子我们写一个小游戏，一个列表中有很多不同颜色的小方块，通过拖动这些方块来进行颜色的重排序。效果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210616225647.gif" alt="345" style="zoom:50%;" />

通过点击按钮来打乱顺序，然后长按方框拖动进行重新排序；

下面我们来写一下代码：

```dart
final boxes = [
  Box(Colors.red[100], key: UniqueKey()),
  Box(Colors.red[300], key: UniqueKey()),
  Box(Colors.red[500], key: UniqueKey()),
  Box(Colors.red[700], key: UniqueKey()),
  Box(Colors.red[900], key: UniqueKey()),
];

  _shuffle() {
    setState(() => boxes.shuffle());
  }

 @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        ///可重排序的列表
        child: Container(
          child: ReorderableListView(
              onReorder: (int oldIndex, newIndex) {
                if (newIndex > oldIndex) newIndex--;
                final box = boxes.removeAt(oldIndex);
                boxes.insert(newIndex, box);
              },
              children: boxes),
          width: 60,
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _shuffle(),
        child: Icon(Icons.refresh),
      ),
    );
  }
```

ReorderableListView：可重排序的列表，支持拖动排序

- onReorder：拖动后的回调，会给出新的 index 和 旧的 index，通过这两个参数就可以对位置就行修改，如上所示
- scrollDirection：指定横向或者竖向

还有一个需要注意的是 ReorderableListView 的 Item 必须需要一个 key，否则就会报错。

```dart
class Box extends StatelessWidget {
  final Color color;

  Box(this.color, {Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return UnconstrainedBox(
      child: Container(
        margin: EdgeInsets.all(5),
        width: 50,
        height: 50,
        decoration: BoxDecoration(
            color: color, borderRadius: BorderRadius.circular(10)),
      ),
    );
  }
}
```

上面是列表中 item 的 widget，需要注意的是里面使用到了 UnconstrainedBox，因为在 ReorderableListView 中可能使用到了尺寸限制，导致在 item 中设置的宽高无法生效，所以使用了 UnconstrainedBox。

体验了几次之后就发现了一些问题，

- 比如拖动的时候只能是一维的，只能上下或者左右，
- 拖动的时候是整个 item 拖动，并且会有一些阴影效果等，
- 必须是长按才能拖动

___



因为 ReorderableListView  没有提供属性去修改上面的这些问题，所以我们可以自己实现一个类似的效果。如下：

```dart
class _MyHomePageState extends State<MyHomePage> {
  final colors = [
    Colors.red[100],
    Colors.red[300],
    Colors.red[500],
    Colors.red[700],
    Colors.red[900],
  ];

  _shuffle() {
    setState(() => colors.shuffle());
  }

  int _slot;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Listener(
        onPointerMove: (event) {
          //获取移动的位置
          final x = event.position.dx;
          //如果大于抬起位置的下一个，则互换
          if (x > (_slot + 1) * Box.width) {
            if (_slot == colors.length - 1) return;
            setState(() {
              final temp = colors[_slot];
              colors[_slot] = colors[_slot + 1];
              colors[_slot + 1] = temp;
              _slot++;
            });
          } else if (x < _slot * Box.width) {
            if (_slot == 0) return;
            setState(() {
              final temp = colors[_slot];
              colors[_slot] = colors[_slot - 1];
              colors[_slot - 1] = temp;
              _slot--;
            });
          }
        },
        child: Stack(
          children: List.generate(colors.length, (i) {
            return Box(
              colors[i],
              x: i * Box.width,
              y: 300,
              onDrag: (Color color) => _slot = colors.indexOf(color),
              key: ValueKey(colors[i]),
            );
          }),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _shuffle(),
        child: Icon(Icons.refresh),
      ),
    );
  }
}

class Box extends StatelessWidget {
  final Color color;
  final double x, y;
  static final width = 50.0;
  static final height = 50.0;
  static final margin = 2;

  final Function(Color) onDrag;

  Box(this.color, {this.x, this.y, this.onDrag, Key key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return AnimatedPositioned(
      child: Draggable(
        child: box(color),
        feedback: box(color),
        onDragStarted: () => onDrag(color),
        childWhenDragging: box(Colors.transparent),
      ),
      duration: Duration(milliseconds: 100),
      top: y,
      left: x,
    );
  }

  box(Color color) {
    return Container(
      width: width - margin * 2,
      height: height - margin * 2,
      decoration:
          BoxDecoration(color: color, borderRadius: BorderRadius.circular(10)),
    );
  }
}

```



可以看到上面我们将 `ReorderableListView ` 直接改成了 `Stack` , 这是因为在 Stack 中我们可以再 子元素中通过 `Positioned` 来自由的控制其位置。并且在 Stack 外面套了一层 Listener，这是用来监听移动的事件。

接着我们看 Box，Box 就是可以移动的小方块。在最外层使用了 带动画的 `Positioned`，在  ` Positioned ` 的位置发生变化之后就会产生平移的动画效果。

接着看一下 `Draggable `组件，`Draggable` 是一个可拖拽组件，常用的属性如下：

- feedback：跟随拖拽的组件
- childWhenDragging：拖拽时 chilid 子组件显示的样式
- onDargStarted：第一次按下的回调

上面的代码工作流程如下：

1，当手指按住 `Box` 之后，计算  `Box` 的 index 。

2，当手指开始移动时通过移动的位置和按下时的位置进行比较。

3，如果大于，则 index 和 index +1 进行互换，小于则 index 和 index-1互换。

4，进行判决处理，如果处于第一个或最后一个时直接 return。

> 需要注意的是上面并没有使用 UniqueKey，因为 UniqueKey 是惟一的，在重新 build 的时候 因为 key 不相等，之前的状态就会丢失，导致 AnimatedPositioned  的动画无法执行，所以这里使用 ValueKey。这样就能保证不会出现状态丢失的问题。
>
> 当然也可以给每一个 Box 创建一个惟一的 UniqueKey  也可以。

上面例子中执行效果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210619194608.gif" alt="345" style="zoom:50%;" />

由于是 gif 图，所以就会显得比较卡顿。

### 问题

其实在上面最终完成的例子中，还是有一些问题，例如只能是横向的，如果是竖着的，就需要重新修改代码。

并且 x 的坐标是从 0 开始计算的，如果在前面还有一些内容就会出现问题了。例如如果是竖着的，在最上面有一个 appbar，则就会出现问题。

修改代码如下所示：

```dart
class _MyHomePageState extends State<MyHomePage> {
 ///...

  int _slot;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Listener(
        onPointerMove: (event) {
          //获取移动的位置
          final y = event.position.dy;
          //如果大于抬起位置的下一个，则互换
          if (y > (_slot + 1) * Box.height) {
            if (_slot == colors.length - 1) return;
            setState(() {
              final temp = colors[_slot];
              colors[_slot] = colors[_slot + 1];
              colors[_slot + 1] = temp;
              _slot++;
            });
          } else if (y < _slot * Box.height) {
            if (_slot == 0) return;
            setState(() {
              final temp = colors[_slot];
              colors[_slot] = colors[_slot - 1];
              colors[_slot - 1] = temp;
              _slot--;
            });
          }
        },
        child: Stack(
          children: List.generate(colors.length, (i) {
            return Box(
              colors[i],
              x: 300,
              y: i * Box.height,
              onDrag: (Color color) => _slot = colors.indexOf(color),
              key: ValueKey(colors[i]),
            );
          }),
        ),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () => _shuffle(),
        child: Icon(Icons.refresh),
      ),
    );
  }
}
```

在上面代码中将原本横着的组件变成了竖着的，然后在拖动就会发现问题，如向上拖动的时候需要拖动两格才能移动，这就是因为y轴不是从0开始的，在最上面会有一个 appbar，我们没有将他的高度计算进去，所以就出现了这个问题。

这个时候我们就可以使用 GlobalKey 来解决这个问题：

```dart
final _globalKey = GlobalKey();
double _offset;

@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(
      title: Text(widget.title),
    ),
    body: Column(
      children: [
        SizedBox(height: 30),
        Text("WelCome", style: TextStyle(fontSize: 28, color: Colors.black)),
        SizedBox(height: 30),
        Expanded(
            child: Listener(
          onPointerMove: (event) {
            //获取移动的位置
            final y = event.position.dy - _offset;
            //如果大于抬起位置的下一个，则互换
            if (y > (_slot + 1) * Box.height) {
              if (_slot == colors.length - 1) return;
              setState(() {
                final temp = colors[_slot];
                colors[_slot] = colors[_slot + 1];
                colors[_slot + 1] = temp;
                _slot++;
              });
            } else if (y < _slot * Box.height) {
              if (_slot == 0) return;
              setState(() {
                final temp = colors[_slot];
                colors[_slot] = colors[_slot - 1];
                colors[_slot - 1] = temp;
                _slot--;
              });
            }
          },
          child: Stack(
            key: _globalKey,
            children: List.generate(colors.length, (i) {
              return Box(
                colors[i],
                x: 180,
                y: i * Box.height,
                onDrag: (Color color) {
                  _slot = colors.indexOf(color);
                  final renderBox = (_globalKey.currentContext
                      .findRenderObject() as RenderBox);
                  //获取距离顶部的距离
                  _offset = renderBox.localToGlobal(Offset.zero).dy;
                },
                key: ValueKey(colors[i]),
              );
            }),
          ),
        ))
      ],
    ),
    floatingActionButton: FloatingActionButton(
      onPressed: () => _shuffle(),
      child: Icon(Icons.refresh),
    ),
  );
}
```

解决的思路非常简单，

通过 GlobalKey 获取到当前 Stack 距离顶部的位置，然后用dy减去这个位置即可。最终效果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210620223806.gif" alt="345" style="zoom:33%;" />

### 优化细节

经过上面的操作，基本的功能都实现了，最后我们优化一下细节，如随机颜色，固定第一个颜色，添加游戏成功检测等。

最终代码如下：

```dart
class _MyHomePageState extends State<MyHomePage> {
  MaterialColor _color;

  List<Color> _colors;

  initState() {
    super.initState();
    _shuffle();
  }

  _shuffle() {
    _color = Colors.primaries[Random().nextInt(Colors.primaries.length)];
    _colors = List.generate(8, (index) => _color[(index + 1) * 100]);
    setState(() => _colors.shuffle());
  }

  int _slot;

  final _globalKey = GlobalKey();
  double _offset;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(widget.title), actions: [
        IconButton(
          onPressed: () => _shuffle(),
          icon: Icon(Icons.refresh, color: Colors.white),
        )
      ]),
      body: Column(
        children: [
          SizedBox(height: 30),
          Text("WelCome", style: TextStyle(fontSize: 28, color: Colors.black)),
          SizedBox(height: 30),
          Container(
            width: Box.width - Box.margin * 2,
            height: Box.height - Box.margin * 2,
            decoration: BoxDecoration(
                color: _color[900], borderRadius: BorderRadius.circular(10)),
            child: Icon(Icons.lock, color: Colors.white),
          ),
          SizedBox(height: Box.margin * 2.0),
          Expanded(
              child: Center(
            child: Listener(
              onPointerMove: event,
              child: SizedBox(
                width: Box.width,
                child: Stack(
                  key: _globalKey,
                  children: List.generate(_colors.length, (i) {
                    return Box(
                      _colors[i],
                      y: i * Box.height,
                      onDrag: (Color color) {
                        _slot = _colors.indexOf(color);
                        final renderBox = (_globalKey.currentContext
                            .findRenderObject() as RenderBox);
                        //获取距离顶部的距离
                        _offset = renderBox.localToGlobal(Offset.zero).dy;
                      },
                      onEnd: _checkWinCondition,
                    );
                  }),
                ),
              ),
            ),
          ))
        ],
      ),
    );
  }

  _checkWinCondition() {
    List<double> lum = _colors.map((e) => e.computeLuminance()).toList();
    bool success = true;
    for (int i = 0; i < lum.length - 1; i++) {
      if (lum[i] > lum[i + 1]) {
        success = false;
        break;
      }
    }
    print(success ? "成功" : "");
  }

  event(event) {
    //获取移动的位置
    final y = event.position.dy - _offset;
    //如果大于抬起位置的下一个，则互换
    if (y > (_slot + 1) * Box.height) {
      if (_slot == _colors.length - 1) return;
      setState(() {
        final temp = _colors[_slot];
        _colors[_slot] = _colors[_slot + 1];
        _colors[_slot + 1] = temp;
        _slot++;
      });
    } else if (y < _slot * Box.height) {
      if (_slot == 0) return;
      setState(() {
        final temp = _colors[_slot];
        _colors[_slot] = _colors[_slot - 1];
        _colors[_slot - 1] = temp;
        _slot--;
      });
    }
  }
}

class Box extends StatelessWidget {
  final double x, y;
  final Color color;
  static final width = 200.0;
  static final height = 50.0;
  static final margin = 2;

  final Function(Color) onDrag;
  final Function onEnd;

  Box(this.color, {this.x, this.y, this.onDrag, this.onEnd})
      : super(key: ValueKey(color));

  @override
  Widget build(BuildContext context) {
    return AnimatedPositioned(
      child: Draggable(
        child: box(color),
        feedback: box(color),
        onDragStarted: () => onDrag(color),
        onDragEnd: (drag) => onEnd(),
        childWhenDragging: box(Colors.transparent),
      ),
      duration: Duration(milliseconds: 100),
      top: y,
      left: x,
    );
  }

  box(Color color) {
    return Container(
      width: width - margin * 2,
      height: height - margin * 2,
      decoration:
          BoxDecoration(color: color, borderRadius: BorderRadius.circular(10)),
    );
  }
}
```

最终效果如下：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210620231803.gif" alt="345" style="zoom:33%;" />

___

### 参考文献

> B站王叔不秃视频
>
> Flutter 实战

> 如果本文有帮助到你的地方，不胜荣幸，如有文章中有错误和疑问，欢迎大家提出!¬
