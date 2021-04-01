---
layout:     post
title:      "Activity"
subtitle:   "AMS的启动流程和Activity的启动流程"
date:       2021-03-11 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-FrameWork
---


# ActivityManagerService详解

## 简介

本文基于Android-R源码 从以下方面介绍AMS
1. AMS的启动流程
2. AMS启动Activity的流程

## AMS启动流程

在经过C层init启动后,SystemServer会开始启动一些引导服务,启动包括ActivityManagerService 。 其中调用了ActivityManagerService的setSystemProcess()方法 如下两段代码

SystemServer # main 

```java
 private void run() {
        TimingsTraceAndSlog t = new TimingsTraceAndSlog();
        try {
           >>>
           
            // Initialize the system context.
            createSystemContext();

            // Call per-process mainline module initialization.
            ActivityThread.initializeMainlineModules();

            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
            // Prepare the thread pool for init tasks that can be parallelized
            SystemServerInitThreadPool.start();
            // Attach JVMTI agent if this is a debuggable build and the system property is set.
            if (Build.IS_DEBUGGABLE) {
                // Property is of the form "library_path=parameters".
                String jvmtiAgent = SystemProperties.get("persist.sys.dalvik.jvmtiagent");
                if (!jvmtiAgent.isEmpty()) {
                    int equalIndex = jvmtiAgent.indexOf('=');
                    String libraryPath = jvmtiAgent.substring(0, equalIndex);
                    String parameterList =
                            jvmtiAgent.substring(equalIndex + 1, jvmtiAgent.length());
                    // Attach the agent.
                    try {
                        Debug.attachJvmtiAgent(libraryPath, parameterList, null);
                    } catch (Exception e) {
                        Slog.e("System", "*************************************************");
                        Slog.e("System", "********** Failed to load jvmti plugin: " + jvmtiAgent);
                    }
                }
            }
        } finally {
            t.traceEnd();  // InitBeforeStartServices
        }

        // Setup the default WTF handler
        RuntimeInit.setDefaultApplicationWtfHandler(SystemServer::handleEarlySystemWtf);

        // Start services.
        try {
            t.traceBegin("StartServices");
            startBootstrapServices(t); 启动引导服务
            startCoreServices(t);
            startOtherServices(t);
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            t.traceEnd(); // StartServices
        }

       >>>
    }
```

SystemServer # startBootstrapServices

```java
 private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
        
        // Activity manager runs the show.
        t.traceBegin("StartActivityManager");
        // TODO: Might need to move after migration to WM.
        ActivityTaskManagerService atm = mSystemServiceManager.startService(
                ActivityTaskManagerService.Lifecycle.class).getService();
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
                
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        mWindowManagerGlobalLock = atm.getGlobalLock();
        t.traceEnd();

       >>>

        // Start the package manager.
        if (!mRuntimeRestart) {
            FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
                    FrameworkStatsLog
                            .BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__PACKAGE_MANAGER_INIT_START,
                    SystemClock.elapsedRealtime());
        }

        t.traceBegin("StartPackageManagerService");
        try {
            Watchdog.getInstance().pauseWatchingCurrentThread("packagemanagermain");
            mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                    mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        } finally {
            Watchdog.getInstance().resumeWatchingCurrentThread("packagemanagermain");
        }
      >>>
      
      
        // Set up the Application instance for the system process and get started.
        t.traceBegin("SetSystemProcess");
        mActivityManagerService.setSystemProcess(); 
        t.traceEnd();

    }

```

经过SystemServiceManager的StartService  先通过反射得到Service对象 。

SystemServiceManager 方法如下

SystemServiceManager # StartService 

```java
  public <T extends SystemService> T startService(Class<T> serviceClass) {
        try {
            final String name = serviceClass.getName();
            Slog.i(TAG, "Starting " + name);
            Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartService " + name);

            // Create the service.
            if (!SystemService.class.isAssignableFrom(serviceClass)) {
                throw new RuntimeException("Failed to create " + name
                        + ": service must extend " + SystemService.class.getName());
            }
            final T service;
            try {
                Constructor<T> constructor = serviceClass.getConstructor(Context.class);
                service = constructor.newInstance(mContext);
            } 
            >>>
            startService(service);
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }
```
SystemServiceManager # StartService 

```java
    public void startService(@NonNull final SystemService service) {
        // Register it.
        mServices.add(service);
        // Start it.
        long time = SystemClock.elapsedRealtime();
        try {
            service.onStart();
        } catch (RuntimeException ex) {
            throw new RuntimeException("Failed to start service " + service.getClass().getName()
                    + ": onStart threw an exception", ex);
        }
        warnIfTooLong(SystemClock.elapsedRealtime() - time, service, "onStart");
    }

```

最后会调用之ActivityMangerService内部类Lifecycle（继承自SystemService）的onStart方法。最终调用ActivityManagerService的start();

ActivityManagerService # ActivityManagerService构造方法

