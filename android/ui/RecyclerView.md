## RecyclerView 核心知识点

### 1，RecyclerView是什么

- 为有限的屏幕显示大量的数据且灵活的View，如下图

<img src="D:\android\note-instance\android\ui\RecyclerView.assets\image-20201023224410224.png" alt="image-20201023224410224" style="zoom:50%;" />

- 相比较 ListView

  ListView：

  - 只有纵向列表一种布局
  - 没有支持动画的 API 
  - 接口设计和系统不一致，如 setOnItemClickListener
  - 没有强制实现 ViewHolder
  - 性能不如 RecyclerView

  RecyclerView：

  - 默认支持 Linear，Grid ，Staggered Grid 布局
  - 友好的 ItemAnimator 动画 Api。在刷新的时候调用对应的刷新 api 即可看到动画
  - 强制实现 ViewHolder
  -  RecyclerView 的源码是非常解耦的，且性能非常好

### 2，RecyclerView 中重要的组件

- RecyclerView：一个特殊的 ViewGroup，他本身不会做太多的工作。重要的工作都会交给下面的三个组件来完成
  - LayoutManager：负责布局和摆放 item 
  - ItemAnimator：负责动画
  - Adapter：适配器模式，对数据进行适配，把数据列表转化成 RecyclerView 需要的 ItemViewAdapter

### 3，简单的使用

