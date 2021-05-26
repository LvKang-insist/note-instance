### 概述

通知(Notification) 是 Flutter 中一个重要的机制，在 Widget 树中，每一个节点都可以分发通知，通知会沿着当前节点向上传递，所有父节点都可以同个 NotificationListener 来监听通知；

Flutter 中将这种由子向父传递的机制叫做通知冒泡(Notification Bubbing) 。通知冒泡和用户触摸事件冒泡是相似的，但有一点不同：通知冒泡可以终止，单用户触摸事件不行

> 通知冒泡和 Web 开发中浏览器事件的冒泡原理是相似的，都是事件源想上层传递，我们可以在上层任意节点任意位置来监听/事件，也可以终止冒泡过程，终止冒泡后，通知将不会再向上传递。

### 源码中的例子

Flutter 中很多地方用到了通知，如可滚动组件(Scrollable Widget) 滑动时就会分发滚动通知(ScrollNotification) ，而Scrollbar 正是通过监听 ScrollNotification 来确定滚动条的位置的。

例：监听可滚动组件

```dart
NotificationListener(
  onNotification: (notification){
    switch (notification.runtimeType){
      case ScrollStartNotification: print("开始滚动"); break;
      case ScrollUpdateNotification: print("正在滚动"); break;
      case ScrollEndNotification: print("滚动停止"); break;
      case OverscrollNotification: print("滚动到边界"); break;
    }
  },
  child: ListView.builder(
      itemCount: 100,
      itemBuilder: (context, index) {
        return ListTile(title: Text("$index"),);
      }
  ),
);
```

上例中的滚动通知 ScrollStartNotification，等都是继承自 ScrollNotification 类，不同的子类会包含不同的信息，如 ScollUpdateNotification 有一个 scrollDelta 属性，他记录了移动的位移等

上例中 通过 NotificationListener 来监听子 Listview 的滚动通知，其定义如下：

```dart
class NotificationListener<T extends Notification> extends StatelessWidget {
  const NotificationListener({
    Key key,
    @required this.child,
    this.onNotification,
  }) : super(key: key);
 ...//省略无关代码 
}  
```

我们以看到：

1. NotificationListener 继承自 StatelessWidget 类，所以他可以直接嵌套到 Widget 树中

2. NotificationListener 可以指定一个泛型，必须继承自 Notification，当现实指定该泛型后，NotificationListener 便只会接受该泛型类型的通知，例如：

   ```dart
   //指定监听通知的类型为滚动结束通知(ScrollEndNotification)
   NotificationListener<ScrollEndNotification>(
     onNotification: (notification){
       //只会在滚动结束时才会触发此回调
       print(notification);
     },
     child: ListView.builder(
         itemCount: 100,
         itemBuilder: (context, index) {
           return ListTile(title: Text("$index"),);
         }
     ),
   );
   ```

   上面的代码运行后便只会在滚动结束时在控制台打印信息

3. onNotification 回调Wie通知处理回调，其函数签名如下：

   ```dart
   typedef NotificationListenerCallback<T extends Notification> = bool Function(T notification);
   ```

   他的返回值为 bool，当返回 true，阻止冒泡，其父 Wdiget 收不到通知，false 则继续向上冒泡

Flutter 的 UI 框架实现中，除了可滚动组件在滚动过程中会发出 ScrollNotification 之外，还有一些其他的通知，如 `SizeChangedLayoutNotification`，`KeepAliveNotification`， `LayoutChangedNotification` 等，Flutter 正是通过这种通知机制来使父元素可以在一些特定时机来做一些事

### 自定义通知

除了 Flutter 内部通知，我们可以使用自定义通知，下面我们看看如何实现：

```dart
class CustomNotification extends Notification {
  final String msg;

  CustomNotification(this.msg);
}
```

