### 简介

`Jetpack Compose` 是用于构建原生 Andorid 界面的新工具包，`Compose` 使用了更少的代码，强大的工具和直观的 Kotlin Api 简化并且加快了 Android 上界面的开发。

在 Compose 中，在构建界面的时候，无需在像之前那么构建 XML 布局，只需要调用 Jetpack Compose 函数来声明你想要的的元素，Compose 编译器就会自动帮你完成后面的工作。

在开始使用 Compose 之前，你需要重新搭建环境，可参考**[官方文档](https://developer.android.google.cn/jetpack/compose/setup)**



### 注解

- @Compose

  所有的组合函数都必须添加 `@Compose` 注解才可以。

  被 `@Compose` 注解的方法只能被同类型的方法调用。

- @Preview

  使用该注解的方法可以不在运行 App 的情况下就可以查看布局。`@Preview` 中常用的参数如下：

  1. `name: String`: 为该Preview命名，该名字会在布局预览中显示。

  2. `showBackground: Boolean`: 是否显示背景，true为显示。

  3. `backgroundColor: Long`: 设置背景的颜色。

  4. `showDecoration: Boolean`: 是否显示Statusbar和Toolbar，true为显示。

  5. `group: String`: 为该Preview设置group名字，可以在UI中以group为单位显示。

  6. `fontScale: Float`: 可以在预览中对字体放大，范围是从0.01。

  7. `widthDp: Int`: 在Compose中渲染的最大宽度，单位为dp。

  8. `heightDp: Int`: 在Compose中渲染的最大高度，单位为dp。

### Compose 编程思想

`Jetpack COmpose` 是一个适用于 android 的新式声明性界面工具包。`Compose` 提供了声明性 API ，可以在不以命令的方式改变前端视图的情况下呈现应用界面，从而使得编写和维护界面变得更加容易。

#### 申明性编程范式

长期以来，android 的视图结构一直可以表示为界面微件数。由于应用的状态会因用户交互等因素而发生变化，因此界面层次结构需要进行更新以显示当前的数据，最常见的就是 `findviewById` 等函数遍历树，并调用设置数据的方法等改变节点，这些方法会改变微件的内部状态

再过去的几年中，整个行业已经转向声明性界面模型，该模型大大的简化了构建和更新界面管理的工程设计，改技术的工作原理是在改建上重头生成整个屏幕，然后执行必要的更改。此方法可以避免手动更新有状态视图结构的复杂性。`Compose` 是一个声明性的界面框架。

重新生成整个屏幕所面临的一个难题是，在时间，计算力和电量方面可能成本高昂，为了减轻这一成本，`Compose` 会智能的选择在任何时间需要重新绘制界面的那些部分。这回对设计界面的组件有一定影响。



#### 组合函数

`Jetpack Compose` 是围绕可组合函数构建的，这些函数就是要显示在界面上的元素，在函数中只需要描述应用界面形状和数据依赖关系，而不用去关系界面的构建过程，

如果需要创建组合函数，只需要将 `@Composeable` 注解添加到对于的函数上即可，需要注意的是组合函数的名称一般都是以大写字母开头的，如下：

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            PrimaryTheme {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    Greeting("Android")
                }
            }
        }
    }
}

@Composable
fun Greeting(name: String) {
    Text(text = "Hello $name!", fontSize = 18.sp, color = Color.Red)
}