```java

// Note: This method is invoked on the main thread but may need to attach various
    // handlers to other threads.  So take care to be explicit about the looper.
    public ActivityManagerService(Context systemContext, ActivityTaskManagerService atm) {
        LockGuard.installLock(this, LockGuard.INDEX_ACTIVITY);
        mInjector = new Injector(systemContext);
        mContext = systemContext;

        mFactoryTest = FactoryTest.getMode();
        mSystemThread = ActivityThread.currentActivityThread();
        mUiContext = mSystemThread.getSystemUiContext();

        Slog.i(TAG, "Memory class: " + ActivityManager.staticGetMemoryClass());

        mHandlerThread = new ServiceThread(TAG,
                THREAD_PRIORITY_FOREGROUND, false /*allowIo*/);
        mHandlerThread.start();
        mHandler = new MainHandler(mHandlerThread.getLooper());
        mUiHandler = mInjector.getUiHandler(this);

        mProcStartHandlerThread = new ServiceThread(TAG + ":procStart",
                THREAD_PRIORITY_FOREGROUND, false /* allowIo */);
        mProcStartHandlerThread.start();
        mProcStartHandler = new Handler(mProcStartHandlerThread.getLooper());

        mConstants = new ActivityManagerConstants(mContext, this, mHandler);
        final ActiveUids activeUids = new ActiveUids(this, true /* postChangesToAtm */);
        mPlatformCompat = (PlatformCompat) ServiceManager.getService(
                Context.PLATFORM_COMPAT_SERVICE);
        mProcessList = mInjector.getProcessList(this);
        mProcessList.init(this, activeUids, mPlatformCompat);
        mLowMemDetector = new LowMemDetector(this);
        mOomAdjuster = new OomAdjuster(this, mProcessList, activeUids);

        // Broadcast policy parameters
        final BroadcastConstants foreConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_FG_CONSTANTS);
        foreConstants.TIMEOUT = BROADCAST_FG_TIMEOUT;

        final BroadcastConstants backConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_BG_CONSTANTS);
        backConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;

        final BroadcastConstants offloadConstants = new BroadcastConstants(
                Settings.Global.BROADCAST_OFFLOAD_CONSTANTS);
        offloadConstants.TIMEOUT = BROADCAST_BG_TIMEOUT;
        // by default, no "slow" policy in this queue
        offloadConstants.SLOW_TIME = Integer.MAX_VALUE;

        mEnableOffloadQueue = SystemProperties.getBoolean(
                "persist.device_config.activity_manager_native_boot.offload_queue_enabled", false);

        mFgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "foreground", foreConstants, false);
        mBgBroadcastQueue = new BroadcastQueue(this, mHandler,
                "background", backConstants, true);
        mOffloadBroadcastQueue = new BroadcastQueue(this, mHandler,
                "offload", offloadConstants, true);
        mBroadcastQueues[0] = mFgBroadcastQueue;
        mBroadcastQueues[1] = mBgBroadcastQueue;
        mBroadcastQueues[2] = mOffloadBroadcastQueue;

        mServices = new ActiveServices(this);
        mProviderMap = new ProviderMap(this);
        mPackageWatchdog = PackageWatchdog.getInstance(mUiContext);
        mAppErrors = new AppErrors(mUiContext, this, mPackageWatchdog);

        final File systemDir = SystemServiceManager.ensureSystemDir();

        // TODO: Move creation of battery stats service outside of activity manager service.
        mBatteryStatsService = new BatteryStatsService(systemContext, systemDir,
                BackgroundThread.get().getHandler());
        mBatteryStatsService.getActiveStatistics().readLocked();
        mBatteryStatsService.scheduleWriteToDisk();
        mOnBattery = DEBUG_POWER ? true
                : mBatteryStatsService.getActiveStatistics().getIsOnBattery();
        mBatteryStatsService.getActiveStatistics().setCallback(this);
        mOomAdjProfiler.batteryPowerChanged(mOnBattery);

        mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));

        mAppOpsService = mInjector.getAppOpsService(new File(systemDir, "appops.xml"), mHandler);

        mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);

        mUserController = new UserController(this);

        mPendingIntentController = new PendingIntentController(
                mHandlerThread.getLooper(), mUserController, mConstants);

        if (SystemProperties.getInt("sys.use_fifo_ui", 0) != 0) {
            mUseFifoUiScheduling = true;
        }

        mTrackingAssociations = "1".equals(SystemProperties.get("debug.track-associations"));
        mIntentFirewall = new IntentFirewall(new IntentFirewallInterface(), mHandler);

        mActivityTaskManager = atm;
        mActivityTaskManager.initialize(mIntentFirewall, mPendingIntentController,
                DisplayThread.get().getLooper());
        mAtmInternal = LocalServices.getService(ActivityTaskManagerInternal.class);

        mProcessCpuThread = new Thread("CpuTracker") {
            @Override
            public void run() {
                synchronized (mProcessCpuTracker) {
                    mProcessCpuInitLatch.countDown();
                    mProcessCpuTracker.init();
                }
                while (true) {
                    try {
                        try {
                            synchronized(this) {
                                final long now = SystemClock.uptimeMillis();
                                long nextCpuDelay = (mLastCpuTime.get()+MONITOR_CPU_MAX_TIME)-now;
                                long nextWriteDelay = (mLastWriteTime+BATTERY_STATS_TIME)-now;
                                //Slog.i(TAG, "Cpu delay=" + nextCpuDelay
                                //        + ", write delay=" + nextWriteDelay);
                                if (nextWriteDelay < nextCpuDelay) {
                                    nextCpuDelay = nextWriteDelay;
                                }
                                if (nextCpuDelay > 0) {
                                    mProcessCpuMutexFree.set(true);
                                    this.wait(nextCpuDelay);
                                }
                            }
                        } catch (InterruptedException e) {
                        }
                        updateCpuStatsNow();
                    } catch (Exception e) {
                        Slog.e(TAG, "Unexpected exception collecting process stats", e);
                    }
                }
            }
        };

        mHiddenApiBlacklist = new HiddenApiSettings(mHandler, mContext);

        Watchdog.getInstance().addMonitor(this);
        Watchdog.getInstance().addThread(mHandler);

        // bind background threads to little cores
        // this is expected to fail inside of framework tests because apps can't touch cpusets directly
        // make sure we've already adjusted system_server's internal view of itself first
        updateOomAdjLocked(OomAdjuster.OOM_ADJ_REASON_NONE);
        try {
            Process.setThreadGroupAndCpuset(BackgroundThread.get().getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
            Process.setThreadGroupAndCpuset(
                    mOomAdjuster.mCachedAppOptimizer.mCachedAppOptimizerThread.getThreadId(),
                    Process.THREAD_GROUP_SYSTEM);
        } catch (Exception e) {
            Slog.w(TAG, "Setting background thread cpuset failed");
        }

        mInternal = new LocalService();
        mPendingStartActivityUids = new PendingStartActivityUids(mContext);
    }
```

ActivityManagerService #  start()

```java
private void start() {
        removeAllProcessGroups();
        mProcessCpuThread.start();

        mBatteryStatsService.publish();
        mAppOpsService.publish();
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, mInternal);
        mActivityTaskManager.onActivityManagerInternalAdded();
        mPendingIntentController.onActivityManagerInternalAdded();
        // Wait for the synchronized block started in mProcessCpuThread,
        // so that any other access to mProcessCpuTracker from main thread
        // will be blocked during mProcessCpuTracker initialization.
        try {
            mProcessCpuInitLatch.await();
        } catch (InterruptedException e) {
            Slog.wtf(TAG, "Interrupted wait during start", e);
            Thread.currentThread().interrupt();
            throw new IllegalStateException("Interrupted wait during start");
        }
    }
```

AMS服务会将自己的服务注册到ServiceManager中，从而支持跨进程的Binder通信

ServiceManager是管理Binder的

LocalServices 缓存了各服务的内部实现LocalService 如 LocalService extends ActivityManagerInternal这些定义在服务的内部类，其他进程可以同过LocalServices.getService()获取实例

## 疑问

1.支线逻辑的作用和AMS有无关联, 和Binder调用的关系,AMS是在systemServer进程启动吗？


## 支线逻辑

    // Initialize the system context.
        createSystemContext();
    
    // Call per-process mainline module initialization.
    ActivityThread.initializeMainlineModules();


### createSystemContext()

ActivityThread  管理着一个应用进程的主线程,用来调度和执行运行在该进程中的Activities，Broadcasrs以及其他的相关操作


## AMS-Activity调用流程

