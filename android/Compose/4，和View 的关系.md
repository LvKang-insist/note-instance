### setContent

#### ComponentActivity.setContent

```kotlin
public fun ComponentActivity.setContent(
    parent: CompositionContext? = null,
    content: @Composable () -> Unit
) {
    val existingComposeView = window.decorView
        .findViewById<ViewGroup>(android.R.id.content)
        .getChildAt(0) as? ComposeView
	//判断 composeView 是否存在，如果不存在则创建
    if (existingComposeView != null) with(existingComposeView) {
        //1
        setParentCompositionContext(parent)
        setContent(content)
    } else ComposeView(this).apply {
        // 2
        setParentCompositionContext(parent)
        setContent(content)
        setContentView(this, DefaultActivityContentLayoutParams)
    }
}
```

上面代码中，会首先从获取 `DecorView` 中的第一个 View，默认是 ComposeView。

注释1：composeView 如果不为 null，就会重新设置内容。

注释2：如果等于 null，就会设置内容，并且调用 `setContentView` 将 ComposeView 添加到 DecorView 中。

通过上面源码，我们可以看出重点的地方有两个，分别是 `setParentCompositionContext(parent)` 和  `setContent(content)` 。 setParentCompositionContext 中主要设置了一下父级的 CompositionContext。所以我们需要重点关注的是 `setContent(content)`。

#### ComposeView.setContent

```kotlin
private val content = mutableStateOf<(@Composable () -> Unit)?>(null)
fun setContent(content: @Composable () -> Unit) {
    shouldCreateCompositionOnAttachedToWindow = true
    this.content.value = content
    if (isAttachedToWindow) {
        createComposition()
    }
}
```

首先是将 content 赋值给 `this.content.value` ，然后观察者就会收到改变通知

然后就是判断是否已经附加到 window 上，如果没有 就调用了  `createComposition()`

#### ComposeView.createComposition

```kotlin
fun createComposition() {
    check(parentContext != null || isAttachedToWindow) {
        "createComposition requires either a parent reference or the View to be attached" +
            "to a window. Attach the View or call setParentCompositionReference."
    }
    ensureCompositionCreated()
}
```

仅当此视图isAttachedToWindow或已显式设置父CompositionContext时才应调用此方法。

#### ComposeView.ensureCompositionCreated()

```kotlin
private fun ensureCompositionCreated() {
    if (composition == null) {
        try {
            creatingComposition = true
            composition = setContent(resolveParentCompositionContext()) {
                Content()
            }
        } finally {
            creatingComposition = false
        }
    }
}
```

composition 默认是为空的，所以第一次 if 是可以进去的。然后将创建组合的标记设置为 true，接着调用 setContent 来创建 ComposeView，最后将标记设为 false。

#### Wrapper.Android.kt ->setContent

```kotlin
internal fun AbstractComposeView.setContent(
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
    GlobalSnapshotManager.ensureStarted()
    val composeView =
        if (childCount > 0) {
            getChildAt(0) as? AndroidComposeView
        } else {
            removeAllViews(); null
        } ?: AndroidComposeView(context).also { addView(it.view, DefaultLayoutParams) }
    return doSetContent(composeView, parent, content)
}
```

如果子 child 大于 0，就直接获取第一个，否则就创建一个 `AndroidComposeView`。

最后调用 `doSetContent` 将 Compose 添加到 `ComposeView` 上。

#### Wrapper.Android.kt -> doSetContent

```kotlin
private fun doSetContent(
    owner: AndroidComposeView,
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
	//...
    //创建 Composeition
    val original = Composition(UiApplier(owner.root), parent)
    val wrapped = owner.view.getTag(R.id.wrapped_composition_tag)
        as? WrappedComposition
        ?: WrappedComposition(owner, original).also {
            owner.view.setTag(R.id.wrapped_composition_tag, it)
        }
    //将 Compose 内容添加到 Composition 中
    wrapped.setContent(content)
    return wrapped
}
```



### Compose 执行过程

我们知道，在 View 中会有一个 `ViewTree`，通过一个树来描述整个 UI 界面。

在 `Compose`中我们写的代码在渲染的时候也会构建成一个 `NodeTree`，每一个组件就是一个 `ComposeNode` ，作为 `NodeTree` 上的一个节点。

`Compose` 对 `NodeTree` 的管理涉及 `Applier` ，`Composition` 和 `ComposeNode`。

 `Composition` 作为起点发起首次的  Composition，通过 `Compose` 的执行，填充 `Slot Table`，并基于 `Table` 创建 `NodeTree`。渲染引擎基于 `ComposeNode` 渲染 `UI` ，每当 `recomposition` 发生时，都会通过 `Applier` 对 `NodeTree` 进行更新。

因此 `Compose` 的执行过程就是创建 `Node` 并构建 `NodeTree` 的过程 。

![image-20220615174708258](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202206151747338.png)



#### Applier

`Applier` 的作用就是增删 `NodeTree` 节点，每个 `NodeTree` 的运算都需要配套一个 `Applier` ，同时 `Applier` 会提供回调，基于回调我们可以对 `NodeTree` 进行能自定义修改：

