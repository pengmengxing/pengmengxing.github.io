---
layout: post
title:  Activity启动流程
date:   2022-11-12 00:00:00 +0800
categories: document
tag: 教程
---


* content
  {:toc}


# Activity的启动流程

**基于Android T**

参考：https://juejin.cn/post/6883088187430797319#heading-3

# 1.根Activity的创建

这里可以理解为是从桌面点击图标首次启动应用的activity，整体流程图可参考processon

桌面是一个进程，也就是一个Launcher进程，是一个系统app。当点击桌面app时，会向AMS(system_server)进程请求启动一个activity，如果此时app进程没有启动，system_server进程会先通过Zygote去fork一个进程，然后返回到(system_server)进程去请求app进程创建一个activity。

桌面是一个进程，也就是一个Launcher进程，是一个系统app。当点击桌面app时，会向AMS(system_server)进程请求启动一个activity，如果此时app进程没有启动，system_server进程会先通过Zygote去fork一个进程，然后返回到(system_server)进程去请求app进程创建一个activity。

## 1.1 主要涉及四个进程

对于一个初次启动的app（进程未创建），主要涉及以下四个进程

1. Launcher进程，桌面进程
2. SystemServer进程，AMS所在进程
3. Zygote进程，孵化器，用于fork进程
4. App进程，应用程序进程，即将启动的进程

## 1.2 主要流程

1. Launcher进程向AMS请求创建一个Activity
2. AMS向Zygote请求创建一个app进程
3. Zygote通过fork创建一个进程，通知AMS创建完成
4. AMS通知App进程去创建根Activity

# 2.普通Activity创建

对于一个已经启动过的进程（不需要再重新去fork一个进程），主要涉及以下两个进程，这里主要是指应用内部启动或者两个app直接跳转。（非从桌面点击图标的热启动）。

普通Activity的创建在代码中通常是startActivity(Intent intent)的方式。

## 2.1 主要涉及两个进程

App进程，指当前启动的app所在的进程

SystemServer进程，为所有应用共用的服务进程

## 2.2 主要流程

1. App进程中activity通过startActivity向Instrumentation请求创建一个Activity
2. Instrumentation通过ATMS获取到app进程的Ibinder接口，通过Binder调用到SystemServer(AMS)进程（AIDL）
3. AMS对app侧传入的intent，ActivityOptions等参数进行一系列的处理
4. 系统侧通过ClientLifecycleManager获取到app进程在systemserver进程中的Binder接口IapplicationThread，访问到app进程的ActivityThread中的ApplicationThead
5. ApplicationThead接收到服务端传递过来的事务后，交由ActivityThread进行处理，再交由其父类ClientTransactionHandler去sendMessage
6. ActivityThread通过Instrumentation利用类加载器创建一个Activity，并利用mInstrumentation去回调activity的生命周期

# 3.应用内Activity启动流程

日志可参考后面 “应用内启动其他activity堆栈信息”

## 3.1 Activity请求AMS的过程(App进程)



这里通常是app侧代码中startActivity(intent)的过程

```java
protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button btn = findViewById(R.id.btn);
        btn.setOnClickListener(this);
    }
    
    public void onClick(View v) {
        switch (v.getId())
        {
            case R.id.btn:
                SwitchToApp1();
                break;
            default:
                break;
        }
    }

    private void SwitchToApp1() {
        Intent intent = new Intent();
        ComponentName cn = new ComponentName("com.android.browser",
                "com.android.browser.BrowserActivity");
        intent.setComponent(cn);
        startActivity(intent);
    }
```

### 3.1.1 Activity.startActivity

```java
//路径：core/java/android/app/Activity.java
   @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        //最终都会跳转到startActivityForResult中去，两者的区别就是是否传入options参数
        getAutofillClientController().onStartActivity(intent, mIntent);
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

```



### 3.1.2  Activity.startActivityForResult

```java
//路径：core/java/android/app/Activity.java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
            //通过mInstrumentation.execStartActivity启动Activity
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```



### 3.1.3 Instrumentation.execStartActivity

```java
//路径:core/java/android/app/Instrumentation.java
public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        android.util.SeempLog.record_str(377, intent.toString());
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
		
        ......
            
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
            //调用到systemserver进程中ATMS的startActivity
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getOpPackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
            notifyStartActivityResult(result, options);
            //检查启动结果，启动失败则抛出异常
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```



### 3.1.4 IActivityTaskManager.startActivity

这里是通过AIDL进行跨进程通信，Binder是android中独有的一种进程间通信的方式，底层依靠mmap，只需要一次数据拷贝，把物理内存同时映射到内核空间和进程所在的用户空间。AIDL的原理就是使用了代理模式对Binder的使用进行了优化，保证了代码的整洁性和一致性，避免重复编写繁琐的代理类相关代码。

