解决什么问题？

学习源码的思想？

有些代码跳过，看不懂就不要看，发现跑偏了就回来。

先泛读流程，在抓细节(怎么开启任务栈已经怎么管理任务栈，启动默认怎么处理的，activity 的实例怎么创建的activity 的生命周期怎么管理的)。



Activity 启动流程

![hot](Activity%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B.assets/hot.jpg)

------

入口

```java
@Override
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
@Override
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

 public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
          // ......
    }
```

​	调用 Instrumentation 的 execStartActivity 方法

```java
public ActivityResult execStartActivity(
    Context who, IBinder contextThread, IBinder token, String target,
    Intent intent, int requestCode, Bundle options) {
    //......
    try {
        intent.migrateExtraStreamToClipData();
        intent.prepareToLeaveProcess(who);
        int result = ActivityTaskManager.getService()
            .startActivity(whoThread, who.getBasePackageName(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()),
                    token, target, requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```

​	调用 ActivityTaskManager 获取服务，然后调用 startActivity，下面看一下获取服务

```java
/** @hide */
public static IActivityTaskManager getService() {
    return IActivityTaskManagerSingleton.get();
}

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

​	看到 IActivityTaskManager.Stub.asInterface(b) 就可以知道是在这里获取的服务了。

​	那么服务类到底是那个呢？其实就是 ActivityManagerService 

​	上面获取完服务调用的是 服务的 startActivity，接着往下看

```java
@Override
public int startActivity(IApplicationThread caller, String callingPackage,
        Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
    return mActivityTaskManager.startActivity(caller, callingPackage, intent, resolvedType,resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions);
}
```

​	这个 mActivityTaskManager 就是 服务端 AIDL 的实现，如下

```java
public class ActivityTaskManagerService extends IActivityTaskManager.Stub {
    
