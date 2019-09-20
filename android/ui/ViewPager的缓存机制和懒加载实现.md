### **在开发的时候经常使用ViewPager + Fragment ，但是你真的了解他是怎么执行的吗，是时候进行一下总结了。**
#### **1,ViewPager的适配器**
---
 FragementPagerAdapter和FragmentStatePagerAdapter
- FragmentPagerAdapter中，即使fragment不可见了，他的事他实例还是存在于内存中，当			  fragment比较多的时候，会占用较大的内存.
- FragmentStatePagerAdapter,当fragment不可见时，可能会将fragment的实例会销毁,随意内存占会小一些

#### **2,ViewPager 的缓存机制**
- 1. ViwePager 的缓存机制会默认缓存旁边的页面，是为了让页面更加流畅.在缓存旁边页面的时候会执行到onCreateView方法，如果你两个碎片的onCreateView 中都有执行请求数据的时候，旁边的页面也会发送请求，这样就会造成网络请求的一些问题。
- 使用setOffscreenPagerLimit方法可以设置缓存页面的个数，默认为1，就算你传入负数他还是默认缓存旁边页面，也就是1
- 由于ViewPager的默认缓存是1，所以ViewPager中至少会预加载一个页面的数据。如果页面有请求数据或者下载数据的时候，这就会影响当前显示的页面。
- 我们知道在碎片中 只有在onCreateView方法中才可以拿到View 然后加载控件进行一些数据的设置.但是当碎片在ViewPager中的时候，缓存机制会导致旁边的页面中的onCreateView方法执行，这就是我们不想看到的结果。
- Fragment里面有一个方法 setUserVisibleHint(boolean isVisibleToUser)方法，这个方法可以判断当前页面是否对用户可见。
- 还有一点，当ViewPager缓存了旁边的那个碎片之后，那个碎片执行onCreateView之后，他碎片的生命周期会失效。当他被缓存抛弃的时候这个碎片就会被销毁。直到他又一次被缓存或者当他可见的时候才会执行onCreateView方法。

### 3，懒加载实现

---
定义所有碎片的基类，也就是让所有碎片继承这个类，他是一个抽象类

```
public abstract class FragmentFactory extends Fragment {

    private boolean isViewInitFinished;
//当Fragment所在的Activity被启动完成后回调该方法
    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);
        isViewInitFinished = true;
    }

// 当前页面是否可见
    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);
        if (isVisibleToUser && isViewInitFinished){
            getData();
        }
    }
    public abstract void getData();
}
```
- 让ViewPager里面要显示的碎片继承这个类并实现他的方法，在这个方法里面就可以去执行你的逻辑了。记住当实现懒加载的时候就不要在onCreateView方法执行逻辑了。当碎片加载的时候会调用父类的setUserVisibleHint方法。后判断是否是当前页面，如果是就可以在重新父类的方法中进行操作了。
- SetUserVisibleHint方法是执行在最前面的，他会在ViewPager里面Fragment生命周期还没执行的时候调用.
-  ==请注意SetUserVisibleHint方法调用的次数，在刚开始的时候会根据你设置的ViewPager的缓存个数去调用SetUserVisibleHint方法，然后Fragment显示后会在调用一次SetUserVisibleHint方法。也就是说在Fragment生命周期之前ViewPager会根据缓存个数去调用SetUserVisibleHint方法。当调用完成后，Fragment的生命周期会开始，这个时候Viewpager就会再次调用SetUserVisibleHint方法判断当前的fragment是否被显示。所以在SetUserVisibleHint中判断是否是当前页面的时候 都要判断一下 在onActivityCreated中的布尔变量是否为真，必须在这个布尔为真的时候才可以去判断当前Fragment有没有显示。==


---
#### **这种懒加载在官方文档中给出了明确的说明**

```
/**
 * Created by hjm on 2017/4/13 15:40.
 */

public abstract class BaseFragment extends Fragment {

    private boolean isViewInitFinished;

    @Override
    public void onActivityCreated(@Nullable Bundle savedInstanceState) {
        super.onActivityCreated(savedInstanceState);

        isViewInitFinished = true;
    }

    @Override
    public void setUserVisibleHint(boolean isVisibleToUser) {
        super.setUserVisibleHint(isVisibleToUser);

        requestData(isVisibleToUser);
    }

    public void requestData(boolean isVisibleToUser){
        if(isViewInitFinished && isVisibleToUser){
            getData();
        }
    }

    /**
     *  this moment the fragment is visiable to user, so request data
##### #### #### #### #### **     */**
    public abstract void getData();
}
```
> 本人也是新手，如有错误请指出，谢谢！