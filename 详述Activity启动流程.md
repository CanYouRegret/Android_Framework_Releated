# 详述Activity启动流程

## 前言

&nbsp; 从手机桌面点击应用图标，到主界面显示出来，经历了什么流程？

&nbsp; 想要分析系统的某个流程，从哪里入手？

&nbsp; 系统为何要这样来设计，这样做的好处是什么，有没有更好的做法或是可以优化的点？

在开始分析之前，先给自己提出几个问题，然后在分析的过程中看能否找到答案

## 从何处入手进行分析

&emsp; 不管是分析bug还是流程，我们往往首先想到的是日志(log)。Activity作为Android四大组件中最为重要的部分，系统在关键节点处都进行了log插桩，以便于定位问题。从framework的角度而言，更为直观的是system和event log。对于Activity启动流程而言event & system log表达的含义直接清楚，本文基于相应的event & system log进行分析。（system log相关log开关是默认关闭的，需要手动打开并编译系统，Activity相关log开关存在于ActivityManagerDebugConfig.java和ActivityTaskManagerDebugConfig.java文件文件中）

&emsp; 首先，自写一个简单的Hello world应用，使用adb logcat -b events -b system命令进行log抓取，在桌面上找到应用图标进行点击。 就会得到下面对应的日志 (本例中启动的Activity为com.northwall.learningdemo.MainActivity)

```
10-26 15:05:27.734  1765  2860 I ActivityTaskManager: START u0 {act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.northwall.learningdemo/.MainActivity bnds=[643,917][850,1247]} from uid 10264
10-26 15:05:27.734  1765  2860 D ActivityTaskManager: Activity start allowed for home app callingUid (10264)
10-26 15:05:27.736  1765  2860 I wm_task_created: [49,-1]
10-26 15:05:27.738  1765  2860 I wm_task_moved: [49,1,4]
10-26 15:05:27.738  1765  2860 I wm_task_to_front: [0,49]
10-26 15:05:27.739  1765  2860 I wm_create_task: [0,49]
10-26 15:05:27.739  1765  2860 I wm_create_activity: [0,220883831,49,com.northwall.learningdemo/.MainActivity,android.intent.action.MAIN,NULL,NULL,270532608]
10-26 15:05:27.739  1765  2860 I wm_task_moved: [49,1,4]
10-26 15:05:27.739  1765  2860 V ActivityTaskManager: Prepare open transition: starting ActivityRecord{d2a6b77 u0 com.northwall.learningdemo/.MainActivity} t49}
10-26 15:05:27.740  1765  2860 I wm_pause_activity: [0,267360229,com.android.launcher3/.uioverrides.QuickstepLauncher,userLeaving=true,pauseBackTasks]
10-26 15:05:27.745  1765  1852 V ActivityManager: Clearing bad process: 10000/com.northwall.learningdemo
10-26 15:05:27.745  1765  1852 V ActivityManager: startProcess: name=com.northwall.learningdemo app=null knownToBeDead=false thread=null pid=-1
10-26 15:05:27.746  2726  2726 I wm_on_top_resumed_lost_called: [267360229,com.android.launcher3.uioverrides.QuickstepLauncher,topStateChangedWhenResumed]
10-26 15:05:27.746  1765  1852 I am_uid_running: 10000
10-26 15:05:27.747  1765  1852 I ActivityManager: Posting procStart msg for 0:com.northwall.learningdemo/u0a0
10-26 15:05:27.747  2726  2726 I wm_on_paused_called: [267360229,com.android.launcher3.uioverrides.QuickstepLauncher,performPause]
10-26 15:05:27.748  1765  2471 V ActivityTaskManager: Prepare open transition: prev=ActivityRecord{fef97e5 u0 com.android.launcher3/.uioverrides.QuickstepLauncher} t46}
10-26 15:05:27.748  1765  2471 I ActivityTaskManager: Applying options for ActivityRecord{d2a6b77 u0 com.northwall.learningdemo/.MainActivity} t49}
10-26 15:05:27.748  1765  1852 V ActivityManager: Clearing bad process: 10000/com.northwall.learningdemo
10-26 15:05:27.748  1765  1852 V ActivityManager: startProcess: name=com.northwall.learningdemo app=ProcessRecord{8504813 0:com.northwall.learningdemo/u0a0} knownToBeDead=false thread=null pid=0
10-26 15:05:27.749  1765  1852 V ActivityManager: Clearing bad process: 10000/com.northwall.learningdemo
10-26 15:05:27.749  1765  1852 V ActivityManager: startProcess: name=com.northwall.learningdemo app=ProcessRecord{8504813 0:com.northwall.learningdemo/u0a0} knownToBeDead=false thread=null pid=0
10-26 15:05:27.749  1765  2471 I wm_add_to_stopping: [0,267360229,com.android.launcher3/.uioverrides.QuickstepLauncher,makeInvisible]
10-26 15:05:27.753  1765  1866 I am_proc_start: [0,4660,10000,com.northwall.learningdemo,next-top-activity,{com.northwall.learningdemo/com.northwall.learningdemo.MainActivity}]
10-26 15:05:27.753  1765  1866 I ActivityManager: Start proc 4660:com.northwall.learningdemo/u0a0 for next-top-activity {com.northwall.learningdemo/com.northwall.learningdemo.MainActivity}
10-26 15:05:27.756  1765  2229 I input_focus: [Focus leaving 2c40b47 com.android.launcher3/com.android.launcher3.uioverrides.QuickstepLauncher (server),reason=NO_WINDOW]
10-26 15:05:27.784  1765  2471 I am_proc_bound: [0,4660,com.northwall.learningdemo]
10-26 15:05:27.786  1765  2471 I wm_restart_activity: [0,220883831,49,com.northwall.learningdemo/.MainActivity]
10-26 15:05:27.786  1765  2471 I ActivityTaskManager: Taking options for ActivityRecord{d2a6b77 u0 com.northwall.learningdemo/.MainActivity} t49} callers=com.android.server.wm.ActivityTaskSupervisor.realStartActivityLocked:910 com.android.server.wm.RootWindowContainer$AttachApplicationHelper.test:3594 com.android.server.wm.RootWindowContainer$AttachApplicationHelper.test:3544 com.android.server.wm.ActivityRecord.forAllActivities:4510 com.android.server.wm.WindowContainer.forAllActivities:1638 com.android.server.wm.WindowContainer.forAllActivities:1632
10-26 15:05:27.786  1765  2471 I wm_set_resumed_activity: [0,com.northwall.learningdemo/.MainActivity,minimalResumeActivityLocked]
10-26 15:05:27.787  1765  1852 I am_uid_active: 10000
10-26 15:05:27.916  4660  4660 I wm_on_create_called: [220883831,com.northwall.learningdemo.MainActivity,performCreate]
10-26 15:05:27.918  4660  4660 I wm_on_start_called: [220883831,com.northwall.learningdemo.MainActivity,handleStartActivity]
10-26 15:05:27.919  4660  4660 I wm_on_resume_called: [220883831,com.northwall.learningdemo.MainActivity,RESUME_ACTIVITY]
10-26 15:05:27.931  4660  4660 I wm_on_top_resumed_gained_called: [220883831,com.northwall.learningdemo.MainActivity,topStateChangedWhenResumed]
10-26 15:05:27.968  1765  1849 I wm_activity_launch_time: [0,220883831,com.northwall.learningdemo/.MainActivity,233]
10-26 15:05:27.979  1765  2229 I input_focus: [Focus entering 264a1b9 com.northwall.learningdemo/com.northwall.learningdemo.MainActivity (server),reason=Window became focusable. Previous reason: NOT_VISIBLE]
10-26 15:05:28.289  1765  1852 I wm_stop_activity: [0,267360229,com.android.launcher3/.uioverrides.QuickstepLauncher]
10-26 15:05:28.303  2726  2726 I wm_on_stop_called: [267360229,com.android.launcher3.uioverrides.QuickstepLauncher,STOP_ACTIVITY_ITEM]
10-26 15:05:44.182  1765  2863 I wm_task_moved: [1,0,0]
10-26 15:05:44.182  1765  2863 I wm_task_moved: [1,0,3]
```