@Preview(showBackground = true)
@Composable
fun DefaultPreview() {
    PrimaryTheme {
        Greeting("Android")
    }
}
```

setContent 块 定义了 Activity 的布局，我们不需要去定义 XML 的布局内容，只需要在其中调用组合函数即可。

上面的 一个简单的示例`Greeting` 微件，它接收 `String` 而发出的一个显示问候消息的 `Text` 微件。此函数不会返回任何内容，因为他们描述所需的屏幕状态，而不是构造界面微件。

其中 Greeting 就是一个非常简单的可组合函数，里面定义了一个 Text，顾名思义，就是用来显示一段文本

并且，我们可以在 Test 函数上添加 @PreView 注释，这样就可以非常方便的进行预览。

#### 声明式范式转变

在 `Compose` 的声明方法中，微件相对无状态，并且不提供 get,set 方法。实际上，微件微件不会以对象的形式提供。你可以通过调用带有不同参数的统一可组合函数来更新界面。这使得架构模式，如 `ViewModel` 变得很容易。

引用逻辑为顶级可组合函数提供数据。该函数通过调用其他可组合函数来使用这些数据来描述界面。将适当的数据传递给这些可组合函数，并沿层次结构向下传递数据。

<img src="https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202205251708931.png" alt="image-20220525170811867" style="zoom:50%;" />

当用户与界面交互时，界面发起 `onClick`事件。这些事件会通知应用逻辑，应用逻辑可以改变应用状态。当状态发生变化时，系统就会重新调用可组合函数。这回导致重新绘制界面描述，此过程称为**重组**。

<img src="https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202205251708261.png" alt="image-20220525170848197" style="zoom:50%;" />

#### 动态内容

由于可组合函是 kotlin 编写的，因此他们可以像任何 kotlin 代码一样动态，例如，假设你想要的构建一个界面，如下：

```kotlin
@Composable
fun Greeting(names: List<String>) {
    for (name in names) {
        Text("Hello $name")
    }
}
```

此函数接受一个列表，每位每个列表元素生成一个 Text。可组合函数可能性非常复杂，你可以使用 `if` 语句来确定是否需要显示特定的界面元素。例如循环，辅助函数等。你拥有地城语言的灵活性，这种强大的功能和灵活性是 `JetpackCompose` 的主要优势之一。



#### 重组

在 `Compose` 中，你可以用新数据再次调用某个可组合函数，这回导致组合函数重新进行重组。系统会根据需要使用新数据重新绘制发出的微件。Compose 框架可以只能的重组已经更改的组件。

例如，下面这个可组合函数，用于显示一个按钮：

```kotlin
@Composable
fun ClickCounter(clicks: Int, onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("I've been clicked $clicks times")
    }
}
```

每次点击按钮，就会更新 `clicks` 的值，Compose 会再次调用 lambda 与 Text 函数以显示新值，此过程称为 `重组`。不依赖该值的其他元素不会重组。

重组是指在输入更改的时候再次调用可组合函数的过程。当函数更改时，会发生这种情况。当 Compose 根据新输入重组时，它仅调用可能已经更改的函数或 lambad，而跳过其余函数或 lambda。通过跳过岂会为更改参数的函数或者 lambda ，Compose 可以高效的重组。

切勿依赖于执行可组合函数所产生的附带效应，因为可能会跳过函数的重组，如果这样做，用户可能在应用中遇到奇怪且不可预测的行为。例如：

- 写入共享对象的属性
- 更新 viewmodel 中的可观察项
- 更新共享偏好设置

可组合函数可能会每一帧一样的频繁执行，例如呈现动画的时候。所以可组合函数需要快速执行，所以避免在组合函数中出现卡顿，如果你需要执行高昂的操作，请在狗太协程中执行，并将结果作为参数传递给可组合函数。

例如下面代码，应该将 sp 读取的操作放在 viewmode 中，然后在回调中触发更新：

```kotlin
@Composable
fun SharedPrefsToggle(
    text: String,
    value: Boolean,
    onValueChanged: (Boolean) -> Unit
) {
    Row {
        Text(text)
        Checkbox(checked = value, onCheckedChange = onValueChanged)
    }
}
```

##### 可组合函数可以按照任何顺序执行

如果你看到了可组合函数的代码，可能会认为他们按照顺序运行。但实际上未必是这样。如果某个可组合函数包含对其他组合代码的调用，这些函数可以按照顺序执行。

Compose 可以选择识别出某些界面元素的优先级高于其他界面元素，因此首先绘制这些元素。

假设你有如下代码：

```kotlin
@Composable
fun ButtonRow() {
    MyFancyNavigation {
        StartScreen()
        MiddleScreen()
        EndScreen()
    }
}
```

对于这三个的调用可以按照任何顺序进行。这意味着你不能让某个函数设置一个全局变量(附带效应)，并让别的函数利用这个全局变量而发生更改。所以每个函数都应该独立。

##### 可组合函数可以并行运行

Compose 可以通过并行运行可组合函数来优化重组。这样依赖，Compose 就可以利用多个核心，并按照较低的优先级运行可组合函数（不在屏幕上）

**这种优化方方式意味着可组合函数可能会在后台的线程池中执行**，如果某个可组合函数对 `viewModel` 调用一个函数，则 Compose 可能会同时从多个线程调动该函数。

为了确保应用可以正常运行，所有的组合都不应该有附带效应，而应该通过始终在界面线程上执行的 `onClick` 等回调触发附带效应。

调用某个可组合函数时，调用可能发生在与调用方不同的线程上。这意味着，应避免修改可组合函数 lambda 中的变量代码，基因为此类代码并非线程安全代码，又因为他是可组合 lambda 不允许的附带效应。

下面展示了一个可组合函数，他显示了一个列表已经数量。

```kotlin
@Composable
fun ListComposable(myList: List<String>) {
    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
            }
        }
        Text("Count: ${myList.size}")
    }
}
```

此函数没有附带效应，他会将输出列表转为界面。才代码非常适合展示小列表。不过此函数写入局部变量，则这并不是非线程安全或者正确的代码：

```kotlin
@Composable
@Deprecated("Example with bug")
fun ListWithBug(myList: List<String>) {
    var items = 0

    Row(horizontalArrangement = Arrangement.SpaceBetween) {
        Column {
            for (item in myList) {
                Text("Item: $item")
                items++ // Avoid! Side-effect of the column recomposing.
            }
        }
        Text("Count: $items")
    }
}
```

在上面例子中，每次重组都会修改 items。这可以在动画的第一帧，或者在列表更新的时候。但不管怎么样，界面都会显示出错误的数量。因此 Compose 不支持这样的写入操作。通过静止此类操作，我们允许框架更改线程以执行可组合 lambda。



##### 重组跳过尽可能多的内容

如果界面某些部分无需，Compose 会尽力只重组需要更新的部分。这意味着，他可以跳过某些内容以重新运行单个按钮的可组合项，而不执行树中其上面或下面的任何可组合项。

每个可组合函数和 lambda 都可以自行重组。以下演示了在呈现列表时重组如何跳过某些元素：

```kotlin
/**
 * Display a list of names the user can click with a header
 */