### AMS启动Activity的流程简述
    
    Activity的启动过程，我们可以从Context的startActivity说起，其实现是ContextImpl的startActivity，然后内部会通过Instrumentation来尝试启动Activity，
    
    这是一个跨进程过程，它会调用ams的startActivity方法，当ams校验完activity的合法性后，会通过ApplicationThread回调到我们的进程，这也是一次跨进程过程，
    
    而applicationThread就是一个binder，回调逻辑是在binder线程池中完成的，所以需要通过Handler H将其切换到ui线程，
    
    第一个消息是LAUNCH_ACTIVITY，它对应handleLaunchActivity，在这个方法里完成了Activity的创建和启动，
    
    接着，在activity的onResume中，activity的内容将开始渲染到window上，然后开始绘制直到我们看见。 
                                                                                        ---任玉刚

### 代码跟踪


Instrumentation # execStartActivity 

```java
@UnsupportedAppUsage
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getBasePackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

ActivityTaskManager getService 最终调用的是ACTIVITY_TASK_SERVICE 此service  。ActivityTaskManager 是 ActivityTaskManagerService的binder代理

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

通过上述代码可以看出 最终调用的是 ActivityTaskManagerService的startActivity方法 .此过程为IPC调用。

#### 共同ATMSBinder代理进入系统进程调用

ActivityTaskManagerService # startActivity

```java
public final int startActivity(IApplicationThread caller, String callingPackage,
		Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode,
		int startFlags, ProfilerInfo profilerInfo, Bundle bOptions) {
	return startActivityAsUser(caller, callingPackage, intent, resolvedType, resultTo,
			resultWho, requestCode, startFlags, profilerInfo, bOptions,
			UserHandle.getCallingUserId());
}

```

ActivityTaskManagerService # startActivityAsUser

```java
    private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
        assertPackageMatchesCallingUid(callingPackage);
        enforceNotIsolatedCaller("startActivityAsUser");

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
getActivityStartController最终会获得 ActivityStarter调用execute方法, ActivityStarter 准备启动参数 mRequest,并处理相关启动参数。


ActivityStarter # execute

```java
int execute() {
	try {
		>>>
			res = executeRequest(mRequest);

			>>>

			// Notify ActivityMetricsLogger that the activity has launched.
			// ActivityMetricsLogger will then wait for the windows to be drawn and populate
			// WaitResult.
			mSupervisor.getActivityMetricsLogger().notifyActivityLaunched(launchingState, res,
					mLastStartActivityRecord);
			return getExternalResult(mRequest.waitResult == null ? res
					: waitForResult(res, mLastStartActivityRecord));
		}
	} finally {
		onExecutionComplete();
	}
}
```

ActivityStarter # executeRequest
 
```java
private int executeRequest(Request request) {
      >>>
      
        mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions, inTask,
                restrictedBgActivity, intentGrants);

        if (request.outActivity != null) {
            request.outActivity[0] = mLastStartActivityRecord;
        }

        return mLastStartActivityResult;
    }
```

会调用startActivityUnchecked

```java
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
                IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
                int startFlags, boolean doResume, ActivityOptions options, Task inTask,
                boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        int result = START_CANCELED;
        final ActivityStack startedActivityStack;
        try {
            mService.deferWindowLayout();
            Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
            result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                    startFlags, doResume, options, inTask, restrictedBgActivity, intentGrants);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
            startedActivityStack = handleStartResult(r, result);
            mService.continueWindowLayout();
        }

        postStartActivityProcessing(r, result, startedActivityStack);

        return result;
    }
```

ActivityStarter # startActivityInner

```java
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            boolean restrictedBgActivity, NeededUriGrants intentGrants) {
        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor, restrictedBgActivity);

        computeLaunchingTaskFlags();

        computeSourceStack();

        mIntent.setFlags(mLaunchFlags);

        final Task reusedTask = getReusableTask();
       
        >>> 省略 mTargetStack 为 ActivityStack
        
        mTargetStack.startActivityLocked(mStartActivity, topStack.getTopNonFinishingActivity(),
                newTask, mKeepCurTransition, mOptions);
        //处理上一个 Activity 的OnResume事件       
        if (mDoResume) {
            final ActivityRecord topTaskActivity =
                    mStartActivity.getTask().topRunningActivityLocked();
            if (!mTargetStack.isTopActivityFocusable()
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
                mTargetStack.ensureActivitiesVisible(null /* starting */,
                        0 /* configChanges */, !PRESERVE_WINDOWS);
                // Go ahead and tell window manager to execute app transition for this activity
                // since the app transition will not be triggered through the resume channel.
                mTargetStack.getDisplay().mDisplayContent.executeAppTransition();
            } else {
                // If the target stack was not previously focusable (previous top running activity
                // on that stack was not visible) then any prior calls to move the stack to the
                // will not update the focused stack.  If starting the new activity now allows the
                // task stack to be focusable, then ensure that we now update the focused stack
                // accordingly.
                if (mTargetStack.isTopActivityFocusable()
                        && !mRootWindowContainer.isTopDisplayFocusedStack(mTargetStack)) {
                    mTargetStack.moveToFront("startActivityInner");
                }
                mRootWindowContainer.resumeFocusedStacksTopActivities(
                        mTargetStack, mStartActivity, mOptions);
            }
        }
        mRootWindowContainer.updateUserStack(mStartActivity.mUserId, mTargetStack);

        // Update the recent tasks list immediately when the activity starts
        mSupervisor.mRecentTasks.add(mStartActivity.getTask());
        mSupervisor.handleNonResizableTaskIfNeeded(mStartActivity.getTask(),
                mPreferredWindowingMode, mPreferredTaskDisplayArea, mTargetStack);

        return START_SUCCESS;
    }
```

主要涉及栈和Activity联系 ,主要负责上一个activity的pause和下一个activity的resume相关的一系列操作。

ActivityStack：Activity在AMS的栈管理类，activity的单个堆栈的状态和管理在这个类里。

RootWindowContainer # resumeFocusedStacksTopActivities

