### Flutter 布局过程

Layout(布局)过程中是确定每一个组件的信息(大小和位置)，Flutter 中的布局过程如下：

1，父节点向子节点传递约束信息，限制子节点的最大和最小宽高。

2，子节点根据自己的约束信息来确定自己的大小（Szie）。

3，父节点根据特定的规则（不同的组件会有不同的布局算法）确定每一个子节点在父节点空间中的位置，用偏移 offset表示。

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

#### Layout流程

如果组件有子组件，则需要在 performLayout 中调用子组件的 layout 先对子组件进行布局，如下：

```dart
void layout(Constraints constraints, { bool parentUsesSize = false }) {
  RenderObject? relayoutBoundary;
  
  //先确定当前布局边界  
  if (!parentUsesSize || sizedByParent || constraints.isTight || parent is! RenderObject) {
    relayoutBoundary = this;
  } else {
    relayoutBoundary = (parent! as RenderObject)._relayoutBoundary;
  }
  // _neessLayout  标记当前组件是否被标记为需要布局
  // _constraints 是上次布局时父组件传递给当前组件的约束
  // _relayoutBoundary 为上次布局时当前组件的布局边界 
  // 所以，当当前组件没有被标记为需要布局，且父组件传递的约束没有发生变化
  // 和布局边界也没有发生变化时则不需要重新布局，直接返回即可  
  if (!_needsLayout && constraints == _constraints && relayoutBoundary == _relayoutBoundary) {
	....///
    return;
  }
  // 如果需要布局，缓存约束和布局边界  
  _constraints = constraints;
  _relayoutBoundary = relayoutBoundary;
  assert(!_debugMutationsLocked);
  assert(!_doingThisLayoutWithCallback);
  assert(() {
    _debugMutationsLocked = true;
    if (debugPrintLayouts)
      debugPrint('Laying out (${sizedByParent ? "with separate resize" : "with resize allowed"}) $this');
    return true;
  }());
  
  // 后面解释
  if (sizedByParent) {
     performResize();
  }

  // 执行布局
  performLayout();

   //布局结束后将 _needsLayotu 置位 false
  _needsLayout = false;
  
  // 将当前组件标记为重绘，因为布局发生变化后，需要重新绘制
  markNeedsPaint();
}
```

简单的讲一下布局的过程：

1. 确定当前组件的布局边界
2. 判断是否需要重新布局，如果没有必要会直接返回，反之才需要重新布局。不需要布局时需要满足三个条件
   - 单签组件没有被标记为需要重新布局。
   - 父组件传递的约束没有发生变化。
   - 当前组件布局边界也没有发生变化时。
3. 调用 performLayout 进行布局，因为 performLayout 中又会调用子组件的 layout 方法，所以这是一个递归的过程，递归结束后整个组件的布局也就完成了。
4. 请求重绘

#### sizedByParent

在 layout 方法中，有以下逻辑：

```dart
  if (sizedByParent) {
     performResize();
  }
```

上面我们说过，sizeByParent 为 true 是表示：当前组件的大小值取决于父组件传递的约束，而不会依赖后组件的大小。前面我们说过，performLayout 中确定当前组件大小时通常会依赖子组件的大小，如果 `sizedByParent` 为 true，则当前组件大小就不会依赖于子组件的大小。

为了清晰逻辑，Flutter 框架中约定，当 sizedByParent 为 true 时，确定当前组件大小的逻辑应该抽离到 performResize() 中，这种情况下 performLayout 主要任务便只有两个：对子组件进行布局和确定子组件在当前组件中的偏移。

下面通过一个 AccurateSizedBox 示例来演示一下 sizebyParent 为 true 时我们应该如何布局：

##### AccurateSizeBox

Flutter 中的 SizeBox 会将其父组件的约束传递给其子组件，这也就意味着，如果父组件限制了最新的宽度为 100，即使我们通过 SizeBox 指定宽度为 50 也是没有用的。

因为 SizeBox 中的实现会让 SizedBox 的子组件先满足 SizeBox 父组件的约束。例如：

```dart
 AppBar(
    title: Text(title),
    actions: <Widget>[
      SizedBox( // 使用SizedBox定制loading 宽高
        width: 20, 
        height: 20,
        child: CircularProgressIndicator(
          strokeWidth: 3,
          valueColor: AlwaysStoppedAnimation(Colors.white70),
        ),
      )
    ],
 )
```

实际结果还是 progress 的高度为 appbar 的高度。

通过查看 SizedBox 源码，如下所示：