@Composable
fun NamePicker(
    header: String,
    names: List<String>,
    onNameClicked: (String) -> Unit
) {
    Column {
        // this will recompose when [header] changes, but not when [names] changes
        Text(header, style = MaterialTheme.typography.h5)
        Divider()

        // LazyColumn is the Compose version of a RecyclerView.
        // The lambda passed to items() is similar to a RecyclerView.ViewHolder.
        LazyColumn {
            items(names) { name ->
                // When an item's [name] updates, the adapter for that item
                // will recompose. This will not recompose when [header] changes
                NamePickerItem(name, onNameClicked)
            }
        }
    }
}

/**
 * Display a single name the user can click.
 */
@Composable
private fun NamePickerItem(name: String, onClicked: (String) -> Unit) {
    Text(name, Modifier.clickable(onClick = { onClicked(name) }))
}
```

这些作用域中的每一个都可能是在重组期间执行唯一一个作用域。当 header 发生更改时，Compose 可能会跳至 `Column lambda` 。二部执行他的任何父项。此外，执行 `Colum` 时，如果 names 未更改，`Compose` 可能会旋转跳过 LazyColum 的项。

同样，执行所有组合函数或者 lambda 都应该没有附带效应。当需要执行附带效应时，应该通过回调触发。

##### 重组是乐观操作

只要 Compose 任务某个可组合函数可能已经更改，就会开始重组。重组是乐观操作，也就是说 Compose 预计会在参数再次更改之前完成重组。如果某个参数在重组完成之间发生改变，Compose 可能会取消重组，并使用新的参数重新开始。

取消重组后，Compose 会从重组中舍弃界面树。如有附带效应依赖于显示的界面，即使取消了组成操作，也会应用该附带效应。这可能导致应用状态不一致。

确保每个可组合函数和 lambda 都幂等，且没有附带效应，以处理乐观的重组

##### 可组合函数可能会非常频繁的运行

在某些情况下，可能针对界面每一帧运行一个可组合函数，如果该函数成本高昂，可能会导致界面卡顿。

例如，你的微件重试读取设备配置，或者读取 sp，他可能会在一秒钟内读取这些数据上百次，这回对性能造成灾难性的影响。

如果您的可组合函数需要数据，它应为相应的数据定义参数。然后，您可以阿静成本高昂的工作移到其他线程，并使用 `mutableStateOf` 或者 `LiveData` 将相应的数据传递给 Compose。



### 主题

```kotlin
//深色
val DarkColorScheme = darkColors(
    primary = Purple80,
    onPrimary = Color(0xFFFFFFFF),
    secondary = PurpleGrey80,
)