```java
boolean resumeFocusedStacksTopActivities(
            ActivityStack targetStack, ActivityRecord target, ActivityOptions targetOptions) {

        if (!mStackSupervisor.readyToResume()) {
            return false;
        }

        boolean result = false;
        if (targetStack != null && (targetStack.isTopStackInDisplayArea()
                || getTopDisplayFocusedStack() == targetStack)) {
            result = targetStack.resumeTopActivityUncheckedLocked(target, targetOptions);
        }

        for (int displayNdx = getChildCount() - 1; displayNdx >= 0; --displayNdx) {
            boolean resumedOnDisplay = false;
            final DisplayContent display = getChildAt(displayNdx);
            for (int tdaNdx = display.getTaskDisplayAreaCount() - 1; tdaNdx >= 0; --tdaNdx) {
                final TaskDisplayArea taskDisplayArea = display.getTaskDisplayAreaAt(tdaNdx);
                for (int sNdx = taskDisplayArea.getStackCount() - 1; sNdx >= 0; --sNdx) {
                    final ActivityStack stack = taskDisplayArea.getStackAt(sNdx);
                    final ActivityRecord topRunningActivity = stack.topRunningActivity();
                    if (!stack.isFocusableAndVisible() || topRunningActivity == null) {
                        continue;
                    }
                    if (stack == targetStack) {
                        // Simply update the result for targetStack because the targetStack had
                        // already resumed in above. We don't want to resume it again, especially in
                        // some cases, it would cause a second launch failure if app process was
                        // dead.
                        resumedOnDisplay |= result;
                        continue;
                    }
                    if (taskDisplayArea.isTopStack(stack) && topRunningActivity.isState(RESUMED)) {
                        // Kick off any lingering app transitions form the MoveTaskToFront
                        // operation, but only consider the top task and stack on that display.
                        stack.executeAppTransition(targetOptions);
                    } else {
                        resumedOnDisplay |= topRunningActivity.makeActiveIfNeeded(target);
                    }
                }
            }
            if (!resumedOnDisplay) {
                // In cases when there are no valid activities (e.g. device just booted or launcher
                // crashed) it's possible that nothing was resumed on a display. Requesting resume
                // of top activity in focused stack explicitly will make sure that at least home
                // activity is started and resumed, and no recursion occurs.
                final ActivityStack focusedStack = display.getFocusedStack();
                if (focusedStack != null) {
                    result |= focusedStack.resumeTopActivityUncheckedLocked(target, targetOptions);
                } else if (targetStack == null) {
                    result |= resumeHomeActivity(null /* prev */, "no-focusable-task",
                            display.getDefaultTaskDisplayArea());
                }
            }
        }

        return result;
    }
```

ActivityStack #  resumeTopActivityUncheckedLocked

```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options) {
        if (mInResumeTopActivity) {
            // Don't even start recursing.
            return false;
        }

        boolean result = false;
        try {
            // Protect against recursion.
            mInResumeTopActivity = true;
            result = resumeTopActivityInnerLocked(prev, options);

            // When resuming the top activity, it may be necessary to pause the top activity (for
            // example, returning to the lock screen. We suppress the normal pause logic in
            // {@link #resumeTopActivityUncheckedLocked}, since the top activity is resumed at the
            // end. We call the {@link ActivityStackSupervisor#checkReadyForSleepLocked} again here
            // to ensure any necessary pause logic occurs. In the case where the Activity will be
            // shown regardless of the lock screen, the call to
            // {@link ActivityStackSupervisor#checkReadyForSleepLocked} is skipped.
            final ActivityRecord next = topRunningActivity(true /* focusableOnly */);
            if (next == null || !next.canTurnScreenOn()) {
                checkReadyForSleep();
            }
        } finally {
            mInResumeTopActivity = false;
        }

        return result;
    }
```

ActivityStack #  resumeTopActivityInnerLocked

