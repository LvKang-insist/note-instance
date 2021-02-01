









![0005b72e6a0746b0be300e6117d1595b_tplv-k3u1fbpfcp-zoom-1](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210104125353.jpg)

### 基础 Widget

在 Fluter 中，几乎所有的都是一个 widget ，与原生开发不同的是，widget 的范围更加广阔，他不仅可以表示 UI 元素，也可以表示一些功能的组件，如手势检测的 widget，用于主题数据传递的 `Theme` 等等。所以，在大多数时候，可以认为 widget 就是一个控件，不必纠结于概念

Widget 的功能是 “描述一个 UI 元素的配置数据”，widget 并不是表示最终绘制在屏幕上的显示元素，正在代绘制屏幕上的是 `Element` ，下面就看一下 `Element`

### Widget 与  Element

在 Flutter 中，Widget 的功能是 "描述一个 UI 元素的配置数据"，也就是说，**Widget 其实并不是表示最终绘制在设备屏幕上的显示元素，它只是描述显示元素的一个配置数据**

实际上，Flutter 中真正代表屏幕上显示元素的类是  `Element` ，也就是说 Widget 只是描述 `Element` 的配置数据，前期读者只需要知道：**Widget 只是 UI 元素的一个配置数据，并且一个 Widget 可以对应多个 Element**。这是因为同一个 Widget 可以被添加到 UI 树的不同部分，而真正渲染时，UI 树的每一个 `Element` 都会对应一个 Widget 对象 。总结一下：

- Widget 实际上就是 `Element` 的配置数据。Widget 树实际上是一个配置数，而真正渲染 UI 树是由 `Element` 构成

  不过由于是 `Element` 是通过 Widget 生成的，所以他们之间是有对应关系，在大多数场景，我们可以广泛的认为 Widget 树就是指 UI 控件树或 UI 渲染树

- 一个 Widget 对象可以对应多个 Element。这个很好理解，根据同一份配置（Widget），可以创建多个实例（Element）

### Widget 类

```dart
abstract class Widget extends DiagnosticableTree {
  /// Initializes [key] for subclasses.
  const Widget({ this.key });

  final Key? key;

  @protected
  @factory
  Element createElement();

  @override
  String toStringShort() {
    final String type = objectRuntimeType(this, 'Widget');
    return key == null ? type : '$type-$key';
  }

  @override
  void debugFillProperties(DiagnosticPropertiesBuilder properties) {
    super.debugFillProperties(properties);
    properties.defaultDiagnosticsTreeStyle = DiagnosticsTreeStyle.dense;
  }

  @override
  @nonVirtual
  bool operator ==(Object other) => super == other;

  @override
  @nonVirtual
  int get hashCode => super.hashCode;

  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }

  static int _debugConcreteSubtype(Widget widget) {
    return widget is StatefulWidget ? 1 :
           widget is StatelessWidget ? 2 :
           0;
    }
}
```

- Widget 类继承自 `DiagnosticableTree`，`DiagnosticableTree` 即诊断树，主要作用是提供调试信息
- Key：这个 key 属性 类似于 React/Vue 中的 key，主要的作用是决定下一次 build 时复用旧的 widget，决定的条件在 `canUpdate()` 方法中。
- createElement()：正如前文所述，一个 Widget 可以对应多个 Element，Flutter Framework 在构建 UI 树时，会先调用此方法生成对应节点的 Element 对象。此方法是 Flutter FrameWork 隐式调用的，在我们开发过程中基本不会调用到。
- _debugConcreteSubtype 复写父类的方法，主要是诊断树的一些特性
-  canUpdate() 是一个静态方法，主要用于在 Widget 树重新更新 build 时服复用的 widget，其实具体来说，应该是：**是否用新的 Widget 对象去更新旧 UI 树上所对应的 `Element` 对象的配置**；通过其源码我们可以看到，只要 `newWidet `与 `oldWidget` 的 `runtimeType` 和 `key` 同时相等时就会用 `newWidget` 去更新 `Element` 对象的配置，否则就会创建新的 `Element`。