​	[Demo](https://blog.csdn.net/baidu_40389775/article/details/82933053)

### 4，ViewHoder 究竟是什么

​	ViewHolder 和 item 是一一对应的关系，在创建一个item的时候就会创建一个 ViewHolder，这样当 Item 进行复用的时候就可以直接拿到 ViewHolder，从而防止重复进行 findViewById 。

​	所以说就算你没有使用 ViewHolder，你的 item 还是会被复用，不同的是他会重新进行 findViewById 的操作。

​	ViewHolder 的实践：一般情况下我们是在 onBindViewHolder 方法中绑定数据，但是如果是多个条目，那么这种写法就会非常臃肿，这种情况下就可以吧绑定数据的代码写在 ViewHolder 中。

### 5，RecyclerView 的缓存机制

RecyclerView 中缓存的其实是 ViewHolder。ViewHolder和 item 实际上是绑定的，所以缓存了 ViewHolder 也就相当于缓存了 item。

- 1，Scrap

  屏幕内部的 itemView，可直接进行使用

- 2，Cache

  被滑出的 View 会放在 Cache 中，当用户倒着滑的时候就会直接从 Cache 中获取 viewHolder，避免重复创建 ViewHolder。从Cache 中拿到的缓存可直接进行使用，无需重新创建可绑定数据。

- 3，ViewCacheExtension

  用户自定义的Cache策略，用户如果定义了，就会去里面找缓存，如果没有则直接去下面的缓存池。

- 4，RecyclerViewPool

  缓存池，里面保存的都是被废弃的 ItemView。如果从上面的缓存都没有找到，则就会从 RecyclerViewPoll 中查找
  
  **在 RecyclerViewPoll 中保存的数据都是脏数据，即使在 RecyclerViewPoll  中找到了，虽然不会重新创建 ViewHolder，但是会重新执行onBindView 的操作。这也是 Poll 和前面1和2中不一样的地方。**
  
  如果在上面的 4 级缓存中都没有，则会重新创建 ViewHolder。最终调用的是 onCreateViewHolder，由用户自行创建。

### 6，RecyclerView 中 item 广告的统计

- 在 ListView 中通过 getView() 方法进行统计是没有问题的。每次滑动的时候都会调用 getView() 方法。

- 在 RecyclerView 中 通过 onBindViewHolder() 统计？**可能错误！**

  onBindViewHolde 这个方法不是每次都调用的，有可能你看到了item 10 多次，但是只统计了 5,6次。这种情况下数据就是错误的。

  如何解决呢？

  通过 onViewAttachedToWindow() 统计即可。每看到一次，这个方法就会执行一次

### 7，你可能不知道的 RecyclerView 性能优化策略

- 不要在 onBindViewHolder 方法中创建点击事件

  在创建 ViewHolder 的时候创建 点击事件，如在 new ViewHolder() 或者在 ViewHolder 的初始化方法中创建点击事件即可。

- LinearLayoutManager.setInitialPrefetchltemCount() 方法

  如果是 RecyclerView 嵌套横向的 ReyclerView，当用户滑动的时候，由于需要创建更复杂的 RecyclerView 以及多个子View，可能会导致页面卡顿

  由于 RenderThread 的存在，RecyclerView 会进行 prefetch(RenderThread 是一个专门用于 UI 渲染的线程，把原来很多放在主线程的操作放在了这个线程) 。这样在渲染的时候主线程就会有更多的空闲时间，那么在这个空闲的状态，recyclerView 就可以用来做 prefetch

  setInitialPrefetchltemCount(横向列表初次显示时可见的 item 个数)，调用这个方法后，由于 prefetch，用户在滑动的时候就不会那么卡顿了。

  需要注意的：

  - 只有 LinearLayoutManager 有这个 API 
  - 只有嵌套在内部的 RecyclerView 才会生效

- RecyclerView.setHasFixedSize()

  ```JAVA
  //伪代码
  void onContentsChanged(){
  	if(mHasFixedSize){
  		layoutChildren();
  	}else{
  		requestLayout();
  	}
  }
  ```

  通过上面的伪代码可以看到，如果有固定大小，则直接 layoutChildren，否则 requestLayout()。

  requestLayout() 会让 RecyclerView 重新走一遍绘制流程。

  所以如果 recycleView 的数据是固定的，则可以将此方法设置为 true。

- 多个 RecyclerView 共用 RecycledViewPoll

  注意这个 RecycledViewPool 不是 四级缓存中的 RecyclerViewPool

  RecyclerView 会默认给自己创建一个 RecycledViewPool 

  使用场景：如果是一个 tab 页面，并且有很多个子页面，他们的 item 大致都相同，那么就可以设置一个共享的 RecycledViewPoll，这样就可以提升一定的性能。通过下面代码即可设置

  ```
  RecyclerView.RecycledViewPool pool = new RecyclerView.RecycledViewPool();
  recycler1.setRecyclerViewPool(pool);
  recycler2.setRecyclerViewPool(pool);
  recycler3.setRecyclerViewPool(pool);
  ```

- DiffUtil

  计算两个不同列表的差异，根据计算出的差异输出一段操作，把第一个 list 变成第二个list

  - 局部更新方法：notifyItemXXX() 不适用于所有情况

    有可能你不确定你要更新的 item 是哪个了，那么只能通过 notifyDataSetChange() 进行刷新，这样会导致整个布局重绘，重新绑定所有的 ViewHolder，而且会失去可能的动画效果

  - DiffUtil 适用于整个页面需要刷新，但是有部分数据可能相同的情况。

    DiffUtili.Callback，他是用于给系统计算 diff 的callback

    ```java
    /**
     *一个由DiffUtil在计算两个列表之间的差异时使用的回调类
     */
    public abstract static class Callback {
        /**
         * 旧数据的大小
         */
        public abstract int getOldListSize();
    
        /**
         * 新数据的大小
         */
        public abstract int getNewListSize();
    
        /**
         * 由DiffUtil调用，以确定两个对象是否表示同一项
         * <p>
         * 例如，如果条目具有惟一的id，该方法应该检查它们的id是否相等
         *
         * @param oldItemPosition 旧数据在列表中的位置
         * @param newItemPosition 新数据在列表中的位置
         * @return 如果两项表示同一对象，则为真;如果两项不同，则为假
         */
        public abstract boolean areItemsTheSame(int oldItemPosition, int newItemPosition);
    
        /**
         * 当需要检查两个项是否具有相同的数据时，由DiffUtil调用。DiffUtil使用此信息检测项的内容是否已更改
         * <p>
         * areItemsTheSame 返回true时才会调用此方法，例如，两个 User 的id是一样的，但是他的数据可能发生了变化，所以此方法会被调用。
         * <p>
         * @param oldItemPosition  旧数据在列表中的位置
         * @param newItemPosition 新数据在列表中的位置                   
         * return true 表示这两个列表的数据相同，false 表示数据发生了更改
         */
        public abstract boolean areContentsTheSame(int oldItemPosition, int newItemPosition);
    
        /**
         * 当上面 areItemsTheSame 返回true，areContentsTheSame 返回false时调用
         * <p>
         *	更新数据，如果不实现此方法，则永远也不会有 item 内部的增量更新了
         * <p>
         * Default implementation returns {@code null}.
         *
         * @param oldItemPosition 旧数据在列表中的位置
         * @param newItemPosition 新数据在列表中的位置           
         *
         * @return 一个有效的对象，表示两项之间的更改。
         */
        @Nullable
        public Object getChangePayload(int oldItemPosition, int newItemPosition) {
            Bundle b = new Bundle()
            if(.....){
                b.put("key",value)
            }
            return b;
        }
    }
    ```

    那么如何使用呢？

    ```java
    private List<User> list ;
    public void swapData(List<User> newList,boolean diff){
    	if(diff){
    		//UserDiffCallBack实现类，
    		DiffUtili.DiffResult diffResult = DiffUtil.calculateDiff(new UserDiffCallBack(userList,newList),false)
    		//..... 
    		list = newList
    		//添加回调，this 是当前的 adapter
    		diffResult.dispatchUpdatesTo(this)
    	}else{
    		//更新数据
    		list = newList
    		//刷新数据
    		notifyDataSetChanged()
    	}
    }
    ```

    ```java
    public void onBindViewHolder(@NonNull VH holder, int position,
            @NonNull List<Object> payloads) {
       if(payloads.isEmpty()){
           //默认的，全量更新
           onBindViewHolder(holder,position);
       }else{
           //根据差异计算出来的增量更新
           Bundle payload = (Bundle)payloads.get(0);
           String value =  payload.get("key");
           holder.name.setText(value);
       }
    }
    ```

    一般我们使用的是两个参数的onBindViewHolder，但是有一个三个参数的 onBindViewHolder 方法，通过第三个参数可以判断出来差异，然后在确定是否要使用具体的内容

  - 如果在列表差异很大的时候计算 diff

    - 使用 Thread 将 DiffResult 发送到主线程
    - 使用 RxJava 将 calculateDiff 操作放在后台线程
    - 使用 Google 提供的 AsyncListDiffer(Executor)/ListAdapter(Recycler包下的 ListAdapter，不是平常使用的 adapter)。他把这件事进行了封装。

    这三个方法都做了同一件事，将计算差异放在后台线程执行。

### 8，为什么 ItemDecoration 可以绘制风分割线







