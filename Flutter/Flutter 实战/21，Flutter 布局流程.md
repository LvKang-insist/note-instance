### Flutter 布局过程

Layout(布局)过程朱啊哟是确定每一个组件的信息(大小和位置)，Flutter 中的布局过程如下：

1，父节点向子节点传递约束信息，限制子节点的最大和最小宽高。

2，子节点根据自己的约束信息来确定自己的大小（Szie）。

3，父节点根据特定的规则（不同的组件会有不同的布局算法）确定每一个子节点在父节点空间中的位置，用便宜 offset 中，

4，递归整个过程，确定出每一个节点的位置和大小。

可以看到，组件的大小是由自身来决定的，而组件的位置是由父组件来决定的。

Flutter 中布局类组件有很多，根据孩子数量可以分为单子组件和多子组件，下面我们分别定义一个单子组件和多子组件来深入理解一下 Fluuter 布局过程。

___



### 单子组件布局示例

我们自定义一个单子组件 CustomCenter。公告基本和 Center 一样，通过这个示例我们演示一下布局的主要流程。

为了展示原理，我们不采用组合的方式来实现组件，而是通过定制的 RenderObject 的方式来实现。因为居中组件需要包含一个子节点，所以我们继承 SingleChildRenderObjectWidget。

```dart
class CustomCenter extends SingleChildRenderObjectWidget {
  const CustomCenter({Key key, @required Widget child})
      : super(key: key, child: child);

  @override
  RenderObject createRenderObject(BuildContext context) {
    return RenderCustomCenter();
  }
}
```

接着实现 RenderCustomCenter：

```dart
class RenderCustomCenter extends RenderShiftedBox {
  RenderCustomCenter({RenderBox child}) : super(child);

  @override
  void performLayout() {
    //1,先对子组件进行 layout，随后获取他的 size
    child.layout(constraints.loosen(), //将约束传递给子节点
        parentUsesSize: true //因为接下来要使用 child 的 size，所以不能为 false
        );
    //2，根据子组件的大小确定自身的大小
    size = constraints.constrain(Size(
        constraints.maxWidth == double.infinity
            ? child.size.width
            : double.infinity,
        constraints.maxHeight == double.infinity
            ? child.size.height
            : double.infinity));

    //3，根据父节点大小，算出子节点在父节点中居中后的偏移，
    //然后将这个偏移保存在子节点的 parentData 中，在后续的绘制节点会用到
    BoxParentData parentData = child.parentData as BoxParentData;
    parentData.offset = ((size - child.size) as Offset) / 2;
  }
}
```

上面代码本来继承 `RenderObject` 会更底层一点，但是这需要我们手动实现一些和布局无关的东西，比如事件分发等逻辑。为了更聚焦布局本身，我们选择继承 `RenderShiftedBox`，他会帮我们实现布局之外的功能，这样我们只需要重写 performLayout。在改函数中实现居中算法即可。

布局过程如上注释所示，在此之外还有三点需要说明：

1. 在对子组件进行 Layout 的时候，`constraints` 是 CustomCenter 的父组件传递给自己的约束信息，我们传递给字节的的约束信息是 `constraints.loosen()`，下面看一下 lossen 的实现：

   ```dart
   BoxConstraints loosen() {
     assert(debugAssertIsValid());
     return BoxConstraints(
       minWidth: 0.0,
       maxWidth: maxWidth,
       minHeight: 0.0,
       maxHeight: maxHeight,
     );
   }
   ```

   很明显，CustomCenter 约束字节的最大宽高不能超过自身的最大宽高

2. 子节点在父节点(CustomCenter) 的约束下，确定自己的宽高。此时 CustomCenter 会根据子节点的宽高来确定自己的宽高。

   上面的代码逻辑是，如果父节点的约束是无限大，他的宽高就是字节的宽高，否则自己宽高为无限大。

   > 需要注意的是，如果这个时候将 CustomCenter 的宽高也设置为无限大就会有问题，因为在一个无限大的范围内自己的宽高也是无限大的话，那么自己的父节点会懵逼的。屏幕的大小是固定的，这显然很不合理。
   >
   > 如果CustomCenter 父节点传递的宽高不是无限大，那么这个时候是可以设置自己的宽高为无限大，因为在一个有限的空间内，子节点设置无限大也就是父节点的大小。
   >
   > 简而言之，CustomCenter 会尽可能让自己填满父元素的空间

