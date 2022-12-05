### 前言

`Activity` 类是 android 应用的关键组件，在日常开发中，绝对少不了组件。既然用了这么久，你知道他的启动流程🐴？作为一个应用层开发者，大多数人可能觉得学习这些对日常开发可能没有太大帮助。但是多了解一下 framework 的代码还是很有必要的，了解系统组件机制，对于一些问题我们也能快速的定位找到问题的所在点，并且在面试的时候也是一个加分项。

如果你对 Activity 启动流程不懂或者也是一知半解的可以读一下这篇文章，说不定可以学到一些东西。

本文基于 Android 12 版本源码，从 `startActivity` 作为切入点，对整个启动流程进行分析。

### 客户端

众所周知，一般情况下Activity 的启动方式有下面种：

- startActivity(Intent intent)：直接启动一个 Activity
- startActivityForResult(Intent intent, int requestCode)：带返回值的启动方式，这种启动方式已经被官方所废弃，取而代之的是 `registerForActivityResult(contract, mActivityResultRegistry, callback)`
- registerForActivityResult(contract, mActivityResultRegistry, callback)

我们从 startActivity 来一步步往下看：

```java
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
```

```java
public void startActivity(Intent intent, @Nullable Bundle options) {
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        startActivityForResult(intent, -1);
    }
}
```

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}
```

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        ......
        //execStartActivity
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
            //分析启动结果
            mMainThread.sendActivityResult(
                mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                ar.getResultData());
        }
        ......
    } else {
        if (options != null) {
            //这里最终也是调用 execStartActivity 方法
            mParent.startActivityFromChild(this, intent, requestCode, options);
        } else {
            mParent.startActivityFromChild(this, intent, requestCode);
        }
    }
}
```

上面代码中，调用了 execStartActivity 方法，该方法会返回一个启动结果。最下面的的 startActivityFromChild 方法最终也是调用的 execStartActivity。