参考：[不得不说的 Android Binder 机制与 AIDL](https://juejin.cn/post/6994057245113729038)

------

上述代码中 ActivityTaskManager.getService().startActivity，先拿到了ATMS的代理对象，这样就通过AIDL调用到systemserver侧，把启动任务交给ATMS。

```java
//路径：core/java/android/app/ActivityTaskManager.java
//这里使用单利模式
private ActivityTaskManager() {
    }

    /** @hide */
    public static ActivityTaskManager getInstance() {
        return sInstance.get();
    }

    /** @hide */
    public static IActivityTaskManager getService() {
        return IActivityTaskManagerSingleton.get();
    }

    @UnsupportedAppUsage(trackingBug = 129726065)
    private static final Singleton<IActivityTaskManager> IActivityTaskManagerSingleton =
            new Singleton<IActivityTaskManager>() {
                @Override
                protected IActivityTaskManager create() {
                    //得到AMS的Ibinder接口
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_TASK_SERVICE);
                    //转换成IActivityTaskManager对象，systemserver中ATMS继承于IActivityTaskManager.Stub接口并进行了具体实现
                    return IActivityTaskManager.Stub.asInterface(b);
                }
            };
```

------



## 3.2 AMS处理请求的过程（systemserver进程）

### 3.2.1 ActivityTaskManagerService.startActivity

```java
//路径：services/core/java/com/android/server/wm/ActivityTaskManagerService.java
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    Slog.e("SETTINGERROR"," ActivityTaskManagerService#startActivity "
            + " callingPackage = " + callingPackage
            + " 当前的进程号 = " + Process.myPid()
            + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}


```



### 3.2.2 ActivityTaskManagerService.startActivityAsUser

```java
//路径：services/core/java/com/android/server/wm/ActivityTaskManagerService.java
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}

private int startActivityAsUser(IApplicationThread caller, String callingPackage,
        @Nullable String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
    assertPackageMatchesCallingUid(callingPackage);
    enforceNotIsolatedCaller("startActivityAsUser");
    if (Process.isSdkSandboxUid(Binder.getCallingUid())) {
        SdkSandboxManagerLocal sdkSandboxManagerLocal = LocalManagerRegistry.getManager(
                SdkSandboxManagerLocal.class);
        if (sdkSandboxManagerLocal == null) {
            throw new IllegalStateException("SdkSandboxManagerLocal not found when starting"
                    + " an activity from an SDK sandbox uid.");
        }
        sdkSandboxManagerLocal.enforceAllowedToStartActivity(intent);
    }

    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // TODO: Switch to user app stacks here.
    //通过ActivityStartController创建并获取到ActivityStarter，通过ActivityStarter的execute，把相关的Activity启动的逻辑(通过setXXXXX方法)放到ActivityStarter中
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

//路径：services/core/java/com/android/server/wm/ActivityStartController.java
 ActivityStarter obtainStarter(Intent intent, String reason) {
        return mFactory.obtain().setIntent(intent).setReason(reason);
    }

//路径：services/core/java/com/android/server/wm/ActivityStarter.java
ActivityStarter obtain();
//ActivityStarter中的工厂类
static class DefaultFactory implements Factory {
        private ActivityStartController mController;
        private ActivityTaskManagerService mService;
        private ActivityTaskSupervisor mSupervisor;
        private ActivityStartInterceptor mInterceptor;

        private SynchronizedPool<ActivityStarter> mStarterPool =
                new SynchronizedPool<>(MAX_STARTER_COUNT);
    
        public ActivityStarter obtain() {
            ActivityStarter starter = mStarterPool.acquire();

            if (starter == null) {
                if (mService.mRootWindowContainer == null) {
                    throw new IllegalStateException("Too early to start activity.");
                }
                starter = new ActivityStarter(mController, mService, mSupervisor, mInterceptor);
            }

            return starter;
        }
}

```

接下来就是ActivityStarter的源码流程，AMS把启动的逻辑都放在ActivityStarter去处理，主要包括启动前的处理，进程的创建等等。

### 3.2.3 ActivityStarter.setXXXXX

将从app进程传递过来的值都赋给mRequest变量，Request是ActivityStarter的一个内部类。

```JAVA
ActivityStarter setCaller(IApplicationThread caller) {
        mRequest.caller = caller;
        return this;
    }
    
ActivityStarter setResolvedType(String type) {
    mRequest.resolvedType = type;
    return this;
}

ActivityStarter setActivityInfo(ActivityInfo info) {
    mRequest.activityInfo = info;
    return this;
}
```



### 3.2.4 ActivityStarter.execute

```java
int execute() {
    try {

        final LaunchingState launchingState;
        synchronized (mService.mGlobalLock) {
            final ActivityRecord caller = ActivityRecord.forTokenLocked(mRequest.resultTo);
            final int callingUid = mRequest.realCallingUid == Request.DEFAULT_REAL_CALLING_UID
                    ?  Binder.getCallingUid() : mRequest.realCallingUid;
            launchingState = mSupervisor.getActivityMetricsLogger().notifyActivityLaunching(
                    mRequest.intent, caller, callingUid);
        }

		......
            
        int res;
        synchronized (mService.mGlobalLock) {
            res = resolveToHeavyWeightSwitcherIfNeeded();
            if (res != START_SUCCESS) {
                return res;
            }
            
            res = executeRequest(mRequest);

         
}
```

### 3.2.5 ActivityStarter.executeRequest

创建一个activityrecord记录得到的activity信息

```java
private int executeRequest(Request request) {
    	.......
         //创建一个activityrecord记录得到的activity信息
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

                if (r.appTimeTracker == null && sourceRecord != null) {
                    // If the caller didn't specify an explicit time tracker, we want to continue
                    // tracking under any it has.
                    r.appTimeTracker = sourceRecord.appTimeTracker;
                }

                // Only allow app switching to be resumed if activity is not a restricted background
                // activity and target app is not home process, otherwise any background activity
                // started in background task can stop home button protection mode.
                // As the targeted app is not a home process and we don't need to wait for the 2nd
                // activity to be started to resume app switching, we can just enable app switching
                // directly.
                WindowProcessController homeProcess = mService.mHomeProcess;
                boolean isHomeProcess = homeProcess != null
                        && aInfo.applicationInfo.uid == homeProcess.mUid;
                if (!restrictedBgActivity && !isHomeProcess) {
                    mService.resumeAppSwitches();
                }

                mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                        request.voiceInteractor, startFlags, true /* doResume */, checkedOptions,
                        inTask, inTaskFragment, restrictedBgActivity, intentGrants);

                if (request.outActivity != null) {
                    request.outActivity[0] = mLastStartActivityRecord;
                }

                return mLastStartActivityResult;
    }

```



### 3.2.6 ActivityStarter.startActivityUnchecked

```java
   private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            TaskFragment inTaskFragment, boolean restrictedBgActivity,
            NeededUriGrants intentGrants) {
        int result = START_CANCELED;
        final Task startedActivityRootTask;

        // Create a transition now to record the original intent of actions taken within
        // startActivityInner. Otherwise, logic in startActivityInner could start a different
        // transition based on a sub-action.
        // Only do the create here (and defer requestStart) since startActivityInner might abort.
        final TransitionController transitionController = r.mTransitionController;
        Transition newTransition = (!transitionController.isCollecting()
                && transitionController.getTransitionPlayer() != null)
                ? transitionController.createTransition(TRANSIT_OPEN) : null;
        RemoteTransition remoteTransition = r.takeRemoteTransition();
        if (newTransition != null && remoteTransition != null) {
            newTransition.setRemoteTransition(remoteTransition);
        }
        transitionController.collect(r);
        try {
            mService.deferWindowLayout();
            Slog.e("SETTINGERROR"," ActivityStarter#startActivityUnchecked "
                    + " 当前的进程号 = " + Process.myPid()
                    + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
            result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, inTaskFragment, restrictedBgActivity,
                    intentGrants);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
            startedActivityRootTask = handleStartResult(r, options, result, newTransition,
                    remoteTransition);
            mService.continueWindowLayout();
        }

        postStartActivityProcessing(r, result, startedActivityRootTask);

        return result;
    }
```



###  3.2.7 ActivityStarter.startActivityInner

```java
    int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                TaskFragment inTaskFragment, boolean restrictedBgActivity,
                NeededUriGrants intentGrants) {
            
                if (mDoResume) {
                            final ActivityRecord topTaskActivity = startedTask.topRunningActivityLocked();
                            if (!mTargetRootTask.isTopActivityFocusable()
                                    || (topTaskActivity != null && topTaskActivity.isTaskOverlay()
                                    && mStartActivity != topTaskActivity)) {
                                // If the activity is not focusable, we can't resume it, but still would like to
                                // make sure it becomes visible as it starts (this will also trigger entry
                                // animation). An example of this are PIP activities.
                                // Also, we don't want to resume activities in a task that currently has an overlay
                                // as the starting activity just needs to be in the visible paused state until the
                                // over is removed.
                                // Passing {@code null} as the start parameter ensures all activities are made
                                // visible.
                                mTargetRootTask.ensureActivitiesVisible(null /* starting */,
                                        0 /* configChanges */, !PRESERVE_WINDOWS);
                                // Go ahead and tell window manager to execute app transition for this activity
                                // since the app transition will not be triggered through the resume channel.
                                mTargetRootTask.mDisplayContent.executeAppTransition();
                            } else {
                                // If the target root-task was not previously focusable (previous top running
                                // activity on that root-task was not visible) then any prior calls to move the
                                // root-task to the will not update the focused root-task.  If starting the new
                                // activity now allows the task root-task to be focusable, then ensure that we
                                // now update the focused root-task accordingly.
                                if (mTargetRootTask.isTopActivityFocusable()
                                        && !mRootWindowContainer.isTopDisplayFocusedRootTask(mTargetRootTask)) {
                                    mTargetRootTask.moveToFront("startActivityInner");
                                }
                                mRootWindowContainer.resumeFocusedTasksTopActivities(
                                        mTargetRootTask, mStartActivity, mOptions, mTransientLaunch);
                            }
                        }
                        mRootWindowContainer.updateUserRootTask(mStartActivity.mUserId, mTargetRootTask);

                        // Update the recent tasks list immediately when the activity starts
                        mSupervisor.mRecentTasks.add(startedTask);
                        mSupervisor.handleNonResizableTaskIfNeeded(startedTask,
                                mPreferredWindowingMode, mPreferredTaskDisplayArea, mTargetRootTask);

                        // If Activity's launching into PiP, move the mStartActivity immediately to pinned mode.
                        // Note that mStartActivity and source should be in the same Task at this point.
                        if (mOptions != null && mOptions.isLaunchIntoPip()
                                && sourceRecord != null && sourceRecord.getTask() == mStartActivity.getTask()
                                && !mRestrictedBgActivity) {
                            mRootWindowContainer.moveActivityToPinnedRootTask(mStartActivity,
                                    sourceRecord, "launch-into-pip");
                        }

                        return START_SUCCESS;
    }			
```



### 3.2.8 RootWindowContainer.resumeFocusedTasksTopActivities

接下来进行一系列的调用

RootWindowContainer.resumeFocusedTasksTopActivities->Task.resumeTopActivityUncheckedLocked->Task.resumeTopActivityUncheckedLocked->Task.resumeTopActivityInnerLocked->TaskFragment.resumeTopActivity->ActivityTaskSupervisor.startSpecificActivity

从这里就调用到了关键流程ActivityTaskSupervisor.startSpecificActivity，在startSpecificActivity里面会去判断app进程是否创建，没有创建就去创建app进程；已经创建进程的话，就去调用realStartActivityLocked去创建事务Transaction，交给app进程去执行，事务的内容就是里面的item。



### 3.2.9 ActivityTaskSupervisor.startSpecificActivity

```java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
                if (mPerfBoost != null) {
                    Slog.i(TAG, "The Process " + r.processName + " Already Exists in BG. So sending its PID: " + wpc.getPid());
                    mPerfBoost.perfHint(BoostFramework.VENDOR_HINT_FIRST_LAUNCH_BOOST, r.processName, wpc.getPid(), BoostFramework.Launch.TYPE_START_APP_FROM_BG);
                }
                Slog.e("SETTINGERROR"," ActivityTaskSupervisor#startSpecificActivity+realStartActivityLocked "
                        + " 当前的activity = " + r
                        + " 当前的进程号 = " + Process.myPid()
                        + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
                //进程已经存在
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
            knownToBeDead = true;
            // Remove the process record so it won't be considered as alive.
            mService.mProcessNames.remove(wpc.mName, wpc.mUid);
            mService.mProcessMap.remove(wpc.getPid());
        }

        r.notifyUnknownVisibilityLaunchedForKeyguardTransition();

        final boolean isTop = andResume && r.isTopRunningActivity();
        Slog.e("SETTINGERROR"," ActivityTaskSupervisor#startSpecificActivity+startProcessAsync "
                + " 当前的activity = " + r
                + " 当前的进程号 = " + Process.myPid()
                + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
         //app进程不存在，创建一个APP进程
        mService.startProcessAsync(r, knownToBeDead, isTop,
                isTop ? HostingRecord.HOSTING_TYPE_TOP_ACTIVITY
                        : HostingRecord.HOSTING_TYPE_ACTIVITY);
    }
```

以当前app进程已经存在为例，直接调用realStartActivityLocked函数

### 3.2.10 ActivityTaskSupervisor.realStartActivityLocked

```java
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {

              ......
              EventLogTags.writeWmRestartActivity(r.mUserId, System.identityHashCode(r),
                                task.mTaskId, r.shortComponentName);
              //创建activity启动的事务，这里将ActivityRecord的token封装在ClientTransaction，并将这个token传递到客户端
              final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.token);

                final boolean isTransitionForward = r.isTransitionForward();
                final IBinder fragmentToken = r.getTaskFragment().getFragmentToken();
        		//通过LaunchActivityItem.obtain创建一个LaunchActivityItem添加到clientTransaction的mActivityCallbacks列表中，会在handlmessage中的execute时进行处理
                clientTransaction.addCallback(LaunchActivityItem.obtain(new Intent(r.intent),
                        System.identityHashCode(r), r.info,
                        // TODO: Have this take the merged configuration instead of separate global
                        // and override configs.
                        mergedConfiguration.getGlobalConfiguration(),
                        mergedConfiguration.getOverrideConfiguration(), r.compat,
                        r.getFilteredReferrer(r.launchedFromPackage), task.voiceInteractor,
                        proc.getReportedProcState(), r.getSavedState(), r.getPersistentSavedState(),
                        results, newIntents, r.takeOptions(), isTransitionForward,
                        proc.createProfilerInfoIfNeeded(), r.assistToken, activityClientController,
                        r.shareableActivityToken, r.getLaunchedFromBubble(), fragmentToken));

                // 设置最终要切换的状态
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    //通过ResumeActivityItem.obtain创建一个ResumeActivityItem
                    lifecycleItem = ResumeActivityItem.obtain(isTransitionForward);
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
        		//设置客户端执行事务后应处于的生命周期状态
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

        		//安排事务
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);
                
                ......
}

//路径：core/java/android/app/servertransaction/ClientTransaction.java
    /** Obtain an instance initialized with provided params. */
    public static ClientTransaction obtain(IApplicationThread client, IBinder activityToken) {
        ClientTransaction instance = ObjectPool.obtain(ClientTransaction.class);
        if (instance == null) {
            instance = new ClientTransaction();
        }
        instance.mClient = client;
        instance.mActivityToken = activityToken;

        return instance;
    }

    /**
     * Add a message to the end of the sequence of callbacks.
     * @param activityCallback A single message that can contain a lifecycle request/callback.
     */
    public void addCallback(ClientTransactionItem activityCallback) {
        if (mActivityCallbacks == null) {
            mActivityCallbacks = new ArrayList<>();
        }
        mActivityCallbacks.add(activityCallback);
    }


    /**
     * Set the lifecycle state in which the client should be after executing the transaction.
     * @param stateRequest A lifecycle request initialized with right parameters.
     */
    public void setLifecycleStateRequest(ActivityLifecycleItem stateRequest) {
        mLifecycleStateRequest = stateRequest;
    }

```



### 3.2.11 ClientLifecycleManager.scheduleTransaction

```java
/**
 * Schedule a transaction, which may consist of multiple callbacks and a lifecycle request.
 * @param transaction A sequence of client transaction items.
 * @throws RemoteException
 *
 * @see ClientTransaction
 */
void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
    final IApplicationThread client = transaction.getClient();
    //调用到客户端，开始进行事务的处理
    transaction.schedule();
    if (!(client instanceof Binder)) {
        // If client is not an instance of Binder - it's a remote call and at this point it is
        // safe to recycle the object. All objects used for local calls will be recycled after
        // the transaction is executed on client in ActivityThread.
        transaction.recycle();
    }
}
```



## 3.3 ActivityThread创建Activity过程(App进程)

### 3.3.1 ClientTransaction.schedule

```java
/** Target client. */
    private IApplicationThread mClient;

/**
 * Schedule the transaction after it was initialized. It will be send to client and all its
 * individual parts will be applied in the following sequence:
 * 1. The client calls {@link #preExecute(ClientTransactionHandler)}, which triggers all work
 *    that needs to be done before actually scheduling the transaction for callbacks and
 *    lifecycle state request.
 * 2. The transaction message is scheduled.
 * 3. The client calls {@link TransactionExecutor#execute(ClientTransaction)}, which executes
 *    all callbacks and necessary lifecycle transitions.
 */
public void schedule() throws RemoteException {
    mClient.scheduleTransaction(this);
}
```



### 3.3.2 ActivityThread.ApplicationThread.scheduleTransaction

```java
public final class ActivityThread extends ClientTransactionHandler
        implements ActivityThreadInternal {
    
    	private class ApplicationThread extends IApplicationThread.Stub {
					public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
   							 ActivityThread.this.scheduleTransaction(transaction);
               	 }
        }
}
```



### 3.3.3 ClientTransactionHandler.scheduleTransaction

调用到ActivityThread的父级ClientTransactionHandler的scheduleTransaction，这里去sendMessage

```java
/** Prepare and schedule transaction for execution. */
void scheduleTransaction(ClientTransaction transaction) {
    transaction.preExecute(this);
    sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
}
```



### 3.3.4 ActivityThread.H.handleMessage

```java
public void handleMessage(Message msg) {
    switch (msg.what) {
            case EXECUTE_TRANSACTION:
                    final ClientTransaction transaction = (ClientTransaction) msg.obj;
                    mTransactionExecutor.execute(transaction);
                    if (isSystem()) {
                    // Client transactions inside system process are recycled on the client side
                    // instead of ClientLifecycleManager to avoid being cleared before this
                    // message is handled.
                    transaction.recycle();
         		   }
                    // TODO(lifecycler): Recycle locally scheduled transactions.
                    break;
            }
}
```



### 3.3.5 TransactionExecutor.execute

```java
public void execute(ClientTransaction transaction) {
    if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Start resolving transaction");

    final IBinder token = transaction.getActivityToken();
    if (token != null) {
        final Map<IBinder, ClientTransactionItem> activitiesToBeDestroyed =
                mTransactionHandler.getActivitiesToBeDestroyed();
        final ClientTransactionItem destroyItem = activitiesToBeDestroyed.get(token);
        if (destroyItem != null) {
            if (transaction.getLifecycleStateRequest() == destroyItem) {
                // It is going to execute the transaction that will destroy activity with the
                // token, so the corresponding to-be-destroyed record can be removed.
                activitiesToBeDestroyed.remove(token);
            }
            if (mTransactionHandler.getActivityClient(token) == null) {
                // The activity has not been created but has been requested to destroy, so all
                // transactions for the token are just like being cancelled.
                Slog.w(TAG, tId(transaction) + "Skip pre-destroyed transaction:\n"
                        + transactionToString(transaction, mTransactionHandler));
                return;
            }
        }
    }

    if (DEBUG_RESOLVER) Slog.d(TAG, transactionToString(transaction, mTransactionHandler));

    executeCallbacks(transaction);

    executeLifecycleState(transaction);
    mPendingActions.clear();
    if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "End resolving transaction");
}
```



### 3.3.5 TransactionExecutor.executeCallbacks

```java
public void executeCallbacks(ClientTransaction transaction) {
    final List<ClientTransactionItem> callbacks = transaction.getCallbacks();
    //在3.2.10 realStartActivityLocked会注册一个LaunchActivityItem的callback，如果此处获取的为null，直接返回
    if (callbacks == null || callbacks.isEmpty()) {
        // No callbacks to execute, return early.
        return;
    }
    if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callbacks in transaction");

    final IBinder token = transaction.getActivityToken();
    ActivityClientRecord r = mTransactionHandler.getActivityClient(token);

    // In case when post-execution state of the last callback matches the final state requested
    // for the activity in this transaction, we won't do the last transition here and do it when
    // moving to final state instead (because it may contain additional parameters from server).
    final ActivityLifecycleItem finalStateRequest = transaction.getLifecycleStateRequest();
    final int finalState = finalStateRequest != null ? finalStateRequest.getTargetState()
            : UNDEFINED;
    // Index of the last callback that requests some post-execution state.
    final int lastCallbackRequestingState = lastCallbackRequestingState(transaction);

    final int size = callbacks.size();
    for (int i = 0; i < size; ++i) {
        final ClientTransactionItem item = callbacks.get(i);
        if (DEBUG_RESOLVER) Slog.d(TAG, tId(transaction) + "Resolving callback: " + item);
        final int postExecutionState = item.getPostExecutionState();
        final int closestPreExecutionState = mHelper.getClosestPreExecutionState(r,
                item.getPostExecutionState());
        if (closestPreExecutionState != UNDEFINED) {
            cycleToPath(r, closestPreExecutionState, transaction);
        }
		
        //这里的item指的是LaunchActivityItem
        item.execute(mTransactionHandler, token, mPendingActions);
        item.postExecute(mTransactionHandler, token, mPendingActions);
        if (r == null) {
            // Launch activity request will create an activity record.
            r = mTransactionHandler.getActivityClient(token);
        }

        if (postExecutionState != UNDEFINED && r != null) {
            // Skip the very last transition and perform it by explicit state request instead.
            final boolean shouldExcludeLastTransition =
                    i == lastCallbackRequestingState && finalState == postExecutionState;
            cycleToPath(r, postExecutionState, shouldExcludeLastTransition, transaction);
        }
    }
}
```



### #####onCreate生命周期#####

在MainActivity3的onCreate进行断点

```java
public class MainActivity3 extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main3);
    }
}
```



打印的堆栈信息如下

```
onCreate:12, MainActivity3 (com.example.myapp)
performCreate:8593, Activity (android.app)
performCreate:8557, Activity (android.app)
callActivityOnCreate:1445, Instrumentation (android.app)
performLaunchActivity:3905, ActivityThread (android.app)
handleLaunchActivity:4077, ActivityThread (android.app)
execute:101, LaunchActivityItem (android.app.servertransaction)
executeCallbacks:135, TransactionExecutor (android.app.servertransaction)
execute:95, TransactionExecutor (android.app.servertransaction)
handleMessage:2426, ActivityThread$H (android.app)
dispatchMessage:106, Handler (android.os)
loopOnce:211, Looper (android.os)
loop:300, Looper (android.os)
main:8512, ActivityThread (android.app)
invoke:-1, Method (java.lang.reflect)
run:561, RuntimeInit$MethodAndArgsCaller (com.android.internal.os)
main:954, ZygoteInit (com.android.internal.os)
```



#### 3.3.5.1 LaunchActivityItem.execute

```java
public void execute(ClientTransactionHandler client, IBinder token,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
    ActivityClientRecord r = new ActivityClientRecord(token, mIntent, mIdent, mInfo,
            mOverrideConfig, mCompatInfo, mReferrer, mVoiceInteractor, mState, mPersistentState,
            mPendingResults, mPendingNewIntents, mActivityOptions, mIsForward, mProfilerInfo,
            client, mAssistToken, mShareableActivityToken, mLaunchedFromBubble,
            mTaskFragmentToken);
    client.handleLaunchActivity(r, pendingActions, null /* customIntent */);
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```



#### 3.3.5.2 ActivityThread.handleLaunchActivity

```java
/**
 * Extended implementation of activity launch. Used when server requests a launch or relaunch.
 */
@Override
public Activity handleLaunchActivity(ActivityClientRecord r,
        PendingTransactionActions pendingActions, Intent customIntent) {
    
  //进一步去调用performLaunchActivity
    final Activity a = performLaunchActivity(r, customIntent);

    if (a != null) {
        r.createdConfig = new Configuration(mConfigurationController.getConfiguration());
        reportSizeConfigurations(r);
        if (!r.activity.mFinished && pendingActions != null) {
            pendingActions.setOldState(r.state);
            pendingActions.setRestoreInstanceState(true);
            pendingActions.setCallOnPostCreate(true);
        }
    } else {
        // If there was an error, for any reason, tell the activity manager to stop us.
        ActivityClient.getInstance().finishActivity(r.token, Activity.RESULT_CANCELED,
                null /* resultData */, Activity.DONT_FINISH_TASK_WITH_ACTIVITY);
    }

    return a;
}
```



#### 3.3.5.3 ActivityThread.performLaunchActivity

```java
/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

    ......

    ContextImpl appContext = createBaseContextForActivity(r);
    Activity activity = null;
    try {
        java.lang.ClassLoader cl = appContext.getClassLoader();
        //通过类加载器的形式创建activity
        activity = mInstrumentation.newActivity(
                cl, component.getClassName(), r.intent);
        Slog.e("SETTINGERROR"," ActivityThread#performLaunchActivity "
                + " 当前的Activity是 = " + activity
                + " 当前的进程号 = " + Process.myPid()
                + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());

        StrictMode.incrementExpectedActivityCount(activity.getClass());
        r.intent.setExtrasClassLoader(cl);
        r.intent.prepareToEnterProcess(isProtectedComponent(r.activityInfo),
                appContext.getAttributionSource());
        if (r.state != null) {
            r.state.setClassLoader(cl);
        }
    } catch (Exception e) {
        if (!mInstrumentation.onException(activity, e)) {
            throw new RuntimeException(
                "Unable to instantiate activity " + component
                + ": " + e.toString(), e);
        }
    }

    try {
        Application app = r.packageInfo.makeApplicationInner(false, mInstrumentation);

        if (activity != null) {
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());
            Configuration config =
                    new Configuration(mConfigurationController.getCompatConfiguration());
            if (r.overrideConfig != null) {
                config.updateFrom(r.overrideConfig);
            }
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "
                    + r.activityInfo.name + " with config " + config);
            Window window = null;
            if (r.mPendingRemoveWindow != null && r.mPreserveWindow) {
                window = r.mPendingRemoveWindow;
                r.mPendingRemoveWindow = null;
                r.mPendingRemoveWindowManager = null;
            }

            // Activity resources must be initialized with the same loaders as the
            // application context.
            appContext.getResources().addLoaders(
                    app.getResources().getLoaders().toArray(new ResourcesLoader[0]));

            appContext.setOuterContext(activity);
            activity.attach(appContext, this, getInstrumentation(), r.token,
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,
                    r.embeddedID, r.lastNonConfigurationInstances, config,
                    r.referrer, r.voiceInteractor, window, r.activityConfigCallback,
                    r.assistToken, r.shareableActivityToken);
			
            //进一步调用到Instrumentation.callActivityOnCreate
            if (r.isPersistable()) {
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
            } else {
                mInstrumentation.callActivityOnCreate(activity, r.state);
            }
            if (!activity.mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + r.intent.getComponent().toShortString() +
                    " did not call through to super.onCreate()");
            }
            mLastReportedWindowingMode.put(activity.getActivityToken(),
                    config.windowConfiguration.getWindowingMode());
        }
        r.setState(ON_CREATE);

    } 
    }

    return activity;
}
```



#### 3.3.5.4 Instrumentation.callActivityOnCreate

```java
public void callActivityOnCreate(Activity activity, Bundle icicle,
        PersistableBundle persistentState) {
    prePerformCreate(activity);
    activity.performCreate(icicle, persistentState);
    postPerformCreate(activity);
}
```



#### 3.3.5.5 Activity.performCreate

```java
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
    if (Trace.isTagEnabled(Trace.TRACE_TAG_WINDOW_MANAGER)) {
        Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "performCreate:"
                + mComponent.getClassName());
    }
    dispatchActivityPreCreated(icicle);
    mCanEnterPictureInPicture = true;
    // initialize mIsInMultiWindowMode and mIsInPictureInPictureMode before onCreate
    final int windowingMode = getResources().getConfiguration().windowConfiguration
            .getWindowingMode();
    mIsInMultiWindowMode = inMultiWindowMode(windowingMode);

    mIsInPictureInPictureMode = windowingMode == WINDOWING_MODE_PINNED;
    mShouldDockBigOverlays = getResources().getBoolean(R.bool.config_dockBigOverlayWindows);
    restoreHasCurrentPermissionRequest(icicle);

	//这里就回调到onCreate
    if (persistentState != null) {
        onCreate(icicle, persistentState);
    } else {
        onCreate(icicle);
    }
    
    EventLogTags.writeWmOnCreateCalled(mIdent, getComponentName().getClassName(),
              "performCreate")
    mActivityTransitionState.readState(icicle);

    mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
            com.android.internal.R.styleable.Window_windowNoDisplay, false);
    mFragments.dispatchActivityCreated();
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    dispatchActivityPostCreated(icicle);
    Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
}
```



### ##########





### 3.3.6 TransactionExecutor.executeLifecycleState

```java
private void executeLifecycleState(ClientTransaction transaction) {
    final ActivityLifecycleItem lifecycleItem = transaction.getLifecycleStateRequest();
    if (lifecycleItem == null) {
        // No lifecycle request, return early.
        return;
    }

    final IBinder token = transaction.getActivityToken();
    final ActivityClientRecord r = mTransactionHandler.getActivityClient(token);
    if (DEBUG_RESOLVER) {
        Slog.d(TAG, tId(transaction) + "Resolving lifecycle state: "
                + lifecycleItem + " for activity: "
                + getShortActivityName(token, mTransactionHandler));
    }

    if (r == null) {
        // Ignore requests for non-existent client records for now.
        return;
    }

    // 循环到最终状态之前的状态，在首次创建启动时这里是start的状态
    cycleToPath(r, lifecycleItem.getTargetState(), true /* excludeLastState */, transaction);

    // Execute the final transition with proper parameters.
    //这里的lifecycleItem就是3.2.10设置客户端执行事务后应处于的生命周期状态,是ResumeActivityItem
    lifecycleItem.execute(mTransactionHandler, token, mPendingActions);
    lifecycleItem.postExecute(mTransactionHandler, token, mPendingActions);
}
```



### #####onStart生命周期#####

#### TransactionExecutor.cycleToPath

#### TransactionExecutor.performLifecycleSequence

#### ActivityThread.handleStartActivity

#### Activity.performStart

#### Instrumentation.callActivityOnStart

在app代码中的onStart处进行断点调试，堆栈信息如下

```
at com.demoapp.activitydemo.MainActivity.onStart(MainActivity.java:36)
at android.app.Instrumentation.callActivityOnStart(Instrumentation.java:1436)
at android.app.Activity.performStart(Activity.java:8266)
at android.app.ActivityThread.handleStartActivity(ActivityThread.java:3613)
at android.app.servertransaction.TransactionExecutor.performLifecycleSequence(TransactionExecutor.java:221)
at android.app.servertransaction.TransactionExecutor.cycleToPath(TransactionExecutor.java:201)
at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:173)
at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2173)
at android.os.Handler.dispatchMessage(Handler.java:106)
at android.os.Looper.loop(Looper.java:236)
at android.app.ActivityThread.main(ActivityThread.java:8168)
at java.lang.reflect.Method.invoke(Native Method)
at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:656)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:967)
```

### ##########











### #####onResume生命周期#####

在app代码中的onResume进行断点调试，堆栈信息如下

```
at com.demoapp.activitydemo.MainActivity.onResume(MainActivity.java:43)
at android.app.Instrumentation.callActivityOnResume(Instrumentation.java:1457)
at android.app.Activity.performResume(Activity.java:8390)
at android.app.ActivityThread.performResumeActivity(ActivityThread.java:4601)
at android.app.ActivityThread.handleResumeActivity(ActivityThread.java:4643)
at android.app.servertransaction.ResumeActivityItem.execute(ResumeActivityItem.java:52)
at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:176)
at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2173)
at android.os.Handler.dispatchMessage(Handler.java:106)
at android.os.Looper.loop(Looper.java:236)
at android.app.ActivityThread.main(ActivityThread.java:8168)
at java.lang.reflect.Method.invoke(Native Method)
at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:656)
at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:967)
```



#### 3.3.6.1 ResumeActivityItem.execute

```java
public void execute(ClientTransactionHandler client, ActivityClientRecord r,
        PendingTransactionActions pendingActions) {
    Trace.traceBegin(TRACE_TAG_ACTIVITY_MANAGER, "activityResume");
    client.handleResumeActivity(r, true /* finalStateRequest */, mIsForward,
            "RESUME_ACTIVITY");
    Trace.traceEnd(TRACE_TAG_ACTIVITY_MANAGER);
}
```



#### 3.3.6.2 ActivityThread.handleResumeActivity

```java
public void handleResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
        boolean isForward, String reason) {
    // If we are getting ready to gc after going to the background, well
    // we are back active so skip it.
    unscheduleGcIdler();
    mSomeActivitiesChanged = true;

    // TODO Push resumeArgs into the activity for consideration
    // skip below steps for double-resume and r.mFinish = true case.
    if (!performResumeActivity(r, finalStateRequest, reason)) {
        return;
    }
}
```



#### 3.3.6.3 ActivityThread.performResumeActivity

```java
public boolean performResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
        String reason) {

    if (r.activity.mFinished) {
        return false;
    }

    if (r.getLifecycleState() == ON_RESUME) {
        if (!finalStateRequest) {
            final RuntimeException e = new IllegalStateException(
                    "Trying to resume activity which is already resumed");
            Slog.e(TAG, e.getMessage(), e);
            Slog.e(TAG, r.getStateString());
            // TODO(lifecycler): A double resume request is possible when an activity
            // receives two consequent transactions with relaunch requests and "resumed"
            // final state requests and the second relaunch is omitted. We still try to
            // handle two resume requests for the final state. For cases other than this
            // one, we don't expect it to happen.
        }
        return false;
    }
    if (finalStateRequest) {
        r.hideForNow = false;
        r.activity.mStartedActivity = false;
    }
    try {
        r.activity.onStateNotSaved();
        r.activity.mFragments.noteStateNotSaved();
        checkAndBlockForNetworkAccess();
        if (r.pendingIntents != null) {
            deliverNewIntents(r, r.pendingIntents);
            r.pendingIntents = null;
        }
        if (r.pendingResults != null) {
            deliverResults(r, r.pendingResults, reason);
            r.pendingResults = null;
        }
        Slog.e("SETTINGERROR"," ActivityThread#performResumeActivity "
                + " 当前的Activity是 = " + r.activity
                + " 当前的进程号 = " + Process.myPid()
                + " 当前的进程名称 = " + Process.myProcessName()
                + " 当前的生命周期状态是 = " + r.getLifecycleState(),new Throwable());
        r.activity.performResume(r.startsNotResumed, reason);

        r.state = null;
        r.persistentState = null;
        r.setState(ON_RESUME);

        reportTopResumedActivityChanged(r, r.isTopResumedActivity, "topWhenResuming");
    } catch (Exception e) {
        if (!mInstrumentation.onException(r.activity, e)) {
            throw new RuntimeException("Unable to resume activity "
                    + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
        }
    }
    return true;
}
```



#### 3.3.6.4 Activity.performResume

```JAVA
    final void performResume(boolean followedByPause, String reason) {
        if (Trace.isTagEnabled(Trace.TRACE_TAG_WINDOW_MANAGER)) {
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "performResume:"
                    + mComponent.getClassName());
        }
        dispatchActivityPreResumed();
        performRestart(true /* start */, reason);

        mFragments.execPendingActions();

        mLastNonConfigurationInstances = null;

        getAutofillClientController().onActivityPerformResume(followedByPause);

        mCalled = false;
        // mResumed is set by the instrumentation

        mInstrumentation.callActivityOnResume(this);
        EventLogTags.writeWmOnResumeCalled(mIdent, getComponentName().getClassName(), reason);

        if (!mCalled) {
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onResume()");
        }

        // invisible activities must be finished before onResume) completes
        if (!mVisibleFromClient && !mFinished) {
            Log.w(TAG, "An activity without a UI must call finish() before onResume() completes");
            if (getApplicationInfo().targetSdkVersion
                    > android.os.Build.VERSION_CODES.LOLLIPOP_MR1) {
                throw new IllegalStateException(
                        "Activity " + mComponent.toShortString() +
                        " did not call finish() prior to onResume() completing");
            }
        }

        // Now really resume, and install the current status bar and menu.
        mCalled = false;

        mFragments.dispatchResume();
        mFragments.execPendingActions();

        onPostResume();
        if (!mCalled) {
            throw new SuperNotCalledException(
                "Activity " + mComponent.toShortString() +
                " did not call through to super.onPostResume()");
        }
        dispatchActivityPostResumed();
        Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
    }
```

#### 3.3.6.5 Instrumentation.callActivityOnResume

```JAVA
public void callActivityOnResume(Activity activity) {
    activity.mResumed = true;
    activity.onResume();
    
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                am.match(activity, activity, activity.getIntent());
            }
        }
    }
}
```

#### 3.3.6.5  Activity.onResume()

```java
protected void onResume() {
    if (DEBUG_LIFECYCLE) Slog.v(TAG, "onResume " + this);
    dispatchActivityResumed();
    mActivityTransitionState.onResume(this);
    getAutofillClientController().onActivityResumed();

    notifyContentCaptureManagerIfNeeded(CONTENT_CAPTURE_RESUME);

    mCalled = true;
}
```



### ##########

# 4.EventsLog、System、MainLog打印

如果要打印这些所有的信息，可以用下面的两个命令进行过滤，all表示的信息更加全面。

-  adb logcat -b events,system,main

-  adb logcat -b all

## 4.1 events

命令：adb logcat -b events

events log关键字常包含：wm_,input_focus,input_interaction

例如：从桌面点击demo

```
11-04 11:59:46.471  2430  3107 I input_interaction: Interaction with: ce2ec2b com.miui.home/com.miui.home.launcher.Launcher (server), {touchableRegion=[0,0][1080,2400], inputConfig=DUPLICATE_TOUCH_TO_WALLPAPER}, [Gesture Monitor] miui-gesture (server), PointerEventDispatcher0 (server), 
11-04 11:59:46.585  2430  4154 I wm_task_moved: [17,1,4]
11-04 11:59:46.585  2430  4154 I wm_task_to_front: [0,17]
11-04 11:59:46.586  2430  4154 I wm_focused_root_task: [0,0,17,1,bringingFoundTaskToFront]
11-04 11:59:46.587  2430  4154 I input_focus: displayId :0, focusApplication has changed to ActivityRecord{67b2b99 u0 com.example.myapp/.MainActivity} t17}
11-04 11:59:46.588  2430  4154 I wm_set_resumed_activity: [0,com.example.myapp/.MainActivity,bringingFoundTaskToFront]
11-04 11:59:46.589  2430  4154 I wm_new_intent: [0,108735385,17,com.example.myapp/.MainActivity,android.intent.action.MAIN,NULL,NULL,270532608]
11-04 11:59:46.594  2430  4154 I wm_pause_activity: [0,89564982,com.miui.home/.launcher.Launcher,userLeaving=true,pauseBackTasks]

11-04 11:59:46.609 20818 20818 I wm_on_restart_called: [0,108735385,com.example.myapp.MainActivity,performRestartActivity,0]
11-04 11:59:46.610 20818 20818 I wm_on_start_called: [0,108735385,com.example.myapp.MainActivity,handleStartActivity,1]
11-04 11:59:46.614 20818 20818 I wm_on_resume_called: [0,108735385,com.example.myapp.MainActivity,RESUME_ACTIVITY,2]
11-04 11:59:46.615 20818 20818 I wm_on_top_resumed_gained_called: [108735385,com.example.myapp.MainActivity,topWhenResuming]
11-04 11:59:46.620  2430  4154 I input_focus: [Focus receive :<null>,reason=setFocusedWindow]

11-04 11:59:47.195  2430  2584 I wm_stop_activity: [0,89564982,com.miui.home/.launcher.Launcher]
11-04 11:59:47.223 23544 23544 I wm_on_stop_called: [0,89564982,com.miui.home.launcher.Launcher,STOP_ACTIVITY_ITEM,8]

```

events log常包含：

### 4.1.1 system_server端EventsLog

#### wm_create_activity

```java
//路径：services/core/java/com/android/server/wm/ActivityStarter.java
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            TaskFragment inTaskFragment, boolean restrictedBgActivity,
            NeededUriGrants intentGrants) {
    
        setInitialState(r, options, inTask, inTaskFragment, doResume, startFlags, sourceRecord,
                voiceSession, voiceInteractor, restrictedBgActivity);
                
         ......
                
        mStartActivity.logStartActivity(EventLogTags.WM_CREATE_ACTIVITY, startedTask);
        
        ......
        
}
```

#### wm_restart_activity

```java
//路径：services/core/java/com/android/server/wm/ActivityTaskSupervisor.java
boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {

			......
            EventLogTags.writeWmRestartActivity(r.mUserId, System.identityHashCode(r),
                            task.mTaskId, r.shortComponentName);
                            
            
          ......
                
}
```

####  wm_resume_activity

```java
   //路径：services/core/java/com/android/server/wm/TaskFragment.java
   final boolean resumeTopActivity(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
        ActivityRecord next = topRunningActivity(true /* focusableOnly */);
        
        ......
        
        EventLogTags.writeWmResumeActivity(next.mUserId, System.identityHashCode(next),
        next.getTask().mTaskId, next.shortComponentName);
        
        ......
        
        }
```

#### wm_set_resumed_activity

```java
/** Update AMS states when an activity is resumed. */
    void setLastResumedActivityUncheckLocked(ActivityRecord r, String reason) {


       ......
		
        EventLogTags.writeWmSetResumedActivity(
                r == null ? -1 : r.mUserId,
                r == null ? "NULL" : r.shortComponentName,
                reason);
    }
```



####  wm_pause_activity

```java
   //路径：services/core/java/com/android/server/wm/TaskFragment.java
   void schedulePauseActivity(ActivityRecord prev, boolean userLeaving,
            boolean pauseImmediately, boolean autoEnteringPip, String reason) {
        ProtoLog.v(WM_DEBUG_STATES, "Enqueueing pending pause: %s", prev);
        try {
            EventLogTags.writeWmPauseActivity(prev.mUserId, System.identityHashCode(prev),
                    prev.shortComponentName, "userLeaving=" + userLeaving, reason);

            mAtmService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
                    prev.token, PauseActivityItem.obtain(prev.finishing, userLeaving,
                            prev.configChangeFlags, pauseImmediately, autoEnteringPip));
        } catch (Exception e) {
            // Ignore exception, if process died other code will cleanup.
            Slog.w(TAG, "Exception thrown during pause", e);
            mPausingActivity = null;
            mLastPausedActivity = null;
            mTaskSupervisor.mNoHistoryActivities.remove(prev);
        }
    }
```



#### wm_add_to_stopping

```java
//路径：services/core/java/com/android/server/wm/ActivityRecord.java
void addToStopping(boolean scheduleIdle, boolean idleDelayed, String reason) {
        if (!mTaskSupervisor.mStoppingActivities.contains(this)) {
            EventLogTags.writeWmAddToStopping(mUserId, System.identityHashCode(this),
                    shortComponentName, reason);
            mTaskSupervisor.mStoppingActivities.add(this);
        }

        final Task rootTask = getRootTask();
        // If we already have a few activities waiting to stop, then give up on things going idle
        // and start clearing them out. Or if r is the last of activity of the last task the root
        // task will be empty and must be cleared immediately.
        boolean forceIdle = mTaskSupervisor.mStoppingActivities.size() > MAX_STOPPING_TO_FORCE
                || (isRootOfTask() && rootTask.getChildCount() <= 1);
        if (scheduleIdle || forceIdle) {
            ProtoLog.v(WM_DEBUG_STATES,
                    "Scheduling idle now: forceIdle=%b immediate=%b", forceIdle, !idleDelayed);

            if (!idleDelayed) {
                mTaskSupervisor.scheduleIdle();
            } else {
                mTaskSupervisor.scheduleIdleTimeout(this);
            }
        } else {
            rootTask.checkReadyForSleep();
        }
    }
```



####  wm_stop_activity

```java
//路径：services/core/java/com/android/server/wm/ActivityRecord.java 
 void stopIfPossible() {
    
            ......
            EventLogTags.writeWmStopActivity(
            mUserId, System.identityHashCode(this), shortComponentName);
            ......
            
 }
```



####  wm_finish_activity

有两个地方调用wm_finish_activity，路径均是在：services/core/java/com/android/server/wm/ActivityRecord.java。

第一个是在ActivityRecord.finishIfPossible，这是正常的结束调用。

```java
@FinishRequest int finishIfPossible(int resultCode, Intent resultData,
            NeededUriGrants resultGrants, String reason, boolean oomAdj) {
        ProtoLog.v(WM_DEBUG_STATES, "Finishing activity r=%s, result=%d, data=%s, "
                + "reason=%s", this, resultCode, resultData, reason);

		......
         EventLogTags.writeWmFinishActivity(mUserId, System.identityHashCode(this),
                    task.mTaskId, shortComponentName, reason);
         .......
                    
 }
```

第二个是在app挂掉的时候调用

```java
/**
     * Detach this activity from process and clear the references to it. If the activity is
     * finishing or has no saved state or crashed many times, it will also be removed from history.
     */
void handleAppDied() {
    
    ......
            if (!finishing || (app != null && app.isRemoved())) {
                        Slog.w(TAG, "Force removing " + this + ": app died, no saved state");
                        EventLogTags.writeWmFinishActivity(mUserId, System.identityHashCode(this),
                        task != null ? task.mTaskId : -1, shortComponentName,
                        "proc died without state saved");
            }
    ......
            
  }
```



####  wm_destroy_activity

```java
boolean destroyImmediately(String reason) {
        if (DEBUG_SWITCH || DEBUG_CLEANUP) {
            Slog.v(TAG_SWITCH, "Removing activity from " + reason + ": token=" + this
                    + ", app=" + (hasProcess() ? app.mName : "(null)"));
        }

        if (isState(DESTROYING, DESTROYED)) {
            ProtoLog.v(WM_DEBUG_STATES, "activity %s already destroying, skipping "
                    + "request with reason:%s", this, reason);
            return false;
        }

        EventLogTags.writeWmDestroyActivity(mUserId, System.identityHashCode(this),
                task.mTaskId, shortComponentName, reason);
                
                
        ......
        
        }
```



### 4.1.2 app端EventsLog

app侧所有的log路径均是在：frameworks/base/core/java/android/app/Activity.java

#### wm_on_create_called

```
final void performCreate(Bundle icicle, PersistableBundle persistentState) {
		.....
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        
         EventLogTags.writeWmOnCreateCalled(mIdent, getComponentName().getClassName(),
                "performCreate")

        mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
                com.android.internal.R.styleable.Window_windowNoDisplay, false);
        mFragments.dispatchActivityCreated();
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
        dispatchActivityPostCreated(icicle);

		.....
    }

```



#### wm_on_start_called

```
final void performStart(String reason) {
		......
        mInstrumentation.callActivityOnStart(this);
         EventLogTags.writeWmOnStartCalled(mIdent, getComponentName().getClassName(), reason);
       	......
}
```



#### wm_on_restart_called

```

/**
     * Restart the activity.
     * @param start Indicates whether the activity should also be started after restart.
     *              The option to not start immediately is needed in case a transaction with
     *              multiple lifecycle transitions is in progress.
     */
    final void performRestart(boolean start, String reason) {
			......
            mInstrumentation.callActivityOnRestart(this);
            EventLogTags.writeWmOnRestartCalled(mIdent, getComponentName().getClassName(), reason);

            if (!mCalled) {
                throw new SuperNotCalledException(
                    "Activity " + mComponent.toShortString() +
                    " did not call through to super.onRestart()");
            }
            if (start) {
                performStart(reason);
            }
        	......
    }
```



#### wm_on_resume_called

```
final void performResume(boolean followedByPause, String reason) {

		......
		
        dispatchActivityPreResumed();
        performRestart(true /* start */, reason);

		......

        mInstrumentation.callActivityOnResume(this);
        EventLogTags.writeWmOnResumeCalled(mIdent, getComponentName().getClassName(), reason);
        ......
}

```



#### wm_on_paused_called

```
final void performPause() {
		......
        onPause();
         EventLogTags.writeWmOnPausedCalled(mIdent, getComponentName().getClassName(),
                "performPause");
         ......

    }
```



#### wm_on_stop_called

```
final void performStop(boolean preserveWindow, String reason) {
			......
            long startTime = SystemClock.uptimeMillis();
            mInstrumentation.callActivityOnStop(this);
             EventLogTags.writeWmOnStopCalled(mIdent, getComponentName().getClassName(), reason);
             ......
    }
```



#### wm_on_destroy_called

```
final void performDestroy() {
		......
        onDestroy();
         EventLogTags.writeWmOnDestroyCalled(mIdent, getComponentName().getClassName(),
                 "performDestroy");
         
		......
    }
```







## 4.2 system

命令：adb logcat -b system

events log关键字常包含： START u0

```JAVA
11-04 12:02:43.604  2430  7052 I ActivityTaskManager: START u0 {act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.example.myapp/.MainActivity bnds=[582,763][754,935] (has extras)} from uid 10133 from pid 23544 callingPackage com.miui.home

```

在ActivityStarter的executeRequest方法中，在打印完start u0信息之后，后续会进一步调用startActivityUnchecked函数

```java
  //路径：services/core/java/com/android/server/wm/ActivityStarter.java
  
  /**
     * Executing activity start request and starts the journey of starting an activity. Here
     * begins with performing several preliminary checks. The normally activity launch flow will
     * go through {@link #startActivityUnchecked} to {@link #startActivityInner}.
     */
     private int executeRequest(Request request) {
         		.......
                    
				if (err == ActivityManager.START_SUCCESS) {
                	Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true,
                            true, false) + "} from uid " + callingUid + " on display "
                            + (focusedRootTask == null ? DEFAULT_DISPLAY
                            : focusedRootTask.getDisplayId()));
						}
      			 }
				......
       
}
```



# 5.关键类以及关键方法

| 类名                       | 所属进程         | 路径                                                         |
| -------------------------- | ---------------- | ------------------------------------------------------------ |
| Instrumentation            | app进程          | core/java/android/app/Instrumentation.java                   |
| ActivityThread             | app进程          | core/java/android/app/ActivityThread.java                    |
| ClientTransactionHandler   | app进程          | core/java/android/app/ClientTransactionHandler.java          |
| ClientTransaction          | app进程          | core/java/android/app/servertransaction/ClientTransaction.java |
| TransactionExecutor        | app进程          | core/java/android/app/servertransaction/TransactionExecutor.java |
| Activity                   | app进程          | core/java/android/app/Activity.java                          |
| ApplicationThread          | app进程          | core/java/android/app/ActivityThread.java                    |
| IApplicationThread         | app进程          | core/java/android/app/IApplicationThread.aidl                |
| ActivityTaskManager        | app进程          | core/java/android/app/ActivityTaskManager.java               |
| LaunchActivityItem         | app进程          | core/java/android/app/servertransaction/LaunchActivityItem.java |
| ResumeActivityItem         | app进程          | core/java/android/app/servertransaction/ResumeActivityItem.java |
| ActivityTaskManagerService | systemserver进程 | services/core/java/com/android/server/wm/ActivityTaskManagerService.java |
| ActivityManagerService     | systemserver进程 | services/core/java/com/android/server/am/ActivityManagerService.java |
| ActivityTaskSupervisor     | systemserver进程 | services/core/java/com/android/server/wm/ActivityTaskSupervisor.java |
| ClientLifecycleManager     | systemserver进程 | services/core/java/com/android/server/wm/ClientLifecycleManager.java |
| ActivityStarter            | systemserver进程 | services/core/java/com/android/server/wm/ActivityStarter.java |
| RootWindowContainer        | systemserver进程 | services/core/java/com/android/server/wm/RootWindowContainer.java |
|                            | systemserver进程 |                                                              |
|                            | systemserver进程 |                                                              |
|                            | systemserver进程 |                                                              |



##  Instrumentation

> 官方释义：
>
> /**
>
> Base class for implementing application instrumentation code.  When running  with instrumentation turned on, this class will be instantiated for you  before any of the application code, allowing you to monitor all of the  interaction the system has with the application.  An Instrumentation implementation is described to the system through an AndroidManifest.xml's  &lt;instrumentation&gt; tag.
>
> */
>
> 谷歌渣翻
>
> 用于实现应用程序检测代码的基类。 当打开检测运行时，此类将在任何应用程序代码之前为您实例化，从而允许您监视系统与应用程序的所有交互。 通过 AndroidManifest.xml 的 <instrumentation> 向系统描述 Instrumentation 实现。 标签。

是Activity与外界联系的类，activity通过Instrumentation请求构建，ActivityThread通过Instrumentation来创建和调用activity的生命周期。

| 关键函数                | 释义                                                         |
| ----------------------- | ------------------------------------------------------------ |
| newActivity             | activity通过请求mInstrumentation构建，在ActivityThread.performLaunchActivity被调用 |
| callActivityOnCreate    | 进一步调用到app的oncreate方法，在ActivityThread.performLaunchActivity被调用 |
| callActivityOnXXXXXX    | 会有很多类似写法的方法进一步调用到activity的生命周期中，在ActivityThread.performXXXXXActivity被调用 |
| callApplicationOnCreate | 用于构造app进程，在ActivityThread.handleBindApplication被调用 |



## ActivityThread

> 官方释义
>
> /**
> This manages the execution of the main thread in an application process, scheduling and executing activities, broadcasts, and other operations on it as the activity manager requests.
>
> {@hide}
> */
>
> 谷歌渣翻
>
> 它管理应用程序进程中主线程的执行，根据活动管理器的请求调度和执行活动、广播和其他操作。

ActivityThread就是应用进程中的唯一实例，负责对Activity的创建和管理，其内部类ApplicationThread是app进程和系统进程（systemserver）通信的类，只负责通信，把AMS的任务交个ActivityThread。

同时，它继承于ClientTransactionHandler，实现了handleXXXXXActivity等等的抽象类。

| 关键函数            | 释义                                                         |
| ------------------- | ------------------------------------------------------------ |
| attach              | 在ActivityThread.systemMain或者main函数调用                  |
| handleXXXXXActivity | 准备去执行生命周期相关的操作，例如create，resume，pause等等，在XXXXXActivityItem类中被调用 |
|                     |                                                              |
|                     |                                                              |



## ApplicationThread

是ActivityThread的一个内部类，继承IApplicationThread.Stub，是一个IBinder，是ActivityThread与ActivityManagerService通信的桥梁；



## ClientTransactionHandler

> 官方释义
>
> /**
>  Defines operations that a {@link  android.app.servertransaction.ClientTransaction} or its items  can perform on client.
>
> @hide
> */
>
> 谷歌渣翻
>
> 定义 {@link android.app.servertransaction.ClientTransaction} 或其具体的items可以在客户端上执行的操作。

被ActiivtyThread所继承

| 关键函数            | 释义                                                         |
| ------------------- | ------------------------------------------------------------ |
| sendMessage         | 定义了一个抽象类，被ActivityThread具体实现                   |
| scheduleTransaction | 发送一个EXECUTE_TRANSACTION的消息，被ActivityThread.scheduleTransaction所调用 |
|                     |                                                              |
|                     |                                                              |



## ClientTransaction

> ## 官方释义
>
> /**
> A container that holds a sequence of messages, which may be sent to a client.
> This includes a list of callbacks and a final lifecycle state.
>
> @see com.android.server.am.ClientLifecycleManager
>
> @see ClientTransactionItem
>
> @see ActivityLifecycleItem
>
> @hide
> */
>
> 谷歌渣翻
>
> 保存一系列消息的容器，这些消息可以发送到客户端。这包括回调列表和最终生命周期状态。

等待补充：



| 关键函数                 | 释义                                                         |
| ------------------------ | ------------------------------------------------------------ |
| schedule                 | 进一步调用ActivityThread去sendMeassage，被ClientLifecycleManager.scheduleTransaction所调用 |
| addCallback              | 将消息添加到回调序列的末尾，在activity启动过程中被ActivityTaskSupervisor.realStartActivityLocked所调用，添加的回调是LaunchActivityItem |
| getCallbacks             | 获取到回调的列表mActivityCallbacks                           |
| setLifecycleStateRequest | 设置客户端执行事务后应处于的生命周期状态，在activity启动过程中被ActivityTaskSupervisor.realStartActivityLocked所调用，设置的最终状态是ResumeActivityItem |
| getLifecycleStateRequest | 获取目标状态生命周期请求。                                   |



## TransactionExecutor

> 官方释义
>
> /**
>
> Class that manages transaction execution in the correct order.
>
> @hide
> */
>
> 谷歌渣翻
>
> 以正确的顺序管理事务执行的类。

等待补充：



| 关键函数                 | 释义                                                         |
| ------------------------ | ------------------------------------------------------------ |
| execute                  | 在ActivityThread去handleMeassage时，处理EXECUTE_TRANSACTION消息 |
| executeCallbacks         | 循环遍历回调请求的所有状态并在适当的时间执行它们，在TransactionExecutor.execute所调用。这里执行的是LaunchActivityItem |
| executeLifecycleState    | 如果满足转换请求，则转换到最终状态，在TransactionExecutor.execute所调用。这里执行的是ResumeActivityItem |
| cycleToPath              | 设置客户端执行事务后应处于的生命周期状态，在activity启动过程中被ActivityTaskSupervisor.realStartActivityLocked所调用，设置的最终状态是ResumeActivityItem |
| performLifecycleSequence | 通过先前初始化的状态序列转换客户端，被cycleToPath调用        |



## Activity

生命周期相关的类

| 关键函数      | 释义                                      |
| ------------- | ----------------------------------------- |
| startActivity | 生命周期开始的入口，像AMS请求创建Activity |
| onCreate      |                                           |
| onResume      |                                           |
|               |                                           |
|               |                                           |



## ActivityTaskManager

> 官方释义
>
> /**
>
> This class gives information about, and interacts with activities and their containers like task,stacks, and displays.
> *@hide
> */
>
> 谷歌渣翻
>
> 此类提供有关Activity及其WindowContainer（ task,stacks, and displays）的信息并与之交互。

待补充



## LaunchActivityItem

请求启动Activity的类。继承于ClientTransactionItem



| 关键函数 | 释义                                                         |
| -------- | ------------------------------------------------------------ |
| obtain   | 根据提供的参初始化一个LaunchActivityItem的实例               |
| execute  | 进一步调用ClientTransactionHandler.handleLaunchActivity,被TransactionExecutor.execute所调用 |
|          |                                                              |
|          |                                                              |
|          |                                                              |



## ResumeActivityItem

请求将activity的状态切换为resume的状态的类，继承于ActivityLifecycleItem。



| 关键函数 | 释义                                                         |
| -------- | ------------------------------------------------------------ |
| obtain   | 根据提供的参初始化一个ResumeActivityItem的实例               |
| execute  | 进一步调用ClientTransactionHandler.handleResumeActivity,被TransactionExecutor.execute所调用 |
|          |                                                              |
|          |                                                              |
|          |                                                              |



## ActivityTaskManagerService

> 官方释义
>
> /**
>
> System service for managing activities and their containers (task, displays,... ).
> *{@hide}
> */
>
> 谷歌渣翻
>
> 用于管理活动及其容器（任务、显示等）的系统服务。 

ATMS的主要作用是管理Activity的生命周期，消息派发，内存管理等等。

待补充



## ActivityTaskSupervisor







## ClientLifecycleManager

> 官方释义
>
> /**
> Class that is able to combine multiple client lifecycle transition requests and/or callbacks, and execute them as a single transaction.
>  @see ClientTransaction
>  */
>
> 谷歌渣翻
>
> 能够组合多个客户端生命周期转换的请求或者回调，探后在每一个transaction中去执行

关键函数就是scheduleTransaction，它是系统进程调用到app进程的一个函数。

```java
    void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
        final IApplicationThread client = transaction.getClient();
        transaction.schedule();
        if (!(client instanceof Binder)) {
            // If client is not an instance of Binder - it's a remote call and at this point it is
            // safe to recycle the object. All objects used for local calls will be recycled after
            // the transaction is executed on client in ActivityThread.
            transaction.recycle();
        }
    }
```



## ActivityStarter

> 官方释义
>
> /**
> Controller for interpreting how and then launching an activity.
>
> This class collects all the logic for determining how an intent and flags should be turned into an activity and associated task and root task.
>  */
>
> 谷歌渣翻
>
> 用于解释如何启动Activity的控制器。
>
> 此类收集用于确定如何将intent和flag转换为Activity以及关联任务和根任务的所有逻辑。



| 关键函数       | 释义                                                         |
| -------------- | ------------------------------------------------------------ |
| execute        | 根据前面提供的请求参数解析必要的信息，并执行mRequest，然后进一步调用executeRequest |
| executeRequest | 执行Activity启动请求，开始启动Activity,被ActivityStarter.execute调用，接下来调用startActivityUnchecked。 |
|                |                                                              |
|                |                                                              |
|                |                                                              |





## RootWindowContainer

设备的根，继承于WindowContainer。







# 6.一些相关日志

## 从桌面点击图标测试Myapp

在core/java/android/app/Activity.java中添加log打印堆栈

```java
    public void startActivity(Intent intent, @Nullable Bundle options) {
        Slog.e("SETTINGERROR"," Activity#startActivity "
                + " this = " + this
                + " 当前的进程号 = " + Process.myPid()
                + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
        getAutofillClientController().onStartActivity(intent, mIntent);
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }
```

在core/java/android/app/ActivityThread.java添加log打印堆栈

```java
/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }
		......
        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            Slog.e("SETTINGERROR"," ActivityThread#performLaunchActivity "
                    + " 当前的Activity是 = " + activity
                    + " 当前的进程号 = " + Process.myPid()
                    + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
            ......
        }


/**
     * Resume the activity.
     * @param r Target activity record.
     * @param finalStateRequest Flag indicating if this is part of final state resolution for a
     *                          transaction.
     * @param reason Reason for performing the action.
     *
     * @return {@code true} that was resumed, {@code false} otherwise.
     */
    @VisibleForTesting
public boolean performResumeActivity(ActivityClientRecord r, boolean finalStateRequest,
            String reason) {
        if (localLOGV) {
            Slog.v(TAG, "Performing resume of " + r + " finished=" + r.activity.mFinished);
        }
        if (r.activity.mFinished) {
            return false;
        }

       ......
           
        try {
            Slog.e("SETTINGERROR"," ActivityThread#performResumeActivity "
                    + " 当前的Activity是 = " + r.activity
                    + " 当前的进程号 = " + Process.myPid()
                    + " 当前的进程名称 = " + Process.myProcessName()
                    + " 当前的生命周期状态是 = " + r.getLifecycleState(),new Throwable());
            r.activity.performResume(r.startsNotResumed, reason);

            r.setState(ON_RESUME);

            reportTopResumedActivityChanged(r, r.isTopResumedActivity, "topWhenResuming");
        } catch (Exception e) {
            if (!mInstrumentation.onException(r.activity, e)) {
                throw new RuntimeException("Unable to resume activity "
                        + r.intent.getComponent().toShortString() + ": " + e.toString(), e);
            }
        }
        return true;
    }

```

路径：services/core/java/com/android/server/wm/ActivityTaskSupervisor.java

```java
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        final WindowProcessController wpc =
                mService.getProcessController(r.processName, r.info.applicationInfo.uid);

        boolean knownToBeDead = false;
        if (wpc != null && wpc.hasThread()) {
            try {
                if (mPerfBoost != null) {
                    Slog.i(TAG, "The Process " + r.processName + " Already Exists in BG. So sending its PID: " + wpc.getPid());
                    mPerfBoost.perfHint(BoostFramework.VENDOR_HINT_FIRST_LAUNCH_BOOST, r.processName, wpc.getPid(), BoostFramework.Launch.TYPE_START_APP_FROM_BG);
                }
                Slog.e("SETTINGERROR"," ActivityTaskSupervisor#startSpecificActivity+realStartActivityLocked "
                        + " 当前的activity = " + r
                        + " 当前的进程号 = " + Process.myPid()
                        + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
                realStartActivityLocked(r, wpc, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
            knownToBeDead = true;
            // Remove the process record so it won't be considered as alive.
            mService.mProcessNames.remove(wpc.mName, wpc.mUid);
            mService.mProcessMap.remove(wpc.getPid());
        }

        r.notifyUnknownVisibilityLaunchedForKeyguardTransition();
        // MIUI ADD:
        r.isColdStart = true;

        final boolean isTop = andResume && r.isTopRunningActivity();
        Slog.e("SETTINGERROR"," ActivityTaskSupervisor#startSpecificActivity+startProcessAsync "
                + " 当前的activity = " + r
                + " 当前的进程号 = " + Process.myPid()
                + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
        mService.startProcessAsync(r, knownToBeDead, isTop,
                isTop ? HostingRecord.HOSTING_TYPE_TOP_ACTIVITY
                        : HostingRecord.HOSTING_TYPE_ACTIVITY);
    }
```

路径：services/core/java/com/android/server/wm/ActivityStarter.java

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            TaskFragment inTaskFragment, boolean restrictedBgActivity,
            NeededUriGrants intentGrants) {
        int result = START_CANCELED;
        final Task startedActivityRootTask;

        // Create a transition now to record the original intent of actions taken within
        // startActivityInner. Otherwise, logic in startActivityInner could start a different
        // transition based on a sub-action.
        // Only do the create here (and defer requestStart) since startActivityInner might abort.
        final TransitionController transitionController = r.mTransitionController;
        Transition newTransition = (!transitionController.isCollecting()
                && transitionController.getTransitionPlayer() != null)
                ? transitionController.createTransition(TRANSIT_OPEN) : null;
        RemoteTransition remoteTransition = r.takeRemoteTransition();
        if (newTransition != null && remoteTransition != null) {
            newTransition.setRemoteTransition(remoteTransition);
        }
        transitionController.collect(r);
        try {
            mService.deferWindowLayout();
            Slog.e("SETTINGERROR"," ActivityStarter#startActivityUnchecked "
                    + " 当前的进程号 = " + Process.myPid()
                    + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
            result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, inTaskFragment, restrictedBgActivity,
                    intentGrants);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
            startedActivityRootTask = handleStartResult(r, options, result, newTransition,
                    remoteTransition);
            mService.continueWindowLayout();
        }

        postStartActivityProcessing(r, result, startedActivityRootTask);

        return result;
    }
