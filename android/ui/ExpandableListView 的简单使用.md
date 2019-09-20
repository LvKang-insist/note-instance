> 简单的来说，它是一种用于滚动的展示的两级列表视图。也就是ListView的二级选择加强版这种。一种用于垂直滚动展示两级列表的视图，和 ListView 的不同之处就是它可以展示两级列表，分组可以单独展开显示子选项。这些选项的数据是通过 ExpandableListAdapter 关联的。 


>  这个 ExpandableListAdapter 又是什么呢？和 ListView 使用的 BaseAdapter 差不多，都是用来给 View 提供数据、 实例化子布局的。

#### **实现过程:**

---
#### 1. 创建布局
```
<ExpandableListView
        android:id="@+id/frat2_expand_list"
        android:layout_width="match_parent"
        android:layout_height="match_parent">
    </ExpandableListView>
```
一级列表

```
<?xml version="1.0" encoding="utf-8"?>
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/frag2_expand_parent_item"
    android:layout_width="match_parent"
    android:layout_height="50dp"
    android:gravity="center_vertical"
    android:paddingBottom="8dp"
    android:paddingLeft="32dp"
    android:textSize="25dp"
    android:text="  "
    android:textColor="#000"
    tools:ignore="HardcodedText,RtlHardcoded,RtlSymmetry,SpUsage" />
```
二级列表

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="50dp"
    android:paddingTop="8dp"
    android:paddingBottom="8dp"
    android:orientation="horizontal">

    <RelativeLayout
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1">

        <ImageView
            android:id="@+id/frag2_child_image"
            android:layout_width="40dp"
            android:layout_height="40dp"
            android:src="@drawable/bus_2"
            android:layout_marginLeft="100dp"/>

        <TextView
            android:id="@+id/frag2_child_tv1"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:text="1号（101人）"
            android:textSize="25sp"
            android:layout_toRightOf="@id/frag2_child_image"/>

        <TextView
            android:id="@+id/frag2_child_tv2"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:layout_toRightOf="@id/frag2_child_tv1"
            android:text="5分钟到达"
            android:textSize="25sp"
            android:layout_marginLeft="50dp"
            />
    </RelativeLayout>
    <LinearLayout
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:gravity="center"
        android:layout_weight="1">
        <TextView
            android:id="@+id/frag2_child_tv3"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:text="距站台1000米"
            android:textSize="25sp"/>
    </LinearLayout>
</LinearLayout>
```

---

#### 2. 加载控件，编写要显示数据到的base类
```
public class Frag2_Data {
    private String stand;
    private String carCode;
    private String Per_num;
    private String time;
    private String distance;
    public Frag2_Data(String stand, String carCode, String per_num, String time, String distance) {
        this.stand = stand;
        this.carCode = carCode;
        Per_num = per_num;
        this.time = time;
        this.distance = distance;
    }
    public String getStand() {
        return stand;
    }
    public void setStand(String stand) {
        this.stand = stand;
    }
    public String getCarCode() {
        return carCode;
    }
    public void setCarCode(String carCode) {
        this.carCode = carCode;
    }
    public String getPer_num() {
        return Per_num;
    }
    public void setPer_num(String per_num) {
        Per_num = per_num;
    }
    public String getTime() {
        return time;
    }
    public void setTime(String time) {
        this.time = time;
    }
    public String getDistance() {
        return distance;
    }
    public void setDistance(String distance) {
        this.distance = distance;
    }
}
```
#### 3，设置数据
```
private void init(View view) {
		expandableListView = view.findViewById(R.id.frat2_expand_list);
		expandableListView.setAdapter(new Frag2_ExpandableAdapter(getContext(),maps));
	}
private void getData() {
	for (int i = 0; i <= 1; i++) {
		Frag2_Data frag2Data = new Frag2_Data("中医院站",
				(i+1)+"号","("+101+"人)",
				6+"分钟到达","距离站台"+(1000+i*5)+"米");
		map.put(i,frag2Data);
	}
	maps.put(0,map);
	map = new HashMap<>();
	for (int i = 0; i <= 1; i++) {
		Frag2_Data frag2Data = new Frag2_Data("联想大厦站",
				(i+1)+"号","("+101+"人)",
				6+"分钟到达","距离站台"+(8000-i*8)+"米");
		map.put(i,frag2Data);
	}
	maps.put(1,map);
}
	
```
设置了两个Map，map里面存的是 对象，第二个maps里面存的是第一个map，这每个maps里面的map都对应着一个 数据项.

---
#### 4,设置适配器
```
package com.mad.trafficclient.adapter;