&emsp; 从得到的event & system log可以很清晰的看到Activity的生命周期变化，当中也有task相关的操作(task可以理解为装填activity的容器)。待分析完Activity的启动流程之后，我们也可以根据event log来进行梳理。

## Activity启动流程

### 1. 从Launcher点击应用图标

&emsp; Launcher对于显示的应用快捷方式图标均有点击监听，具体位置位于[packages/apps/Launcher3/src/com/android/launcher3/touch/ItemClickHandler.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:packages/apps/Launcher3/src/com/android/launcher3/touch/ItemClickHandler.java)

```java
    private static void onClick(View v) {
        // Make sure that rogue clicks don't get through while allapps is launching, or after the
        // view has detached (it's possible for this to happen if the view is removed mid touch).
        if (v.getWindowToken() == null) return;

        Launcher launcher = Launcher.getLauncher(v.getContext());
        if (!launcher.getWorkspace().isFinishedSwitchingState()) return;

        Object tag = v.getTag();
        if (tag instanceof WorkspaceItemInfo) {
            onClickAppShortcut(v, (WorkspaceItemInfo) tag, launcher);
        } else if (tag instanceof FolderInfo) {
            if (v instanceof FolderIcon) {
                onClickFolderIcon(v);
            }
        } else if (tag instanceof AppInfo) {
            startAppShortcutOrInfoActivity(v, (AppInfo) tag, launcher);
        } else if (tag instanceof LauncherAppWidgetInfo) {
            if (v instanceof PendingAppWidgetHostView) {
                onClickPendingWidget((PendingAppWidgetHostView) v, launcher);
            }
        } else if (tag instanceof SearchActionItemInfo) {
            onClickSearchAction(launcher, (SearchActionItemInfo) tag);
        }
    }
```

&emsp; 从上述代码可以看出，Launcher对于其上各种组件的点击监听，如文件夹、快捷方式图标、小部件、搜索框等。 当快捷方式图标被点击时会调用startAppShortcutOrInfoActivity方法，该方法的作用就是启动该快捷图标注册的Activity(对于正常情况来说就是应用的MAIN Activity)。 沿着这个方法追踪下去，我们会发现在构造Intent之后实质上时调用了Context.java#startActivity。(限于篇幅未贴出详细代码，可根据源码查看相关调用链)

### 2. Context.java#startActivity

&emsp; 正常来说想要启动一个Activity， 均需要基于Context，Context是上下文的含义，如Application、Activity、Service等均属于上下文。使用哪个Context就决定了该次启动的caller。 而当Context是Activity时是与其他Context存在差异的

[frameworks/base/core/java/android/app/ContextImpl.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/android/app/ContextImpl.java)

```java
    @Override
    public void startActivity(Intent intent, Bundle options) {
        warnIfCallingFromSystemProcess();

        // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
        // generally not allowed, except if the caller specifies the task id the activity should
        // be launched in. A bug was existed between N and O-MR1 which allowed this to work. We
        // maintain this for backwards compatibility.
        final int targetSdkVersion = getApplicationInfo().targetSdkVersion;

        if ((intent.getFlags() & Intent.FLAG_ACTIVITY_NEW_TASK) == 0
                && (targetSdkVersion < Build.VERSION_CODES.N
                        || targetSdkVersion >= Build.VERSION_CODES.P)
                && (options == null
                        || ActivityOptions.fromBundle(options).getLaunchTaskId() == -1)) {
            throw new AndroidRuntimeException(
                    "Calling startActivity() from outside of an Activity "
                            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                            + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, options);
    }
```

[frameworks/base/core/java/android/app/Activity.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/android/app/Activity.java)