```

路径： services/core/java/com/android/server/wm/ActivityTaskManagerService.java

```
    public final int startActivity(IApplicationThread caller, String callingPackage,
            String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
            String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
            Bundle bOptions) {
        Slog.e("SETTINGERROR"," ActivityTaskManagerService#startActivity "
                + " callingPackage = " + callingPackage
                + " 当前的进程号 = " + Process.myPid()
                + " 当前的进程名称 = " + Process.myProcessName(),new Throwable());
        return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
                resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
                UserHandle.getCallingUserId());
    }
```





### 冷启动 打印的堆栈信息

```
//点击桌面图标，从launcher进程开始向systemserver进程发起请求【当前进程是launcher进程】
11-01 10:28:07.717  3635  3635 E SETTINGERROR:  Activity#startActivity  this = com.miui.home.launcher.Launcher@672e82f 当前的进程号 = 3635 当前的进程名称 = com.miui.home
11-01 10:28:07.717  3635  3635 E SETTINGERROR: java.lang.Throwable
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.app.Activity.startActivity(Activity.java:6137)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.startActivity(Launcher.java:9891)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.startActivity(Launcher.java:5034)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher$PerformLaunchAction.run(Launcher.java:4937)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher$PerformLaunchAction.launch(Launcher.java:4918)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.launch(Launcher.java:4900)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.ShortcutInfo.handleClick(ShortcutInfo.java:798)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.appOnClick(Launcher.java:4865)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.onClick(Launcher.java:4846)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.View.performClick(View.java:7600)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.View.performClickInternal(View.java:7577)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.View.-$$Nest$mperformClickInternal(Unknown Source:0)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.View$PerformClick.run(View.java:30186)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.ItemIcon.post(ItemIcon.java:429)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.View.onTouchEvent(View.java:16637)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.ItemIcon.onTouchEvent(ItemIcon.java:407)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.View.dispatchTouchEvent(View.java:15160)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3139)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2802)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3188)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.CellLayout.dispatchTouchEvent(CellLayout.java:480)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Workspace.dispatchTouchEvent(Workspace.java:732)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.SuperDraglayer.dispatchTouchEvent(SuperDraglayer.java:257)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.ShortcutMenuLayer.dispatchTouchEvent(ShortcutMenuLayer.java:140)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.android.internal.policy.DecorView.superDispatchTouchEvent(DecorView.java:579)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.android.internal.policy.PhoneWindow.superDispatchTouchEvent(PhoneWindow.java:1903)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.app.Activity.dispatchTouchEvent(Activity.java:4456)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.dispatchTouchEvent(Launcher.java:4706)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.android.internal.policy.DecorView.dispatchTouchEvent(DecorView.java:537)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.View.dispatchPointerEvent(View.java:15435)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$ViewPostImeInputStage.processPointerEvent(ViewRootImpl.java:7315)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$ViewPostImeInputStage.onProcess(ViewRootImpl.java:7085)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:6504)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:6561)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:6527)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:6692)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:6535)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$AsyncInputStage.apply(ViewRootImpl.java:6749)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:6508)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:6561)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:6527)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:6535)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:6508)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl.deliverInputEvent(ViewRootImpl.java:9713)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl.doProcessInputEvents(ViewRootImpl.java:9664)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl.enqueueInputEvent(ViewRootImpl.java:9632)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$WindowInputEventReceiver.onInputEvent(ViewRootImpl.java:9869)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.view.InputEventReceiver.dispatchInputEvent(InputEventReceiver.java:298)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.os.MessageQueue.nativePollOnce(Native Method)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.os.MessageQueue.next(MessageQueue.java:341)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.os.Looper.loopOnce(Looper.java:169)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.os.Looper.loop(Looper.java:300)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at android.app.ActivityThread.main(ActivityThread.java:8512)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at java.lang.reflect.Method.invoke(Native Method)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:561)
11-01 10:28:07.717  3635  3635 E SETTINGERROR: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:954)