```dart
@override
void performLayout() {
  final BoxConstraints constraints = this.constraints;
  if (child != null) {
    child!.layout(_additionalConstraints.enforce(constraints), parentUsesSize: true);
    size = child!.size;
  } else {
    size = _additionalConstraints.enforce(constraints).constrain(Size.zero);
  }
}
//返回尊重给定约束同时尽可能接近原始约束的新框约束
BoxConstraints enforce(BoxConstraints constraints) {
    return BoxConstraints(
        // clamp ：根据数字返回一个介于低和高之间的值
        minWidth: minWidth.clamp(constraints.minWidth, constraints.maxWidth),
        maxWidth: maxWidth.clamp(constraints.minWidth, constraints.maxWidth),
        minHeight: minHeight.clamp(constraints.minHeight, constraints.maxHeight),
        maxHeight: maxHeight.clamp(constraints.minHeight, constraints.maxHeight),
    );
}
```

可以发现，之所以不生效，是应为父组件限制了最小高度，SizeBox 中的子组件会先满足父组件的约束。当然，我们也可以通过使用 UnconstrainedBox + SizedBox 来实现我们想要的效果，但是这里我们希望使用一个布局搞定，为此我们自定义一个 AccurateSizeBox 组件。

它和 SizedBox 主要的区别就是 AccurateSizedBox 自身会遵守其父组件传递的约束，而不是让子组件去满足 AccureateSizeBox 父组件的约束，具体：

1. AccurateSizedBox 自身大小只取决于父组件的约束和自身的宽高。
2. AccurateSizedBox 确定自身大小后，限制其子组件的大小。

```dart
class AccurateSizedBox extends SingleChildRenderObjectWidget {
  const AccurateSizedBox(
      {Key key, this.width = 0, this.height = 0, @required Widget child})
      : super(key: key, child: child);

  final double width;
  final double height;

  @override
  RenderObject createRenderObject(BuildContext context) {
    return RenderAccurateSizeBox(width, height);
  }

  @override
  void updateRenderObject(
      BuildContext context, covariant RenderAccurateSizeBox renderObject) {
    renderObject
      ..width = width
      ..height = height;
  }
}

class RenderAccurateSizeBox extends RenderProxyBoxWithHitTestBehavior {
  RenderAccurateSizeBox(this.width, this.height);

  double width;
  double height;

  //当前组件的大小只取决于父组件传递的约束
  @override
  bool get sizedByParent => true;
    
  // performResize 中会调用
  @override
  Size computeDryLayout(BoxConstraints constraints) {
    //设置当前元素的宽高，遵守父组件的约束
    return constraints.constrain(Size(width, height));
  }  

  @override
  void performLayout() {
    child.layout(
        BoxConstraints.tight(
            Size(min(size.width, width), min(size.height, height))),
        //父容器是固定大小，子元素大小改变时不影响父元素
        //parentUserSize 为 false时，子组件的布局边界会是他自身，子组件布局发生变化后不会影响当前组件
        parentUsesSize: false);
  }
}
```

上面代码有三点需要注意：

1. 我们的 RenderAccurateSizedBox 不在继承自 RenderBox，而是继承 `RenderProxyBoxWithHitTestBehavior` ，`RenderProxyBoxWithHitTestBehavior` 是间接继承自 `RenderBox` 的，它里面包含了默认的命中测试和绘制相关逻辑，继承它以后则不需要我们手动实现了。

2. 我们将确定当前组件大小的逻辑挪到了 `computeDryLayout` 方法中，因为 RenderBox 的 performResize 方法会调用 computeDryLayout，并将返回结果作为当前组件大小。

   按照 Flutter 框架约定，我们应该重写 computeDryLayout 方法，而不是 performResize 方法。就行我们在布局时应该重写 performLayout 方法而不是 layout 方法；不过，这只是一个约定，并非强制，但我们应该尽可能遵守这个约定，除非你清楚的知道自己在干什么并且能确保之后维护你代码的人也清楚。

3. RenderAccurateSizedBox 在调用子组件 layout 时，将 `parentUserSize` 置为 false，这样的话子组件就会变成一个布局边界。

测试如下：

```dart
class AccurateSizedBoxRoute extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final child = GestureDetector(
      onTap: () => print("tap"),
      child: Container(width: 300, height: 30, color: Colors.red),
    );
    return Row(
      children: [
        ConstrainedBox(
          //限制高度为 100x100
          constraints: BoxConstraints.tight(Size(100, 100)),
          child: SizedBox(
            width: 50,
            height: 50,
            child: child,
          ),
        ),
        Padding(
          padding: const EdgeInsets.only(left: 8),
          child: ConstrainedBox(
            constraints: BoxConstraints.tight(Size(100, 100)),
            child: AccurateSizedBox(width: 50, height: 50, child: child),
          ),
        )
      ],
    );
  }
}
```