```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
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

&emsp; 从上述代码可以看到，无论是Activity还是其他的Context后续均调用到了Instrumentation.java#execStartActivity。差异在于，如果caller为Activity时，则会关注将要启动Activity的启动结果，设置启动参数mToken、target及requestCode，而mToken将会对应后续ActivityStarter.java#executeRequest时的sourceRecord。(读者可先略过处，待到后续流程用到sourceRecord时再回过头来查看)

### 3. Instrumentation.java#execStartActivity

&emsp; Instrumentation被译为“仪器仪表”，可以理解为系统和应用交互的一层钩子，往往在编写自动化测试用例时会用到，或者应用侧可通过反射替换Instrumentation实例的方式起到监控相关生命周期的效果。

[frameworks/base/core/java/android/app/Instrumentation.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/android/app/Instrumentation.java)

```java
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        Uri referrer = target != null ? target.onProvideReferrer() : null;
        if (referrer != null) {
            intent.putExtra(Intent.EXTRA_REFERRER, referrer);
        }
        ... ...
        try {
            intent.migrateExtraStreamToClipData(who);
            intent.prepareToLeaveProcess(who);
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

&emsp; 通过上述代码可以看到，实质上是通过ActivityTaskManager进行binder调用startActivity从而进行后续流程。

### 4. ActivityTaskManagerService.java#startActivity

&emsp; ActivityTaskManagerService位于Server一侧，跑在System Server进程中。是用来管理Activity的系统服务。(该系统服务是从ActivityManagerService中解耦出来的，Android10之前ActivityManagerService是管理四大组件的系统服务，Android10之后将Activity的管理独立成为ActivityTaskManagerService)

[frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java)

```java
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

&emsp; 通过上述代码可以看到，Server端的做法是从ActivityStartController拿到一个ActivityStarter，并将相关信息装填然后运行execute进行启动。(此处获取ActivityStarter时，用到了工厂类以及Pool的概念，ActivityStarter并非每次都创建新的对象，如发现池子里已有空闲实例化对象时，则直接拿出并填充数据，使用完之后再release放回池子)

### 5. ActivityStarter.java#execute

&emsp; 此时，ActivityStarter数据填充完毕，做预启动的准备工作。

[frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java)

```java
    int execute() {
        try {
            ... ...
            // If the caller hasn't already resolved the activity, we're willing
            // to do so here. If the caller is already holding the WM lock here,
            // and we need to check dynamic Uri permissions, then we're forced
            // to assume those permissions are denied to avoid deadlocking.
            if (mRequest.activityInfo == null) {
                mRequest.resolveActivity(mSupervisor);
            }
            ... ...

            int res;
            synchronized (mService.mGlobalLock) {
                final boolean globalConfigWillChange = mRequest.globalConfig != null
                        && mService.getGlobalConfiguration().diff(mRequest.globalConfig) != 0;
                final Task rootTask = mRootWindowContainer.getTopDisplayFocusedRootTask();
                if (rootTask != null) {
                    rootTask.mConfigWillChange = globalConfigWillChange;
                }
                ProtoLog.v(WM_DEBUG_CONFIGURATION, "Starting activity when config "
                        + "will change = %b", globalConfigWillChange);

                final long origId = Binder.clearCallingIdentity();

                res = resolveToHeavyWeightSwitcherIfNeeded();
                if (res != START_SUCCESS) {
                    return res;
                }
                res = executeRequest(mRequest);

                ... ...
                return getExternalResult(res);
            }
        } finally {
            onExecutionComplete();
        }
    }
```

&emsp; 笔者去掉了相关细节的代码，只保留了我们关注的比较重要的部分，后续亦是如此。

&emsp; 从该段代码中可以看到，如果activityInfo还未被指定，则会进行resolveActivity的操作；我们知道启动Activity时有显式启动和隐式启动两种，其中显式启动就是Intent已经指定了Activity的ComponentName，系统只需去寻找该ComponentName是否存在，存在则启动即可；而隐式启动则只是通过IntentFilter列出了将要启动Activity的相关条件，如action、category、data等，resolveActivity函数就是在隐式启动的情况下会被执行去寻找相匹配的Activity。

&emsp; resolveActivity函数最终是PackageManager来执行的，毕竟在应用安装的时候，组件的注册是由PackageManager来管理的。具体resolve流程可参考[resolveIntentInternal](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/pm/ResolveIntentHelper.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=108?gsn=resolveIntentInternal&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.server.pm.ResolveIntentHelper%238761de458fab8fc3aade746020dc75f99a21f1074f22d5182bac867d76ec3976) 和 [chooseBestActivity](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/core/java/com/android/server/pm/ResolveIntentHelper.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=146?gsn=chooseBestActivity&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.server.pm.ResolveIntentHelper%237cd6e4aec7ed033520f58b914b329796e8c50ea09b25e77dbf2bfa2d146a84ce) 函数，大概就是根据给定的相关条件去寻找可以匹配上的Activity组件，如果只有一个满足条件的则直接返回；如果存在多个，查看用户是否有设置偏好Activity，否则启动ResolverActivity将满足条件的Activity均列出来供用户选择。 读者可自行研究相关Code，这里不再赘述。

### 6. ActivityStarter.java#executeRequest

&emsp; 到此处，要启动的目标Activity已经明确。

```java
    private int executeRequest(Request request) {
        ... ...
        if (err == ActivityManager.START_SUCCESS) {
            Slog.i(TAG, "START u" + userId + " {" + intent.toShortString(true, true, true, false)
                    + "} from uid " + callingUid);
        }
    
        ActivityRecord sourceRecord = null;
        ActivityRecord resultRecord = null;
        if (resultTo != null) {
            sourceRecord = ActivityRecord.isInAnyTask(resultTo);
            if (DEBUG_RESULTS) {
                Slog.v(TAG_RESULTS, "Will send result to " + resultTo + " " + sourceRecord);
            }
            if (sourceRecord != null) {
                if (requestCode >= 0 && !sourceRecord.finishing) {
                    resultRecord = sourceRecord;
                }
            }
        }
        ... ...

        if (err == ActivityManager.START_SUCCESS && intent.getComponent() == null) {
            // We couldn't find a class that can handle the given Intent.
            // That's the end of that!
            err = ActivityManager.START_INTENT_NOT_RESOLVED;
        }

        if (err == ActivityManager.START_SUCCESS && aInfo == null) {
            // We couldn't find the specific class specified in the Intent.
            // Also the end of the line.
            err = ActivityManager.START_CLASS_NOT_FOUND;
        }

        ... ...

        boolean abort = !mSupervisor.checkStartAnyActivityPermission(intent, aInfo, resultWho,
                requestCode, callingPid, callingUid, callingPackage, callingFeatureId,
                request.ignoreTargetSecurity, inTask != null, callerApp, resultRecord,
                resultRootTask);
        abort |= !mService.mIntentFirewall.checkStartActivity(intent, callingUid,
                callingPid, resolvedType, aInfo.applicationInfo);
        abort |= !mService.getPermissionPolicyInternal().checkStartActivity(intent, callingUid,
                callingPackage);

        // Merge the two options bundles, while realCallerOptions takes precedence.
        ActivityOptions checkedOptions = options != null
                ? options.getOptions(intent, aInfo, callerApp, mSupervisor) : null;

        boolean restrictedBgActivity = false;
        if (!abort) {
            try {
                Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER,
                        "shouldAbortBackgroundActivityStart");
                BackgroundActivityStartController balController =
                        mController.getBackgroundActivityLaunchController();
                restrictedBgActivity =
                        balController.shouldAbortBackgroundActivityStart(
                                callingUid,
                                callingPid,
                                callingPackage,
                                realCallingUid,
                                realCallingPid,
                                callerApp,
                                request.originatingPendingIntent,
                                request.allowBackgroundActivityStart,
                                intent,
                                checkedOptions);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
            }
        }

        ... ...

        mInterceptor.setStates(userId, realCallingPid, realCallingUid, startFlags, callingPackage,
                callingFeatureId);
        if (mInterceptor.intercept(intent, rInfo, aInfo, resolvedType, inTask, callingPid,
                callingUid, checkedOptions)) {
            // activity start was intercepted, e.g. because the target user is currently in quiet
            // mode (turn off work) or the target application is suspended
            intent = mInterceptor.mIntent;
            rInfo = mInterceptor.mRInfo;
            aInfo = mInterceptor.mAInfo;
            resolvedType = mInterceptor.mResolvedType;
            inTask = mInterceptor.mInTask;
            callingPid = mInterceptor.mCallingPid;
            callingUid = mInterceptor.mCallingUid;
            checkedOptions = mInterceptor.mActivityOptions;

            // The interception target shouldn't get any permission grants
            // intended for the original destination
            intentGrants = null;
        }

        if (abort) {
            if (resultRecord != null) {
                resultRecord.sendResult(INVALID_UID, resultWho, requestCode, RESULT_CANCELED,
                        null /* data */, null /* dataGrants */);
            }
            // We pretend to the caller that it was really started, but they will just get a
            // cancel result.
            ActivityOptions.abort(checkedOptions);
            return START_ABORTED;
        }
        ... ...

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

        ... ...

        mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, true /* doResume */, checkedOptions,
                inTask, inTaskFragment, restrictedBgActivity, intentGrants);

        if (request.outActivity != null) {
            request.outActivity[0] = mLastStartActivityRecord;
        }

        return mLastStartActivityResult;
    }