3. CustomCenter 确定了自己的大小和子节点的大小之后就可以确定子节点的位置了。根据居中算法，将子节点的原点坐标计算出来后保存在子节点的  parentData 中，在后续的绘制阶段会用到，具体如何使用我们看一下 RenderShiftedBox 中的默认实现：

   ```dart
     @override
     void paint(PaintingContext context, Offset offset) {
       if (child != null) {
         final BoxParentData parentData = child.parentData as BoxParentData;
         //从 child。parentData 中取出子节点相对当前节点的偏移，加上当前节点在屏幕中的偏移
         //便是子节点在屏幕中的偏移
         context.paintChild(child, parentData.offset + offset);
       }
     }
   ```

#### PerformLayout

通过上面可以看到，布局的逻辑是在 `performLayout` 方法中实现的，我们总结一下 `performLayout` 中具体做的事：

1. 如果有子组件，则对子组件进行递归排序
2. 确定当前组件大小(size)，通知会依赖于子组件的大小
3. 确定子组件在当前组件中的起始偏移

在Flutter 组件库中，有很多常用的单子组件，如 Align，SizeBox，DecoratedBox 等，都可以打开源码去看一下具体实现。

___



### 多子组件布局示例

在实际开发中，我们经常会用到左右贴边的布局，现在我们就来实现一个 LeftRightBox 组件，来实现左右贴边。

首先我们定义组件，与单子组件不同的是，多子组件需要继承自 `MultiChildRenderObjectWidget` ：

```
class LeftRightBox extends MultiChildRenderObjectWidget {
  LeftRightBox({Key key, @required List<Widget> list})
      : assert(list.length == 2, "只能传两个 child"),
        super(key: key, children: list);

  @override
  RenderObject createRenderObject(BuildContext context) {
    return RenderLeftRight();
  }
}
```

接下来在 RenderLeftRight 中的 performLayout 实现左右布局的算法：

```dart
class LeftRightParentData extends ContainerBoxParentData<RenderBox> {}

class RenderLeftRight extends RenderBox
    with
        ContainerRenderObjectMixin<RenderBox, LeftRightParentData>,
        RenderBoxContainerDefaultsMixin<RenderBox, LeftRightParentData> {
  /// 初始化每一个 child 的 parentData
  @override
  void setupParentData(covariant RenderObject child) {
    if (child.parentData is! LeftRightParentData)
      child.parentData = LeftRightParentData();
  }

  @override
  void performLayout() {
    //获取当前约束(从父组件传入的)，
    final BoxConstraints constraints = this.constraints;

    //获取第一个组件，和他父组件传的约束
    RenderBox leftChild = firstChild;
    LeftRightParentData childParentData =
        leftChild.parent as LeftRightParentData;
    //获取下一个组件
    //至于这里为什么可以获取到下一个组件，是因为在 多子组件的 mount 中，遍历创建所有的 child 然后将其插入到到 child 的 childParentData 中
    //这里需要对 renderObject 的构建流程有一个了解，https://juejin.cn/post/7031858186567188511#heading-11  
    RenderBox rightChild = childParentData.nextSibling;

    //限制右孩子宽度不超过总宽度的一半
    rightChild.layout(constraints.copyWith(maxWidth: constraints.maxWidth / 2),
        parentUsesSize: true);

    //设置右子节点的 offset
    childParentData = rightChild.parentData as LeftRightParentData;
    //位于最右边
    childParentData.offset =
        Offset(constraints.maxWidth - rightChild.size.width, 0);

    //左子节点的 offset 默认为 (0,0),为了确保左子节点能始终显示，我们不修改他的 offset
    leftChild.layout(
        constraints.copyWith(
            //左侧剩余的最大宽度
            maxWidth: constraints.maxWidth - rightChild.size.width),
        parentUsesSize: true);

    //设置 leftRight 自身的 size
    size = Size(constraints.maxWidth,
        max(leftChild.size.height, rightChild.size.height));
  }

  double max(double height, double height2) {
    if (height > height2)
      return height;
    else
      return height2;
  }

  @override
  void paint(PaintingContext context, Offset offset) {
    defaultPaint(context, offset);
  }

  @override
  bool hitTestChildren(BoxHitTestResult result, {Offset position}) {
    return defaultHitTestChildren(result, position: position);
  }
}
```
