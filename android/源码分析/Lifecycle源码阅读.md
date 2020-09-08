### Lifecycle

生命周期感知型组件，可执行操作来响应另一个组件，（如 Activity，Fragment）的生命周期变化。这些组件有助于开发者写出更有条理且往往精简的代码。

由于 lifecycle 已经在 AndroidX 中集成，所以只要在 AndroidX 的项目下，就可以直接获取到 lifecycle

#### 1，查看 Lifecycle 类

​		在 activity 中可以通过 getLifecycle 直接调用到 lifecycle。这里获取到的是 lifecycle 的订阅类，如下：

```kotlin
private final LifecycleRegistry mLifecycleRegistry = new LifecycleRegistry(this);

@NonNull
@Override
public Lifecycle getLifecycle() {
    return mLifecycleRegistry;
}
```

​	这些代码是写在 ComponentActivity 类中的，这个类是活动的基类



#### 2，ComponentActivity类

该类实现了 LifecycleOwner，下面看一下他的 onCreate 方法：

```kotlin
protected void onCreate(@Nullable Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ReportFragment.injectIfNeededIn(this);
}
```

其中 ReportFragment 就是一个 Fragment 类，是用来分派初始化事件的内部类

injectIfNeededIn 方法就是给 ComponentActivity 添加了一个 fragment。下面看一下是如何进行分派事件的

```kotlin
@Override
public void onActivityCreated(Bundle savedInstanceState) {
    super.onActivityCreated(savedInstanceState);
    dispatchCreate(mProcessListener);
    dispatch(Lifecycle.Event.ON_CREATE);
}

@Override
public void onStart() {
    super.onStart();
    dispatchStart(mProcessListener);
    dispatch(Lifecycle.Event.ON_START);
}

@Override
public void onResume() {
    super.onResume();
    dispatchResume(mProcessListener);
    dispatch(Lifecycle.Event.ON_RESUME);
}

@Override
public void onPause() {
    super.onPause();
    dispatch(Lifecycle.Event.ON_PAUSE);
}

@Override
public void onStop() {
    super.onStop();
    dispatch(Lifecycle.Event.ON_STOP);
}

@Override
public void onDestroy() {
    super.onDestroy();
    dispatch(Lifecycle.Event.ON_DESTROY);
}

private void dispatch(Lifecycle.Event event) {
    Activity activity = getActivity();
 
    if (activity instanceof LifecycleRegistryOwner) {
        ((LifecycleRegistryOwner) activity).getLifecycle().handleLifecycleEvent(event);
        return;
    }

    if (activity instanceof LifecycleOwner) {
        Lifecycle lifecycle = ((LifecycleOwner) activity).getLifecycle();
        if (lifecycle instanceof LifecycleRegistry) {
            ((LifecycleRegistry) lifecycle).handleLifecycleEvent(event);
        }
    }
}
```

这些对应着生命周期的方法都调用了 dispatch 方法，接着就会判断 activity 是否实现了 LifecycleOwner ，接着就会获取到对应的 lifecycle。

接着就会判断 lifecycle 是否是 LifecycleRegistry，如果是就**调用 handleLifecycleEvent 方法设置当前的状态，并且通知对应的观察者。**

其实这里的 lifecycle 肯定是 LifecycleRegistry。应为 getLifecycle 获取到的就是 LifecycleRegistry，这个在文章的开始就说过了。

通过上面这些代码就可以知道生命周期的事件是如何发送出去的了。那么观察者是如果添加的呢，这就需要看一下LifecycleRegistry 类了



#### 3，LifecycleRegistry 类

​	LifecycleRegistry 是一个订阅类，并且继承自 Lifecycle 抽象类

​	Lifecycle 抽象类中包括：添加观察者，取消观察者，获取当前生命周期的状态的抽象方法，以及对应生命周期的枚举，和 当前状态枚举

​	接着看一下 LifecycleRegistry： 一个可以处理多个观察者的「生命周期」的实现。看一下其中比较重要的方法：

​	**添加观察者**

```kotlin
    private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap = new FastSafeIterableMap<>();
		

		@Override
    public void addObserver(@NonNull LifecycleObserver observer) {
        //如果当前状态是销毁状态，则设置为销毁状态，否则为初始化状态
        State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
        //将当前的状态 和 观察者进行封装，
        ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
        //以观察者为 key 的方式，保存 statefulObserver
        //如果已存在，在返回存在的 VALUE，否则进行保存，返回 null
        ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);
				//不为空，则return
        if (previous != null) {
            return;
        }

        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            // it is null we should be destroyed. Fallback quickly
            return;
        }
			  //mAddingObserverCounter 表示正在添加观察者的数量
        boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
       //获取到上一个的状态
        State targetState = calculateTargetState(observer);
        mAddingObserverCounter++;
       //循环，将新添加的观察者的状态更新到指定的 targetState
        while ((statefulObserver.mState.compareTo(targetState) < 0
                && mObserverMap.contains(observer))) {
          	//将状态保存到 mParentState 中
            pushParentState(statefulObserver.mState);
            statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
            popParentState();
            // mState / subling may have been changed recalculate
            targetState = calculateTargetState(observer);
        }
		
      	//没有事件正在处理，或者是添加了 observer ，则需要同步一次状态
        if (!isReentrance) {
            // we do sync only on the top level.
            sync();
        }
        mAddingObserverCounter--;
    }
//计算目标状态
 private State calculateTargetState(LifecycleObserver observer) {
   		// 拿到当前 observer 的上一个的 observer
        Entry<LifecycleObserver, ObserverWithState> previous = mObserverMap.ceil(observer);
				//获取到上一个 observer 的状态
        State siblingState = previous != null ? previous.getValue().mState : null;
   			//如果父级状态不为 null，则获取他的状态
        State parentState = !mParentStates.isEmpty() ? mParentStates.get(mParentStates.size() - 1)
                : null;
   		//获取到最小状态
        return min(min(mState, siblingState), parentState);
    }
```

**删除观察者**

```java
@Override
public void removeObserver(@NonNull LifecycleObserver observer) {
	  //移除对应的 observer
    mObserverMap.remove(observer);
}
```

**获取当前状态**

```
@Override
public State getCurrentState() {
    return mState;
}
```

更新状态  **handleLifecycleEvent 方法** 

通过 ReportFragment 类可以知道状态改变后最终调用的是 handleLifecycleEvent 方法：

```java
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
		//拿到下一个状态
    State next = getStateAfter(event);
    moveToState(next);
}

private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
  	//如果正在处理上一次的事件，就标记 mNewEventOccurred 为 true，然后退出
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
  //更新正在处理的状态为 true
    mHandlingEvent = true;
    sync();
  //处理结束，false
    mHandlingEvent = false;
}

private void sync() {
        LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
        if (lifecycleOwner == null) {
            throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                    + "garbage collected. It is too late to change lifecycle state.");
        }
        while (!isSynced()) {
            mNewEventOccurred = false;
            // no need to check eldest for nullability, because isSynced does it for us.
            if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
                backwardPass(lifecycleOwner);
            }
            Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
            if (!mNewEventOccurred && newest != null
                    && mState.compareTo(newest.getValue().mState) > 0) {
                forwardPass(lifecycleOwner);
            }
        }
        mNewEventOccurred = false;
    }
```