```

&emsp; 从该executeRequest函数来看：

* "START u"日志的打印；此处也可以与文章开头打印的log联系起来，该log标志系统确实收到了Activity的启动请求，如果Activity启动失败且该日志未打印，需要检查应用启动Activity的请求是否真的发出了。而"START u0"中0的含义表示机主，在多用户模式下(如访客)会变成其他数字。

* 确定sourceRecord；ActivityRecord为System Server一侧用来表示Activity的类，对应Client一侧的ActivityClientRecord及Activity。如在启动Activity时caller Context为Activity，则会通过mToken(也就是resultTo)来确定caller Activity在System Server一侧的ActivityRecord。(从这里我们也就可以看出System Server一侧的ActivityRecord时如何与Client一侧的Activity对应起来的，也就是ActivityRecord.java(WindowToken.java)#token 与 Activity.java#mToken。这里简单描述下联系绑定规则，一个新的ActivityRecord创建时会先new一个Token出来，然后将该ActivityRecord绑定至该Token，而后创建Client一侧的Activity实例时，再将该Token传入，这样System和Client两侧的数据结构就对应起来了)

* Activity启动失败常见的几种情况；如：根据启动的Intent未找到匹配的Activity、因为权限问题或者后台启动被中止、被设置的Interceptor(拦截器)拦截等。正常来说启动失败时，ActivityTaskManager会有相关的日志打印，我们可以根据相应的日志定位启动失败的原因。

* 创建ActivityRecord；如该Activity未启动失败，则此时创建ActivityRecord实例，

* 调用startActivityUnchecked函数，而该函数又调用了startActivityInner。



### 7. ActivityStarter.java#startActivityInner

&emsp; 接下来到了比较重要的环节，一些初始化的相关操作均在startActivityInner函数中进行。

```java
    int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, Task inTask,
            TaskFragment inTaskFragment, boolean restrictedBgActivity,
            NeededUriGrants intentGrants) {
        setInitialState(r, options, inTask, inTaskFragment, doResume, startFlags, sourceRecord,
                voiceSession, voiceInteractor, restrictedBgActivity);

        computeLaunchingTaskFlags();

        computeSourceRootTask();

        ... ...

        // Get top task at beginning because the order may be changed when reusing existing task.
        final Task prevTopRootTask = mPreferredTaskDisplayArea.getFocusedRootTask();
        final Task prevTopTask = prevTopRootTask != null ? prevTopRootTask.getTopLeafTask() : null;
        final Task reusedTask = getReusableTask();

        ... ...

        // Compute if there is an existing task that should be used for.
        final Task targetTask = reusedTask != null ? reusedTask : computeTargetTask();
        final boolean newTask = targetTask == null;
        mTargetTask = targetTask;

        computeLaunchParams(r, sourceRecord, targetTask);

        ... ...
        if (targetTask != null) {
            mPriorAboveTask = TaskDisplayArea.getRootTaskAbove(targetTask.getRootTask());
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

        // If the activity being launched is the same as the one currently at the top, then
        // we need to check if it should only be launched once.
        final Task topRootTask = mPreferredTaskDisplayArea.getFocusedRootTask();
        if (topRootTask != null) {
            startResult = deliverToCurrentTopIfNeeded(topRootTask, intentGrants);
            if (startResult != START_SUCCESS) {
                return startResult;
            }
        }

        if (mTargetRootTask == null) {
            mTargetRootTask = getOrCreateRootTask(mStartActivity, mLaunchFlags, targetTask,
                    mOptions);
        }
        if (newTask) {
            final Task taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                    ? mSourceRecord.getTask() : null;
            setNewTask(taskToAffiliate);
        } else if (mAddingToTask) {
            addOrReparentStartingActivity(targetTask, "adding to task");
        }

        if (!mAvoidMoveToFront && mDoResume) {
            mTargetRootTask.getRootTask().moveToFront("reuseOrNewTask", targetTask);
            if (!mTargetRootTask.isTopRootTaskInDisplayArea() && mService.isDreaming()
                    && !dreamStopping) {
                // Launching underneath dream activity (fullscreen, always-on-top). Run the launch-
                // -behind transition so the Activity gets created and starts in visible state.
                mLaunchTaskBehind = true;
                r.mLaunchTaskBehind = true;
            }
        }

        ... ...
        final Task startedTask = mStartActivity.getTask();
        if (newTask) {
            EventLogTags.writeWmCreateTask(mStartActivity.mUserId, startedTask.mTaskId);
        }
        mStartActivity.logStartActivity(EventLogTags.WM_CREATE_ACTIVITY, startedTask);

        mStartActivity.getTaskFragment().clearLastPausedActivity();

        mRootWindowContainer.startPowerModeLaunchIfNeeded(
                false /* forceSend */, mStartActivity);

        final boolean isTaskSwitch = startedTask != prevTopTask && !startedTask.isEmbedded();
        mTargetRootTask.startActivityLocked(mStartActivity, topRootTask, newTask, isTaskSwitch,
                mOptions, sourceRecord);
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
        ... ...

        return START_SUCCESS;
    }
```

* setInitialState 进行初始化赋值的相关操作

* computeLaunchingTaskFlags 根据Activity启动时设置的LaunchMode及Intent的flags计算出启动的LaunchingTaskFlags。 这会影响到Activity启动时Task的选择，如使用已存在的Task还是创建新的Task等。

* computeSourceRootTask 计算出caller所在的RootTask， 如Caller是Activity，则就是Activity所在的RootTask， 否则为null。

* getReusableTask和computeTargetTask 此处是比较重要的一环，顾名思义是"获取可重用的Task"。在Android13已经没有Stack的概念了，此前的ActivityStackSupervisor也变成了ActivityTaskSupervisor。Task是装载Activity的栈结构，在启动Activity时需要根据启动参数以及当前系统的Task状态确定将其放置在哪个Task中。简单来说就是根据将要启动Activity的Launch Flag以及与当前系统已有Task的关联性决定是使用已有Task还是创建一个新的Task。 对于当前我们分析的这种场景，首次启动自然是创建新的Task，相应的会有wm_create_task及wm_create_activity的日志打印。且此时Activity已与Task绑定，Activity处于Task的top focus

* resumeFocusedTasksTopActivities  当前新创建的Task处于Focus，调用此函数来对Task的当前TopActivity(也就时我们将要启动的Activity)进行resume。 最终调用的是Task.java#resumeTopActivityUncheckedLocked

### 8. Task.java#resumeTopActivityUncheckedLocked

&emsp; 注意，该方法至关重要，ActivityRecord以及Task此时已经准备好了，开始真正的启动工作。理一下后续的函数调用链，我们只关注重要的部分。Task.java#resumeTopActivityUnchecked -> Task.java#resumeTopActivityInnerLocked -> TaskFragment.java#resumeTopActivity

[frameworks/base/services/core/java/com/android/server/wm/TaskFragment.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/services/core/java/com/android/server/wm/TaskFragment.java)

```java
    final boolean resumeTopActivity(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
        ActivityRecord next = topRunningActivity(true /* focusableOnly */);
        if (next == null || !next.canResumeByCompat()) {
            return false;
        }

        ... ...

        boolean pausing = !deferPause && taskDisplayArea.pauseBackTasks(next);
        if (mResumedActivity != null) {
            ProtoLog.d(WM_DEBUG_STATES, "resumeTopActivity: Pausing %s", mResumedActivity);
            pausing |= startPausing(mTaskSupervisor.mUserLeaving, false /* uiSleeping */,
                    next, "resumeTopActivity");
        }
        if (pausing) {
            ProtoLog.v(WM_DEBUG_STATES, "resumeTopActivity: Skip resume: need to"
                    + " start pausing");
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
                final boolean isTop = this == taskDisplayArea.getFocusedRootTask();
                mAtmService.startProcessAsync(next, false /* knownToBeDead */, isTop,
                        isTop ? HostingRecord.HOSTING_TYPE_NEXT_TOP_ACTIVITY
                                : HostingRecord.HOSTING_TYPE_NEXT_ACTIVITY);
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
            ProtoLog.d(WM_DEBUG_STATES, "resumeTopActivity: Top activity resumed "
                    + "(dontWaitForPause) %s", next);
            return true;
        }

        ... ...

        if (next.attachedToProcess()) {
            if (DEBUG_SWITCH) {
                Slog.v(TAG_SWITCH, "Resume running: " + next + " stopped=" + next.stopped
                        + " visibleRequested=" + next.mVisibleRequested);
            }

            ... ...

            try {
                final ClientTransaction transaction =
                        ClientTransaction.obtain(next.app.getThread(), next.token);
                // Deliver all pending results.
                ArrayList<ResultInfo> a = next.results;
                if (a != null) {
                    final int size = a.size();
                    if (!next.finishing && size > 0) {
                        if (DEBUG_RESULTS) {
                            Slog.v(TAG_RESULTS, "Delivering results to " + next + ": " + a);
                        }
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

                mAtmService.getAppWarningsLocked().onResumeActivity(next);
                next.app.setPendingUiCleanAndForceProcessStateUpTo(mAtmService.mTopProcessState);
                next.abortAndClearOptionsAnimation();
                transaction.setLifecycleStateRequest(
                        ResumeActivityItem.obtain(next.app.getReportedProcState(),
                                dc.isNextTransitionForward()));
                mAtmService.getLifecycleManager().scheduleTransaction(transaction);

                ProtoLog.d(WM_DEBUG_STATES, "resumeTopActivity: Resumed %s", next);
            } catch (Exception e) {
                // Whoops, need to restart this activity!
                ProtoLog.v(WM_DEBUG_STATES, "Resume failed; resetting state to %s: "
                        + "%s", lastState, next);
                next.setState(lastState, "resumeTopActivityInnerLocked");

                // lastResumedActivity being non-null implies there is a lastStack present.
                if (lastResumedActivity != null) {
                    lastResumedActivity.setState(RESUMED, "resumeTopActivityInnerLocked");
                }

                Slog.i(TAG, "Restarting because process died: " + next);
                if (!next.hasBeenLaunched) {
                    next.hasBeenLaunched = true;
                } else if (SHOW_APP_STARTING_PREVIEW && lastFocusedRootTask != null
                        && lastFocusedRootTask.isTopRootTaskInDisplayArea()) {
                    next.showStartingWindow(false /* taskSwitch */);
                }
                mTaskSupervisor.startSpecificActivity(next, true, false);
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
                    next.showStartingWindow(false /* taskSwich */);
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH, "Restarting: " + next);
            }
            ProtoLog.d(WM_DEBUG_STATES, "resumeTopActivity: Restarting %s", next);
            mTaskSupervisor.startSpecificActivity(next, true, true);
        }

        return true;
    }
```

* startPausing； 正常来说，启动next Activity需要先pause current Activity，此处是一个知识点(当然也有例外，对于Launcher而言，为了回到Home界面的用户体验，不会等待上个Activity Pause完成，Launcher设置了resumeWhilePausing来达到此种效果)；从代码中可以看到，pause上个Activity之后该方法就直接返回了，需要等待上个Activity Pause完成后，下个Activity的启动流程才会继续。(另外此处还有个Android版本间的优化差异，那就是进程的启动时机，在Android10以及之前，是pause完成后再走resumeTopActivity进行启动Activity所在进程的操作， 但在Android11及之后，开始Pause之后，发现要启动的下个Activity所在进程不存在就开始异步启动startProcessAsync)。

* 思考一个问题，开始Pause上个Activity后，该流程就直接return了，那么后续的启动流程是如何继续的？ System Server通过binder调用 通知 Client端的Activity进行pause，Client端的Activity Pause完成之后，则会再通过binder调用(activityPaused)告知SystemServer Pause已经完成，而后SystemServer会继续调用resumeTopActivity进行resume流程。
  
  调用链为
  
  TaskFragment.java#startPausing 
  
  -> TaskFragment.java#schedulePauseActivity
  
  -> ClientLifecycleManager.java#scheduleTransaction
  
  -> (binder调用) ClientTransaction.java#schedule 
  
  -> ClientTransactionHandler.java#scheduleTransaction 
  
  -> sendMessage(ActivityThread.H.EXECUTE_TRANSACTION, transaction) 
  
  -> TransactionExecutor.java#execute 
  
  -> TransactionExecutor.java#executeCallbacks 
  
  -> PauseActivityItem.java#execute 
  
  -> PauseActivityItem.java#postExecute 
  
  -> ActivityClient.java#activityPaused 
  
  -> (binder调用)ActivityClientController.java#activityPaused 
  
  -> ActivityRecord.java#activityPaused 
  
  -> TaskFragment.java#completePause 
  
  -> RootWindowContainer.java#resumeFocusedTasksTopActivities 
  
  -> Task.java#resumeTopActivityUncheckedLocked
  
  该调用链较长，读者可以据此在源码中将该段流程捋顺，总之，上个Activity Pause完成后又回到了下个Activity的resume流程。

* 进程启动。Android组件是依附进程存在的，因此组件在启动前都会检查所依附的进程是否存在，如不存在则先启动进程。这里总结下进程启动的调用链：
  
  ActivityTaskManagerService.java#startProcessAsync 
  
  -> ActivityManagerService.java\$LocalService#startProcess 
  
  -> ActivityManagerService.java#startProcessLocked 
  
  -> ProcessList.java#startProcessLocked(此处会传入进程的入口启动类entryPoint android.app.ActivityThread) 
  
  -> ProcessList.java#startProcess 
  
  -> Process.java#start
  
  简述下进程启动的流程，构造空的ProcessRecord其pid为0，并将进程相关的参数填入(比较关键的就是进程的入口函数android.app.ActivityThread#main)，然后通过Scoket与Zygote通信，将进程通过zygote孵化出来，然后将生成的pid装入ProcessRecord，开始执行入口函数ActivityThread#main。 这只是个简略的过程，进程的启动详细说起来也比较复杂，放在单独专题再详细进行分析。

* 进程启动之后。 还是先总结下调用链：
  
  ActivityThread.java#main 
  
  -> ActivityThread.java#attach 
  
  ->  (binder调用) ActivityManagerService.java#attachApplication 
  
  -> ActivityManagerService.java#attachApplicationLocked 
  
  -> (binder调用)ActivityThread.java#bindApplication
  
  Client通过binder调用将自身attach至SystemServer，SystemServer通过Binder的callingPid与ProcessRecord记录相匹配，再次通过binder调用bindApplication绑定Client，这样新创建的进程就与SystemServer建立了链接，为后续的Client-Server通信做好准备。

* ActivityTaskSupervisor.java#startSpecificActivity 此时Activity依附的进程已经启动完毕，并且可以正常与SystemServer进行通信，但是Activity与进程的连接还未建立，因此会走startSpecificActivity的流程。

### 9. ActivityTaskSupervisor.java#startSpecificActivity

&emsp; 此时进程已经存在，接下来就是启动Activity，并建立Activity与进程的关系。调用至ActivityTaskSupervisor.java#realStartActivityLocked

[frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java)

```java
    boolean realStartActivityLocked(ActivityRecord r, WindowProcessController proc,
            boolean andResume, boolean checkConfig) throws RemoteException {

            ... ...

            try {
                if (!proc.hasThread()) {
                    throw new RemoteException();
                }
                List<ResultInfo> results = null;
                List<ReferrerIntent> newIntents = null;
                if (andResume) {
                    // We don't need to deliver new intents and/or set results if activity is going
                    // to pause immediately after launch.
                    results = r.results;
                    newIntents = r.newIntents;
                }
                if (DEBUG_SWITCH) Slog.v(TAG_SWITCH,
                        "Launching: " + r + " savedState=" + r.getSavedState()
                                + " with results=" + results + " newIntents=" + newIntents
                                + " andResume=" + andResume);
                EventLogTags.writeWmRestartActivity(r.mUserId, System.identityHashCode(r),
                        task.mTaskId, r.shortComponentName);
                ... ...

                // Create activity launch transaction.
                final ClientTransaction clientTransaction = ClientTransaction.obtain(
                        proc.getThread(), r.token);

                final boolean isTransitionForward = r.isTransitionForward();
                final IBinder fragmentToken = r.getTaskFragment().getFragmentToken();
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

                // Set desired final state.
                final ActivityLifecycleItem lifecycleItem;
                if (andResume) {
                    lifecycleItem = ResumeActivityItem.obtain(isTransitionForward);
                } else {
                    lifecycleItem = PauseActivityItem.obtain();
                }
                clientTransaction.setLifecycleStateRequest(lifecycleItem);

                // Schedule transaction.
                mService.getLifecycleManager().scheduleTransaction(clientTransaction);

                ... ...
        return true;
    }


```

* wm_restart_activity 该event log在此处进行打印，表示开始真正的启动Activity，因此re是real而非重新的意思。

* 通过LaunchActivityItem和ResumeActivityItem构建ClientTransaction和ActivityLifecycleItem进行Activity的启动。



### 10. LaunchActivityItem.java#execute

&emsp; 此时，SystemServer通过binder调用至Client端，Client收到启动信息后开始在自身进程中进行Activity的启动。

[frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/android/app/servertransaction/LaunchActivityItem.java)

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

&emsp; 从代码中可以看到，基于Server传过来的信息，Client端构造了ActivityClientRecord。而后调用了ActivityThread.java#handleLaunchActivity(ActivityThread extends ClientTransactionHandler)。

### 11. ActivityThread#handleLaunchActivity

&emsp; ActivityThread可以理解为进程的主线程，此时开始在主线程上进行launch activity的工作。后续调用了ActivityThread.java#performLaunchActivity，下面我们来看该核心代码

[frameworks/base/core/java/android/app/ActivityThread.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/android/app/ActivityThread.java)

```java
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

            if (localLOGV) Slog.v(TAG, "Performing launch of " + r);
            if (localLOGV) Slog.v(
                    TAG, r + ": app=" + app
                    + ", appName=" + app.getPackageName()
                    + ", pkg=" + r.packageInfo.getPackageName()
                    + ", comp=" + r.intent.getComponent().toShortString()
                    + ", dir=" + r.packageInfo.getAppDir());

            // updatePendingActivityConfiguration() reads from mActivities to update
            // ActivityClientRecord which runs in a different thread. Protect modifications to
            // mActivities to avoid race.
            synchronized (mResourcesManager) {
                mActivities.put(r.token, r);
            }

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

                if (customIntent != null) {
                    activity.mIntent = customIntent;
                }
                r.lastNonConfigurationInstances = null;
                checkAndBlockForNetworkAccess();
                activity.mStartedActivity = false;
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);
                }

                if (r.mActivityOptions != null) {
                    activity.mPendingOptions = r.mActivityOptions;
                    r.mActivityOptions = null;
                }
                activity.mLaunchedFromBubble = r.mLaunchedFromBubble;
                activity.mCalled = false;
                // Assigning the activity to the record before calling onCreate() allows
                // ActivityThread#getActivity() lookup for the callbacks triggered from
                // ActivityLifecycleCallbacks#onActivityCreated() or
                // ActivityLifecycleCallback#onActivityPostCreated().
                r.activity = activity;
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

        } catch (SuperNotCalledException e) {
            throw e;

        } catch (Exception e) {
            if (!mInstrumentation.onException(activity, e)) {
                throw new RuntimeException(
                    "Unable to start activity " + component
                    + ": " + e.toString(), e);
            }
        }

        return activity;
    }