    int startActivityAsUser(IApplicationThread caller, String callingPackage,
            Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,int startFlags, ProfilerInfo profilerInfo, Bundle bOptions, int userId,
            boolean validateIncomingUser) {
        // ......
        return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
                .setCaller(caller)
                .setCallingPackage(callingPackage)
                .setResolvedType(resolvedType)
                .setResultTo(resultTo)
                .setResultWho(resultWho)
                .setRequestCode(requestCode)
                .setStartFlags(startFlags)
                .setProfilerInfo(profilerInfo)
                .setActivityOptions(bOptions)
            	// 这个方法中会执行   mRequest.mayWait = true;
                .setMayWait(userId)
                .execute();
    }
}
```

​	最终会调到 startActivityAsUser ，然后就会执行 execute 方法

```java
int execute() {
    try {
        // 上面在执行 setMayWait 方法时将  mayWait 设置为了 true，所以这里直接进入
        if (mRequest.mayWait) {
            return startActivityMayWait(mRequest.caller, mRequest.callingUid,
                    mRequest.callingPackage, mRequest.realCallingPid, mRequest.realCallingUid,
                    mRequest.intent, mRequest.resolvedType,
                    mRequest.voiceSession, mRequest.voiceInteractor, mRequest.resultTo,
                    mRequest.resultWho, mRequest.requestCode, mRequest.startFlags,
                    mRequest.profilerInfo, mRequest.waitResult, mRequest.globalConfig,
                    mRequest.activityOptions, mRequest.ignoreTargetSecurity, mRequest.userId,
                    mRequest.inTask, mRequest.reason,
                    mRequest.allowPendingRemoteAnimationRegistryLookup,
                    mRequest.originatingPendingIntent, mRequest.allowBackgroundActivityStart);
        } else {
          //......
        }
    } finally {
        onExecutionComplete();
    }
}
```

​	上面调用 startActivityMayWait 如下：

```java
private int startActivityMayWait(IApplicationThread caller, int callingUid,
        String callingPackage, int requestRealCallingPid, int requestRealCallingUid,
        Intent intent, String resolvedType, IVoiceInteractionSession voiceSession,
        IVoiceInteractor voiceInteractor, IBinder resultTo, String resultWho, int requestCode,
        int startFlags, ProfilerInfo profilerInfo, WaitResult outResult,
        Configuration globalConfig, SafeActivityOptions options, boolean ignoreTargetSecurity,
        int userId, TaskRecord inTask, String reason,
        boolean allowPendingRemoteAnimationRegistryLookup,
        PendingIntentRecord originatingPendingIntent, boolean allowBackgroundActivityStart) {
  //......
	
    //解析 intent ，返回一个 ResolveInfo
    ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
            0 /* matchFlags */,
                    computeResolveFilterUid(
                            callingUid, realCallingUid, mRequest.filterCallingUid));
    if (rInfo == null) {
        UserInfo userInfo = mSupervisor.getUserInfo(userId);
        if (userInfo != null && userInfo.isManagedProfile()) {
            // Special case for managed profiles, if attempting to launch non-cryto aware
            // app in a locked managed profile from an unlocked parent allow it to resolve
            // as user will be sent via confirm credentials to unlock the profile.
            UserManager userManager = UserManager.get(mService.mContext);
            boolean profileLockedAndParentUnlockingOrUnlocked = false;
            long token = Binder.clearCallingIdentity();
            try {
                UserInfo parent = userManager.getProfileParent(userId);
                profileLockedAndParentUnlockingOrUnlocked = (parent != null)
                        && userManager.isUserUnlockingOrUnlocked(parent.id)
                        && !userManager.isUserUnlockingOrUnlocked(userId);
            } finally {
                Binder.restoreCallingIdentity(token);
            }
            if (profileLockedAndParentUnlockingOrUnlocked) {
                rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId,
                        PackageManager.MATCH_DIRECT_BOOT_AWARE
                                | PackageManager.MATCH_DIRECT_BOOT_UNAWARE,
                        computeResolveFilterUid(
                                callingUid, realCallingUid, mRequest.filterCallingUid));
            }
        }
    }
    // Collect information about the target of the Intent.
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

    synchronized (mService.mGlobalLock) {
        final ActivityStack stack = mRootActivityContainer.getTopDisplayFocusedStack();
        stack.mConfigWillChange = globalConfig != null
                && mService.getGlobalConfiguration().diff(globalConfig) != 0;
        if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                "Starting activity when config will change = " + stack.mConfigWillChange);

        final long origId = Binder.clearCallingIdentity();

        if (aInfo != null &&
                (aInfo.applicationInfo.privateFlags
                        & ApplicationInfo.PRIVATE_FLAG_CANT_SAVE_STATE) != 0 &&
                mService.mHasHeavyWeightFeature) {
            // This may be a heavy-weight process!  Check to see if we already
            // have another, different heavy-weight process running.
            if (aInfo.processName.equals(aInfo.applicationInfo.packageName)) {
                final WindowProcessController heavy = mService.mHeavyWeightProcess;
                if (heavy != null && (heavy.mInfo.uid != aInfo.applicationInfo.uid
                        || !heavy.mName.equals(aInfo.processName))) {
                    int appCallingUid = callingUid;
                    if (caller != null) {
                        WindowProcessController callerApp =
                                mService.getProcessController(caller);
                        if (callerApp != null) {
                            appCallingUid = callerApp.mInfo.uid;
                        } else {
                            Slog.w(TAG, "Unable to find app for caller " + caller
                                    + " (pid=" + callingPid + ") when starting: "
                                    + intent.toString());
                            SafeActivityOptions.abort(options);
                            return ActivityManager.START_PERMISSION_DENIED;
                        }
                    }

                    IIntentSender target = mService.getIntentSenderLocked(
                            ActivityManager.INTENT_SENDER_ACTIVITY, "android",
                            appCallingUid, userId, null, null, 0, new Intent[] { intent },
                            new String[] { resolvedType }, PendingIntent.FLAG_CANCEL_CURRENT
                                    | PendingIntent.FLAG_ONE_SHOT, null);

                    Intent newIntent = new Intent();
                    if (requestCode >= 0) {
                        // Caller is requesting a result.
                        newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_HAS_RESULT, true);
                    }
                    newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_INTENT,
                            new IntentSender(target));
                    heavy.updateIntentForHeavyWeightActivity(newIntent);
                    newIntent.putExtra(HeavyWeightSwitcherActivity.KEY_NEW_APP,
                            aInfo.packageName);
                    newIntent.setFlags(intent.getFlags());
                    newIntent.setClassName("android",
                            HeavyWeightSwitcherActivity.class.getName());
                    intent = newIntent;
                    resolvedType = null;
                    caller = null;
                    callingUid = Binder.getCallingUid();
                    callingPid = Binder.getCallingPid();
                    componentSpecified = true;
                    rInfo = mSupervisor.resolveIntent(intent, null /*resolvedType*/, userId,
                            0 /* matchFlags */, computeResolveFilterUid(
                                    callingUid, realCallingUid, mRequest.filterCallingUid));
                    aInfo = rInfo != null ? rInfo.activityInfo : null;
                    if (aInfo != null) {
                        aInfo = mService.mAmInternal.getActivityInfoForUser(aInfo, userId);
                    }
                }
            }
        }

        final ActivityRecord[] outRecord = new ActivityRecord[1];
        int res = startActivity(caller, intent, ephemeralIntent, resolvedType, aInfo, rInfo,
                voiceSession, voiceInteractor, resultTo, resultWho, requestCode, callingPid,
                callingUid, callingPackage, realCallingPid, realCallingUid, startFlags, options,
                ignoreTargetSecurity, componentSpecified, outRecord, inTask, reason,
                allowPendingRemoteAnimationRegistryLookup, originatingPendingIntent,
                allowBackgroundActivityStart);

        Binder.restoreCallingIdentity(origId);

        if (stack.mConfigWillChange) {
            // If the caller also wants to switch to a new configuration,
            // do so now.  This allows a clean switch, as we are waiting
            // for the current activity to pause (so we will not destroy
            // it), and have not yet started the next activity.
            mService.mAmInternal.enforceCallingPermission(android.Manifest.permission.CHANGE_CONFIGURATION,
                    "updateConfiguration()");
            stack.mConfigWillChange = false;
            if (DEBUG_CONFIGURATION) Slog.v(TAG_CONFIGURATION,
                    "Updating to new configuration after starting activity.");
            mService.updateConfigurationLocked(globalConfig, null, false);
        }

        // Notify ActivityMetricsLogger that the activity has launched. ActivityMetricsLogger
        // will then wait for the windows to be drawn and populate WaitResult.
        mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(res, outRecord[0]);
        if (outResult != null) {
            outResult.result = res;

            final ActivityRecord r = outRecord[0];

            switch(res) {
                case START_SUCCESS: {
                    mSupervisor.mWaitingActivityLaunched.add(outResult);
                    do {
                        try {
                            mService.mGlobalLock.wait();
                        } catch (InterruptedException e) {
                        }
                    } while (outResult.result != START_TASK_TO_FRONT
                            && !outResult.timeout && outResult.who == null);
                    if (outResult.result == START_TASK_TO_FRONT) {
                        res = START_TASK_TO_FRONT;
                    }
                    break;
                }
                case START_DELIVERED_TO_TOP: {
                    outResult.timeout = false;
                    outResult.who = r.mActivityComponent;
                    outResult.totalTime = 0;
                    break;
                }
                case START_TASK_TO_FRONT: {
                    outResult.launchState =
                            r.attachedToProcess() ? LAUNCH_STATE_HOT : LAUNCH_STATE_COLD;
                    // ActivityRecord may represent a different activity, but it should not be
                    // in the resumed state.
                    if (r.nowVisible && r.isState(RESUMED)) {
                        outResult.timeout = false;
                        outResult.who = r.mActivityComponent;
                        outResult.totalTime = 0;
                    } else {
                        final long startTimeMs = SystemClock.uptimeMillis();
                        mSupervisor.waitActivityVisible(
                                r.mActivityComponent, outResult, startTimeMs);
                        // Note: the timeout variable is not currently not ever set.
                        do {
                            try {
                                mService.mGlobalLock.wait();
                            } catch (InterruptedException e) {
                            }
                        } while (!outResult.timeout && outResult.who == null);
                    }
                    break;
                }
            }
        }

        return res;
    }
}
```

​	