另外 `Widget` 类本身是一个抽象类，其中最核心的就是定义了 `createElement()` 接口，在 Flutter 开发中，我们一般都不用直接继承 `Widget` 类来 实现一个新组建，想法，我们经常会通过继承 StatelessWidget 或 StatefulWidget 来间接继承 `Widget` 类，这两个类都继承自 `Widget` 类，并且这两个是非常重要的抽象类，它们引入了 Widget 中的两种模型。接下来 将重点介绍一下这两个类

### StatelessWidget

- 无状态组件

- 继承自 Widget 类，重写了 `createElement()` 方法 

  ```dart
  @override
  StatelessElement createElement() => StatelessElement(this);
  ```

  创建 `StatelessElement` 对象，间接继承自 `Element` 类，与 StatelessWidget 相对应（作为其配置数据）

- `StatelessWidget` **用于不需要维护状态的场景（也就是UI不可修改）**，它通常在 `build`方法中通过嵌套其他 Widget 来构建 UI ，在构建过程中会递归的构建其嵌套的 Widget。

- **栗子**

  ```dart
  class Echo extends StatelessWidget {
    final String text;
    final Color backgroundColor;
  
    const Echo({Key key, @required this.text, this.backgroundColor: Colors.green})
        : super(key: key);
  
    @override
    Widget build(BuildContext context) {
      return Center(
        child: Container(
          child: Text("hello word"),
          color: backgroundColor,
        )
      );
    }
  }
  ```

  上面代码，实现了一个回显字符串的 `Echo` Widget

  > widget 的构造函数参数应使用命名参数，命名参数中的必要参数要添加 `@required` 标注，这样有利于静态代码分析器进行检查。另外，在继承 `widget` 时，第一个参数通常  `key` ，另外，如果 Widget 需要接收自 Widget，那么 child 或者 children 参数通常应该放在参数列表的最后。widget 的属性应该尽肯能被声明为 final，防止被意外改变

  可以使用如下方式去使用它

  ```dart
  void main() {
    runApp(MyApp());
  }
  
  class MyApp extends StatelessWidget {
    @override
    Widget build(BuildContext context) {
      return MaterialApp(
          title: "Widget相关",
          theme: ThemeData(primaryColor: Colors.blue),
          home: Echo(text: "hello word"));
    }
  }
  ```

- Context

  `build` 有一个 `context` 参数，他是 BuildContext 类的实例，表示当前 widget 在 widget 树种的上下文，每个 widget 都会对应一个 context 对象(因为每个 widget都是 widget 树上的一个节点)。实际上，context 是当前 widget 在 widget 树中位置中执行 “相关操作”的一个句柄，比如它提供了从当前 widget 开始向上遍历widget树，以及查找父类 widget 方法

  ```dart
  class ContextRoute extends StatelessWidget {
    @override
    Widget build(BuildContext context) {
      return Scaffold(
        appBar: AppBar(
          title: Text("Context测试"),
        ),
        body: Container(
          child: Builder(builder: (context) {
            // 在Widget树中向上查找最近的父级`Scaffold` widget
            Scaffold scaffold = context.findAncestorWidgetOfExactType<Scaffold>();
            // 直接返回 AppBar的title， 此处实际上是Text("Context测试")
            return (scaffold.appBar as AppBar).title;
          }),
        ),
      );
    }
  }
  ```

### StatefulWidget

- 有状态的组件
- 和 StatelessWidget 一样，StatefulWidget 也是继承自 widget 类，并重写了 createElement 方法，不同的是返回的 Element 对象并不相同；另外 StatefulWidget 类中添加了一个新的接口 createState()
- 至少由两个类组成，一个 StatefulWidget ，一个 state 类
- StatefulWidget 类本身是不变的，但是 State 类中持有的状态在 widget 生命周期中可能会发生变化

