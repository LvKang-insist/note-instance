### ListWheelScrollView

一个可滚动的列表，不同的是他的滚动是一个类似于转轮的效果，默认效果如下：

```dart
Widget build(BuildContext context) {
  return Container(
    height: 500,
    child: ListWheelScrollView(
      itemExtent: 70,
      children: List.generate(
          10,
          (index) => Container(
            height: 100,
            color: Colors.red,
            alignment: Alignment.center,
            child: Text(
                  "$index",
                  style: TextStyle(fontSize: 40,color: Colors.blue,),
                ),
              )).toList(),
    ),
  );
```

<img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210622230839.png" alt="image-20210622230839108" style="zoom:33%;" />

比较重要的属性如下：

- offAxisFraction

  轴心的偏移量，默认为0，可以为负数。如果为 0.5，则效果如下：

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210622233511.png" alt="image-20210622233511322" style="zoom:50%;" />

  如果是负数，则弧度就是向左了

- diameterRatio 直径比例。默认为2.0如果改为 1.0，效果如下：

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210622233811.png" alt="image-20210622233811271" style="zoom:33%;" />

- overAndUnderCenterOpacity 设置中间元素上下的不透明度，如 0.5：

  <img src="https://gitee.com/lvknaginist/pic-go-picure-bed/raw/master/images/20210622234035.png" alt="image-20210622234035443" style="zoom:50%;" />

- magnification：中间元素的放大倍数，默认是 1倍

- useMagnifier：是否启用放大镜，如果没有使用 overAndUnderCenterOpacity 属性，则此属性必须设置为 true。

- physics：滑动的效果
  - NeverScrollableScrollPhysics ：禁止滑动
  - FixedExtentScrollPhysics：保证每次滑动结束后，都能停到中间 Item 的上面。

- onSelectedItemChanged：滑动停止之后选择的某个 item 的下标回调