//通过Binder调用到systemServer进程，调用到ATMS的startActivity【当前的进程名称 是system_server】
11-01 10:28:07.719  2430  5074 E SETTINGERROR:  ActivityTaskManagerService#startActivity  callingPackage = com.miui.home 当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:28:07.719  2430  5074 E SETTINGERROR: java.lang.Throwable
11-01 10:28:07.719  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivity(ActivityTaskManagerService.java:1310)
11-01 10:28:07.719  2430  5074 E SETTINGERROR: 	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:1224)
11-01 10:28:07.719  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.onTransact(ActivityTaskManagerService.java:5613)
11-01 10:28:07.719  2430  5074 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1290)
11-01 10:28:07.719  2430  5074 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)

//进一步调用去 ActivityStarter#startActivityUnchecked
11-01 10:28:07.761  2430  5074 E SETTINGERROR:  ActivityStarter#startActivityUnchecked  当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:28:07.761  2430  5074 E SETTINGERROR: java.lang.Throwable
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1863)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.executeRequest(ActivityStarter.java:1405)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.execute(ActivityStarter.java:805)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1373)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1336)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivity(ActivityTaskManagerService.java:1311)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:1224)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.onTransact(ActivityTaskManagerService.java:5613)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1290)
11-01 10:28:07.761  2430  5074 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)