```

* 为Activity创建上下文Context createBaseContextForActivity

* 调用Instrumentation.java#newActivity并传入ClassLoader、Component、Intent构造Activity实例

* 通过LoadedApk.java#makeApplicationInner构造Application实例，并做好Application相关的初始化操作，如attath、create等。具体情况读者可自行查看该方法内容

* 调用Activity.java#attach将Activity实例与进程主线程建立链接

* 通过Instrumentation调用callActivityIOnCreate使Activity实例进入onCreate生命周期。在event log中，wm_on_xxx\_called就表示某生命周期已被调用完毕。



### 12. ActivityThread.java#handleResumeActivity

&emsp; 生命周期的调度大都是类似的，基本都是SystemServer检测到场景变化，调度Client端Activity实例进行相应生命周期的变化，Client端生命周期执行完毕后再回调告知Server端已完成，Server端将调度的超时取消掉。下面来看下比较重要的Resume生命周期

[frameworks/base/core/java/android/app/ActivityThread.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/android/app/ActivityThread.java)

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
        if (mActivitiesToBeDestroyed.containsKey(r.token)) {
            // Although the activity is resumed, it is going to be destroyed. So the following
            // UI operations are unnecessary and also prevents exception because its token may
            // be gone that window manager cannot recognize it. All necessary cleanup actions
            // performed below will be done while handling destruction.
            return;
        }

        final Activity a = r.activity;

        if (localLOGV) {
            Slog.v(TAG, "Resume " + r + " started activity: " + a.mStartedActivity
                    + ", hideForNow: " + r.hideForNow + ", finished: " + a.mFinished);
        }

        final int forwardBit = isForward
                ? WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION : 0;

        // If the window hasn't yet been added to the window manager,
        // and this guy didn't finish itself or start another activity,
        // then go ahead and add the window.
        boolean willBeVisible = !a.mStartedActivity;
        if (!willBeVisible) {
            willBeVisible = ActivityClient.getInstance().willActivityBeVisible(
                    a.getActivityToken());
        }
        if (r.window == null && !a.mFinished && willBeVisible) {
            r.window = r.activity.getWindow();
            View decor = r.window.getDecorView();
            decor.setVisibility(View.INVISIBLE);
            ViewManager wm = a.getWindowManager();
            WindowManager.LayoutParams l = r.window.getAttributes();
            a.mDecor = decor;
            l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
            l.softInputMode |= forwardBit;
            if (r.mPreserveWindow) {
                a.mWindowAdded = true;
                r.mPreserveWindow = false;
                // Normally the ViewRoot sets up callbacks with the Activity
                // in addView->ViewRootImpl#setView. If we are instead reusing
                // the decor view we have to notify the view root that the
                // callbacks may have changed.
                ViewRootImpl impl = decor.getViewRootImpl();
                if (impl != null) {
                    impl.notifyChildRebuilt();
                }
            }
            if (a.mVisibleFromClient) {
                if (!a.mWindowAdded) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                } else {
                    // The activity will get a callback for this {@link LayoutParams} change
                    // earlier. However, at that time the decor will not be set (this is set
                    // in this method), so no action will be taken. This call ensures the
                    // callback occurs with the decor set.
                    a.onWindowAttributesChanged(l);
                }
            }

            // If the window has already been added, but during resume
            // we started another activity, then don't yet make the
            // window visible.
        } else if (!willBeVisible) {
            if (localLOGV) Slog.v(TAG, "Launch " + r + " mStartedActivity set");
            r.hideForNow = true;
        }

        // Get rid of anything left hanging around.
        cleanUpPendingRemoveWindows(r, false /* force */);

        // The window is now visible if it has been added, we are not
        // simply finishing, and we are not starting another activity.
        if (!r.activity.mFinished && willBeVisible && r.activity.mDecor != null && !r.hideForNow) {
            if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward=" + isForward);
            ViewRootImpl impl = r.window.getDecorView().getViewRootImpl();
            WindowManager.LayoutParams l = impl != null
                    ? impl.mWindowAttributes : r.window.getAttributes();
            if ((l.softInputMode
                    & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                    != forwardBit) {
                l.softInputMode = (l.softInputMode
                        & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                        | forwardBit;
                if (r.activity.mVisibleFromClient) {
                    ViewManager wm = a.getWindowManager();
                    View decor = r.window.getDecorView();
                    wm.updateViewLayout(decor, l);
                }
            }

            r.activity.mVisibleFromServer = true;
            mNumVisibleActivities++;
            if (r.activity.mVisibleFromClient) {
                r.activity.makeVisible();
            }
        }

        r.nextIdle = mNewActivities;
        mNewActivities = r;
        if (localLOGV) Slog.v(TAG, "Scheduling idle handler for " + r);
        Looper.myQueue().addIdleHandler(new Idler());
    }
```

