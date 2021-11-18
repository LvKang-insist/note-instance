### 引言

在 `Flutter` 中，很多人都知道三棵树，最熟悉就是其中的 `Widget` 树了，这也是平常开发的过程中最多用到的东西，那么其他两棵树你知道是什么吗，了解他们的构建流程吗？

 

### Widget 树

在开发过程中，与我们息息相关的就是 `widget` 了，几乎所有页面上显示的都是 `widget` ，Widget  是 Flutter 的核心，**是用户界面的不可变描述。**

事实上，widget 的功能就是`描述一个 UI 元素的配置数据` ，也就是说 widget 并不是最终绘制到屏幕上的元素，它只是描述显示元素的一个配置而已。

在代码的运行过程中并没有明确的 `widget` 树的概念，这棵树是我我们在开发的过程中对 widget 嵌套的描述，因为确实长得像是一棵树

```dart
abstract class Widget {
  const Widget({ this.key });
  final Key key;
  
  @protected
  Element createElement();//注释1
  
  static bool canUpdate(Widget oldWidget, Widget newWidget) {//注释2
   return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
}

```

Widget 本身是一个抽象类，接收一个 key，[至于key的原理和使用可查看这篇文章](https://juejin.cn/post/6976069994270588941)，

- 注释1

  createElement 是抽象方法，子类必须实现，该方法创建了一个 Element，所以每个 Element 都会对应一个 widget 对象。

- 注释2

  判断 oldWidget 和 newWidget 是不是同一个 widget，如何 runtimeType 和 key 相同则认为是同一个 widget。

>需要注意的是 widget 不能被修改，如果要修改只能重新创建了，因为 wdiget 并不参与渲染，它只是一个配置文件而已，只需要告诉渲染层自己的样式即可。



### Element 树

Flutter 中真正显示到屏幕上的元素是 `Element` 类，也就是说 widget 只是描述 `Element` 的配置数据，并且 `widget` 可以对应多个 `Element`。这是因为同一个 `widget` 可以被添加到  `Element` 树的不同部分。而真正渲染的时候，每一个 `Element` 都会对应着一个 `widget` 对象。

所谓的 UI 树就是由一个个 `Element` 节点构成。组件的最终 Layout，渲染都是通过  `RenderObject` 来完成的，从创建到渲染的大体流程就是：**根据 `widget` 生成 `Element`，然后在创建相应的 `RenderObject` 并关联到 `Element.renderObject` 属性上，最后在通过 `RenderObject` 来完成布局排列和绘制。**

Element 表示一个 widget 树中特定位置的实例，大多数的 `Element` 只有惟一的 `RenderObject`，但是还有一些 Element 会有多个子节点，如继承自 `RenderObjectElement` 的一些类，比如 `MultiChildRenderObjectObject`。最终所有的 Element 的 RenderObject 构成一棵树，我们称之为 渲染树。

总结一下，我们可以认为 Flutter 的 UI 系统中包含了三棵树：Widget 树，Element 树，渲染树，他们的对应关系式  Element 树是根据 Widget 树生成，而渲染树有依赖于 Element 树，如图所示：

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20211115184004.png" alt="image-20211115184004464" style="zoom:50%;" />

#### Element 类源代码

```dart
abstract class Element extends DiagnosticableTree implements BuildContext {
  
  Element(Widget widget)
    : assert(widget != null),
      _widget = widget;

  Element _parent;

  @override
  Widget get widget => _widget;
  Widget _widget;

  RenderObject get renderObject { ... }

  @mustCallSuper
  void mount(Element parent, dynamic newSlot) { ... }

  @mustCallSuper
  void activate() { ... }

  @mustCallSuper
  void deactivate() { ... }

  @mustCallSuper
  void unmount() { ... }
}
```

#### Element 的生命周期

- initial

  初始状态

  ```dart
  _ElementLifecycle _lifecycleState = _ElementLifecycle.initial;
  ```

- active

  ```dart
  //RenderObjectElement 的 mount 方法
  @override
  void mount(Element? parent, Object? newSlot) {
    super.mount(parent, newSlot);
    //.....
    _renderObject = widget.createRenderObject(this);
    assert(_slot == newSlot);
    attachRenderObject(newSlot);
    _dirty = false;
  }
  ```

  当 fragment 调用 `element.mount` 方法后，mount 方法中会首先调用 element 对应 widget 的 `createRenderObject `方法来创建与 element 对应的 RenderObject 对象。

  然后调用 element.attachRenderObject 将  element.renderObject 添加到渲染树插槽的位置(这一步不是必须的，一般发生在 Element 树结构发生变化时才需要重新 attach。

  插入到渲染后的 element 就处于 `active` 状态，处于 `active`状态后就可以显示在屏幕上了(可以隐藏)。

  ```dart
  super.mount(parent,newslot)
  _lifecycleState = _ElementLifecycle.active
  ```

  当 widget 更新时，为了避免重新创建 element 会判断是否可以更新，会调用 updateChild 方法

  ```dart
  @protected
  @pragma('vm:prefer-inline')
  Element? updateChild(Element? child, Widget? newWidget, Object? newSlot) {
    //当没有新 widget，并且有原来的widget，则移出原来的child,因为他不再有配置
    if (newWidget == null) {
      if (child != null)
        deactivateChild(child);
      return null;
    }
    final Element newChild;
    //原来有 child
    if (child != null) {
      bool hasSameSuperclass = true;
  	assert(() {
          final int oldElementClass = Element._debugConcreteSubtype(child);
          final int newWidgetClass = Widget._debugConcreteSubtype(newWidget);
          hasSameSuperclass = oldElementClass == newWidgetClass;
          return true;
        }());
        // 如果父控件类型相同，子控件也相同，直接更新
      if (hasSameSuperclass && child.widget == newWidget) {
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        newChild = child;
      } else if (hasSameSuperclass && Widget.canUpdate(child.widget, newWidget)) 
          //父控件类型相同，并且可以更新 widget，则更新child
        if (child.slot != newSlot)
          updateSlotForChild(child, newSlot);
        child.update(newWidget);
        assert(child.widget == newWidget);
        assert(() {
          child.owner!._debugElementWasRebuilt(child);
          return true;
        }());
        newChild = child;
      } else {
        //不能更新，需要先移出原有的 child，并创建新的 child 并添加
        deactivateChild(child);
        assert(child._parent == null);
        newChild = inflateWidget(newWidget, newSlot);
      }
    } else {
      //没有 child，直接创建新的 child 并添加
      newChild = inflateWidget(newWidget, newSlot);
    }
    return newChild;
  }
  ```

  weidget.canUpdate ，主要判断 type 和 key 是否相同。如果我们需要强制更新，只需要修改key即可，官方不推荐修改 runtimetype

  ```dart
  static bool canUpdate(Widget oldWidget, Widget newWidget) {
    return oldWidget.runtimeType == newWidget.runtimeType
        && oldWidget.key == newWidget.key;
  }
  ```

  **从“非活动”到“活动”生命周期状态的转换**

  在上面的 `updateChild` 方法中，最后如果调用了 inflateWidget() 方法后，就需要将状态从 inactive 转到 active 状态

  ```dart
  @mustCallSuper
  void activate() {
    assert(_lifecycleState == _ElementLifecycle.inactive);
    assert(widget != null);
    assert(owner != null);
    assert(depth != null);
    final bool hadDependencies = (_dependencies != null && _dependencies!.isNotEmpty) || _hadUnsatisfiedDependencies;
    _lifecycleState = _ElementLifecycle.active;
    // We unregistered our dependencies in deactivate, but never cleared the list.
    // Since we're going to be reused, let's clear our list now.
    _dependencies?.clear();
    _hadUnsatisfiedDependencies = false;
    _updateInheritance();
    if (_dirty)
      owner!.scheduleBuildFor(this);
    if (hadDependencies)
      didChangeDependencies();
  }
  ```

- inactive

  从“活动”到“非活动”生命周期状态的转换

  在上面的 `updateChild` 方法中我们可以看到新的 widget 为空并且存在旧的，就会调用deactiveChild 移除 child，然后调用 `deactivate` 方法将 `_lifecycleState` 设置为 `inactive`

  ```dart
  @mustCallSuper
  void deactivate() {
    assert(_lifecycleState == _ElementLifecycle.active);
    assert(_widget != null); // Use the private property to avoid a CastError during hot reload.
    assert(depth != null);
    if (_dependencies != null && _dependencies!.isNotEmpty) {
      for (final InheritedElement dependency in _dependencies!)
        dependency._dependents.remove(this);
    }
    _inheritedWidgets = null;
    _lifecycleState = _ElementLifecycle.inactive;
  }
  ```

- defunct

  从“非活动”到“已失效”生命周期状态的转换

  ```dart
  @mustCallSuper
  void unmount() {
    assert(_lifecycleState == _ElementLifecycle.inactive);
    assert(_widget != null); // Use the private property to avoid a CastError during hot reload.
    assert(depth != null);
    assert(owner != null);
    // Use the private property to avoid a CastError during hot reload.
    final Key? key = _widget!.key;
    if (key is GlobalKey) {
      owner!._unregisterGlobalKey(key, this);
    }
    // Release resources to reduce the severity of memory leaks caused by
    // defunct, but accidentally retained Elements.
    _widget = null;
    _dependencies = null;
    _lifecycleState = _ElementLifecycle.defunct;
  }
  ```

#### StatelessElement

`Container` 创建出来的是 `StatelessElement`，下面我们浅析一下他的调用过程，部分代码省略，只显示核心代码!

```dart
class StatelessElement extends ComponentElement {
  //通过 createElement 创建时传入的 widget
  StatelessElement(StatelessWidget widget) : super(widget);

  @override
  StatelessWidget get widget => super.widget as StatelessWidget;

  //这里调用的 build 就是我们自己实现的 build 方法
  @override
  Widget build() => widget.build(this);
}


abstract class ComponentElement extends Element {
  /// Creates an element that uses the given widget as its configuration.
  ComponentElement(Widget widget) : super(widget);

  Element? _child;

  bool _debugDoingBuild = false;
  @override
  bool get debugDoingBuild => _debugDoingBuild;

  @override
  void mount(Element? parent, Object? newSlot) {
    _firstBuild();
  }

  void _firstBuild() {
    rebuild();
  }


  @override
  @pragma('vm:notify-debugger-on-exception')
  void performRebuild() {
   //.....
  }

  @protected
  Widget build();
}

abstract class Element extends DiagnosticableTree implements BuildContext {
  // 构造方法, 接收一个widget参数
  Element(Widget widget)
    : assert(widget != null),
      _widget = widget;

  @override
  Widget get widget => _widget;
  Widget _widget;
  
  void rebuild() {
    if (!_active || !_dirty)
      return;
      
    Element debugPreviousBuildTarget;
    
    // 这里调用的performRebuild方法, 在当前类并没有实现, 只能去自己的类里面查找实现
    performRebuild();
  }

  /// Called by rebuild() after the appropriate checks have been made.
  @protected
  void performRebuild();
}
```

梳理一下流程，如下：

1. 这里创建了 `StatelessElement` ，创建成功后，`framework` 就会调用 `mount`方法，因为 `StatelessElement` 没有实现 `mount`，所以这里调用的是 `ComponentElement` 的 `mount`。

2. 在 `mount` 中调用了 `_firstBuild` 方法进行第一次构建。(这里调用的是实现类(StatelessElement) 的 `_firstBuild` 方法)

3. `_firstBuild` 方法最后调用的是 `super._firstBuild()` ，也就是`ComponentElement` 的 `_firstBuild` 方法，在其中调用了 `rebuild()` 。由于 `ComponentElement` 没有重写，所以最终调用的是 `Element` 的 `rebuild` 方法。

4. `rebuild` 最终又会调用到 `ComponentElement` 的 `performRebuild` 方法中。 如下：

   ```dart
    @override
     @pragma('vm:notify-debugger-on-exception')
     void performRebuild() {
       if (!kReleaseMode && debugProfileBuildsEnabled)
         Timeline.startSync('${widget.runtimeType}',  arguments: timelineArgumentsIndicatingLandmarkEvent);
       assert(_debugSetAllowIgnoredCallsToMarkNeedsBuild(true));
       Widget? built;
       try {
         assert(() {
           _debugDoingBuild = true;
           return true;
         }());
         // 调用 build ，这里调用的是实现类 StatelessElement 的，最终是调用到我们自己实现的 build 中
         built = build();
         debugWidgetBuilderValue(widget, built);
       } catch (e, stack) {
         // catch
       } finally {
         _dirty = false;
         assert(_debugSetAllowIgnoredCallsToMarkNeedsBuild(false));
       }
       try {
         //最终调用 updateChild 方法
         _child = updateChild(_child, built, slot);
         assert(_child != null);
       } catch (e, stack) {
         //....
         _child = updateChild(null, built, slot);
       }
       if (!kReleaseMode && debugProfileBuildsEnabled)
         Timeline.finishSync();
     }
   ```

   ```dart
   @pragma('vm:prefer-inline')
   Element inflateWidget(Widget newWidget, Object? newSlot) {
     assert(newWidget != null);
     final Key? key = newWidget.key;
     if (key is GlobalKey) {
       final Element? newChild = _retakeInactiveElement(key, newWidget);
       if (newChild != null) {
         assert(newChild._parent == null);
         assert(() {
           _debugCheckForCycles(newChild);
           return true;
         }());
         newChild._activateWithParent(this, newSlot);
         final Element? updatedChild = updateChild(newChild, newWidget, newSlot);
         assert(newChild == updatedChild);
         return updatedChild!;
       }
     }
     //创建对应的 element  
     final Element newChild = newWidget.createElement();
     assert(() {
       _debugCheckForCycles(newChild);
       return true;
     }());
     //  调用 mount 方法
     newChild.mount(this, newSlot);
     assert(newChild._lifecycleState == _ElementLifecycle.active);
     return newChild;
   }
   ```

   上面代码中最终调用到了 `updateChild` 方法，这个方法在上面 Element的生命周期中有提到。

   在 `updateChild` 方法中就会判断 built 是否需要更新或者替换，如果需要替换，就会清除原来的，并且对新的 built 创建 对应的 `Element` ，并且在最后调用 built 对应 Element 的 `mount` 方法。这里的 Element 就不一定是 `StatelessElement` 了，而是看在  build 方法中的 widget 对应的 Element 是什么。

**总结一下**

从上面的流程分析中我们可以看出来整个流程就像是一个环，最开始的`framework` 调用 `mount` 。在 `mount` 中最终调用到了 `performRebuild` ，在 performRebuild 中通过调用我们实现的 `build` 方法，拿到对应的 widget 后，如果需要替换，就会重新创建 widget 的 element，并且调用这个 element 的 mount 方法。

![image-20211116191710072](https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20211116191710.png)

整个流程大致如上图所示

___

#### RenderObjectElement

我们以 `Flex` 为例来看一下是如何创建 `Element` 的。

```dart
@override
MultiChildRenderObjectElement createElement() => MultiChildRenderObjectElement(this);
```

`Fiex` 是通过父类 `SingleChildRenderObjectWidget` 来创建的 `Element`，、`MultiChildRenderObjectElement` 是继承自 `RenderObjectElement` 的。

接下来我们浅析一下 `RenderObjectElement `的调用过程

```dart
class MultiChildRenderObjectElement extends RenderObjectElement {
    
      @override
  void mount(Element? parent, Object? newSlot) {
    //调用super.mount 将传入的 parent 插入到树中  
    super.mount(parent, newSlot);
     // 
    final List<Element> children = List<Element>.filled(widget.children.length, _NullElement.instance, growable: false);
    Element? previousChild;
     //遍历所有的child
    for (int i = 0; i < children.length; i += 1) {
      //加载child  
      final Element newChild = inflateWidget(widget.children[i], IndexedSlot<Element?>(i, previousChild));
      children[i] = newChild;
      previousChild = newChild;
    }
    _children = children;
  }
}  

abstract class RenderObjectElement extends Element {
  @override
  void mount(Element? parent, Object? newSlot) {
    //将传入的 parent 插入到树中  
    super.mount(parent, newSlot);
      
    //创建与element相关联的renderObject 对象
    _renderObject = widget.createRenderObject(this);
    //将element.renderObjectZ 插入到渲染树中的指定位置  
    attachRenderObject(newSlot);
    _dirty = false;
  }
}    
```

我们大致分析一下

1. 首先是调用到 `MultiChildRenderObjectElement` 的 `mount` 方法中，先将传入的 parent 插入到树中，
2. 接着在 `RenderObjectElement` 中 创建了一个 `renderObject` 并添加到渲染树中插槽的指定位置
3. 最后回到 `MultiChildRenderObjectElement` 中遍历所有的 child，并且调用 `inflateWidget` 创建 child并插入到指定的插槽中。

___

#### StatefulElement

相比于 `StatelessWidget` ，`StatefulWidget` 中多了一个 `State`，在 `StatefulWidget` 中会通过 `createState` 来创建一个 `State`，如下所示

```dart
abstract class StatefulWidget extends Widget {
  const StatefulWidget({ Key? key }) : super(key: key);
  
  @override
  StatefulElement createElement() => StatefulElement(this);

  @protected
  @factory
  State createState(); // ignore: no_logic_in_create_state, this is the original sin
}
```

通过上面代码可以看出 `StatefulWidget` 对应的 `Element` 正是 `StatefulElement`。

```dart
class StatefulElement extends ComponentElement {
  /// Creates an element that uses the given widget as its configuration.
  StatefulElement(StatefulWidget widget)
      : _state = widget.createState(),
        super(widget) {
    assert(state._element == null);
    state._element = this;
    state._widget = widget;
  }
    
  State<StatefulWidget> get state => _state!;
  State<StatefulWidget>? _state;
    
  @override
  Widget build() => state.build(this);
} 
```

在 `StatefulElement` 中通过调用 `widget.createState` 获取到了 state 对象

`StatefulElement` 在创建完成后，`framework` 就会调用 `StatelessElement` 中的 `mount` 方法了。

**和 `StatelessElement` 不同的是：**

- `Statelesselement` 中是通过调用 `widget.build(this)` 方法
- `StatefulElement` 中是通过调用 `state.build(this)` 方法

___

#### 总结

通过上面的分析，我们知道了 Element 的生命周期以及它的调用过程。并且如果你仔细观察上面举例的三个 Element 就会发现，上面的这三种 `Element` 可以分为两类，分别是组合类和绘制类。

- 组合类一般都继承自 `StatelessElement` 或者 `StatefulElement`，它们都属于组合类，并不参与绘制，内部嵌套了多层 Widget，查看它们的 mount 方法，就会发现其中并没有创建 renderObject 并添加到渲染树。例如 `StatelessWidget` ，`StatefulWidget`，`Text`，`Container` ,`Image` 等。
- 绘制类的特征顾名思义就是参加绘制了，在mount 方法中，会有创建 renderObject 并 attachRenderObject 渲染树中。如 `Column` ，`SizedBox`，`Transform` 等。

其实还有一种类型，这里借用大佬的一张图来看一下：

<img src="https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/125ab361a1bd40e2bfe1bd457d440a8c~tplv-k3u1fbpfcp-watermark.awebp" style="zoom:50%;" />

代理类对应的 `element` 是 `ProxyElement`。



### RenderObject

在上面我们提到过每一个 `Element` 都对应一个 `RenderObject`，我们可以通过 `Element.renderObject` 来获取。并且 `RenderObject` 的主要职责是 Layout 绘制，所有的 RenderObject 会组成一颗渲染树 `Render Tree`。

通过上面的分析，我们知道树的核心就在 `mount` 方法中，我们直接查看 `RenderObjectElement.mount()` 方法

```dart
@override
void mount(Element? parent, Object? newSlot) {
  super.mount(parent, newSlot);
  _renderObject = widget.createRenderObject(this);
  attachRenderObject(newSlot);
  _dirty = false;
}
```

在执行完 `super.mount`  之后 (将 parent 插入到 element 树中)，执行了 `attachRenderObject` 方法。

```dart
@override
void attachRenderObject(Object? newSlot) {
  assert(_ancestorRenderObjectElement == null);
  _slot = newSlot;
  //查询当前最近的 RenderObject 对象  
  _ancestorRenderObjectElement = _findAncestorRenderObjectElement();
  // 将当前节点的 renderObject 对象插入到上面找的的 RenderObject下面  
  _ancestorRenderObjectElement?.insertRenderObjectChild(renderObject, newSlot);
  //.........	
}

RenderObjectElement? _findAncestorRenderObjectElement() {
   Element? ancestor = _parent;
   while (ancestor != null && ancestor is! RenderObjectElement)
     ancestor = ancestor._parent;
   return ancestor as RenderObjectElement?;
 }
```

上面代码也不难理解，就是一个死循环，退出的添加就是 ancestor 父节点为空和父节点是 RenderObjectElement 。说明这个方法会找到离当前节点最近的一个 RenderObjectElement 对象，然后调用  `insertRenderObjectChild` 方法，这个方法是一个抽象方法，我们查看两个类里面的重写逻辑

- #### SingleChildRenderObjectElement

  ```dart
  void insertRenderObjectChild(RenderObject child, Object? slot) {
    final RenderObjectWithChildMixin<RenderObject> renderObject = this.renderObject as RenderObjectWithChildMixin<RenderObject>;
    renderObject.child = child;
  }
  ```

  在上面代码中，找到了当前 `RenderObjectElement` 的 renderObject ，并且将我们传入的 child 给了 `renderObject` 的 child 。所以这样就将传入的 child 挂在了  RenderObject 树上。

- #### MultiChildRenderObjectElement

  ```dart
  @override
  void insertRenderObjectChild(RenderObject child, IndexedSlot<Element?> slot) {
    final ContainerRenderObjectMixin<RenderObject, ContainerParentDataMixin<RenderObject>> renderObject = this.renderObject;
    assert(renderObject.debugValidateChild(child));
    renderObject.insert(child, after: slot.value?.renderObject);
    assert(renderObject == this.renderObject);
  }
  ```

  上面代码中，找到 renderObject 之后赋值给了 `ContainerRenderObjectMixin<RenderObject, ContainerParentDataMixin<RenderObject>>` 这个类，我们来看一下这个类

  ```dart
  /// Generic mixin for render objects with a list of children.
  /// 具有子项列表的渲染对象的通用混合
  /// Provides a child model for a render object subclass that has a doubly-linked
  /// list of children.
  /// 为具有双向链接的子项列表的渲染对象子类提供子模型
  mixin ContainerRenderObjectMixin<ChildType extends RenderObject, ParentDataType extends ContainerParentDataMixin<ChildType>> on RenderObject {
  }
  ```

  > 泛型 mixin 用于渲染一组子对象的对象。看到这里我们可以知道 MultiChildRenderObjectElement 是可以有子列表的。

  通过上面的注释可以看出一个重点，双向链表。所以 `MultiChildRenderObjectElement` 的子节点通过双向链表连接。上面 insert 最终会调用到 `_insertIntoChildList ` 方法中，如下：

  ```dart
  ChildType? _firstChild;
  ChildType? _lastChild;
  void _insertIntoChildList(ChildType child, { ChildType? after }) {
    final ParentDataType childParentData = child.parentData! as ParentDataType;
    _childCount += 1;
    assert(_childCount > 0);
    if (after == null) {
      // after 为 null，则插入到 _firstChild 中
      childParentData.nextSibling = _firstChild;
      if (_firstChild != null) {
        final ParentDataType _firstChildParentData = _firstChild!.parentData! as ParentDataType;
        _firstChildParentData.previousSibling = child;
      }
      _firstChild = child;
      _lastChild ??= child;
    } else {
      final ParentDataType afterParentData = after.parentData! as ParentDataType;
      if (afterParentData.nextSibling == null) {
        // insert at the end (_lastChild); we'll end up with two or more children 
        // 将 child 插入到末尾  
        assert(after == _lastChild);
        childParentData.previousSibling = after;
        afterParentData.nextSibling = child;
        _lastChild = child;
      } else {
        // insert in the middle; we'll end up with three or more children
        // 插入到中间
        childParentData.nextSibling = afterParentData.nextSibling;
        childParentData.previousSibling = after;
        // set up links from siblings to child
        final ParentDataType childPreviousSiblingParentData = childParentData.previousSibling!.parentData! as ParentDataType;
        final ParentDataType childNextSiblingParentData = childParentData.nextSibling!.parentData! as ParentDataType;
        childPreviousSiblingParentData.nextSibling = child;
        childNextSiblingParentData.previousSibling = child;
        assert(afterParentData.nextSibling == child);
      }
    }
  }
  ```

  根据上面的注释我们可以将其分为三部分，1，在 after 为 null 的时候将 child 插入到第一个节点，2，将 child 插入到 末端，3，将 child 插入到中间。

  我们以一个例子为例：

  ```dart
  Column(
  	children: [
  	    SizedBox(.....),
  	    Text(data),
   	    Text(data),
   	    Text(data),
   	 ],
  )
  ```

  第一个 Stack 向上找到 Column( RenderObjectElement ) 之后， 调用这个方法，目前 after 为 null，则 _firstchild 就是 SizedBox( SizedBox 对应的 renderObject 是 `RenderConstrainedBox`)

  第二个是 Text，我们知道 Text 是组合类型的，所以他不会被挂载到 树中，通过查询源码可以看出最终的 text 用的是 `RichText`。RichText 向上查找到 Column( RenderObjectElement ) 之后 ，调用这个方法，传入了两个参数，第一个 child 是 RichText 对应的 `RenderParagraph` ，第二个 after 是 SizedBox 对应的 `RenderConstrainedBox` 。依照上面逻辑，执行下面的代码

  ```dart
  final ParentDataType afterParentData = after.parentData! as ParentDataType;
  if (afterParentData.nextSibling == null) {
    // insert at the end (_lastChild); we'll end up with two or more children
    assert(after == _lastChild);
    childParentData.previousSibling = after;
    afterParentData.nextSibling = child;
    _lastChild = child;
  } 
  ```

  将 child 的 `childParentData.previousSibling` 指向第一个节点，将第一个接的的  `afterParentData.nextSibling` 指向 child，最后让 _lastchild 执行 child。

  后面也是如此，当流程结束后就可以得到一个 `RenderTree`。

### 总结

本文主要介绍了三棵树的构建过程以及 `elemnt` 的生命周期，这些虽然我们在开发过程中用的比较少，但是却是通向 flutter 内部世界的大门。

其实在写这篇文章的时候也是一知半解，通过不断的查看源码，翻看博客等才慢慢的有些了解。并且在最后产出了这篇文章，可能文章中会有一些错误，如果你看到的话麻烦在下面提出来，感谢！！

### 推荐阅读

- [一文搞懂 BuildContext](https://juejin.cn/post/6991734742672474119#heading-0)
- [Key 的原理和使用](https://juejin.cn/post/6976069994270588941)

### 参考资料

- https://juejin.cn/post/6921493845330886670#heading-7
- https://juejin.cn/post/6844904194822914061#heading-14
- flutter 中文网



> 如果本文有帮助到你的地方，不胜荣幸，如有文章中有错误和疑问，欢迎大家提出