```dart
abstract class StatefulWidget extends Widget {
    
  const StatefulWidget({ Key? key }) : super(key: key);

  @override
  StatefulElement createElement() => StatefulElement(this);

  @protected
  @factory
  State createState();
}
```

- `StatefulElement ` 间接继承 Element 类，与 StatefulWidget 相对应（作为配置数据），S他特氟龙Element中可能会多次调用 createState 来创建状态(State)对象

- `createState` 用于创建 Stateful widget 相关的状态，他在 Stateful widget 的生命周期中可能会被多次调用。例如，当一个 Stateful widget同时插入到 widget 树的多个未值日时，Flutter framework 就会调用该方法为每一个位置生成一个独立的 State 实例，其实，本质上就是一个 StatefulElement 对应一个 State 实例

  > Widget 树他可以指 widget 结构树，但是由于 widget 与 Element 有对应关系(一可能对多)，在有些场景(Flutter 的 Sdk 文档中) 也代指 "UI树" 的意思

### State

一个 StatefulWidget 会对应一个 State 类。State 表示与其对应的 StatefulWidget 要维护的状态，State 中保存的状态信息可以：

- 在 widget 构建时可以被同步读取
- 在 Widget 生命周期中可以被改变，当 State 被改变时，可以手动调用 setState() 方法通知 Flutter framework 状态发生改变，flutter framework 收到消息后，会调用其 build 方法重新构建 widget 树，从而达到更新 UI 的目的

State 中两个常用的属性

- widget ：他表示与之关联的 widget 实例，由 Flutter framework 动态设置，不过这种关联并发永久，因为在生命周期中， UI 树上的某一节点 widget 实例自重新构建时可能会发生变化。但 State 实例只会在第一次插入到树中时被创建，当在重新构建时，如果 widget 被修改了，flutter framework 会动态设置 state，widget 为最新的 widget 实例
- context StatefulWidget 对应的 BuildContext，作用同 StatelessWidget 的 BuildContext 一致

### State 生命周期

- `nitState`

  当 Widget 第一次插入到树中 Widget 时调用，对于每一个 State 对象，Flutter framework 只会调用一次该回调，所以通常在该回调中做一些一次性的操作，如状态初始化，订阅子树的时间通知等

  不能再回调中调用 BuildContext.dependOnInheritedWidgetOfExactType ，原因是在初始化完成后，Widget 树中的 `InheritFromoWidget` 也可以会发生变化，所以正确的做法应该是在 build 方法或者 didChangeDependencies 中调用它  

- `didChangeDependencies()`

  当 State 对象依赖发生变化时会被调用

  例如：build 中包含了一个 InheritedWidget，在之后的 build 中 InheritedWidget发生了变化，那么此时 InheritedWidget 的子 widget 的 didChangeDependencies 回调都会被调用。

  典型的场景是当系统语言 Locale 或应用主题改变时， Flutter framework 会 调用 widget 进行回调

- `build()`

  主要是用来构建 Widget 子树的，会在如下场景被调用

  1，在调用 `initState`  之后

  2，在调用 `didUpdateWidget()` 之后

  3，在调用 `setState()` 之后

  4，在调用 `didChangeDependencies()` 之后

  5，在 State 对象树中一个位置移除后(会调用 deactivate) 又重新插入到树的其他位置之后

- `reassemble()`

   此回调是专门为了开发调试而提供的，在热重载(hot reload) 时会被调用，此回调在 Release 模式下永远不会被调用

- `didUpdateWidget()` 

  在 widget 重新构建时，Flutter framework 会调用  `Widget.canUpdate` 来检测 Widget 树中同一个位置的新旧节点，然后去确定是否需要更新，如果 widget.canUpdate 返回 true 则会调用此回调。

  正如之前所说， widget.canUpdate 会在新旧 widget 的 key 和 runtimeType 同时相等时返回 true，也就是说在新旧相等的  widget 的 额可以 和 runtimeType 同时相等时 此方法会被调用