&emsp; 相应的，Resume的流程跟Create流程基本一致。我们需要关注的是resume之后还有哪些操作。

* ViewManager.addView 这部分逻辑是关于界面绘制的，这里也不详细赘述，读者只要知道是在resume之后开始界面的真正绘制就可以。可能会在其他专题进行详细的分析

* addIdleHandler 这是MessageQueue的一个机制，加入一个IdleHandler进去，待到MessageQueue空闲时，执行其queueIdle方法，目的是将一些次重要的操作放到主线程空闲的时候再去完成。那么在Activity启动流程中，主线程空闲后做了什么操作

[Idler](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/android/app/ActivityThread.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=2350?gsn=Idler&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dandroid.app.ActivityThread.Idler%23cda894538a8b71087449caf30b78f802190c1f7ba77209b46fe772fc3b3e0920)

```java
    private class Idler implements MessageQueue.IdleHandler {
        @Override
        public final boolean queueIdle() {
            ActivityClientRecord a = mNewActivities;
            boolean stopProfiling = false;
            if (mBoundApplication != null && mProfiler.profileFd != null
                    && mProfiler.autoStopProfiler) {
                stopProfiling = true;
            }
            if (a != null) {
                mNewActivities = null;
                final ActivityClient ac = ActivityClient.getInstance();
                ActivityClientRecord prev;
                do {
                    if (localLOGV) Slog.v(
                        TAG, "Reporting idle of " + a +
                        " finished=" +
                        (a.activity != null && a.activity.mFinished));
                    if (a.activity != null && !a.activity.mFinished) {
                        ac.activityIdle(a.token, a.createdConfig, stopProfiling);
                        a.createdConfig = null;
                    }
                    prev = a;
                    a = a.nextIdle;
                    prev.nextIdle = null;
                } while (a != null);
            }
            if (stopProfiling) {
                mProfiler.stopProfiling();
            }
            return false;
        }
    }
```