//亮色
val LightColorScheme = lightColors(
    primary = Purple40,
    onPrimary = Color(0xFF333333),
    secondary = PurpleGrey40,
)

@Composable
fun PrimaryTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    // Dynamic color is available on Android 12+
    dynamicColor: Boolean = true,
    content: @Composable () -> Unit
) {

    MaterialTheme(
        colors = LightColorScheme,
        typography = Typography,
        content = content
    )
}
```

默认的主题定义如上所示，最终会调用 `MaterialTheme`。

Material 主题主要包含三个属性，分别是 颜色，排版，和内容，Api 如下：

```kotlin
@Composable
fun MaterialTheme(
    colors: Colors = MaterialTheme.colors, // 颜色集合
    typography: Typography = MaterialTheme.typography, // 排版集合
    shapes: Shapes = MaterialTheme.shapes, // 形状集合
    content: @Composable () -> Unit // 要展示的内容
)
```

#### 颜色

```kotlin
class Colors(
    primary: Color, // 主颜色，屏幕和元素都用这个颜色
    primaryVariant: Color, // 用于区分主颜色，比如app bar和system bar
    secondary: Color, // 强调色，悬浮按钮，单选/复选按钮，高亮选中的文本，链接和标题
    secondaryVariant: Color, // 用于区分强调色
    background: Color, // 背景色，在可滚动项下面展示
    surface: Color, // 表层色，展示在组件表层，比如卡片，清单和菜单(CardView,SheetLayout,Menu)等
    error: Color, // 错误色，展示错误信息，比如TextField的提示信息
    onPrimary: Color, // 在主颜色primary之上的文本和图标的颜色
    onSecondary: Color, // 在强调色secondary之上的文本和图标的颜色
    onBackground: Color, // 在背景色background之上的文本和图标的颜色
    onSurface: Color, // 在表层色surface之上的文本和图标的颜色
    onError: Color, // 在错误色error之上的文本和图标的颜色
    isLight: Boolean // 是否是浅色模式
) 
```

更多的可以查看 `lightColorScheme` 函数。

#### 排版

```kotlin
@Immutable
class Typography internal constructor(
    val h1: TextStyle,
    val h2: TextStyle,
    val h3: TextStyle,
    val h4: TextStyle,
    val h5: TextStyle,
    val h6: TextStyle,
    val subtitle1: TextStyle,
    val subtitle2: TextStyle,
    val body1: TextStyle,
    val body2: TextStyle,
    val button: TextStyle,
    val caption: TextStyle,
    val overline: TextStyle
) 
```



#### 形状

```kotlin
class Shapes( 
    // 小组件使用的形状，比如: Button，SnackBar，悬浮按钮等
    val small: CornerBasedShape = RoundedCornerShape(4.dp),
    
    // 中组件使用的形状，比如Card(就是CardView)，AlertDialog等
    val medium: CornerBasedShape = RoundedCornerShape(4.dp),
    
    // 大组件使用的形状，比如ModalDrawer或者ModalBottomSheetLayout(就是抽屉布局和清单布局)
    val large: CornerBasedShape = RoundedCornerShape(0.dp),
)
```



#### 使用

```kotlin
setContent {
    PrimaryTheme {
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.color.background
        ) {
            Greeting("Android")
        }
    }
}
```

```kotlin
@Composable
fun PrimaryTheme(
    themeType: ThemeType = themeTypeState.value,
    content: @Composable () -> Unit
) {
    val shapes = Shapes(
        small = RoundedCornerShape(4.dp),
        medium = RoundedCornerShape(8.dp),
        large = RoundedCornerShape(12.dp),
    )
    MaterialTheme(
        colors = getThemeForTheme(themeType),
        typography = Typography,
        shapes = shapes,
        content = content
    )
}
```

另外，我写了一个可动态切换的主题，**[有兴趣的可以看一下](https://github.com/LvKang-insist/WanAndroid_Compose/blob/master/app/src/main/java/com/lvkang/wadandroid/ui/theme/ThemeManager.kt)**



### UI

#### SetContent

```kotlin
setContent {
    PrimaryTheme {
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.colorScheme.background
        ) {
            Greeting("Android")
        }
    }
}
```

同 Android 中的 SetContentView。

#### Theme

创建项目之后，就会生成一个 `项目名称+Theme` 的 `@Compose` 方法，我们可以通过更改其中的颜色来完成对主题的修改。具体如上面的主题所示.



#### Modifier

`Modifier` 本质是一个接口，可以用来修饰各种布局，例如 宽高，padding 等，常见的如下：

- padding：有四个重载方法
- plus：将其他的 `Modifer` 加入到当前的 `Modifer` 中。
- fillMaxHeight，fillMaxWidth，fillmaxSize：类似于 `match_parent`，填充整个父 Layout
- with，height，size ：设置宽高度
- rtl，ltr：开始布局的方向
- widthIn，heightIn，sizeIn 设置布局的宽度和高度的最大值和最小值
- gravity：元素的位置，
- 等等

需要注意的是 `Modifier` 系列的方法都支持链式调用



#### Column，Row

类似于 `LinearLayout`，` Column ` 是横向的，`Row` 是竖向的。有四个参数：

- `Modifer`： 具体值如上述所示

- `verticalArrangement`：子元素竖向的排列规则

  常见的就是，上下左右中，比较特殊的就是 

  `SpaceEvenly` 均匀分配，

  `SpaceBetween` 第一个元素前和最后一个元素后没有空隙，其他的按比例放入。

  `SpaceAround` 把整体中的一半空隙凭据放入第一个和最后一个的开始和结束，剩余的一半等比放入各个元素。

- horizontalAlignment：和上面一个，只不过方向不同

- content：要显示的内容

栗子：@Composable () -> Unit

```dart
setContent {
    PrimaryTheme {
        Surface(
            modifier = Modifier.fillMaxSize(),
            color = MaterialTheme.colorScheme.background
        ) {
            Column {
                Row {
                    Button(
                        onClick = { themeTypeState.value = ThemeType.RED_THEME },
                        modifier = Modifier.width(100.dp),
                        colors = ButtonDefaults.buttonColors(containerColor = Color.Yellow)
                    ) {
                        Greeting(name = "按钮1")
                    }
                    Button(
                        onClick = { themeTypeState.value = ThemeType.GREEN_THEME },
                        modifier = Modifier.width(100.dp),
                        colors = ButtonDefaults.elevatedButtonColors()
                    ) {
                        Greeting(name = "按钮2")
                    }
                }
                Greeting(name = "Hello Android")
                Greeting(name = "Hello 345")
            }
        }
    }
}
```

![屏幕截图 2022-05-17 135439](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202205171356078.png)

#### Text

```kotlin
fun Text(
    text: String, //显示内容
    modifier: Modifier = Modifier, //修饰，可修改透明度，边框，背景等
    color: Color = Color.Unspecified, //文字颜色
    fontSize: TextUnit = TextUnit.Unspecified,// size
    fontStyle: FontStyle? = null, //文字样式，粗体，斜体等
    fontWeight: FontWeight? = null,//文字厚度
    fontFamily: FontFamily? = null,//字体
    letterSpacing: TextUnit = TextUnit.Unspecified, //用于与文本相关的维度值的单位。该组件还在测试中
    textDecoration: TextDecoration? = null,//文字装饰，中划线，下划线
    textAlign: TextAlign? = null,对齐方式
    lineHeight: TextUnit = TextUnit.Unspecified,//行高
    overflow: TextOverflow = TextOverflow.Clip,//如何处理溢出，默认裁切
    softWrap: Boolean = true,//是否软换行
    maxLines: Int = Int.MAX_VALUE,//最大行数
    onTextLayout: (TextLayoutResult) -> Unit = {},//计算布局时回调
    style: TextStyle = LocalTextStyle.current //文本的样式配置，如颜色、字体、行高等。
)
```

- modifier：在此处用来修饰 Text，Modifer 提供了很多扩展，如透明度，背景，边框等


 示例：

```kotlin
@Composable
fun Greeting(name: String) {
    Text(
        text = name,
        fontSize = 18.sp,
        fontWeight = FontWeight.Medium,
        color = MaterialTheme.colorScheme.primary,
        modifier = Modifier.height(30.dp)
    )
}
```



#### Button

```kotlin
fun Button(
    onClick: () -> Unit,//点击时调用
    modifier: Modifier = Modifier,//同上
    enabled: Boolean = true,//是否启用
    elevation: ButtonElevation? = ButtonDefaults.buttonElevation(),// z轴上的高度
    shape: Shape = FilledButtonTokens.ContainerShape.toShape(), 
    border: BorderStroke? = null,
    colors: ButtonColors = ButtonDefaults.buttonColors(),
    contentPadding: PaddingValues = ButtonDefaults.ContentPadding,
    content: @Composable RowScope.() -> Unit
)
```

- shape

  调整 button 的样式，例如 `RoundedCornerShape` 是圆角矩形的样式，`CircleShape` 是圆形的样式，`CutCornerShape` 是切角样式

- border

  外边框，默认是 null，Border 有两种使用方式，1 `Border(size: Dp, color: Color)`,2 `Border(size: Dp, brush: Brush)` 。

  第二种需要自己创建一个笔刷，去绘制外边框，例如要实现渐变的外边框。

- colors

  按钮的颜色，默认是 `ButtonDefaults.buttonColors()` 。可选的有：

  ![image-20220517151926468](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202205171519519.png)

​		其中可以设置按钮的背景色，未启用的颜色等。

栗子：

```kotlin
Button(
    onClick = { themeTypeState.value = ThemeType.GREEN_THEME },
    modifier = Modifier.width(100.dp),
    colors = ButtonDefaults.buttonColors(
        containerColor = Color.Yellow,
        contentColor = Color.Red,
        disabledContainerColor = Color.Black,
        disabledContentColor = Color.Green
    )
) {
    Greeting(name = "按钮2")
}
```

#### OutLinedButton

具有外边框的按钮，内部使用的也是 `Button`。默认会有一个边框，其参数和 `Button` 一致，效果如下

![image-20220517163336439](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202205171633495.png)

#### TextButton

默认的 button 在有主题的时候，默认背景是主题颜色，而 textButton 背景默认是透明的。`TextButton` 默认使用的颜色是 `ButtonDefaults.textButtonColors()`



#### Image

```kotlin
@Composable
fun Image(
    painter: Painter,
    bitmap: ImageBitmap, //
    contentDescription: String?,
    modifier: Modifier = Modifier,
    alignment: Alignment = Alignment.Center,
    contentScale: ContentScale = ContentScale.Fit,
    alpha: Float = DefaultAlpha,
    colorFilter: ColorFilter? = null
) 
```

- painter：图片资源，使用 PainterResource 来完成。
- contentDescription：无障碍提示文本信息
- contentScale ：类似于 ImageView 中的 scaleType 属性。
- colorFilter：将某种颜色应用到图片上
- alpha：不透明度

##### 示例

```dart
@Composable
@Preview
fun Image() {
    Image(
        painter = painterResource(id = R.drawable.one),
        contentDescription = "无障碍提示",
        contentScale = ContentScale.Crop,
        modifier = Modifier
            .width(100.dp)
            .height(100.dp)
    )
}

