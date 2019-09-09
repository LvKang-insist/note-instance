##### 话不多说，先看一下效果

---


### 接下来看一下具体实现的步骤

---
#### 1. 添加依赖，这个我就不多说了

---

[如果不会可以参考这篇博客](https://blog.csdn.net/baidu_40389775/article/details/85552113)。
#### 2. 设置布局

---

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".Main3Activity">
    <com.github.mikephil.charting.charts.PieChart
        android:id="@+id/mPieChart"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>
</LinearLayout>
```
#### 3. 加载控件，设置一些属性

---


```
 private void init() {
        mPieChart = findViewById(R.id.mPieChart);
        //以百分比为单位
        mPieChart.setUsePercentValues(true);
        mPieChart.getDescription().setEnabled(false);
        //偏移量
        mPieChart.setExtraOffsets(5, 10, 5, 5);
        //设置饼图旋转的摩擦力
        mPieChart.setDragDecelerationFrictionCoef(0.9f);
        //设置中间值
        mPieChart.setCenterText("我是345");
        //是否将饼心绘制空心
        mPieChart.setDrawHoleEnabled(true);
        //设置饼图中心圆的颜色
        mPieChart.setHoleColor(Color.WHITE);
        //设置饼图中心圆的边颜色
        mPieChart.setTransparentCircleColor(Color.WHITE);
//        设置饼图中心圆的边的透明度，0--255 0表示完全透明
        mPieChart.setTransparentCircleAlpha(110);
        //设置饼图中心圆的半径
        mPieChart.setHoleRadius(30f);
        //设置饼图中心圆的边的半径
        mPieChart.setTransparentCircleRadius(35f);
        //是否在饼图的中心绘制字
        mPieChart.setDrawCenterText(true);
        //设置雷达图旋转的偏移量(以度为单位)。默认270 f
        mPieChart.setRotationAngle(360);
        //设置是否可以触摸旋转
        mPieChart.setRotationEnabled(true);
        //设置是否点击后将对应的区域进行突出
        mPieChart.setHighlightPerTapEnabled(true);
        mPieChart.setDrawSlicesUnderHole(false);
        //为图标设置一个选择监听器
        mPieChart.setOnChartValueSelectedListener(this);
    }
```
    具体的说明都注释在代码里面了，
    强调一下mPieChart.setUsePercentValues(true)方法。调用这个方法会将你设置的值以百分比的形式展示在饼图上面
#### 4. 设置数据

---

```
private void setData() {
        //设置数据
        ArrayList<PieEntry> entries = new ArrayList<>();
        entries.add(new PieEntry(40, "优秀"));
        entries.add(new PieEntry(30, "良好"));
        entries.add(new PieEntry(20, "一般"));
        entries.add(new PieEntry(10, "差"));

        //添加数据
        PieDataSet DataSet = new PieDataSet(entries, "三年级一班");
        //设置饼图每个区域的间隔
        DataSet.setSliceSpace(10f);
        //设置点击某个区域后突出显示的距离，保证要能被点击
        DataSet.setSelectionShift(30f);

        //设置颜色
        ArrayList<Integer> colors = new ArrayList<>();
        for (int c : ColorTemplate.VORDIPLOM_COLORS)
            colors.add(c);
        for (int c : ColorTemplate.JOYFUL_COLORS)
            colors.add(c);
        for (int c : ColorTemplate.COLORFUL_COLORS)
            colors.add(c);
        for (int c : ColorTemplate.LIBERTY_COLORS)
            colors.add(c);
        for (int c : ColorTemplate.PASTEL_COLORS)
            colors.add(c);
        colors.add(ColorTemplate.getHoloBlue());

//      设置在此数据集之前应该使用的颜色。颜色是重用
        DataSet.setColors(colors);

        DataSet.setYValuePosition(PieDataSet.ValuePosition.OUTSIDE_SLICE);
        DataSet.setXValuePosition(PieDataSet.ValuePosition.OUTSIDE_SLICE);
        //当valuePosition位于外部时，指示偏移量为片大小的百分比
        DataSet.setValueLinePart1OffsetPercentage(100.f);
        //当valuePosition位于外部时，指示行前半部分的长度
        DataSet.setValueLinePart1Length(1f);
        //当valuePosition位于外部时，指示行下半部分的长度
        DataSet.setValueLinePart2Length(.1f);
        DataSet.setValueLineWidth(2);
        DataSet.setValueLineColor(Color.BLACK);


        //添加数据
        PieData pieData = new PieData(DataSet);
        //自定义饼图每个区域显示的值，
        pieData.setValueFormatter(new PercentFormatter());
        /*pieData.setValueFormatter(new IValueFormatter() {
            @Override
            public String getFormattedValue(float value, Entry entry, int dataSetIndex, ViewPortHandler viewPortHandler) {
                return "哈哈哈";
            }
        });*/
        //设置值的大小
        pieData.setValueTextSize(50f);
        pieData.setValueTextColor(Color.BLACK);
        //设置数据
        mPieChart.setData(pieData);
        mPieChart.highlightValues(null);
        mPieChart.invalidate();//刷新
    }
```

这段代码主要就是设置数据，刚开始是一个数据的集合。然后就是设置颜色，这个颜色也不一定这个写，可以自定义颜色，比如：

```
ArrayList<Integer> colors = new ArrayList<>();
        colors.add(Color.parseColor("#B44335"));
        colors.add(Color.parseColor("#3035B4"));
        
        PieDataSet dataSet = new PieDataSet(list,"");
        dataSet.setColors(colors);
```
这个就是设置的自定义颜色。设置数据还要说一下的就是
setValueFormantter方法，在这个方法里面实现IValueFormatter接口可以自定义你要显示的值，设置数据的代码中做了一下简单的介绍，具体的可以自己研究。
还有就是100到106行设置的是一个折线，具体的可以看图看代码。
#### 5. 设置动画，还有图例，图例就是图片中右上角的数据

---

```
 private void setFrom() {
 //设置动画
        mPieChart.animateY(1400, Easing.EasingOption.EaseInOutQuad);
//设置图例
        Legend l = mPieChart.getLegend();
        l.setVerticalAlignment(Legend.LegendVerticalAlignment.TOP);
        l.setHorizontalAlignment(Legend.LegendHorizontalAlignment.RIGHT);
        l.setOrientation(Legend.LegendOrientation.VERTICAL);
        l.setDrawInside(false);
        l.setEnabled(true);
//        l.setXEntrySpace(7f);
//        l.setYEntrySpace(0f);
//        l.setYOffset(0f);

        //设置用于绘制条目标签的颜色
        mPieChart.setEntryLabelColor(Color.BLACK);
        mPieChart.setEntryLabelTextSize(20f);
    }
```
到这里我们的饼状图已经完成了，当然这里面的属性是根据自己的需求做出来的。我设置了这么多的属性完全是为了解释一下这些属性到底是干什么的。是不是感到非常贴心呢！

---
好了，下面奉献出全部的代码。

```
package m.com.mpchart;

import android.graphics.Color;
import android.graphics.Typeface;
import android.service.autofill.Dataset;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.text.SpannableString;
import android.text.style.ForegroundColorSpan;
import android.text.style.RelativeSizeSpan;
import android.text.style.StyleSpan;

import com.github.mikephil.charting.animation.Easing;
import com.github.mikephil.charting.charts.PieChart;
import com.github.mikephil.charting.components.Legend;
import com.github.mikephil.charting.data.Entry;
import com.github.mikephil.charting.data.PieData;
import com.github.mikephil.charting.data.PieDataSet;
import com.github.mikephil.charting.data.PieEntry;
import com.github.mikephil.charting.formatter.IValueFormatter;
import com.github.mikephil.charting.formatter.PercentFormatter;
import com.github.mikephil.charting.highlight.Highlight;
import com.github.mikephil.charting.listener.OnChartValueSelectedListener;
import com.github.mikephil.charting.utils.ColorTemplate;
import com.github.mikephil.charting.utils.ViewPortHandler;

import java.util.ArrayList;

public class Main3Activity extends AppCompatActivity implements OnChartValueSelectedListener {
    private PieChart mPieChart;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main3);

        init();
        setData();
        setFrom();
    }

    private void setFrom() {
        mPieChart.animateY(1400, Easing.EasingOption.EaseInOutQuad);
        Legend l = mPieChart.getLegend();
        l.setVerticalAlignment(Legend.LegendVerticalAlignment.TOP);
        l.setHorizontalAlignment(Legend.LegendHorizontalAlignment.RIGHT);
        l.setOrientation(Legend.LegendOrientation.VERTICAL);
        l.setDrawInside(false);
        l.setEnabled(true);
//        l.setXEntrySpace(7f);
//        l.setYEntrySpace(0f);
//        l.setYOffset(0f);

        //设置用于绘制条目标签的颜色
        mPieChart.setEntryLabelColor(Color.BLACK);
        mPieChart.setEntryLabelTextSize(20f);
    }

    private void setData() {
        //设置数据
        ArrayList<PieEntry> entries = new ArrayList<>();
        entries.add(new PieEntry(40, "优秀"));
        entries.add(new PieEntry(30, "良好"));
        entries.add(new PieEntry(20, "一般"));
        entries.add(new PieEntry(10, "差"));

        //添加数据
        PieDataSet DataSet = new PieDataSet(entries, "三年级一班");
        //设置饼图每个区域的间隔
        DataSet.setSliceSpace(10f);
        //设置点击某个区域后突出显示的距离，保证要能被点击
        DataSet.setSelectionShift(30f);

        //设置颜色
        ArrayList<Integer> colors = new ArrayList<>();
        for (int c : ColorTemplate.VORDIPLOM_COLORS)
            colors.add(c);
        for (int c : ColorTemplate.JOYFUL_COLORS)
            colors.add(c);
        for (int c : ColorTemplate.COLORFUL_COLORS)
            colors.add(c);
        for (int c : ColorTemplate.LIBERTY_COLORS)
            colors.add(c);
        for (int c : ColorTemplate.PASTEL_COLORS)
            colors.add(c);
        colors.add(ColorTemplate.getHoloBlue());

//      设置在此数据集之前应该使用的颜色。颜色是重用
        DataSet.setColors(colors);

        DataSet.setYValuePosition(PieDataSet.ValuePosition.OUTSIDE_SLICE);
        DataSet.setXValuePosition(PieDataSet.ValuePosition.OUTSIDE_SLICE);
        //当valuePosition位于外部时，指示偏移量为片大小的百分比
        DataSet.setValueLinePart1OffsetPercentage(100.f);
        //当valuePosition位于外部时，指示行前半部分的长度
        DataSet.setValueLinePart1Length(1f);
        //当valuePosition位于外部时，指示行下半部分的长度
        DataSet.setValueLinePart2Length(.1f);
        DataSet.setValueLineWidth(2);
        DataSet.setValueLineColor(Color.BLACK);

        //添加数据
        PieData pieData = new PieData(DataSet);
        //自定义饼图每个区域显示的值，
        pieData.setValueFormatter(new PercentFormatter());
        /*pieData.setValueFormatter(new IValueFormatter() {
            @Override
            public String getFormattedValue(float value, Entry entry, int dataSetIndex, ViewPortHandler viewPortHandler) {
                return "哈哈哈";
            }
        });*/
        //设置值的大小
        pieData.setValueTextSize(50f);
        pieData.setValueTextColor(Color.BLACK);
        //设置数据
        mPieChart.setData(pieData);
        mPieChart.highlightValues(null);
        mPieChart.invalidate();//刷新
    }

    private void init() {
        mPieChart = findViewById(R.id.mPieChart);
        //以百分比为单位
        mPieChart.setUsePercentValues(true);
        mPieChart.getDescription().setEnabled(false);
        //偏移量
        mPieChart.setExtraOffsets(5, 10, 5, 5);
        //设置饼图旋转的摩擦力
        mPieChart.setDragDecelerationFrictionCoef(0.9f);
        //设置中间值
        mPieChart.setCenterText("我是345");
        //是否将饼心绘制空心
        mPieChart.setDrawHoleEnabled(true);
        //设置饼图中心圆的颜色
        mPieChart.setHoleColor(Color.WHITE);
        //设置饼图中心圆的边颜色
        mPieChart.setTransparentCircleColor(Color.WHITE);
//        设置饼图中心圆的边的透明度，0--255 0表示完全透明
        mPieChart.setTransparentCircleAlpha(110);
        //设置饼图中心圆的半径
        mPieChart.setHoleRadius(30f);
        //设置饼图中心圆的边的半径
        mPieChart.setTransparentCircleRadius(35f);
        //是否在饼图的中心绘制字
        mPieChart.setDrawCenterText(true);
        //设置雷达图旋转的偏移量(以度为单位)。默认270 f
        mPieChart.setRotationAngle(360);
        //设置是否可以触摸旋转
        mPieChart.setRotationEnabled(true);
        //设置是否点击后将对应的区域进行突出
        mPieChart.setHighlightPerTapEnabled(true);
        mPieChart.setDrawSlicesUnderHole(false);
        //为图标设置一个选择监听器
        mPieChart.setOnChartValueSelectedListener(this);
    }

    @Override
    public void onValueSelected(Entry e, Highlight h) {

    }

    @Override
    public void onNothingSelected() {

    }
}

```
> 好了，具体的步骤也说完了，如果有错的地方还请指正，谢谢！