//由于是第一次打开该应用（进程），或者说是冷启动，走了ActivityTaskSupervisor.startSpecificActivity+startProcessAsync 【当前进程是system_server进程】
11-01 10:28:07.790  2430  7121 E SETTINGERROR:  ActivityTaskSupervisor#startSpecificActivity+startProcessAsync  当前的activity = ActivityRecord{fb57046 u0 com.example.myapp/.MainActivity} t9} 当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:28:07.790  2430  7121 E SETTINGERROR: java.lang.Throwable
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskSupervisor.startSpecificActivity(ActivityTaskSupervisor.java:1170)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.TaskFragment.resumeTopActivity(TaskFragment.java:1667)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task.resumeTopActivityInnerLocked(Task.java:5901)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task.resumeTopActivityUncheckedLocked(Task.java:5824)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities(RootWindowContainer.java:2534)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities(RootWindowContainer.java:2520)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.TaskFragment.completePause(TaskFragment.java:1933)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityRecord.activityPaused(ActivityRecord.java:7065)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityClientController.activityPaused(ActivityClientController.java:195)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at android.app.IActivityClientController$Stub.onTransact(IActivityClientController.java:582)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityClientController.onTransact(ActivityClientController.java:133)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1290)
11-01 10:28:07.790  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)
11-01 10:28:07.791  2430  7121 E SETTINGERROR:  ActivityTaskSupervisor#startSpecificActivity+startProcessAsync  当前的activity = ActivityRecord{fb57046 u0 com.example.myapp/.MainActivity} t9} 当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:28:07.791  2430  7121 E SETTINGERROR: java.lang.Throwable
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskSupervisor.startSpecificActivity(ActivityTaskSupervisor.java:1170)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.EnsureActivitiesVisibleHelper.makeVisibleAndRestartIfNeeded(EnsureActivitiesVisibleHelper.java:287)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.EnsureActivitiesVisibleHelper.setActivityVisibilityState(EnsureActivitiesVisibleHelper.java:211)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.EnsureActivitiesVisibleHelper.process(EnsureActivitiesVisibleHelper.java:144)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.TaskFragment.updateActivityVisibilities(TaskFragment.java:1186)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task.lambda$ensureActivitiesVisible$23(Task.java:5720)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task$$ExternalSyntheticLambda23.accept(Unknown Source:10)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task.forAllLeafTasks(Task.java:3727)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task.ensureActivitiesVisible(Task.java:5719)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.DisplayContent.lambda$ensureActivitiesVisible$47$com-android-server-wm-DisplayContent(DisplayContent.java:6741)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.DisplayContent$$ExternalSyntheticLambda41.accept(Unknown Source:13)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task.forAllRootTasks(Task.java:3739)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2165)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2165)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2165)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2165)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2165)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2165)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.WindowContainer.forAllRootTasks(WindowContainer.java:2158)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.DisplayContent.ensureActivitiesVisible(DisplayContent.java:6736)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.RootWindowContainer.ensureActivitiesVisible(RootWindowContainer.java:2002)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.RootWindowContainer.ensureActivitiesVisible(RootWindowContainer.java:1983)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.TaskFragment.completePause(TaskFragment.java:1953)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityRecord.activityPaused(ActivityRecord.java:7065)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityClientController.activityPaused(ActivityClientController.java:195)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at android.app.IActivityClientController$Stub.onTransact(IActivityClientController.java:582)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityClientController.onTransact(ActivityClientController.java:133)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1290)
11-01 10:28:07.791  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)