```

![image-20220517172753037](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202205171727140.png)

像一些圆图或者边框啥的就可以在 modifer 中直接设置了，如下：

```kotlin
@Composable
@Preview
fun Image() {
    Image(
        painter = painterResource(id = R.drawable.one),
        contentDescription = "无障碍提示",
        contentScale = ContentScale.Crop,
        modifier = Modifier
            .width(100.dp)
            .height(100.dp)
            .clip(shape = CircleShape)
            .border(2.dp, color = Color.Red, shape = CircleShape)
    )
}
```

![image-20220517173235028](https://raw.githubusercontent.com/LvKang-insist/PicGo/main/202205171732130.png)

##### 加载网路图片

加载网路图片需要借助第三方库 `coil`，使用方式如下：

```tex
//图片加载库
implementation("io.coil-kt:coil:2.0.0")
implementation("io.coil-kt:coil-compose:2.0.0")
```

```kotlin
@Composable
@Preview
fun Image() {
    AsyncImage(
        model = "https://img0.baidu.com/it/u=3147375221,1813079756&fm=253&fmt=auto&app=120&f=JPEG?w=500&h=836",
        contentDescription = "无障碍提示",
        contentScale = ContentScale.Crop,
        modifier = Modifier
            .width(100.dp)
            .height(100.dp)
            .clip(shape = CircleShape)
            .border(2.dp, color = Color.Red, shape = CircleShape)
    )
}
```



#### Spacer

和原生的一样，需要空白区域时可以使用 `Spacer` ，使用方式如下：

```kotlin
Spacer(modifier = Modifier.height(100.dp))
```



#### Surface

对内容进行装饰，例如设置背景，shape 等

```kotlin
fun Surface(
    modifier: Modifier = Modifier,
    shape: Shape = Shapes.None,
    color: Color = MaterialTheme.colorScheme.surface,
    contentColor: Color = contentColorFor(color),
    tonalElevation: Dp = 0.dp,
    shadowElevation: Dp = 0.dp,
    border: BorderStroke? = null,
    content: @Composable () -> Unit
) 
```

- color ：设置 `Surface` 的背景色，默认是主题中的 `surface` 颜色。
- contentColor：此 Surface 为其子级提供的首选内容颜色。默认为 [color] 的匹配内容颜色，或者如果 [color] 不是来自主题的颜色，这将保持在此 Surface 上方设置的相同值。
- tonalElevation：当 [color] 为 [ColorScheme.surface] 时，高程越高，浅色主题颜色越深，深色主题颜色越浅。
- shadowElevation：阴影大小



#### Scaffold

脚手架的意思，和 `Flutter` 中的 `Scaffold` 是一样的，通过 `Scaffold` 我看可以快速的对页面进行布局，例如设置导航栏，侧滑栏，底部导航等等。

```kotlin
fun Scaffold(
    modifier: Modifier = Modifier,
    topBar: @Composable () -> Unit = {},
    bottomBar: @Composable () -> Unit = {},
    snackbarHost: @Composable () -> Unit = {},
    floatingActionButton: @Composable () -> Unit = {},
    floatingActionButtonPosition: FabPosition = FabPosition.End,
    containerColor: Color = MaterialTheme.colorScheme.background,
    contentColor: Color = contentColorFor(containerColor),
    content: @Composable (PaddingValues) -> Unit
)
```

- topBar：Toolbar，常用的有 `CenterAlignedTopAppBar`,`SmallTopAppBar`，`MediumTopAppBar` 等。
- bootomBar：底部导航栏
- snackbarHost：
- floatingActionButton：按钮
- floatingActionButtonPosition：按钮位置
- containerColor：背景颜色
- contentColor：内容首选颜色

##### 看一个栗子：

```kotlin
Scaffold(
    topBar = {
       //.....
    },
    bottomBar = bottomBar,
) {
    Box(
        modifier = Modifier
            .fillMaxSize()
            .padding(top = it.calculateTopPadding(), bottom = it.calculateBottomPadding())
    ) {
        content.invoke(it)
    }
}
```

需要注意的是，如果使用了 toolbar 或者 bootomBar，就会把 content 中的内容挡住，这个时候就需要使用 PaddingValue 设置内边距了。

还有一点须要注意，如果要使用沉浸式状态栏，就需要自定义 topBar 了，要不然状态栏会被 topBar 覆盖。下面代码是设置沉浸式状态栏的。

```kotlin
///系统 UI 控制器
implementation "com.google.accompanist:accompanist-systemuicontroller:0.24.8-beta"
//正确获取状态栏高度
api "com.google.accompanist:accompanist-insets-ui:0.24.8-beta"
```

```dart
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    WindowCompat.setDecorFitsSystemWindows(window, false)
    setContent {
        SetImmersion()
        PrimaryTheme {
            SetContent()
        }
    }
}