- `deactivate()` 

  当 State 对象从树中被移除时，会调用此回调。

  在一些场景下，Flutter framework 会将 State 对象重新插入到树中，如果包含次 State 对象的子树在树的一个位置移动到另一个位置时(可以通过 GlobalKey 来实现)。如果移除之后没有重新插入到树中则紧接着就会调用 `dispose()` 方法 

- `dispose()` 

  当 State 对象从树中被永久移除时调用；通常子此回调中释放资源

```dart
class CounterWidget extends StatefulWidget {
  final int counter;

  const CounterWidget({Key key, this.counter: 0}) : super(key: key);

  @override
  State<StatefulWidget> createState() {
    return _CounterWidget();
  }
}

class _CounterWidget extends State<CounterWidget> {
  int _counter;

  @override
  void initState() {
    super.initState();
    //初始化
    _counter = widget.counter;
    print("initState：初始化");
  }

  @override
  void didUpdateWidget(covariant CounterWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    print('didUpdateWidget：widget 重新构建');
  }

  @override
  void deactivate() {
    super.deactivate();
    print('deactivate：State 被移除');
  }

  @override
  void reassemble() {
    super.reassemble();
    print('reassemble：热重载');
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    print('didChangeDependencies：State 对象依赖发生变化');
  }

  @override
  void dispose() {
    super.dispose();
    print('dispose：State 永久移除');
  }

  @override
  Widget build(BuildContext context) {
    print('build：构建 widget');
    return Scaffold(
        body: Center(
            child: FlatButton(
      child: Text("$_counter"),
      onPressed: () => setState(() => ++_counter),
    )));
  }
}

```

一个计数器的小栗子，用来观察一下生命周期的变化

1，首先，打开这个页面，查看输出

```dart
I/flutter ( 6725): initState：初始化
I/flutter ( 6725): didChangeDependencies：State 对象依赖发生变化
I/flutter ( 6725): build：构建 widget
```

2，点击热重载按钮，调用如下

```dart
I/flutter ( 6725): reassemble：热重载
I/flutter ( 6725): didUpdateWidget：widget 重新构建
I/flutter ( 6725): build：构建 widget
```

3，点击数字按钮，调用如下

```dart
I/flutter ( 6725): build：构建 widget
```

4，在 widget 树中移除 CountWidget

```dart
class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
        title: "Widget相关",
        theme: ThemeData(primaryColor: Colors.blue),
        // home: CounterWidget(counter: 0)
      	home:Text("hello word")
    );
  }
}
```

然后点击热重载，调用如下：

```dart
I/flutter ( 7366): deactivate：State 被移除
I/flutter ( 7366): dispose：State 永久移除
```

生命周期图如下所示：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210201154740.png" alt="image-20210201154740168" style="zoom:67%;" />

### 获取 State 对象

由于 StatefulWidget 的具体逻辑都在其 State 中，所有很多时候，我们都需要获取 StatefulWidget 对应的 State 对象来调用一些方法，对此，我们有两种方法在子 widget 树中获取父级 StatefulWidget 的 State 对象

#### 通过 Context 获取

context 对象有一个 `findAncestorStateOfType()` 方法，该方法可以从当前节点沿着 widget 树向上查找指定类型的 StatefulWidget 对应 的 State 对象，

```dart
 // 查找父级最近的Scaffold对应的ScaffoldState对象
ScaffoldState _state = context.findAncestorStateOfType<ScaffoldState>();
```

通过 of 静态方法

```dart
 ScaffoldState _state = Scaffold.of(context);
```

#### 通过 GlobalKey

1，给目标 StatefulWidget 添加 GlobalKey

2，通过 GlobalKey 来获取 State 对象

```dart
//定义一个globalKey, 由于GlobalKey要保持全局唯一性，我们使用静态变量存储
static GlobalKey<ScaffoldState> _globalKey= GlobalKey();
...
Scaffold(
    key: _globalKey , //设置key
    ...  
)
```

> 注意：使用 GlobalKey 开销很大，如果有其他方案，应该去避免它，另外同一个 GlobalKey 在整个 widget 树中必须是惟一的，不能重复