通过代码可以看到，idle之后调用了ActivityClient的activityIdle方法，也就是通过binder调用告诉SystemServer此时处于activityIdle状态。SystemServer一侧真正执行的是ActivityClientController.java#activityIdle



### 13. ActivityClientController.java#activityIdle

&emsp; 而activityIdle又调用到了ActivityTaskSupervisor.java#activityIdleInternal，因此我们来看下这段代码。

[frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java)

```java
    void activityIdleInternal(ActivityRecord r, boolean fromTimeout,
            boolean processPausingActivities, Configuration config) {
        if (DEBUG_ALL) Slog.v(TAG, "Activity idle: " + r);

        if (r != null) {
            if (DEBUG_IDLE) Slog.d(TAG_IDLE, "activityIdleInternal: Callers="
                    + Debug.getCallers(4));
            mHandler.removeMessages(IDLE_TIMEOUT_MSG, r);
            r.finishLaunchTickingLocked();
            if (fromTimeout) {
                reportActivityLaunched(fromTimeout, r, INVALID_DELAY, -1 /* launchState */);
            }

            // This is a hack to semi-deal with a race condition
            // in the client where it can be constructed with a
            // newer configuration from when we asked it to launch.
            // We'll update with whatever configuration it now says
            // it used to launch.
            if (config != null) {
                r.setLastReportedGlobalConfiguration(config);
            }

            // We are now idle.  If someone is waiting for a thumbnail from
            // us, we can now deliver.
            r.idle = true;

            // Check if able to finish booting when device is booting and all resumed activities
            // are idle.
            if ((mService.isBooting() && mRootWindowContainer.allResumedActivitiesIdle())
                    || fromTimeout) {
                checkFinishBootingLocked();
            }

            // When activity is idle, we consider the relaunch must be successful, so let's clear
            // the flag.
            r.mRelaunchReason = RELAUNCH_REASON_NONE;
        }

        if (mRootWindowContainer.allResumedActivitiesIdle()) {
            if (r != null) {
                mService.scheduleAppGcsLocked();
                mRecentTasks.onActivityIdle(r);
            }

            if (mLaunchingActivityWakeLock.isHeld()) {
                mHandler.removeMessages(LAUNCH_TIMEOUT_MSG);
                if (VALIDATE_WAKE_LOCK_CALLER &&
                        Binder.getCallingUid() != Process.myUid()) {
                    throw new IllegalStateException("Calling must be system uid");
                }
                mLaunchingActivityWakeLock.release();
            }
            mRootWindowContainer.ensureActivitiesVisible(null, 0, !PRESERVE_WINDOWS);
        }

        // Atomically retrieve all of the other things to do.
        processStoppingAndFinishingActivities(r, processPausingActivities, "idle");

        if (DEBUG_IDLE) {
            Slogf.i(TAG, "activityIdleInternal(): r=%s, mStartingUsers=%s", r, mStartingUsers);
        }

        if (!mStartingUsers.isEmpty()) {
            final ArrayList<UserState> startingUsers = new ArrayList<>(mStartingUsers);
            mStartingUsers.clear();
            // Complete user switch.
            for (int i = 0; i < startingUsers.size(); i++) {
                UserState userState = startingUsers.get(i);
                Slogf.i(TAG, "finishing switch of user %d", userState.mHandle.getIdentifier());
                mService.mAmInternal.finishUserSwitch(userState);
            }
        }

        mService.mH.post(() -> mService.mAmInternal.trimApplications());
    }
```