```java

    @GuardedBy("mService")
    private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options) {
        if (!mAtmService.isBooting() && !mAtmService.isBooted()) {
            // Not ready yet!
            return false;
        }

        // Find the next top-most activity to resume in this stack that is not finishing and is
        // focusable. If it is not focusable, we will fall into the case below to resume the
        // top activity in the next focusable task.
        ActivityRecord next = topRunningActivity(true /* focusableOnly */);

        final boolean hasRunningActivity = next != null;

        // TODO: Maybe this entire condition can get removed?
        if (hasRunningActivity && !isAttached()) {
            return false;
        }

        mRootWindowContainer.cancelInitializingActivities();

        // Remember how we'll process this pause/resume situation, and ensure
        // that the state is reset however we wind up proceeding.
        boolean userLeaving = mStackSupervisor.mUserLeaving;
        mStackSupervisor.mUserLeaving = false;

        if (!hasRunningActivity) {
            // There are no activities left in the stack, let's look somewhere else.
            return resumeNextFocusableActivityWhenStackIsEmpty(prev, options);
        }

        next.delayedResume = false;
        final TaskDisplayArea taskDisplayArea = getDisplayArea();

        // If the top activity is the resumed one, nothing to do.
        if (mResumedActivity == next && next.isState(RESUMED)
                && taskDisplayArea.allResumedActivitiesComplete()) {
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            executeAppTransition(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Top activity resumed " + next);
            return false;
        }

        if (!next.canResumeByCompat()) {
            return false;
        }

        // If we are currently pausing an activity, then don't do anything until that is done.
        final boolean allPausedComplete = mRootWindowContainer.allPausedActivitiesComplete();
        if (!allPausedComplete) {
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) {
                Slog.v(TAG_PAUSE, "resumeTopActivityLocked: Skip resume: some activity pausing.");
            }
            return false;
        }

        // If we are sleeping, and there is no resumed activity, and the top activity is paused,
        // well that is the state we want.
        if (shouldSleepOrShutDownActivities()
                && mLastPausedActivity == next
                && mRootWindowContainer.allPausedActivitiesComplete()) {
            // If the current top activity may be able to occlude keyguard but the occluded state
            // has not been set, update visibility and check again if we should continue to resume.
            boolean nothingToResume = true;
            if (!mAtmService.mShuttingDown) {
                final boolean canShowWhenLocked = !mTopActivityOccludesKeyguard
                        && next.canShowWhenLocked();
                final boolean mayDismissKeyguard = mTopDismissingKeyguardActivity != next
                        && next.containsDismissKeyguardWindow();

                if (canShowWhenLocked || mayDismissKeyguard) {
                    ensureActivitiesVisible(null /* starting */, 0 /* configChanges */,
                            !PRESERVE_WINDOWS);
                    nothingToResume = shouldSleepActivities();
                } else if (next.currentLaunchCanTurnScreenOn() && next.canTurnScreenOn()) {
                    nothingToResume = false;
                }
            }
            if (nothingToResume) {
                // Make sure we have executed any pending transitions, since there
                // should be nothing left to do at this point.
                executeAppTransition(options);
                if (DEBUG_STATES) Slog.d(TAG_STATES,
                        "resumeTopActivityLocked: Going to sleep and all paused");
                return false;
            }
        }

        // Make sure that the user who owns this activity is started.  If not,
        // we will just leave it as is because someone should be bringing
        // another user's activities to the top of the stack.
        if (!mAtmService.mAmInternal.hasStartedUserState(next.mUserId)) {
            Slog.w(TAG, "Skipping resume of top activity " + next
                    + ": user " + next.mUserId + " is stopped");
            return false;
        }

        // The activity may be waiting for stop, but that is no longer
        // appropriate for it.
        mStackSupervisor.mStoppingActivities.remove(next);
        next.setSleeping(false);

        if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resuming " + next);

        // If we are currently pausing an activity, then don't do anything until that is done.
        if (!mRootWindowContainer.allPausedActivitiesComplete()) {
            if (DEBUG_SWITCH || DEBUG_PAUSE || DEBUG_STATES) Slog.v(TAG_PAUSE,
                    "resumeTopActivityLocked: Skip resume: some activity pausing.");

            return false;
        }

        mStackSupervisor.setLaunchSource(next.info.applicationInfo.uid);

        ActivityRecord lastResumed = null;
        final ActivityStack lastFocusedStack = taskDisplayArea.getLastFocusedStack();
        if (lastFocusedStack != null && lastFocusedStack != this) {
            // So, why aren't we using prev here??? See the param comment on the method. prev doesn't
            // represent the last resumed activity. However, the last focus stack does if it isn't null.
            lastResumed = lastFocusedStack.mResumedActivity;
            if (userLeaving && inMultiWindowMode() && lastFocusedStack.shouldBeVisible(next)) {
                // The user isn't leaving if this stack is the multi-window mode and the last
                // focused stack should still be visible.
                if(DEBUG_USER_LEAVING) Slog.i(TAG_USER_LEAVING, "Overriding userLeaving to false"
                        + " next=" + next + " lastResumed=" + lastResumed);
                userLeaving = false;
            }
        }

        boolean pausing = taskDisplayArea.pauseBackStacks(userLeaving, next);
        if (mResumedActivity != null) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Pausing " + mResumedActivity);
            pausing |= startPausingLocked(userLeaving, false /* uiSleeping */, next);
        }
        if (pausing) {
            if (DEBUG_SWITCH || DEBUG_STATES) Slog.v(TAG_STATES,
                    "resumeTopActivityLocked: Skip resume: need to start pausing");
            // At this point we want to put the upcoming activity's process
            // at the top of the LRU list, since we know we will be needing it
            // very soon and it would be a waste to let it get killed if it
            // happens to be sitting towards the end.
            if (next.attachedToProcess()) {
                next.app.updateProcessInfo(false /* updateServiceConnectionActivities */,
                        true /* activityChange */, false /* updateOomAdj */,
                        false /* addPendingTopUid */);
            } else if (!next.isProcessRunning()) {
                // Since the start-process is asynchronous, if we already know the process of next
                // activity isn't running, we can start the process earlier to save the time to wait
                // for the current activity to be paused.
                final boolean isTop = this == taskDisplayArea.getFocusedStack();
                mAtmService.startProcessAsync(next, false /* knownToBeDead */, isTop,
                        isTop ? "pre-top-activity" : "pre-activity");
            }
            if (lastResumed != null) {
                lastResumed.setWillCloseOrEnterPip(true);
            }
            return true;
        } else if (mResumedActivity == next && next.isState(RESUMED)
                && taskDisplayArea.allResumedActivitiesComplete()) {
            // It is possible for the activity to be resumed when we paused back stacks above if the
            // next activity doesn't have to wait for pause to complete.
            // So, nothing else to-do except:
            // Make sure we have executed any pending transitions, since there
            // should be nothing left to do at this point.
            executeAppTransition(options);
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "resumeTopActivityLocked: Top activity resumed (dontWaitForPause) " + next);
            return true;
        }

        // If the most recent activity was noHistory but was only stopped rather
        // than stopped+finished because the device went to sleep, we need to make
        // sure to finish it as we're making a new activity topmost.
        if (shouldSleepActivities() && mLastNoHistoryActivity != null &&
                !mLastNoHistoryActivity.finishing) {
            if (DEBUG_STATES) Slog.d(TAG_STATES,
                    "no-history finish of " + mLastNoHistoryActivity + " on new resume");
            mLastNoHistoryActivity.finishIfPossible("resume-no-history", false /* oomAdj */);
            mLastNoHistoryActivity = null;
        }

        if (prev != null && prev != next && next.nowVisible) {

            // The next activity is already visible, so hide the previous
            // activity's windows right now so we can show the new one ASAP.
            // We only do this if the previous is finishing, which should mean
            // it is on top of the one being resumed so hiding it quickly
            // is good.  Otherwise, we want to do the normal route of allowing
            // the resumed activity to be shown so we can decide if the
            // previous should actually be hidden depending on whether the
            // new one is found to be full-screen or not.
            if (prev.finishing) {
                prev.setVisibility(false);
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Not waiting for visible to hide: " + prev
                        + ", nowVisible=" + next.nowVisible);
            } else {
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Previous already visible but still waiting to hide: " + prev
                        + ", nowVisible=" + next.nowVisible);
            }

        }

        // Launching this app's activity, make sure the app is no longer
        // considered stopped.
        try {
            mAtmService.getPackageManager().setPackageStoppedState(
                    next.packageName, false, next.mUserId); /* TODO: Verify if correct userid */
        } catch (RemoteException e1) {
        } catch (IllegalArgumentException e) {
            Slog.w(TAG, "Failed trying to unstop package "
                    + next.packageName + ": " + e);
        }

        // We are starting up the next activity, so tell the window manager
        // that the previous one will be hidden soon.  This way it can know
        // to ignore it when computing the desired screen orientation.
        boolean anim = true;
        final DisplayContent dc = taskDisplayArea.mDisplayContent;
        if (prev != null) {
            if (prev.finishing) {
                if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                        "Prepare close transition: prev=" + prev);
                if (mStackSupervisor.mNoAnimActivities.contains(prev)) {
                    anim = false;
                    dc.prepareAppTransition(TRANSIT_NONE, false);
                } else {
                    dc.prepareAppTransition(
                            prev.getTask() == next.getTask() ? TRANSIT_ACTIVITY_CLOSE
                                    : TRANSIT_TASK_CLOSE, false);
                }
                prev.setVisibility(false);
            } else {
                if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION,
                        "Prepare open transition: prev=" + prev);
                if (mStackSupervisor.mNoAnimActivities.contains(next)) {
                    anim = false;
                    dc.prepareAppTransition(TRANSIT_NONE, false);
                } else {
                    dc.prepareAppTransition(
                            prev.getTask() == next.getTask() ? TRANSIT_ACTIVITY_OPEN
                                    : next.mLaunchTaskBehind ? TRANSIT_TASK_OPEN_BEHIND
                                            : TRANSIT_TASK_OPEN, false);
                }
            }
        } else {
            if (DEBUG_TRANSITION) Slog.v(TAG_TRANSITION, "Prepare open transition: no previous");
            if (mStackSupervisor.mNoAnimActivities.contains(next)) {
                anim = false;
                dc.prepareAppTransition(TRANSIT_NONE, false);
            } else {
                dc.prepareAppTransition(TRANSIT_ACTIVITY_OPEN, false);
            }
        }

        if (anim) {
            next.applyOptionsLocked();
        } else {
            next.clearOptionsLocked();
        }

        mStackSupervisor.mNoAnimActivities.clear();

        if (next.attachedToProcess()) {
            if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Resume running: " + next
                    + " stopped=" + next.stopped
                    + " visibleRequested=" + next.mVisibleRequested);

            // If the previous activity is translucent, force a visibility update of
            // the next activity, so that it's added to WM's opening app list, and
            // transition animation can be set up properly.
            // For example, pressing Home button with a translucent activity in focus.
            // Launcher is already visible in this case. If we don't add it to opening
            // apps, maybeUpdateTransitToWallpaper() will fail to identify this as a
            // TRANSIT_WALLPAPER_OPEN animation, and run some funny animation.
            final boolean lastActivityTranslucent = lastFocusedStack != null
                    && (lastFocusedStack.inMultiWindowMode()
                    || (lastFocusedStack.mLastPausedActivity != null
                    && !lastFocusedStack.mLastPausedActivity.occludesParent()));

            // This activity is now becoming visible.
            if (!next.mVisibleRequested || next.stopped || lastActivityTranslucent) {
                next.setVisibility(true);
            }

            // schedule launch ticks to collect information about slow apps.
            next.startLaunchTickingLocked();

            ActivityRecord lastResumedActivity =
                    lastFocusedStack == null ? null : lastFocusedStack.mResumedActivity;
            final ActivityState lastState = next.getState();

            mAtmSeattachrvice.updateCpuStats();

            if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to RESUMED: " + next
                    + " (in existing)");

            next.setState(RESUMED, "resumeTopActivityInnerLocked");

            next.app.updateProcessInfo(false /* updateServiceConnectionActivities */,
                    true /* activityChange */, true /* updateOomAdj */,
                    true /* addPendingTopUid */);

            // Have the window manager re-evaluate the orientation of
            // the screen based on the new activity order.
            boolean notUpdated = true;

            // Activity should also be visible if set mLaunchTaskBehind to true (see
            // ActivityRecord#shouldBeVisibleIgnoringKeyguard()).
            if (shouldBeVisible(next)) {
                // We have special rotation behavior when here is some active activity that
                // requests specific orientation or Keyguard is locked. Make sure all activity
                // visibilities are set correctly as well as the transition is updated if needed
                // to get the correct rotation behavior. Otherwise the following call to update
                // the orientation may cause incorrect configurations delivered to client as a
                // result of invisible window resize.
                // TODO: Remove this once visibilities are set correctly immediately when
                // starting an activity.
                notUpdated = !mRootWindowContainer.ensureVisibilityAndConfig(next, getDisplayId(),
                        true /* markFrozenIfConfigChanged */, false /* deferResume */);
            }

            if (notUpdated) {
                // The configuration update wasn't able to keep the existing
                // instance of the activity, and instead started a new one.
                // We should be all done, but let's just make sure our activity
                // is still at the top and schedule another run if something
                // weird happened.
                ActivityRecord nextNext = topRunningActivity();
                if (DEBUG_SWITCH || DEBUG_STATES) Slog.i(TAG_STATES,
                        "Activity config changed during resume: " + next
                                + ", new next: " + nextNext);
                if (nextNext != next) {
                    // Do over!
                    mStackSupervisor.scheduleResumeTopActivities();
                }
                if (!next.mVisibleRequested || next.stopped) {
                    next.setVisibility(true);
                }
                next.completeResumeLocked();
                return true;
            }

            try {
                final ClientTransaction transaction =
                        ClientTransaction.obtain(next.app.getThread(), next.appToken);
                // Deliver all pending results.
                ArrayList<ResultInfo> a = next.results;
                if (a != null) {
                    final int N = a.size();
                    if (!next.finishing && N > 0) {
                        if (DEBUG_RESULTS) Slog.v(TAG_RESULTS,
                                "Delivering results to " + next + ": " + a);
                        transaction.addCallback(ActivityResultItem.obtain(a));
                    }
                }

                if (next.newIntents != null) {
                    transaction.addCallback(
                            NewIntentItem.obtain(next.newIntents, true /* resume */));
                }

                // Well the app will no longer be stopped.
                // Clear app token stopped state in window manager if needed.
                next.notifyAppResumed(next.stopped);

                EventLogTags.writeWmResumeActivity(next.mUserId, System.identityHashCode(next),
                        next.getTask().mTaskId, next.shortComponentName);

                next.setSleeping(false);
                mAtmService.getAppWarningsLocked().onResumeActivity(next);
                next.app.setPendingUiCleanAndForceProcessStateUpTo(mAtmService.mTopProcessState);
                next.clearOptionsLocked();
                transaction.setLifecycleStateRequest(
                        ResumeActivityItem.obtain(next.app.getReportedProcState(),
                                dc.isNextTransitionForward()));
                mAtmService.getLifecycleManager().scheduleTransaction(transaction);

                if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Resumed "
                        + next);
            } catch (Exception e) {
                // Whoops, need to restart this activity!
                if (DEBUG_STATES) Slog.v(TAG_STATES, "Resume failed; resetting state to "
                        + lastState + ": " + next);
                next.setState(lastState, "resumeTopActivityInnerLocked");

                // lastResumedActivity being non-null implies there is a lastStack present.
                if (lastResumedActivity != null) {
                    lastResumedActivity.setState(RESUMED, "resumeTopActivityInnerLocked");
                }

                Slog.i(TAG, "Restarting because process died: " + next);
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else  if (SHOW_APP_STARTING_PREVIEW && lastFocusedStack != null
                        && lastFocusedStack.isTopStackInDisplayArea()) {
                    next.showStartingWindow(null /* prev */, false /* newTask */,
                            false /* taskSwitch */);
                }
                mStackSupervisor.startSpecificActivity(next, true, false);
                return true;
            }

            // From this point on, if something goes wrong there is no way
            // to recover the activity.
            try {
                next.completeResumeLocked();
            } catch (Exception e) {
                // If any exception gets thrown, toss away this
                // activity and try the next one.
                Slog.w(TAG, "Exception thrown during resume of " + next, e);
                next.finishIfPossible("resume-exception", true /* oomAdj */);
                return true;
            }
        } else {
            // Whoops, need to restart this activity!
            if (!next.hasBeenLaunched) {
                next.hasBeenLaunched = true;
            } else {
                if (SHOW_APP_STARTING_PREVIEW) {
                    next.showStartingWindow(null /* prev */, false /* newTask */,
                            false /* taskSwich */);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
            }
            if (DEBUG_STATES) Slog.d(TAG_STATES, "resumeTopActivityLocked: Restarting " + next);
            mStackSupervisor.startSpecificActivity(next, true, true);
        }

        return true;
    }

  
```