@Composable
private fun SetImmersion() {
    if (isImmersion()) {
        val systemUiController = rememberSystemUiController()
        SideEffect {
            systemUiController.run {
                setSystemBarsColor(color = Color.Transparent, darkIcons = isDark())
                setNavigationBarColor(color = Color.Black)
            }
        }
    }
}
```

##### 底部导航栏

```kotlin
@Composable
fun MainCompose(navController: NavHostController, mainBottomState: MutableState<Int>) {
    SetScaffold(
        bottomBar = {
            BottomBar(mainBottomState)
        }
    ) {
        when (mainBottomState.value) {
            0 -> HomeCompos(navController)
            1 -> ProjectCompos()
            2 -> FLCompos()
            else -> UserCompos()
        }
    }
}


@Composable
private fun BottomBar(mainBottomState: MutableState<Int>) {
    BottomNavigation(
        backgroundColor = MaterialTheme.colors.background,
    ) {

        navigationItems.forEachIndexed { index, navigationItem ->
            BottomNavigationItem(
                selected = mainBottomState.value == index,
                onClick = {
                    mainBottomState.value = index
                },
                icon = {
                    Icon(
                        imageVector = navigationItem.icon,
                        contentDescription = navigationItem.name
                    )
                },
                label = {
                    BottomText(
                        isSelect = mainBottomState.value == index,
                        name = navigationItem.name
                    )
                },
                selectedContentColor = Color.White,
                unselectedContentColor = Color.Black
            )
        }
    }
}

@Composable
fun BottomText(isSelect: Boolean, name: String) {
    if (isSelect) {
        Text(
            text = name,
            color = MaterialTheme.colors.primary,
            fontSize = 12.sp
        )
    } else {
        Text(
            text = name,
            color = Color.Black,
            fontSize = 12.sp
        )
    }
}
```

如果看的不是特别清楚，[可以直接点这里看](https://github.com/LvKang-insist/WanAndroid_Compose)



### 最后

到这里，这篇文章也完了。这篇文章主要讲了一下 `Compose` 中最基本的一些 核心思想以及 UI 函数以及主题啥的。这也是我最开始接触到 `Compose` 学到的东西，所以这也算是我的学习笔记吧。



### 参考资料

> https://developer.android.google.cn/jetpack/compose/documentation
>
> 以及网上的一些文章



> 如果本文对你有帮助，请点赞支持，谢谢！如果有任何问题，可直接在下方评论，谢谢!