我们先看一下该方法的参数：

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
}
```

1. Context who ：传入的是 this，用来启动 Activity 的对象

2. Ibinder contextThread：Binder 对象，具有跨进程通信的能力，传入的是 mMainThread.getApplicationThread()

    ```kotlin
    public ApplicationThread getApplicationThread(){
        return mAppThread;
    }
    final ApplicationThread mAppThread = new ApplicationThread();
    
    private class ApplicationThread extends IApplicationThread.Stub {
    	.....
    }
    ```

    ApplicationThread 是 Activitythread 的内部类，就是通过 AIDL 创建的一个远程服务的接口，用来与服务端进行交互，该对象会被传入到 AMS 中，在 AMS 中回保存他的 client（客户端），这样 AMS 就可以与应用进程进行通信了

3.  IBinder token：Binder 对象，指向了服务端一个 ActivityRecord 对象

4. Activity target：当前的 Activity

5. Intent intent, int requestCode, Bundle options ：Intent 对象，请求码和参数。

下面我们来看一下 `execStartActivity` 方法：

```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    //应用端 AIDL 实现类
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    .......
    try {
        intent.migrateExtraStreamToClipData(who);
        intent.prepareToLeaveProcess(who);
        //通过 Binder 调用 AMS 启动 Activity
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getOpPackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

```java
public static IActivityTaskManager getService() {
    return IActivityTaskManagerSingleton.get();
}
//获取单例
@UnsupportedAppUsage(trackingBug = 129726065)
private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
        new Singleton<IActivityTaskManager>() {
            @Override
            protected IActivityTaskManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                return IActivityTaskManager.Stub.asInterface(b);
            }
        };
```

上面代码中通过 `getService` 获取到 Binder 对象，然后将 Binder 转成 AIDL 接口所属的类型，接着就可以调用 AIDL 中的方法与服务端进行通信了。

最后我们可以看到的是调用的是 ATMS 中的 `startActivity()` 方法，我们可以用一张图来表示上述的流程：

### 服务端

通过上面的代码可以看出已经调用到了系统的 ATMS 当中，我们来看一下具体的流程

```java
#ActivityTaskManagerService.java
@Override
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```

```java
#ActivityTaskManagerService.java
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}
```

```java
#ActivityTaskManagerService.java
private int startActivityAsUser(IApplicationThread caller, String callingPackage,
        @Nullable String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");
    // TODO: Switch to user app stacks here.
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setCallingFeatureId(callingFeatureId)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setUserId(userId)
            .execute();
}
```

上面代码最终调用到了 `startActivityAsUser` 方法，在内部将所有点的参数都交给了 ActivityStarter ，该类包含了启动的所有逻辑，比如 Intent 解析以及任务栈等。

接着调用了`obtainStarter` ，该方法通过工厂模式创建了 ActivityStarter 对象，如下所示：

```java
#ActivityStarter.java
static class DefaultFactory implements Factory {
    /**
     * ActivitySatrter 最大数量
     */
    private final int MAX_STARTER_COUNT = 3;
    ......
		//同步池
    private SynchronizedPool<ActivityStarter> mStarterPool =
            new SynchronizedPool<>(MAX_STARTER_COUNT);

    DefaultFactory(ActivityTaskManagerService service,
            ActivityTaskSupervisor supervisor, ActivityStartInterceptor interceptor) {
        mService = service;
        mSupervisor = supervisor;
        mInterceptor = interceptor;
    }

    @Override
    public void setController(ActivityStartController controller) {
        mController = controller;
    }

    @Override
    public ActivityStarter obtain() {
        //从同步池中获取 ActivityStarter 对象
        ActivityStarter starter = mStarterPool.acquire();
        if (starter == null) {
            if (mService.mRootWindowContainer == null) {
                throw new IllegalStateException("Too early to start activity.");
            }
            starter = new ActivityStarter(mController, mService, mSupervisor, mInterceptor);
        }
        return starter;
    }

    @Override
    public void recycle(ActivityStarter starter) {
        starter.reset(true /* clearRequest*/);
        mStarterPool.release(starter);
    }
}
```

可以看到，默认的工厂在提供了一个容量为 3 的同步缓存池来缓存 ActivityStarter 对象，该对象创建完成之后，该对象创建完成之后，AMTS 就会将接下来启动 Activity 的操作交给 ActivityStarter 来完成。

___



```java
#ActivityStarter.java
//根据前面传入的参数解析一下必要的信息，并开始启动 Activity
int execute() {
    try {
        int res;
        synchronized (mService.mGlobalLock) {
            .....
            res = executeRequest(mRequest);//开始执行请求
            .....
            return getExternalResult(res);
        }
    } finally {
        onExecutionComplete();
    }
}
```

```java
#ActivityStarter.java
private int executeRequest(Request request) {
  
    .......
  
  	//检测Activity启动的权限
    boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, 							resultWho,requestCode, callingPid, callingUid, callingPackage, 										callingFeatureId,request.ignoreTargetSecurity, inTask != null, 										callerApp, resultRecord, resultRootTask);
    abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
            callingPid, resolvedType, aInfo.applicationInfo);
    abort |= !mService.getPermissionPolicyInternal().checkStartActivity(intent, callingUid,
            callingPackage);

   
    final ActivityRecord r = new ActivityRecord.Builder(mService)
            .setCaller(callerApp)
            .setLaunchedFromPid(callingPid)
            .setLaunchedFromUid(callingUid)
            .setLaunchedFromPackage(callingPackage)
            .setLaunchedFromFeature(callingFeatureId)
            .setIntent(intent)
            .setResolvedType(resolvedType)
            .setActivityInfo(aInfo)
            .setConfiguration(mService.getGlobalConfiguration())
            .setResultTo(resultRecord)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setComponentSpecified(request.componentSpecified)
            .setRootVoiceInteraction(voiceSession != null)
            .setActivityOptions(checkedOptions)
            .setSourceRecord(sourceRecord)
            .build();

    mLastStartActivityRecord = r;


    mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
            request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
            restrictedBgActivity, intentGrants);

    return mLastStartActivityResult;
}
```

上面代码中回进行一下校验和判断权限等，后面就会创建 `ActivityRecord` ，每个 Activity 都会对应一个 `ActivityRecord` 对象，接着就会调用 `startActivityUnchecked` 继续进行启动流程

```java
#ActivityStarter.java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            boolean restrictedBgActivity, NeededUriGrants intentGrants) {
    int result = START_CANCELED;
    final Task startedActivityRootTask;
    try {
        mService.deferWindowLayout();
        Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
        result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
    } finally {
        Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
        startedActivityRootTask = handleStartResult(r, result);
        mService.continueWindowLayout();
    }

    postStartActivityProcessing(r, result, startedActivityRootTask);

    return result;
}
```

在大多数初步检查已经完成的情况下开始进行下一步确认拥有必要的权限。 上面的核心方法就是 `startActivityInner()` 用来检查启动所必须要有的权限

```java
#ActivityStarter.java
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, boolean doResume, ActivityOptions options, Task inTask,
        boolean restrictedBgActivity, NeededUriGrants intentGrants) {
  	//设置初始化状态
    setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
            voiceInteractor, restrictedBgActivity);
		//判断启动模式，并且在 mLaunchFlags 上追加对应标记
    computeLaunchingTaskFlags();
		//设置 Activity 的栈
    computeSourceRootTask();
		//设置 LaunchFlags 到 intent 上
    mIntent.setFlags(mLaunchFlags);

  	//决定是否应将新活动插入现有任务中。返回null, 如果不是则应将新活动添加到其中的任务进行活动记录
    final Task reusedTask = getReusableTask();

    ......

    //reusedTask 为 null 则计算是否存在可以使用的任务栈
    final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
    //是否需要创建栈  
    final boolean newTask = targetTask == null;
    mTargetTask = targetTask;

    computeLaunchParams(r, sourceRecord, targetTask);

    //检查是否允许在给定任务或新任务上启动活动
    int startResult = isAllowedToStart(r, newTask, targetTask);
    if (startResult != START_SUCCESS) {
        return startResult;
    }

    final ActivityRecord targetTaskTop = newTask
            ? null : targetTask.getTopNonFinishingActivity();
    if (targetTaskTop != null) {
        // Recycle the target task for this launch.
        startResult = recycleTask(targetTask, targetTaskTop, reusedTask, intentGrants);
        if (startResult != START_SUCCESS) {
            return startResult;
        }
    } else {
        mAddingToTask = true;
    }

  
  	//如果正在启动的活动与当前位于顶部的活动相同
    //则需要检查它是否应该只启动一次
    final Task topRootTask = mPreferredTaskDisplayArea.getFocusedRootTask();
    if (topRootTask != null) {
        startResult = deliverToCurrentTopIfNeeded(topRootTask, intentGrants);
        if (startResult != START_SUCCESS) {
            return startResult;
        }
    }
		
    //复用或者创建新栈
    if (mTargetRootTask == null) {
        mTargetRootTask = getLaunchRootTask(mStartActivity, mLaunchFlags, targetTask, mOptions);
    }
    if (newTask) {
      	//新建一个 Task
        final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTask() : null;
        setNewTask(taskToAffiliate);
    } else if (mAddingToTask) {
        //复用之前的 Task
        addOrReparentStartingActivity(targetTask, "adding to task");
    }

 		......
    if (mDoResume) {
			// 调用 resumeFocusedTasksTopActivities方法
      mRootWindowContainer.resumeFocusedTasksTopActivities(
                    mTargetRootTask, mStartActivity, mOptions, mTransientLaunch);
    }
    mRootWindowContainer.updateUserRootTask(mStartActivity.mUserId, mTargetRootTask);
    .....
    return START_SUCCESS;
}
```

在上面方法中，根据启动模式计算出 flag，然后在根据 flag 等条件判断要启动的 Activity 的 ActivityRecord 是需要新创建 Task 栈 还是加入到现有的 Task 栈。

在为 Activity 准备好 Task 栈之后，调用了 mRootWindowContainer.resumeFocuredTasksTopActivities 方法。

___



```java
#RootWindowContainer.java
boolean resumeFocusedTasksTopActivities(
        Task targetRootTask, ActivityRecord target, ActivityOptions targetOptions,
        boolean deferPause) {
    boolean result = false;
    if (targetRootTask != null && (targetRootTask.isTopRootTaskInDisplayArea()
            || getTopDisplayFocusedRootTask() == targetRootTask)) {
        result = targetRootTask.resumeTopActivityUncheckedLocked(target, targetOptions,deferPause);
    }
    return result;
}
```

___



```java
#Task.java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options,boolean deferPause) {
    if (mInResumeTopActivity) {
        // Don't even start recursing.
        return false;
    }

    boolean someActivityResumed = false;
    try {
        // Protect against recursion.
        mInResumeTopActivity = true;
        ....
        someActivityResumed = resumeTopActivityInnerLocked(prev,options,deferPause);
    } finally {
        mInResumeTopActivity = false;
    }

    return someActivityResumed;
}
```

```java
#Task.java
private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options,
        boolean deferPause) {
    if (!mAtmService.isBooting() && !mAtmService.isBooted()) {
        // Not ready yet!
        return false;
    }

    // 在当前 Task 栈中找到最上层正在运行的 Activity
    // 如果这个 Activity 没有获取焦点，那这个 Activity 将会被重新启动
    ActivityRecord next = topRunningActivity(true /* focusableOnly */);


    if (next.attachedToProcess()) {
        ......
    } else {
        ......
        //调用 ActivityTaskSupervisor.startSpecificActivity
        mTaskSupervisor.startSpecificActivity(next, true, true);
    }

    return true;
}
```

___



```java
#ActivityTaskSupervisor.java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // 获取目标进程
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
                //如果进程存在，启动 Activity 并返回
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
               
            }
            knownToBeDead = true;
        }
        r.notifyUnknownVisibilityLaunchedForKeyguardTransition();
        final boolean isTop = andResume && r.isTopRunningActivity();
        // 进程不存在，则创建进程，并启动 Activity
        mService.startProcessAsync(r, knownToBeDead, isTop, isTop ? "top-activity" : "activity");
    }

```

#### 进程存在

如果进程存在，则执行此方法

```java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
        boolean andResume, boolean checkConfig) throws RemoteException {

    // 创建启动 Activity 的事务
    // proc.getThread() 获取的是一个 IApplicationThread 对象
    final ClientTransaction clientTransaction = ClientTransaction.obtain(
              proc.getThread(), r.appToken);

    final boolean isTransitionForward = r.isTransitionForward();
    // 为事务设置 Callback LaunchActivityItem，在客户端时会被调用
    clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent), 											System.identityHashCode(r), r.info,
            mergedConfiguration.getGlobalConfiguration(),
            mergedConfiguration.getOverrideConfiguration(), r.compat,
            r.launchedFromPackage, task.voiceInteractor, proc.getReportedProcState(),
            r.getSavedState(), r.getPersistentSavedState(), results, newIntents,
            r.takeOptions(), isTransitionForward,
            proc.createProfilerInfoIfNeeded(), r.assistToken, 					
            activityClientController,
            r.createFixedRotationAdjustmentsIfNeeded(), r.shareableActivityToken,
            r.getLaunchedFromBubble()));

     // 设置所需的最终状态
     final ActivityLifecycleItem lifecycleItem;
     if (andResume) {
         lifecycleItem = ResumeActivityItem.obtain(isTransitionForward);
     } else {
         lifecycleItem = PauseActivityItem.obtain();
     }
     // 设置生命周期请求
     clientTransaction.setLifecycleStateRequest(lifecycleItem);

     // 执行事务
     mService.getLifecycleManager().scheduleTransaction(clientTransaction);

    return true;
}
```

#### 进程不存在