```kotlin
interface Applier<N> {
    //当前处理的节点
    val current: N

    //更改时调用
    fun onBeginChanges() {}

    //更改结束调用
    fun onEndChanges() {}

    //正在“向下”遍历树。当这个被调用时， node应该是current的一个 child，并且在这个操作之后， node应该是新的current 。
    fun down(node: N)

    //向上遍历
    fun up()

    //将实例作为子项插入到当前 index 处（自顶向下）
    fun insertTopDown(index: Int, instance: N)

    //添加节点，自底向上
    fun insertBottomUp(index: Int, instance: N)

    //删除节点
    fun remove(index: Int, count: Int)

    //移动节点
    fun move(from: Int, to: Int, count: Int)

    //移动到根节点并删除所有节点
    fun clear()
}
```

`Applier` 有一个默认的实现类 `AbstractApplier` 这个实现类是抽象的，我们可以通过继承这个抽象类来做自定义的修改

`Applier` 需要在  **组成/重组** 的过程中进行调用。组成是通过 `Composition` 中对 Root Composable 的调用发起的，进而调用全部 Composable 最终形成的 NodeTree 。

 



#### Composition

`Composition` 是 Compose 执行的起点，下面我们看一下如何创建一个 `Composition`

```kotlin
val composition = Composition(applier: Applier<*>,  parent: CompositionContext)

composition.setContent{
	//可组合函数调用
}
```

如上，`Comopsition` 中需要传两个参数，`Applier`  和  `CompositionContext`

`Aplier` 上面已经说了，是用来增删节点的。

`CompositionContext` 是一个抽象类，实现类是 `Recomposer` ，`Recomposer` 非常重要，他负责 Compose 的重组。

当 NodeTree 首次创建完成后，与 state 建立关联，监听 state 的变化发生重组。这个关联的建立是通过 `Recompose` 的 "快照系统"  完成的。重组后，Recompose 通过调用 Applier 完成 NodeTree 的变更。

`Composition#setContent` 为后续的可组合函数提供了容器：

```kotlin
interface Composition {

    val hasInvalidations: Boolean

    val isDisposed: Boolean

    fun dispose()

    fun setContent(content: @Composable () -> Unit)
}
```

#### ComposeNode

理论上每个 `Composeable` 的执行都对应一个 Node 的创建，但是由于 NodeTree 无需全量重建，所以也不是每次都需要创建新的 Node。大多的 Composeable 都会调用 `ComposeNode()`接受一个 factory，仅在必要的时候创建 Node。

以 `Layout` 的实现为例

```kotlin
@Composable inline fun Layout(
    content: @Composable @UiComposable () -> Unit,
    modifier: Modifier = Modifier,
    measurePolicy: MeasurePolicy
) {
    val density = LocalDensity.current
    val layoutDirection = LocalLayoutDirection.current
    val viewConfiguration = LocalViewConfiguration.current
    ReusableComposeNode<ComposeUiNode, Applier<Any>>(
        factory = ComposeUiNode.Constructor,
        update = {
            set(measurePolicy, ComposeUiNode.SetMeasurePolicy)
            set(density, ComposeUiNode.SetDensity)
            set(layoutDirection, ComposeUiNode.SetLayoutDirection)
            set(viewConfiguration, ComposeUiNode.SetViewConfiguration)
        },
        skippableUpdate = materializerOf(modifier),
        content = content
    )
}
```

- **factory**：创建 Node 的工厂
- **update**：接受 receiver 为 `Update<T> ` 的 lambda，用来更新当前属性
- **content**：调用子 Composable



```kotlin
@Composable @ExplicitGroupsComposable
inline fun <T, reified E : Applier<*>> ReusableComposeNode(
    noinline factory: () -> T,
    update: @DisallowComposableCalls Updater<T>.() -> Unit,
    noinline skippableUpdate: @Composable SkippableUpdater<T>.() -> Unit,
    content: @Composable () -> Unit
) {
    if (currentComposer.applier !is E) invalidApplier()
    currentComposer.startReusableNode()
    if (currentComposer.inserting) {
        currentComposer.createNode(factory)
    } else {
        currentComposer.useNode()
    }
    currentComposer.disableReusing()
    Updater<T>(currentComposer).update()
    currentComposer.enableReusing()
    SkippableUpdater<T>(currentComposer).skippableUpdate()
    currentComposer.startReplaceableGroup(0x7ab4aae9)
    content()
    currentComposer.endReplaceableGroup()
    currentComposer.endNode()
}
```

在 composition 过程中，通过调用 Composer 上下文，更新 `SlotTable` ，`content()` 递归创建子 Node

SlotTable 在更新过程中，通过 diff 决定是否需要对 Node 进行 add/update/remove 等操作。此处的 `startReusableNode`，`useNode`，`endNode` 等就是对 SlotTable 的遍历过程。

SlotTable 的 diff 结果通过 `Applier` 的回调处理 NodeTree 结构的变化。通过调用 `Update<T>.update()` 来处理 Node 属性的变化。