import android.annotation.SuppressLint;
import android.content.Context;
import android.util.Log;
import android.view.View;
import android.view.ViewGroup;
import android.widget.BaseExpandableListAdapter;
import android.widget.TextView;
import com.mad.trafficclient.R;
import com.mad.trafficclient.base.Frag2_Data;
import java.util.Map;

public class Frag2_ExpandableAdapter extends BaseExpandableListAdapter {

    private Context context ;
    Map<Integer,Map<Integer,Frag2_Data> > maps;

    Map<Integer,Frag2_Data> map ;

    public Frag2_ExpandableAdapter(Context context, Map<Integer,Map<Integer,Frag2_Data>> maps){
        this.maps = maps;
    }

    //获取分组的个数
    @Override
    public int getGroupCount() {
        return maps.size();
    }
    //获取分组中每个子选项的 个数
    @Override
    public int getChildrenCount(int groupPosition) {
        return maps.get(groupPosition).size();
    }

    //获取指定分组的数据
    @Override
    public Object getGroup(int groupPosition) {
        return maps.get(groupPosition);
    }

    //获取指定分组的指定子选项数据
    @Override
    public Object getChild(int groupPosition, int childPosition) {
        return maps.get(groupPosition).get(childPosition);
    }

    //获取指定分组的ID，这个ID必须是唯一的
    @Override
    public long getGroupId(int groupPosition) {
        return groupPosition;
    }

    //获取子选项的ID ,这个ID必须是唯一的
    @Override
    public long getChildId(int groupPosition, int childPosition) {
        return childPosition;
    }

    //分组和子选项 是否持有稳定的ID，就是说底层数据的改变会不会影响他们
    @Override
    public boolean hasStableIds() {
        return true;
    }
    /**
     *获取显示指定组 的视图对象
     * @param groupPosition 组的位置
     * @param isExpanded 该组是展开还是伸缩状态
     * @param convertView 重用已有的 视图对象
     * @param parent 返回的视图对象始终依附于 的视图组
     * @return 返回View
     */
    //获取显示指定分组的 视图
    @Override
    public View getGroupView(int groupPosition, boolean isExpanded, View convertView, ViewGroup parent) {

        if (convertView == null){
            convertView = View.inflate(parent.getContext(), R.layout.frag2_expand_partent_item,null);
        }
        TextView textView = convertView.findViewById(R.id.frag2_expand_parent_item);
        textView.setText(maps.get(groupPosition).get(0).getStand());
        return convertView;
    }
    /**
     *获取一个视图对象，显示指定组中的指定元素数据
     * @param groupPosition 组的位置
     * @param childPosition 子元素的位置
     * @param isLastChild  子元素是否处于组中的最后一个
     * @param convertView 重用已有的视图View
     * @param parent 返回的视图对象始终依附于视图组
     * @return 返回View
     */
//  获取的显示给定分组给定子位置的数据用的视图
    @SuppressLint("SetTextI18n")
    @Override
    public View getChildView(int groupPosition, int childPosition, boolean isLastChild, View convertView, ViewGroup parent) {

        map = maps.get(groupPosition);
        if (convertView == null){
            convertView = View.inflate(parent.getContext(),R.layout.frag2_expandable_child_item,null);
        }

        TextView tv1 = convertView.findViewById(R.id.frag2_child_tv1);
        TextView tv2 = convertView.findViewById(R.id.frag2_child_tv2);
        TextView tv3 = convertView.findViewById(R.id.frag2_child_tv3);

        tv1.setText(map.get(childPosition).getCarCode()+""
                +map.get(childPosition).getPer_num());
        tv2.setText(map.get(childPosition).getTime());
        tv3.setText(map.get(childPosition).getDistance());
        return convertView;
    }

    //指定位置上的子元素 是否可选中
    @Override
    public boolean isChildSelectable(int groupPosition, int childPosition) {
        return true;
    }
}

```
适配器里面方法的作用已经加了注释。其实不难理解。主要就是给适配器传递数据的时候，一定要注意设置数据的方式。每一个标题对应着好几个子选项。我装数据的方式是使用两个map集合，先把子选项据的数据装好，然后把这个map集合给第二个maps。第二个maps里面对应着多个map.这样一个maps里面装的每个map就对应着一个子选项。

---
当然了，装数据也有好多种办法。这里就看你自己的习惯了。
> 如有错误，还请指出，谢谢！
