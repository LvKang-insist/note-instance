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

```dart
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
        leftChild.parentData as LeftRightParentData;
    //获取下一个组件
    //至于这里为什么可以获取到下一个组件，是因为在 多子组件的 mount 中，遍历创建所有的 child 然后将其插入到到 child 的 childParentData 中了
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

使用如下：

```dart
 Container(
  child: LeftRightBox(
    list: [
      Text("左"),
      Text("右"),
    ],
  ),
),
```

![image-20211209155013117](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20211209155013.png)

我们对上面流程进行一个简单分析：

1，获取当前组件的约束信息

2，获取两个子组件

3，对两个子组件进行layout，并且右组件的宽度不能超过总宽度的一半，设置又组件的偏移为最右边。接着对左组件进行布局，左子组件的宽度为总宽度-右子组件的宽度，并且没有设置偏移，默认偏移为0

4，设置当前组件自身的大小，高度为子组件的 max。

可以看到，实际布局流程和单子组件没太大区别，只不过多子组件需要对多个组件进行布局。

l另外和 `RenderCustomCenter` 不同的是，RenderLeftRight 是直接继承了 `RenderBox`，同时混入了 `ContainerRenderObjectMixin ` 和 `RenderBoxContainerDefaultsMixin ` 两个 mixin ，这两个 mixin 中帮我们实现了磨人的绘制和事件处理的相关逻辑。



### 布局更新

理论上，当某个组件的布局发生变化之后，会影响到其他的组件布局，所以当有组件布局发生改变之后，最笨的办法就是对整棵组件树进行重新布局。但是对所有的组件进行 `reLayout` 的成本还是比较大，所以我们需要探索一下降低 `reLayout` 成本的方案，事实上，在一些特定的场景下，组件发生变化之后只需要对特定的组件进行重新布局即可，无需对整棵树进行 `reLayout`  。

#### 布局边界

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20211214113606.png" alt="image-20211214113606334" style="zoom:33%;" />

假如有一个页面的组件树结构如上所示：

假如 Text3 的文本长度发生变化，就会导致 Text4 的位置发生变化，相应的 Column2 的高度也会发生变化。又因为 SizedBox 的宽高已经固定。所以最终需要 reLayout 的组件是：Text3，Colum2，这里需要注意的是：

1. Text4 是不需要进行重新布局的，因为 Text4 的大小没有发生变化，只是位置发生了变化，而它的位置是在父组件 Colum2 布局时确定的。
2. 很容易发现：假如 Text3 和 Column2 之间还有其他组件，则这些组件也都是需要 reLayout 的。

在本例中，Column2 就是 Text3 的 relayoutBoundary(重新布局的边界点)。每个组件的 renderObject 中都有一个 `_relayoutBoundary` 属性指向自身布局，如果当前节点布局发生变化后，自身到 `_relayoutBoundary` 路径上的所有节点都需要 reLayout。

那么一个组件的是否是 relayoutBoundary 的条件是什么呢？ 这里有一个原则和四个场景，原则是 "组件自身的大小变化不会影响父组件"，如果一个组件满足下面四种情况之一，则它便是 relayoutBoundary：

1. 当前组件的父组件大小不依赖当前组件大小时；这种情况下父组件在布局时会调用子组件布局函数时并会给子组件传递一个 parentUserSize 参数，该参数为 false 是表示父组件的布局算法不会依赖子组件的大小。
2. 组件大小只取决于父组件传递的约束，而不会依赖后代组件的大小。这样的话后代组件的大小变化就不会影响到自身的大小了，这种情况组件的 sizedByParent 属性必须为 true。
3. 父组件传递给自身的约束是一个严格约束(固定宽高)；这种情况下即使自身大小依赖后代元素，但也不会影响父组件。
4. 组件Wie根组件；Fluuter 应用的根组件是 RenderView ，他的默认大小是当前设备屏幕的大小。

对应实现的代码是：

```dart
if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
  _relayoutBoundary = this;
} else {
  _relayoutBoundary = (parent! as RenderObject)._relayoutBoundary;
}
```

代码中的 if 的判断条件和上面的四条一一对应，其中除了第二个条件之外(sizeByParent 为 true)，其他的都很直观。第二个条件在后面会讲到。

#### markNeedsLayout

当布局发生变化的时候，他需要调用 markNeedsLayout  方法来更新布局，它的主要功能有两个：

1，将自身到其 relayoutBoundary 路径上的所有节点标记为"需要布局"

2，其请求新的 frame；在新的 frame 中会对标记为 "需要布局" 的节点重新布局

```dart
void markNeedsLayout() {
  //如果当前组件不是布局边界节点  
  if (_relayoutBoundary != this) {
    //递归标记将当前节点到布局边界节点  
    markParentNeedsLayout();
  } else {
    //如果是布局边界节点  
    _needsLayout = true;
    if (owner != null) {
      //将布局边界节点加入到 piplineOwner._nodesNeedingLayout 列表中  
      owner!._nodesNeedingLayout.add(this);
      //改函数最终会请求新的 frame  
      owner!.requestVisualUpdate();
    }
  }
}
```

#### flushLayout

markNeedsLayout 执行完成后，就会将其 relayoutBoundary 添加到 piplineOwner._nodesNeedingLayout 列表中，然后请求新的 frame。

当新的 frame 到来时，就会执行 piplineOwner.drawFrame 方法：

```dart
void drawFrame() {
  assert(renderView != null);
  pipelineOwner.flushLayout();
  pipelineOwner.flushCompositingBits();
  pipelineOwner.flushPaint();
  ...///
}
```

flushLayout 中会对之前添加到 _nodesNeedingLayout 中的节点进行重新布局，如下：

```dart
void flushLayout() {
    while (_nodesNeedingLayout.isNotEmpty) {
      final List<RenderObject> dirtyNodes = _nodesNeedingLayout;
      _nodesNeedingLayout = <RenderObject>[];
      //安装节点在树中的深度从小到大排序后在重新 layout  
      for (final RenderObject node in dirtyNodes..sort((RenderObject a, RenderObject b) => a.depth - b.depth)) {
        if (node._needsLayout && node.owner == this)
          //重新布局  
          node._layoutWithoutResize();
      }
    }
}
```

看一下  _layoutwithoutResize() 的实现

```dart
void _layoutWithoutResize() {
  try {
    //递归重新布局  
    performLayout();
    markNeedsSemanticsUpdate();
  } catch (e, stack) {
    _debugReportException('performLayout', e, stack);
  }
  _needsLayout = false;
  //布局更新后，更新UI  
  markNeedsPaint();
}
```

到此布局更新完成。