//systemserver进程handlemessage，调用到LaunchActivityItem去创建一个activity。
//注意这里是从TransactionExecutor.execute->TransactionExecutor.executeCallbacks【当前进程是app进程】
11-01 10:28:07.929  8599  8599 E SETTINGERROR:  ActivityThread#performLaunchActivity  当前的Activity是 = com.example.myapp.MainActivity@1cd8cc6 当前的进程号 = 8599 当前的进程名称 = com.example.myapp
11-01 10:28:07.929  8599  8599 E SETTINGERROR: java.lang.Throwable
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3813)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:4077)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:101)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2426)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.os.Handler.dispatchMessage(Handler.java:106)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.os.Looper.loopOnce(Looper.java:211)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.os.Looper.loop(Looper.java:300)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.main(ActivityThread.java:8512)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at java.lang.reflect.Method.invoke(Native Method)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:561)
11-01 10:28:07.929  8599  8599 E SETTINGERROR: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:954)


//在launch新的activitty之后，继续通过TransactionExecutor.execute->TransactionExecutor.executeLifecycleState开始resume的过程
11-01 09:39:37.704  8723  8723 E SETTINGERROR:  ActivityThread#performResumeActivity  当前的Activity是 = com.example.myapp.MainActivity@d99ad51 当前的进程号 = 8723 当前的进程名称 = com.example.myapp 当前的生命周期状态是 = 2
11-01 10:28:08.018  8599  8599 E SETTINGERROR:  ActivityThread#performResumeActivity  当前的Activity是 = com.example.myapp.MainActivity@1cd8cc6 当前的进程号 = 8599 当前的进程名称 = com.example.myapp 当前的生命周期状态是 = 2
11-01 10:28:08.018  8599  8599 E SETTINGERROR: java.lang.Throwable
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.performResumeActivity(ActivityThread.java:5087)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.handleResumeActivity(ActivityThread.java:5131)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.ResumeActivityItem.execute(ResumeActivityItem.java:54)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.ActivityTransactionItem.execute(ActivityTransactionItem.java:45)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:176)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2426)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.os.Handler.dispatchMessage(Handler.java:106)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.os.Looper.loopOnce(Looper.java:211)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.os.Looper.loop(Looper.java:300)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.main(ActivityThread.java:8512)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at java.lang.reflect.Method.invoke(Native Method)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:561)
11-01 10:28:08.018  8599  8599 E SETTINGERROR: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:954)