ActivityStack # startPausingLocked

```java
   final boolean startPausingLocked(boolean userLeaving, boolean uiSleeping,
            ActivityRecord resuming) {
        if (mPausingActivity != null) {
            Slog.wtf(TAG, "Going to pause when pause is already pending for " + mPausingActivity
                    + " state=" + mPausingActivity.getState());
            if (!shouldSleepActivities()) {
                // Avoid recursion among check for sleep and complete pause during sleeping.
                // Because activity will be paused immediately after resume, just let pause
                // be completed by the order of activity paused from clients.
                completePauseLocked(false, resuming);
            }
        }
        ActivityRecord prev = mResumedActivity;

        if (prev == null) {
            if (resuming == null) {
                Slog.wtf(TAG, "Trying to pause when nothing is resumed");
                mRootWindowContainer.resumeFocusedStacksTopActivities();
            }
            return false;
        }

        if (prev == resuming) {
            Slog.wtf(TAG, "Trying to pause activity that is in process of being resumed");
            return false;
        }

        if (DEBUG_STATES) Slog.v(TAG_STATES, "Moving to PAUSING: " + prev);
        else if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Start pausing: " + prev);
        mPausingActivity = prev;
        mLastPausedActivity = prev;
        mLastNoHistoryActivity = prev.isNoHistory() ? prev : null;
        prev.setState(PAUSING, "startPausingLocked");
        prev.getTask().touchActiveTime();
        clearLaunchTime(prev);

        mAtmService.updateCpuStats();

        boolean pauseImmediately = false;
        if (resuming != null && (resuming.info.flags & FLAG_RESUME_WHILE_PAUSING) != 0) {
            // If the flag RESUME_WHILE_PAUSING is set, then continue to schedule the previous
            // activity to be paused, while at the same time resuming the new resume activity
            // only if the previous activity can't go into Pip since we want to give Pip
            // activities a chance to enter Pip before resuming the next activity.
            final boolean lastResumedCanPip = prev != null && prev.checkEnterPictureInPictureState(
                    "shouldResumeWhilePausing", userLeaving);
            if (!lastResumedCanPip) {
                pauseImmediately = true;
            }
        }

        if (prev.attachedToProcess()) {
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Enqueueing pending pause: " + prev);
            try {
                EventLogTags.writeWmPauseActivity(prev.mUserId, System.identityHashCode(prev),
                        prev.shortComponentName, "userLeaving=" + userLeaving);

                mAtmService.getLifecycleManager().scheduleTransactionmAtmService.getLifecycleManager().scheduleTransaction(prev.app.getThread(),
                        prev.appToken, PauseActivityItem.obtain(prev.finishing, userLeaving,
                                prev.configChangeFlags, pauseImmediately));
            } catch (Exception e) {
                // Ignore exception, if process died other code will cleanup.
                Slog.w(TAG, "Exception thrown during pause", e);
                mPausingActivity = null;
                mLastPausedActivity = null;
                mLastNoHistoryActivity = null;
            }
        } else {
            mPausingActivity = null;
            mLastPausedActivity = null;
            mLastNoHistoryActivity = null;
        }

        // If we are not going to sleep, we want to ensure the device is
        // awake until the next activity is started.
        if (!uiSleeping && !mAtmService.isSleepingOrShuttingDownLocked()) {
            mStackSupervisor.acquireLaunchWakelock();
        }

        if (mPausingActivity != null) {
            // Have the window manager pause its key dispatching until the new
            // activity has started.  If we're pausing the activity just because
            // the screen is being turned off and the UI is sleeping, don't interrupt
            // key dispatch; the same activity will pick it up again on wakeup.
            if (!uiSleeping) {
                prev.pauseKeyDispatchingLocked();
            } else if (DEBUG_PAUSE) {
                 Slog.v(TAG_PAUSE, "Key dispatch not paused for screen off");
            }

            if (pauseImmediately) {
                // If the caller said they don't want to wait for the pause, then complete
                // the pause now.
                completePauseLocked(false, resuming);
                return false;

            } else {
                prev.schedulePauseTimeout();
                return true;
            }

        } else {
            // This activity failed to schedule the
            // pause, so just treat it as being paused now.
            if (DEBUG_PAUSE) Slog.v(TAG_PAUSE, "Activity not running, resuming next.");
            if (resuming == null) {
                mRootWindowContainer.resumeFocusedStacksTopActivities();
            }
            return false;
        }
    }

```
ClientLifecycleManager # scheduleTransactionmAtmService