```dart
class NotificationTest extends StatefulWidget {
  @override
  _NotificationTestState createState() => _NotificationTestState();
}

class _NotificationTestState extends State<NotificationTest> {
  var _msg = "";

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text("通知测试")),
      body: NotificationListener<CustomNotification>(
        onNotification: (notification) {
          setState(() {
            _msg += notification.msg;
          });
          return true;
        },
        child: Center(
          child: Column(
            children: [
              // RaisedButton(
              //     onPressed: () =>
              //         CustomNotification("Hello world").dispatch(context)),
              Builder(builder: (context) {
                return RaisedButton(
                    onPressed: () =>
                        CustomNotification("Hello world").dispatch(context),
                    child: Text("发送通知"));
              }),
              Text(_msg)
            ],
          ),
        ),
      ),
    );
  }
}
```

效果如下：

![image-20210401164537619](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210401164537.png)

上面代码中，每点击一次按钮，都会分发一个 CustomNotification 类型的通知，我们在 Widget 根上监听通知，收到后打印到屏幕上。

>注意：代码注释的部分是不能正常工作的，因为这个 context 是根 Context，而 NotificationListener 是监听的子树，所以我们通过 Builder 来构建 RaisedButton，来获取按钮位置上的 context



#### 阻止冒泡

```dart
@override
Widget build(BuildContext context) {
  return Scaffold(
    appBar: AppBar(title: Text("通知测试")),
    body: NotificationListener<CustomNotification>(
      onNotification: (notification) {
        print(notification.msg);
        return false;
      },
      child: NotificationListener<CustomNotification>(
        onNotification: (notification) {
          setState(() {
            _msg += notification.msg;
          });
          return true;
        },
        child: Center(
          child: Column(
            children: [
              Builder(builder: (context) {
                return RaisedButton(
                    onPressed: () =>
                        CustomNotification("Hello world").dispatch(context),
                    child: Text("发送通知"));
              }),
              Text(_msg)
            ],
          ),
        ),
      ),
    ),
  );
}
```

修改代码如上，child NotificationListener 返回true 表示拦截，否则表示继续向上传递



#### 通知冒泡原理

为了搞清楚这个问题，就必须看一下源码，所以我们从 Notification 的 dispath 方法出发:

```dart
void dispatch(BuildContext? target) {
  //如果应该在其中分发通知的子树正在处理中，则“ target”可以为null。
  target?.visitAncestorElements(visitAncestor);
}
```

在 dispatch 方法中，调用了 context 的 visitAncestorElements 方法，该方法会从当前 Element向上遍历父级元素；

visitAncestorElements 放入有一个回调参数，在遍历的过程中都会执行该回调。遍历的终止条件是：已经遍历到根 Element 或者某个遍历的回调返回了 false。

我们看一下 visitAncestor 方法的实现

```dart
@protected
@mustCallSuper
bool visitAncestor(Element element) {
  if (element is StatelessElement) {
    final StatelessWidget widget = element.widget;
    if (widget is NotificationListener<Notification>) {
      if (widget._dispatch(this, element)) // that function checks the type dynamically
        return false;
    }
  }
  return true;
}
```

其实代码非常简单，就是判断是不是 NotificationListener ，如果不是，返回 true，否则调用 NotificationListener 的 _dispatch 方法，我们继续跟进一下

```dart
bool _dispatch(Notification notification, Element element) {
  if (onNotification != null && notification is T) {
    final bool result = onNotification!(notification);
    return result == true; // so that null and false have the same effect
  }
  return false;
}
```

判断 如果 onNoaification 不为 null，且 notification 必须是继承自 NotificationListener 的；

之后就会进行回调，拿到 onNotification 的返回值，这个返回值决定了是否继续向上遍历；

我们可以看到最终的回调是在 NotificationListener 中执行的，然后会根据返回值来判断是否继续向上遍历，其实上面的源码并不复杂，我们需要注意的有以下几点：

1. Context 上提供了遍历 Element 树的方法
2. 我们可以通过 Elment.widget 得到对应节点的 widget，我们应该通过这些源码来加深 widget 和 Element 的对应关系

### 总结

Flutter 的冒泡其实就是一套从下往上的遍历，通过阅读源码我们了解了通知冒泡的原理，其实是非常简单的，通过阅读源码可以加深我们对 Flutter 的了解和设计的思想，在此，建议在日常的开发过程中多看看源码，肯定能受益匪浅；



### 参考

> Flutter实战