```



## 热启动 打印的堆栈信息

返回桌面后再次热启动Myapp图标，只有桌面发起，然后resume Myapp的过程

```
//从桌面点击图标热启动
11-01 10:34:02.823  3635  3635 E SETTINGERROR:  Activity#startActivity  this = com.miui.home.launcher.Launcher@672e82f 当前的进程号 = 3635 当前的进程名称 = com.miui.home
11-01 10:34:02.823  3635  3635 E SETTINGERROR: java.lang.Throwable
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.app.Activity.startActivity(Activity.java:6137)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.startActivity(Launcher.java:9891)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.startActivity(Launcher.java:5034)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher$PerformLaunchAction.run(Launcher.java:4937)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher$PerformLaunchAction.launch(Launcher.java:4918)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.launch(Launcher.java:4900)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.ShortcutInfo.handleClick(ShortcutInfo.java:798)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.appOnClick(Launcher.java:4865)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.onClick(Launcher.java:4846)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.View.performClick(View.java:7600)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.View.performClickInternal(View.java:7577)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.View.-$$Nest$mperformClickInternal(Unknown Source:0)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.View$PerformClick.run(View.java:30186)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.ItemIcon.post(ItemIcon.java:429)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.View.onTouchEvent(View.java:16637)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.ItemIcon.onTouchEvent(ItemIcon.java:407)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.View.dispatchTouchEvent(View.java:15160)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3139)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2802)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.CellLayout.dispatchTouchEvent(CellLayout.java:480)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Workspace.dispatchTouchEvent(Workspace.java:732)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.SuperDraglayer.dispatchTouchEvent(SuperDraglayer.java:257)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.ShortcutMenuLayer.dispatchTouchEvent(ShortcutMenuLayer.java:140)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTransformedTouchEvent(ViewGroup.java:3152)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewGroup.dispatchTouchEvent(ViewGroup.java:2820)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.android.internal.policy.DecorView.superDispatchTouchEvent(DecorView.java:579)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.android.internal.policy.PhoneWindow.superDispatchTouchEvent(PhoneWindow.java:1903)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.app.Activity.dispatchTouchEvent(Activity.java:4456)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.miui.home.launcher.Launcher.dispatchTouchEvent(Launcher.java:4706)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.android.internal.policy.DecorView.dispatchTouchEvent(DecorView.java:537)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.View.dispatchPointerEvent(View.java:15435)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$ViewPostImeInputStage.processPointerEvent(ViewRootImpl.java:7315)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$ViewPostImeInputStage.onProcess(ViewRootImpl.java:7085)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:6504)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:6561)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:6527)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$AsyncInputStage.forward(ViewRootImpl.java:6692)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:6535)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$AsyncInputStage.apply(ViewRootImpl.java:6749)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:6508)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.onDeliverToNext(ViewRootImpl.java:6561)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.forward(ViewRootImpl.java:6527)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.apply(ViewRootImpl.java:6535)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$InputStage.deliver(ViewRootImpl.java:6508)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl.deliverInputEvent(ViewRootImpl.java:9713)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl.doProcessInputEvents(ViewRootImpl.java:9664)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl.enqueueInputEvent(ViewRootImpl.java:9632)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.ViewRootImpl$WindowInputEventReceiver.onInputEvent(ViewRootImpl.java:9869)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.view.InputEventReceiver.dispatchInputEvent(InputEventReceiver.java:298)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.os.MessageQueue.nativePollOnce(Native Method)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.os.MessageQueue.next(MessageQueue.java:341)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.os.Looper.loopOnce(Looper.java:169)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.os.Looper.loop(Looper.java:300)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at android.app.ActivityThread.main(ActivityThread.java:8512)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at java.lang.reflect.Method.invoke(Native Method)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:561)
11-01 10:34:02.823  3635  3635 E SETTINGERROR: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:954)