ClientLifecycleManager：该类能够组合多个生命周期转换请求和/或回调，并将它们作为单个事务执行。
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
prev.app.getThread() 参数mClient 的类型是 IApplicationThread，系统进程通过ProcessRecord.IApplicationThread调用app进程相关方法。IApplicationThread是系统进程持有的app进程中ApplicationThread的Binder代理对象，ApplicationThread实现了IApplicationThread.Stub 

#### AMS通过binder代理调用到应用 ApplicationThread scheduleTransaction

IApplicationThread：是系统进程持有的app进程中ApplicationThread的Binder代理对象。系统进程通过ProcessRecord.IApplicationThread调用app进程相关方法。
ApplicationThread：ActivityThread的内部类。AMS通过binder代理调用到ApplicationThread中的方法后，通过主线程（ActivityThread中的main方法）中开启的handler消息轮询来通知主线程调用相关方法。主线程的相关生命周期方法的具体实现会委托给Instrumentation类实现，在Instrumentation类中，会调用具体组件的相关生命周期方法。

ApplicationThread # scheduleTransaction

```
  @Override
        public void scheduleTransaction(ClientTransaction transaction) throws RemoteException {
            ActivityThread.this.scheduleTransaction(transaction);
        }
```

#### 通过Handler机制执行生命周期

ClientTransactionHandler # scheduleTransaction

```
    /** Prepare and schedule transaction for execution. */
    void scheduleTransaction(ClientTransaction transaction) {
        transaction.preExecute(this);
        sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction);
    }
```

sendMessage发送的消息会被ActivityThread中定义的H对象接收，H是Handler的子类

```java
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
```

TransactionExecutor # execute

```java

    /** Transition the client through previously initialized state sequence. */
    private void performLifecycleSequence(ActivityClientRecord r, IntArray path,
            ClientTransaction transaction) {
        final int size = path.size();
        for (int i = 0, state; i < size; i++) {
            state = path.get(i);
            if (DEBUG_RESOLVER) {
                Slog.d(TAG, tId(transaction) + "Transitioning activity: "
                        + getShortActivityName(r.token, mTransactionHandler)
                        + " to state: " + getStateName(state));
            }
            switch (state) {
                case ON_CREATE:
                    mTransactionHandler.handleLaunchActivity(r, mPendingActions,
                            null /* customIntent */);
                    break;
                case ON_START:
                    mTransactionHandler.handleStartActivity(r.token, mPendingActions);
                    break;
                case ON_RESUME:
                    mTransactionHandler.handleResumeActivity(r.token, false /* finalStateRequest */,
                            r.isForward, "LIFECYCLER_RESUME_ACTIVITY");
                    break;
                case ON_PAUSE:
                    mTransactionHandler.handlePauseActivity(r.token, false /* finished */,
                            false /* userLeaving */, 0 /* configChanges */, mPendingActions,
                            "LIFECYCLER_PAUSE_ACTIVITY");
                    break;
                case ON_STOP:
                    mTransactionHandler.handleStopActivity(r.token, 0 /* configChanges */,
                            mPendingActions, false /* finalStateRequest */,
                            "LIFECYCLER_STOP_ACTIVITY");
                    break;
                case ON_DESTROY:
                    mTransactionHandler.handleDestroyActivity(r.token, false /* finishing */,
                            0 /* configChanges */, false /* getNonConfigInstance */,
                            "performLifecycleSequence. cycling to:" + path.get(size - 1));
                    break;
                case ON_RESTART:
                    mTransactionHandler.performRestartActivity(r.token, false /* start */);
                    break;
                default:
                    throw new IllegalArgumentException("Unexpected lifecycle state: " + state);
            }
        }
    }
```

