AppBarLayou 继承自LinearLayout ，布局方向为垂直方向，所以你可以把它当成垂直布局的LinearLayout来使用。AppBarLayout是在LinearLayou上加了一些材料设计的概念，它可以让你定制当**某个可滚动View**的滚动手势发生变化时，其内部的子View实现何种动作。 

​	请注意：上面提到的某个可滚动View，可以理解为某个ScrollView。怎么理解上面的话呢？就是说，当某个ScrollView发生滚动时，你可以定制你的“顶部栏”应该执行哪些动作（如跟着一起滚动、保持不动等等）。那某个可移动的View到底是哪个可移动的View呢？这是由你自己指定的！如何指定，我们后面说。

