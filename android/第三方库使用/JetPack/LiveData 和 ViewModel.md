### 前言

LiveData 是一种持有可被观察的数据存储类,和其他可被观察的类不同的是，LiveData 是就要生命周期感知能力的，这意味着他可以在 Activity ，fragment 或者 service 生命周期活跃状态时 更新这些组件。

在日常开发过程中，LiveData 已经是必不可少的一环了，例如 `MVVM` 以及 `MVI` 开发模式中，都用到了 LiveData。



### 了解 LiveData

如果观察者(Activity/Fragment) 的生命周期处于 `STARTED` 或者 `RESUMED` 状态，LiveData 就会认为是活跃状态。LiveData 只会将数据更新给活跃的观察者。

在添加观察者的时候，可以传入 `LifecycleOwner` 。有了这关系，当 `Lifecycle` 对象状态为 `DESTORYED` 时，便可以移除这个观察者。

使用 LIveData 具有以下优势：

- 确保界面符合数据状态：数据发生变化时，就会通知观察者。我们可以再观察者回调中更新界面，这样就无需在数据改变后手动更新界面了。
- 没有内存泄漏，因为关联了生命周期，页面销毁后会进行自我清理
- 不会因为Activity 停止而导致崩溃，页面处于非活跃状态时，他不会接收到任何 LiveData 事件
- 数据始终保持最新状态，页面如果变为活跃状态，它会在变为活跃状态时接收最新数据
- 配置更改后也会接收到最新的可用数据
- 共享资源，可以使用单例模式扩展 LiveData 对象，以便在应用中共享他们

### LiveData 的使用

LiveData 是一种可用于任何数据的封装容器，通常 LiveData 存储在 ViewModel 对象中。

```kotlin
class JokesDetailViewModel : ViewModel() {
	//创建 LiveData
    private val _state by lazy { MutableLiveData<JokesUIState>() }
    val state : LiveData<JokesUIState> = _state

    private fun loadChildComment(page: Int, commentId: Int, parentPos: Int, curPos: Int) {
        viewModelScope.launch {
            launchHttp {
                jokesApi.jokesCommentListItem(commentId, page)//请求数据
            }.toData {
                //通知观察者
                _state.value = JokesUIState.LoadMoreChildComment(it.data, parentPos, curPos)
            }
        }
     }
}    
```

```kotlin
//观察 LiveData
viewModel.state.observe(this, Observer {
    //更新 UI 
})
```



### LiveData 实现原理分析

#### 添加观察者