//从launcher进程调用到systemserver进程，调用到ActivityTaskManagerService.startActivity，当前进程是systemserver进程
11-01 10:34:02.824  2430  5074 E SETTINGERROR:  ActivityTaskManagerService#startActivity  callingPackage = com.miui.home 当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:34:02.824  2430  5074 E SETTINGERROR: java.lang.Throwable
11-01 10:34:02.824  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivity(ActivityTaskManagerService.java:1310)
11-01 10:34:02.824  2430  5074 E SETTINGERROR: 	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:1224)
11-01 10:34:02.824  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.onTransact(ActivityTaskManagerService.java:5613)
11-01 10:34:02.824  2430  5074 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1290)
11-01 10:34:02.824  2430  5074 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)

//进一步调用到ActivityStarter#startActivityUnchecked
11-01 10:34:02.831  2430  5074 E SETTINGERROR:  ActivityStarter#startActivityUnchecked  当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:34:02.831  2430  5074 E SETTINGERROR: java.lang.Throwable
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1863)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.executeRequest(ActivityStarter.java:1405)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.execute(ActivityStarter.java:805)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1373)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1336)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivity(ActivityTaskManagerService.java:1311)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:1224)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.onTransact(ActivityTaskManagerService.java:5613)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1290)
11-01 10:34:02.831  2430  5074 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)


//热启动，直接开始进行resume的过程。因为在executeCallbacks中获取到的transaction.getCallbacks()为null，因此直接返回，没有执行launchactivity的流程
11-01 10:34:02.872  8599  8599 E SETTINGERROR:  ActivityThread#performResumeActivity  当前的Activity是 = com.example.myapp.MainActivity@1cd8cc6 当前的进程号 = 8599 当前的进程名称 = com.example.myapp 当前的生命周期状态是 = 2
11-01 10:34:02.872  8599  8599 E SETTINGERROR: java.lang.Throwable
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.performResumeActivity(ActivityThread.java:5087)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.handleResumeActivity(ActivityThread.java:5131)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.ResumeActivityItem.execute(ResumeActivityItem.java:54)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.ActivityTransactionItem.execute(ActivityTransactionItem.java:45)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:176)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2426)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.os.Handler.dispatchMessage(Handler.java:106)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.os.Looper.loopOnce(Looper.java:211)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.os.Looper.loop(Looper.java:300)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.main(ActivityThread.java:8512)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at java.lang.reflect.Method.invoke(Native Method)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:561)
11-01 10:34:02.872  8599  8599 E SETTINGERROR: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:954)



```



## 应用内启动其他activity

### 应用内启动其他activity堆栈信息

```
//在myapp应用中点击启动另一个activity【当前进程是app进程】
11-01 10:35:22.530  8599  8599 E SETTINGERROR:  Activity#startActivity  this = com.example.myapp.MainActivity@1cd8cc6 当前的进程号 = 8599 当前的进程名称 = com.example.myapp
11-01 10:35:22.530  8599  8599 E SETTINGERROR: java.lang.Throwable
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.app.Activity.startActivity(Activity.java:6137)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.app.Activity.startActivity(Activity.java:6107)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at com.example.myapp.MainActivity.SwitchToApp7(MainActivity.java:171)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at com.example.myapp.MainActivity.onClick(MainActivity.java:97)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.view.View.performClick(View.java:7600)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at com.google.android.material.button.MaterialButton.performClick(MaterialButton.java:1131)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.view.View.performClickInternal(View.java:7577)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.view.View.-$$Nest$mperformClickInternal(Unknown Source:0)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.view.View$PerformClick.run(View.java:30186)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.os.Handler.handleCallback(Handler.java:942)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.os.Handler.dispatchMessage(Handler.java:99)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.os.Looper.loopOnce(Looper.java:211)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.os.Looper.loop(Looper.java:300)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.main(ActivityThread.java:8512)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at java.lang.reflect.Method.invoke(Native Method)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:561)
11-01 10:35:22.530  8599  8599 E SETTINGERROR: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:954)

//调用到systemserver侧的startActivity
11-01 10:35:22.532  2430  7121 E SETTINGERROR:  ActivityTaskManagerService#startActivity  callingPackage = com.example.myapp 当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:35:22.532  2430  7121 E SETTINGERROR: java.lang.Throwable
11-01 10:35:22.532  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivity(ActivityTaskManagerService.java:1310)
11-01 10:35:22.532  2430  7121 E SETTINGERROR: 	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:1224)
11-01 10:35:22.532  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.onTransact(ActivityTaskManagerService.java:5613)
11-01 10:35:22.532  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1285)
11-01 10:35:22.532  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)

//进一步调用ActivityStarter#startActivityUnchecked
11-01 10:35:22.540  2430  7121 E SETTINGERROR:  ActivityStarter#startActivityUnchecked  当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:35:22.540  2430  7121 E SETTINGERROR: java.lang.Throwable
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.startActivityUnchecked(ActivityStarter.java:1863)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.executeRequest(ActivityStarter.java:1405)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityStarter.execute(ActivityStarter.java:805)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1373)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivityAsUser(ActivityTaskManagerService.java:1336)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.startActivity(ActivityTaskManagerService.java:1311)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at android.app.IActivityTaskManager$Stub.onTransact(IActivityTaskManager.java:1224)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskManagerService.onTransact(ActivityTaskManagerService.java:5613)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1285)
11-01 10:35:22.540  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)


//当前进程已经存在，调用realStartActivityLocked【当前进程是systemserver进程】
11-01 10:35:22.561  2430  7121 E SETTINGERROR:  ActivityTaskSupervisor#startSpecificActivity+realStartActivityLocked  当前的activity = ActivityRecord{ccb20f0 u0 com.example.myapp/.MainActivity3} t9} 当前的进程号 = 2430 当前的进程名称 = system_server
11-01 10:35:22.561  2430  7121 E SETTINGERROR: java.lang.Throwable
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityTaskSupervisor.startSpecificActivity(ActivityTaskSupervisor.java:1146)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.TaskFragment.resumeTopActivity(TaskFragment.java:1667)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task.resumeTopActivityInnerLocked(Task.java:5901)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.Task.resumeTopActivityUncheckedLocked(Task.java:5824)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities(RootWindowContainer.java:2534)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.RootWindowContainer.resumeFocusedTasksTopActivities(RootWindowContainer.java:2520)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.TaskFragment.completePause(TaskFragment.java:1933)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityRecord.activityPaused(ActivityRecord.java:7065)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityClientController.activityPaused(ActivityClientController.java:195)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at android.app.IActivityClientController$Stub.onTransact(IActivityClientController.java:582)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at com.android.server.wm.ActivityClientController.onTransact(ActivityClientController.java:133)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransactInternal(Binder.java:1285)
11-01 10:35:22.561  2430  7121 E SETTINGERROR: 	at android.os.Binder.execTransact(Binder.java:1249)


//处理msg开始launch新的activity
11-01 10:35:22.577  8599  8599 E SETTINGERROR:  ActivityThread#performLaunchActivity  当前的Activity是 = com.example.myapp.MainActivity3@9a1b6a6 当前的进程号 = 8599 当前的进程名称 = com.example.myapp
11-01 10:35:22.577  8599  8599 E SETTINGERROR: java.lang.Throwable
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:3813)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:4077)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.LaunchActivityItem.execute(LaunchActivityItem.java:101)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.executeCallbacks(TransactionExecutor.java:135)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:95)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2426)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.os.Handler.dispatchMessage(Handler.java:106)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.os.Looper.loopOnce(Looper.java:211)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.os.Looper.loop(Looper.java:300)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.main(ActivityThread.java:8512)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at java.lang.reflect.Method.invoke(Native Method)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:561)
11-01 10:35:22.577  8599  8599 E SETTINGERROR: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:954)


//处理msg开始resume activity
11-01 10:35:22.604  8599  8599 E SETTINGERROR:  ActivityThread#performResumeActivity  当前的Activity是 = com.example.myapp.MainActivity3@9a1b6a6 当前的进程号 = 8599 当前的进程名称 = com.example.myapp 当前的生命周期状态是 = 2
11-01 10:35:22.604  8599  8599 E SETTINGERROR: java.lang.Throwable
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.performResumeActivity(ActivityThread.java:5087)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.handleResumeActivity(ActivityThread.java:5131)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.ResumeActivityItem.execute(ResumeActivityItem.java:54)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.ActivityTransactionItem.execute(ActivityTransactionItem.java:45)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.executeLifecycleState(TransactionExecutor.java:176)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.app.servertransaction.TransactionExecutor.execute(TransactionExecutor.java:97)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:2426)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.os.Handler.dispatchMessage(Handler.java:106)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.os.Looper.loopOnce(Looper.java:211)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.os.Looper.loop(Looper.java:300)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at android.app.ActivityThread.main(ActivityThread.java:8512)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at java.lang.reflect.Method.invoke(Native Method)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:561)
11-01 10:35:22.604  8599  8599 E SETTINGERROR: 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:954)


```