&emsp; 还记得启动下一个Activity需要先Pause上一个Activity么？其后续的Stop流程就是在这里进行的(processStoppingAndFinishingActivities)。其实从这里我们就可以看出系统的设计理念，启动的Activity是将要于用户交互的属于重要工作，而pause的Activity将要退到后台，相对而言不是那么重要，因此需要为重要工作让行，待到主线程空闲时(这里是当前Activity Resume完成)再进行处理。

&emsp; 试想一下，如果你的Activity有类似release的操作，放在onStop/onDestory里是否合适，倘若下个Activity的resume执行时间较长，那么上个Activity的stop就会被延后，可能出现问题。



### 总结

&emsp; 至此，其实Activity的启动流程就介绍完了，在介绍的过程中，笔者忽略了分支的细节，将能够看到的比较重要的主干内容做了介绍。至于细节方面，只要对主干流程比较熟悉，在遇到问题时也知道该去看哪部分的代码，具体问题进行具体分析。其中"进程启动"和"界面绘制"部分笔者只是一笔带过，后续可能会有专门的文章来进行介绍。

&emsp; 此时分析完Activity启动流程，我们再回过头来看文章开头附上的日志，整个流程就足够清晰了。如果面试时被问到这个问题，就不会简简单单的回答create-resume生命周期了。










