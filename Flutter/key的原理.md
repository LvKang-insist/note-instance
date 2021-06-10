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



### Widget 和 Element 的对应关系

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