回到ActivityTread # handleLaunchActivity

ActivityTread # performLaunchActivity

```java
  /**  Core implementation of activity launch. */
    private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        ActivityInfo aInfo = r.activityInfo;
        if (r.packageInfo == null) {
            r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,
                    Context.CONTEXT_INCLUDE_CODE);
        }

        ComponentName component = r.intent.getComponent();
        if (component == null) {
            component = r.intent.resolveActivity(
                mInitialApplication.getPackageManager());
            r.intent.setComponent(component);
        }

        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);
        }

        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;
        try {
            java.lang.ClassLoader cl = appContext.getClassLoader();
            activity = mInstrumentation.newActivity(
                    cl, component.getClassName(), r.intent);
            StrictMode.incrementExpectedActivityCount(activity.getClass());
            r.intent.setExtrasClassLoader(cl);
            r.intent.prepareToEnterProcess();
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

       >>>
       
         if (r.isPersistable()) {
                    mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);
                } else {
                    mInstrumentation.callActivityOnCreate(activity, r.state);
                }

        return activity;
    }
```

mInstrumentation # newActivity

```java
 public Activity newActivity(Class<?> clazz, Context context, 
            IBinder token, Application application, Intent intent, ActivityInfo info, 
            CharSequence title, Activity parent, String id,
            Object lastNonConfigurationInstance) throws InstantiationException,
            IllegalAccessException {
        Activity activity = (Activity)clazz.newInstance();
        ActivityThread aThread = null;
        // Activity.attach expects a non-null Application Object.
        if (application == null) {
            application = new Application();
        }
        activity.attach(context, aThread, this, token, 0 /* ident */, application, intent,
                info, title, parent, id,
                (Activity.NonConfigurationInstances)lastNonConfigurationInstance,
                new Configuration(), null /* referrer */, null /* voiceInteractor */,
                null /* window */, null /* activityConfigCallback */, null /*assistToken*/);
        return activity;
    }
```
由此可看出，Activity是通过类加载器去创建的实例。
mInstrumentation # callActivityOnCreate

activity.attach时会创建Window,绘制view等视图

Activity # performCreate

```java
    @UnsupportedAppUsage
    final void performCreate(Bundle icicle, PersistableBundle persistentState) {
        dispatchActivityPreCreated(icicle);
        mCanEnterPictureInPicture = true;
        // initialize mIsInMultiWindowMode and mIsInPictureInPictureMode before onCreate
        final int windowingMode = getResources().getConfiguration().windowConfiguration
                .getWindowingMode();
        mIsInMultiWindowMode = inMultiWindowMode(windowingMode);
        mIsInPictureInPictureMode = windowingMode == WINDOWING_MODE_PINNED;
        restoreHasCurrentPermissionRequest(icicle);
        if (persistentState != null) {
            onCreate(icicle, persistentState);
        } else {
            onCreate(icicle);
        }
        EventLogTags.writeWmOnCreateCalled(mIdent, getComponentName().getClassName(),
                "performCreate");
        mActivityTransitionState.readState(icicle);

        mVisibleFromClient = !mWindow.getWindowStyle().getBoolean(
                com.android.internal.R.styleable.Window_windowNoDisplay, false);
        mFragments.dispatchActivityCreated();
        mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
        dispatchActivityPostCreated(icicle);
    }
```



## 问题

### AMS把栈分成几部分


### AMS 对方进程是什么时候创建出来的？ 主线程是什么时候跑起来的

- AMS先判断是否有相应的 ProcessRecord
- AMS通过Socket去和Zygote协商 ZygoteFock APP进程
- 由于Zygote进程在启动时会创建Java虚拟机，因此通过fork而创建的Launcher程序进程可以在内部获取一个Java虚拟机的实例拷贝 
- 把进程信息通过Socket发给fork进程
- 最终进入ActivityThread的main方法。   

### Activity View是如何绘制的

```java

    @UnsupportedAppUsage
    final void attach(Context context, ActivityThread aThread,
            Instrumentation instr, IBinder token, int ident,
            Application application, Intent intent, ActivityInfo info,
            CharSequence title, Activity parent, String id,
            NonConfigurationInstances lastNonConfigurationInstances,
            Configuration config, String referrer, IVoiceInteractor voiceInteractor,
            Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken) {
        attachBaseContext(context);

        mFragments.attachHost(null /*parent*/);

        mWindow = new PhoneWindow(this, window, activityConfigCallback);
        mWindow.setWindowControllerCallback(mWindowControllerCallback);
        mWindow.setCallback(this);
        mWindow.setOnWindowDismissedCallback(this);
        mWindow.getLayoutInflater().setPrivateFactory(this);
        if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
            mWindow.setSoftInputMode(info.softInputMode);
        }
        if (info.uiOptions != 0) {
            mWindow.setUiOptions(info.uiOptions);
        }
        mUiThread = Thread.currentThread();

        mMainThread = aThread;
        mInstrumentation = instr;
        mToken = token;
        mAssistToken = assistToken;
        mIdent = ident;
        mApplication = application;
        mIntent = intent;
        mReferrer = referrer;
        mComponent = intent.getComponent();
        mActivityInfo = info;
        mTitle = title;
        mParent = parent;
        mEmbeddedID = id;
        mLastNonConfigurationInstances = lastNonConfigurationInstances;
        if (voiceInteractor != null) {
            if (lastNonConfigurationInstances != null) {
                mVoiceInteractor = lastNonConfigurationInstances.voiceInteractor;
            } else {
                mVoiceInteractor = new VoiceInteractor(voiceInteractor, this, this,
                        Looper.myLooper());
            }
        }

        mWindow.setWindowManager(
                (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
                mToken, mComponent.flattenToString(),
                (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
        if (mParent != null) {
            mWindow.setContainer(mParent.getWindow());
        }
        mWindowManager = mWindow.getWindowManager();
        mCurrentConfig = config;

        mWindow.setColorMode(info.colorMode);
        mWindow.setPreferMinimalPostProcessing(
                (info.flags & ActivityInfo.FLAG_PREFER_MINIMAL_POST_PROCESSING) != 0);

        setAutofillOptions(application.getAutofillOptions());
        setContentCaptureOptions(application.getContentCaptureOptions());
    }
```

创建了WindowManager 并绑定了WMS

## 参考

https://cloud.tencent.com/developer/article/1466430 

https://blog.csdn.net/yu749942362/article/details/107978083