![image-20211215183103420](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20211215183103.png)

结果如上所示，当父组件宽高是 100 时，我们通过 SizedBox 指定 Container 大小是 50x50 是不能成功的。而通过 AccurateSizedBox 时成功了。

> 需要注意的是，如果一个组件的 sizeByParent 为 true，那它在布局子组件的时候也是能将 `parentUserSize` 的，sizeByParent 为 true 表示自己是布局边界。
>
> 而将 `parentUsesSize` 置为 true 或者 false 决定的是子组件是否是布局边界，两者并不相矛盾，这一点不能混淆。
>
> 另外，在 Flutter 自带的 OverflowBox 组件中，他的 sizeByParent 为 true，在调用子组件 layout 时，parentUsesSize 也是 true，详情可查看 OverflowBox 的源码



#### Constraints

Constraints(约束)主要描述了最小和最大宽高的限制，理解组件在布局过程中如何根据约束确定自身或子节点的大小对我们理解组件的布局行为有很大的帮助。

我们通过一个 200*200 的 Container 的例子来说明，为了排除干扰，我们让根节点(RenderView) 作为 Container 的父组件，代码如下：

```dart
Container(width: 200, height: 200, color: Colors.red)
```

运行之后，就会发现整个屏幕都为红色，为什么呢，我们看看 RenderView 的实现：

```dart
@override
void performLayout() {
  //configurateion.sieze 为当前设备的屏幕
  _size = configuration.size;
  assert(_size.isFinite);
  if (child != null)
    child!.layout(BoxConstraints.tight(_size));//强制子组件和屏幕一样大
}
```

这里需要介绍一下两种常用的约束：

1. 宽松约束：不限制最小宽高(为 0)，只限制最大宽高，可以通过 `BoxConstraints.loose(Size size)` 来快速创建。
2. 严格约束：限制为固定大小，即最小宽度等于最大宽度，最小高度等于最大高度，可以通过 `BoxConstraints.thght(Size)` 来快速创建。

可以发现，RenderView 中给子组件传递的是一个严格的约束，即强制子组件等于屏幕大小，所以 Container 便撑满了屏幕。

那么我们如何才能让指定的大小生效呢，答案就是 “引入一个中间组件，让中间组件遵守父组件的约束，然后对子组件传递新的约束”。对于这个例子来说，最简单的办法就是使用一个 `Align` 组件来包裹 `Container`:

```dart
@override
Widget build(BuildContext context) {
  var container = Container(width: 200, height: 200, color: Colors.red);
  return Align(
    child: container,
    alignment: Alignment.topLeft,
  );
}
```

Align 会遵守 RenderView 的约束，让自身撑满屏幕，然后会给子组件一个宽松的约束(最小宽度为 0，最大宽度为 200)，这样 Container 就可以变成 200*200 了。

当然我们也可以使用其他组件来代替 Align，例如 `UnconstrainedBox`，但原理是相同的。具体可查看源码进行验证。

例如 Align 的布局过程如下：

```dart
void performLayout() {
  final BoxConstraints constraints = this.constraints;
  final bool shrinkWrapWidth = _widthFactor != null || constraints.maxWidth == double.infinity;
  final bool shrinkWrapHeight = _heightFactor != null || constraints.maxHeight == double.infinity;

  if (child != null) {
    //子组件采用宽松约束，并且设置子组件不是布局边界(表示子组件改变后当前组件也需要重新刷新)
    child!.layout(constraints.loosen(), parentUsesSize: true);
    size = constraints.constrain(Size(
      shrinkWrapWidth ? child!.size.width * (_widthFactor ?? 1.0) : double.infinity,
      shrinkWrapHeight ? child!.size.height * (_heightFactor ?? 1.0) : double.infinity,
    ));
    alignChild();
  } else {
    size = constraints.constrain(Size(
      shrinkWrapWidth ? 0.0 : double.infinity,
      shrinkWrapHeight ? 0.0 : double.infinity,
    ));
  }
}
```

### 总结

到这里我们已经对 flutter 布局流程比较熟悉了，现在我们看一张官网的图：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20220105112857.png" alt="image-20220105112857377" style="zoom:50%;" />



> 在进行布局的时候，Flutter 会以 DFS(深度优先遍历) 的方式遍历渲染树，并限制自上而下的方式从父节点传递给子节点。子节点如果需要确定自身的大小，则必须遵守父节点传递的限制。子节点的响应方式是在父节点建立的约束内将大小以自上而下的方式传递给父节点。

是不是理解的更透彻了一些
