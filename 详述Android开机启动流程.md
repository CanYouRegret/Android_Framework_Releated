# 详述Android开机启动流程

## 前言

&emsp; 长按开机键，到手机屏幕出现开机logo，再到主界面Launcher显示出来，这中间经历了怎样的流程？

&emsp; 我们熟知的Zygote、SystemServer进程是在何时启动的？

&emsp; 各个系统服务(如AMS PKMS等)是在何时启动的？

我们照例先提出几个问题，然后在后续的分析过程中看能否找到答案。

## Android开机流程

### 1. 加载BootLoader

&emsp; 当长按开机键的时候，芯片会执行固化在ROM上的预设代码，将BootLoader加载到RAM，BootLoader被称为引导程序，是在操作系统运行之前运行的一段程序，也是被运行的第一个程序，其作用就是把操作系统映像文件拷贝到RAM中去，然后跳到入口开始执行。

该程序的细节笔者也尚未清楚，我们只需要知道最后会调用到Android的Kernel就行了。

### 2. 启动Kernel

&emsp; Android Kernel的入口函数是start\_kernel，可以理解为Kernel的main函数。具体位置位于[common/init/main.c](https://cs.android.com/android/kernel/superproject/+/common-android-mainline:common/init/main.c) 下面理一下调用链，我们只关注主干重要的部分：

main.c#start\_kernel

-> arch_call_rest_init

-> rest_init

-> kernel_init

-> try_to_run_init_process

在这个过程中，做的基本是初始化内核的操作；最终，通过启动init的二进制可执行文件来启动init进程

### 3. 启动init进程

&emsp; init进程的入口函数为system/core/init/main.cpp#main，是Android系统中用户空间的第一个进程，其进程号为1，最终会通过解析init.rc来启动Zygote。下面我们来看下调用链：

system/core/init/main.cpp#main

-> system/core/init/first_stage_init.cpp#FirstStageMain

-> system/core/init/main.cpp#main(通过execv函数再次调用init并传参selinux\_setup)

-> system/core/init/selinux.cpp#SetupSelinux

-> system/core/init/main.cpp#main(通过execv函数再次调用init并传参second\_stage)

-> system/core/init/init.cpp#SecondStageMain

-> system/core/init/init.cpp#LoadBootScripts(load并解析.rc文件)

-> system/core/init/action_manager.cpp#ExecuteOneCommnad(运行.rc文件里的内容)

&emsp; .rc文件的语法大致分为三种service、on、import，对应三种解析器ServiceParser(解析要启动的服务)、ActionParser(类似于条件监听回调，如当late-init被执行时，on late-init相关内容会回调执行)、ImportParser(导入相关的.rc文件)。

&emsp; 我们这里只关注zygote是如何被启动的。在system/core/init/init.cpp#SecondStageMain（[am](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:system/core/init/init.cpp;drc=b45a2ea782074944f79fc388df20b06e01f265f7;l=1092).[QueueEventTrigger](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:system/core/init/action_manager.cpp;drc=b45a2ea782074944f79fc388df20b06e01f265f7;l=43) ("late-init") ）时会触发late-init。相应的对应on late-init在init.rc文件的内容如下:

[system/core/rootdir/init.rc](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:system/core/rootdir/init.rc) 

```
on late-init
    trigger early-fs

    ... ...

    # Now we can start zygote for devices with file based encryption
    trigger zygote-start


# It is recommended to put unnecessary data/ initialization from post-fs-data
# to start-zygote in device's init.rc to unblock zygote start.
on zygote-start && property:ro.crypto.state=unencrypted
    wait_for_prop odsign.verification.done 1
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start statsd
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=unsupported
    wait_for_prop odsign.verification.done 1
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start statsd
    start netd
    start zygote
    start zygote_secondary

on zygote-start && property:ro.crypto.state=encrypted && property:ro.crypto.type=file
    wait_for_prop odsign.verification.done 1
    # A/B update verifier that marks a successful boot.
    exec_start update_verifier_nonencrypted
    start statsd
    start netd
    start zygote
    start zygote_secondary
```

&emsp; 从上面的code可以看到late-init中又触发了zygote-start，进而start zygote开始启动zygote。从system/core/init/builtins.cpp#GetBuiltinFunctionMap方法中我们可以查到start对应的操作函数为[do_start](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:system/core/init/builtins.cpp;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=762?gsn=do_start&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dsystem%2Fcore%2Finit%2Fbuiltins.cpp%23ZvRc8nB94eJd-kelLkHghS_7DRVYuRCagScRCRej7_0) 进而开始Zygote这个service的Start。实质上就是从[system/core/rootdir/init.zygote64_32.rc](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:system/core/rootdir/init.zygote64_32.rc) 文件中将zygote启动的相关参数信息解析。

```
service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote stream 660 root system
    socket usap_pool_primary stream 660 root system
    onrestart exec_background - system system -- /system/bin/vdc volume abort_fuse
    onrestart write /sys/power/state on
    onrestart restart audioserver
    onrestart restart cameraserver
    onrestart restart media
    onrestart restart media.tuner
    onrestart restart netd
    onrestart restart wificond
    task_profiles ProcessCapacityHigh MaxPerformance
    critical window=${zygote.critical_window.minute:-off} target=zygote-fatal

service zygote_secondary /system/bin/app_process32 -Xzygote /system/bin --zygote --socket-name=zygote_secondary --enable-lazy-preload
    class main
    priority -20
    user root
    group root readproc reserved_disk
    socket zygote_secondary stream 660 root system
    socket usap_pool_secondary stream 660 root system
    onrestart restart zygote
    task_profiles ProcessCapacityHigh MaxPerformance
```

&emsp; 从zygote相关的.rc文件中我们可以看到，启动zygote实际执行的二进制文件为/system/bin/app\_process64，接下来进入Zygote进程的启动流程。

### 3. 启动Zygote进程

&emsp; Zygote有"受精卵"的意思，也是Java一侧启动的首个进程，后续所有的java进程都是由其fork出来的，包括System Server。

Zygote进程的入口main函数位于

[frameworks/base/cmds/app_process/app_main.cpp](frameworks/base/cmds/app_process/app_main.cpp)

```cpp
int main(int argc, char* const argv[])
{
    ... ...
    AppRuntime runtime(argv[0], computeArgBlockSize(argc, argv));
    // Process command line arguments
    // ignore argv[0]
    argc--;
    argv++;

    // Everything up to '--' or first non '-' arg goes to the vm.
    //
    // The first argument after the VM args is the "parent dir", which
    // is currently unused.
    //
    // After the parent dir, we expect one or more the following internal
    // arguments :
    //
    // --zygote : Start in zygote mode
    // --start-system-server : Start the system server.
    // --application : Start in application (stand alone, non zygote) mode.
    // --nice-name : The nice name for this process.
    //
    // For non zygote starts, these arguments will be followed by
    // the main class name. All remaining arguments are passed to
    // the main method of this class.
    //
    // For zygote starts, all remaining arguments are passed to the zygote.
    // main function.
    //
    // Note that we must copy argument string values since we will rewrite the
    // entire argument block when we apply the nice name to argv0.
    //
    // As an exception to the above rule, anything in "spaced commands"
    // goes to the vm even though it has a space in it.
    const char* spaced_commands[] = { "-cp", "-classpath" };
    // Allow "spaced commands" to be succeeded by exactly 1 argument (regardless of -s).
    bool known_command = false;

    int i;
    for (i = 0; i < argc; i++) {
        if (known_command == true) {
          runtime.addOption(strdup(argv[i]));
          // The static analyzer gets upset that we don't ever free the above
          // string. Since the allocation is from main, leaking it doesn't seem
          // problematic. NOLINTNEXTLINE
          ALOGV("app_process main add known option '%s'", argv[i]);
          known_command = false;
          continue;
        }

        for (int j = 0;
             j < static_cast<int>(sizeof(spaced_commands) / sizeof(spaced_commands[0]));
             ++j) {
          if (strcmp(argv[i], spaced_commands[j]) == 0) {
            known_command = true;
            ALOGV("app_process main found known command '%s'", argv[i]);
          }
        }

        if (argv[i][0] != '-') {
            break;
        }
        if (argv[i][1] == '-' && argv[i][2] == 0) {
            ++i; // Skip --.
            break;
        }

        runtime.addOption(strdup(argv[i]));
        // The static analyzer gets upset that we don't ever free the above
        // string. Since the allocation is from main, leaking it doesn't seem
        // problematic. NOLINTNEXTLINE
        ALOGV("app_process main add option '%s'", argv[i]);
    }

    // Parse runtime arguments.  Stop at first unrecognized option.
    bool zygote = false;
    bool startSystemServer = false;
    bool application = false;
    String8 niceName;
    String8 className;

    ++i;  // Skip unused "parent dir" argument.
    while (i < argc) {
        const char* arg = argv[i++];
        if (strcmp(arg, "--zygote") == 0) {
            zygote = true;
            niceName = ZYGOTE_NICE_NAME;
        } else if (strcmp(arg, "--start-system-server") == 0) {
            startSystemServer = true;
        } else if (strcmp(arg, "--application") == 0) {
            application = true;
        } else if (strncmp(arg, "--nice-name=", 12) == 0) {
            niceName.setTo(arg + 12);
        } else if (strncmp(arg, "--", 2) != 0) {
            className.setTo(arg);
            break;
        } else {
            --i;
            break;
        }
    }

    Vector<String8> args;
    if (!className.isEmpty()) {
        // We're not in zygote mode, the only argument we need to pass
        // to RuntimeInit is the application argument.
        //
        // The Remainder of args get passed to startup class main(). Make
        // copies of them before we overwrite them with the process name.
        args.add(application ? String8("application") : String8("tool"));
        runtime.setClassNameAndArgs(className, argc - i, argv + i);

        if (!LOG_NDEBUG) {
          String8 restOfArgs;
          char* const* argv_new = argv + i;
          int argc_new = argc - i;
          for (int k = 0; k < argc_new; ++k) {
            restOfArgs.append("\"");
            restOfArgs.append(argv_new[k]);
            restOfArgs.append("\" ");
          }
          ALOGV("Class name = %s, args = %s", className.string(), restOfArgs.string());
        }
    } else {
        // We're in zygote mode.
        maybeCreateDalvikCache();

        if (startSystemServer) {
            args.add(String8("start-system-server"));
        }

        char prop[PROP_VALUE_MAX];
        if (property_get(ABI_LIST_PROPERTY, prop, NULL) == 0) {
            LOG_ALWAYS_FATAL("app_process: Unable to determine ABI list from property %s.",
                ABI_LIST_PROPERTY);
            return 11;
        }

        String8 abiFlag("--abi-list=");
        abiFlag.append(prop);
        args.add(abiFlag);

        // In zygote mode, pass all remaining arguments to the zygote
        // main() method.
        for (; i < argc; ++i) {
            args.add(String8(argv[i]));
        }
    }

    if (!niceName.isEmpty()) {
        runtime.setArgv0(niceName.string(), true /* setProcName */);
    }

    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
    }
}
```

&emsp; 从上述代码我们可以看到，main函数大致分为以下几个流程

1）解析启动参数。

--zygote : Start in zygote mode
--start-system-server : Start the system server.
--application : Start in application (stand alone, non zygote) mode.
--nice-name : The nice name for this process.

2）启动AppRuntime

--zygote时表示启动的是zygote进程，相应的传入参数为ZygoteInit

--application时表示启动的是应用进程，相应的传入参数为RuntimeInit

那么为什么在init进程的main函数中会有应用进程启动的分支选项呢？是因为Java应用进程启动时也会走init进程的main函数么？ 答案是肯定的，Zygote进程孵化子进程是会把自己的资源复制一份来fork一个新的进程，也就是说子进程也会进入到这个文件的main函数中。

### 4. AndroidRuntime.cpp#start

&emsp; AppRuntime是继承自AndroidRuntime的，因此我们来看下AndroidRuntime的start函数。

[frameworks/base/core/jni/AndroidRuntime.cpp](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/jni/AndroidRuntime.cpp;bpv=0;bpt=1)

```cpp
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    ALOGD(">>>>>> START %s uid %d <<<<<<\n",
            className != NULL ? className : "(unknown)", getuid());

    static const String8 startSystemServer("start-system-server");
    // Whether this is the primary zygote, meaning the zygote which will fork system server.
    bool primary_zygote = false;

    /*
     * 'startSystemServer == true' means runtime is obsolete and not run from
     * init.rc anymore, so we print out the boot start event here.
     */
    for (size_t i = 0; i < options.size(); ++i) {
        if (options[i] == startSystemServer) {
            primary_zygote = true;
           /* track our progress through the boot sequence */
           const int LOG_BOOT_PROGRESS_START = 3000;
           LOG_EVENT_LONG(LOG_BOOT_PROGRESS_START,  ns2ms(systemTime(SYSTEM_TIME_MONOTONIC)));
        }
    }

    ... ...

    //const char* kernelHack = getenv("LD_ASSUME_KERNEL");
    //ALOGD("Found LD_ASSUME_KERNEL='%s'\n", kernelHack);

    /* start the virtual machine */
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote, primary_zygote) != 0) {
        return;
    }
    onVmCreated(env);

    /*
     * Register android functions.
     */
    if (startReg(env) < 0) {
        ALOGE("Unable to register all android natives\n");
        return;
    }

    /*
     * We want to call main() with a String array with arguments in it.
     * At present we have two arguments, the class name and an option string.
     * Create an array to hold them.
     */
    jclass stringClass;
    jobjectArray strArray;
    jstring classNameStr;

    stringClass = env->FindClass("java/lang/String");
    assert(stringClass != NULL);
    strArray = env->NewObjectArray(options.size() + 1, stringClass, NULL);
    assert(strArray != NULL);
    classNameStr = env->NewStringUTF(className);
    assert(classNameStr != NULL);
    env->SetObjectArrayElement(strArray, 0, classNameStr);

    for (size_t i = 0; i < options.size(); ++i) {
        jstring optionsStr = env->NewStringUTF(options.itemAt(i).string());
        assert(optionsStr != NULL);
        env->SetObjectArrayElement(strArray, i + 1, optionsStr);
    }

    /*
     * Start VM.  This thread becomes the main thread of the VM, and will
     * not return until the VM exits.
     */
    char* slashClassName = toSlashClassName(className != NULL ? className : "");
    jclass startClass = env->FindClass(slashClassName);
    if (startClass == NULL) {
        ALOGE("JavaVM unable to locate class '%s'\n", slashClassName);
        /* keep going */
    } else {
        jmethodID startMeth = env->GetStaticMethodID(startClass, "main",
            "([Ljava/lang/String;)V");
        if (startMeth == NULL) {
            ALOGE("JavaVM unable to find main() in '%s'\n", className);
            /* keep going */
        } else {
            env->CallStaticVoidMethod(startClass, startMeth, strArray);

#if 0
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
#endif
        }
    }
    free(slashClassName);

    ALOGD("Shutting down VM\n");
    if (mJavaVM->DetachCurrentThread() != JNI_OK)
        ALOGW("Warning: unable to detach main thread\n");
    if (mJavaVM->DestroyJavaVM() != 0)
        ALOGW("Warning: VM did not shut down cleanly\n");
}
```

&emsp;  从上述代码来看，比较重要的流程有以下几个：

* [startVm](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/jni/AndroidRuntime.cpp;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=1;l=625) 设置虚拟机启动参数，并通过[JNI_CreateJavaVM](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/jni/AndroidRuntime.cpp;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=1152?gsn=JNI_CreateJavaVM&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dlibnativehelper%2Finclude_jni%2Fjni.h%23fjXx9_NCybzyIzTjay9bV9JuvBK-c3Ts28uAuNqq0-A) 方法启动虚拟机。虚拟机部分后续文章会对其进行描述，此处先不再赘述

* [startReg](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/jni/AndroidRuntime.cpp;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=1;l=1670) 通过[register_jni_procs](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/jni/AndroidRuntime.cpp;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=1496?gsn=register_jni_procs&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fbase%2Fcore%2Fjni%2FAndroidRuntime.cpp%23tVLIbctKjfdWMBk4dMBBLjaEIqb40h7k4rQX4CE2Nnk) 函数注册JNI方法，具体注册的JNI函数放在[gRegJNI](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/jni/AndroidRuntime.cpp;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=1509?gsn=gRegJNI&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Dc%252B%252B%3Fpath%3Dframeworks%2Fbase%2Fcore%2Fjni%2FAndroidRuntime.cpp%23AyN2b1sLRQvlD9vjwA5--K2TE3yzgkR0_dH9_PyFc9o) 数组中。

* 调用传入的首个参数类的main方法。

### 5. ZygoteInit.java#main

&emsp; 此时启动的是Zygote进程，因此我们来看下ZygoteInit.java的入口main函数。

[frameworks/base/core/java/com/android/internal/os/ZygoteInit.java](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java)

```java
    public static void main(String[] argv) {
        ZygoteServer zygoteServer = null;

        // Mark zygote start. This ensures that thread creation will throw
        // an error.
        ZygoteHooks.startZygoteNoThreadCreation();

        // Zygote goes into its own process group.
        try {
            Os.setpgid(0, 0);
        } catch (ErrnoException ex) {
            throw new RuntimeException("Failed to setpgid(0,0)", ex);
        }

        Runnable caller;
        try {
            // Store now for StatsLogging later.
            final long startTime = SystemClock.elapsedRealtime();
            final boolean isRuntimeRestarted = "1".equals(
                    SystemProperties.get("sys.boot_completed"));

            String bootTimeTag = Process.is64Bit() ? "Zygote64Timing" : "Zygote32Timing";
            TimingsTraceLog bootTimingsTraceLog = new TimingsTraceLog(bootTimeTag,
                    Trace.TRACE_TAG_DALVIK);
            bootTimingsTraceLog.traceBegin("ZygoteInit");
            RuntimeInit.preForkInit();

            boolean startSystemServer = false;
            String zygoteSocketName = "zygote";
            String abiList = null;
            boolean enableLazyPreload = false;
            for (int i = 1; i < argv.length; i++) {
                if ("start-system-server".equals(argv[i])) {
                    startSystemServer = true;
                } else if ("--enable-lazy-preload".equals(argv[i])) {
                    enableLazyPreload = true;
                } else if (argv[i].startsWith(ABI_LIST_ARG)) {
                    abiList = argv[i].substring(ABI_LIST_ARG.length());
                } else if (argv[i].startsWith(SOCKET_NAME_ARG)) {
                    zygoteSocketName = argv[i].substring(SOCKET_NAME_ARG.length());
                } else {
                    throw new RuntimeException("Unknown command line argument: " + argv[i]);
                }
            }

            final boolean isPrimaryZygote = zygoteSocketName.equals(Zygote.PRIMARY_SOCKET_NAME);
            if (!isRuntimeRestarted) {
                if (isPrimaryZygote) {
                    FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
                            BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__ZYGOTE_INIT_START,
                            startTime);
                } else if (zygoteSocketName.equals(Zygote.SECONDARY_SOCKET_NAME)) {
                    FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
                            BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__SECONDARY_ZYGOTE_INIT_START,
                            startTime);
                }
            }

            if (abiList == null) {
                throw new RuntimeException("No ABI list supplied.");
            }

            // In some configurations, we avoid preloading resources and classes eagerly.
            // In such cases, we will preload things prior to our first fork.
            if (!enableLazyPreload) {
                bootTimingsTraceLog.traceBegin("ZygotePreload");
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_START,
                        SystemClock.uptimeMillis());
                preload(bootTimingsTraceLog);
                EventLog.writeEvent(LOG_BOOT_PROGRESS_PRELOAD_END,
                        SystemClock.uptimeMillis());
                bootTimingsTraceLog.traceEnd(); // ZygotePreload
            }

            // Do an initial gc to clean up after startup
            bootTimingsTraceLog.traceBegin("PostZygoteInitGC");
            gcAndFinalize();
            bootTimingsTraceLog.traceEnd(); // PostZygoteInitGC

            bootTimingsTraceLog.traceEnd(); // ZygoteInit

            Zygote.initNativeState(isPrimaryZygote);

            ZygoteHooks.stopZygoteNoThreadCreation();

            zygoteServer = new ZygoteServer(isPrimaryZygote);

            if (startSystemServer) {
                Runnable r = forkSystemServer(abiList, zygoteSocketName, zygoteServer);

                // {@code r == null} in the parent (zygote) process, and {@code r != null} in the
                // child (system_server) process.
                if (r != null) {
                    r.run();
                    return;
                }
            }

            Log.i(TAG, "Accepting command socket connections");

            // The select loop returns early in the child process after a fork and
            // loops forever in the zygote.
            caller = zygoteServer.runSelectLoop(abiList);
        } catch (Throwable ex) {
            Log.e(TAG, "System zygote died with fatal exception", ex);
            throw ex;
        } finally {
            if (zygoteServer != null) {
                zygoteServer.closeServerSocket();
            }
        }

        // We're in the child process and have exited the select loop. Proceed to execute the
        // command.
        if (caller != null) {
            caller.run();
        }
    }
```

&emsp; 从上述代码来看，总结笔者认为比较重要的部分：

* 解析参数；是否启动system\_server、abi、socket name等

* 进行preload操作；Java进程都是由Zygote进程孵化的，因此在Zygote进程中进行预加载的操作，可以节省Java进程启动的时间。
  
  - [preloadClasses](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=247?gsn=preloadClasses&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.internal.os.ZygoteInit%23b47f3c26f56fdc4cf058d9561f862a5e1f5503f36efdd34533f1b60fa76c4f22) 预加载常用的类，具体位于/system/etc/preloaded-classes文件，加载方式通过Class.forName，类加载的流程放在其他文章来分析，在此不再赘述。
  
  - [cacheNonBootClasspathClassLoaders](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=0;l=373) 预加载可能会被很多其他app用到，但是没有在boot classpath里的。如/system/framework/android.hidl.base-V1.0-java.jar等，该点是为了向后兼容而添加的，具体原因可以看该方法的注释描述。
  
  - [preloadResources](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=0;l=409) 预加载资源文件
  
  - [nativePreloadAppProcessHALs](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=0;l=192) 预加载Hal相关内容
  
  - [maybePreloadGraphicsDriver](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=0;l=203) 预加载GPU驱动相关
  
  - [preloadSharedLibraries](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=0;l=185) 预加载共享库
  
  - [preloadTextResources](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=0;l=209) 预加载字符串资源
  
  - [prepareWebViewInZygote](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/android/webkit/WebViewFactory.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=0;bpt=0;l=540) 初始化webview相关内容

* 创建名为"zygote"的socket用于进程间通信; Zygote.java#nativeInitNativeState、new ZygoteServer。

* 使用forkSystemServer启动SystemServer。这里需要注意的是在Zygote进程中该方法返回为null，在SystemServer进程中该方法不返回null。为什么会这样？SystemServer是从socket孵化出来的，因此连带着zygote socket会一并复制到SystemServer进程，因此需要在SystemServer进程中关闭掉这个socket0，而zygote进程中的这个socket不能关闭，需要用来接收进程启动的请求。如何来实现这个差异的返回结果？因为fork系统调用返回的pid如果为0则表示在子进程，不为零则表示在父进程，就可以判断何时需要关闭socket了。

* 使用ZygoteServer.java#runSelectLoop开启一个死循环，socket不断监听进程启动的请求。

### 6. ZygoteInit.java#forkSystemServer

&emsp; 接下来我们来看下SystemServer的启动流程。

[ZygoteInit.java#forkSystemServer](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=683?gsn=forkSystemServer&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.internal.os.ZygoteInit%234fe8ae1a1e70eab0894831f3841a968852dd1b7ea855c444fa0fd2cbe03fc03f)

```java
    private static Runnable forkSystemServer(String abiList, String socketName,
            ZygoteServer zygoteServer) {
        ... ...
        /* Hardcoded command line to start the system server */
        String[] args = {
                "--setuid=1000",
                "--setgid=1000",
                "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1023,"
                        + "1024,1032,1065,3001,3002,3003,3005,3006,3007,3009,3010,3011,3012",
                "--capabilities=" + capabilities + "," + capabilities,
                "--nice-name=system_server",
                "--runtime-args",
                "--target-sdk-version=" + VMRuntime.SDK_VERSION_CUR_DEVELOPMENT,
                "com.android.server.SystemServer",
        };
        ZygoteArguments parsedArgs;

        int pid;

        try {
            ZygoteCommandBuffer commandBuffer = new ZygoteCommandBuffer(args);
            try {
                parsedArgs = ZygoteArguments.getInstance(commandBuffer);
            } catch (EOFException e) {
                throw new AssertionError("Unexpected argument error for forking system server", e);
            }
            commandBuffer.close();
            Zygote.applyDebuggerSystemProperty(parsedArgs);
            Zygote.applyInvokeWithSystemProperty(parsedArgs);

            if (Zygote.nativeSupportsMemoryTagging()) {
                String mode = SystemProperties.get("arm64.memtag.process.system_server", "");
                if (mode.isEmpty()) {
                  /* The system server has ASYNC MTE by default, in order to allow
                   * system services to specify their own MTE level later, as you
                   * can't re-enable MTE once it's disabled. */
                  mode = SystemProperties.get("persist.arm64.memtag.default", "async");
                }
                if (mode.equals("async")) {
                    parsedArgs.mRuntimeFlags |= Zygote.MEMORY_TAG_LEVEL_ASYNC;
                } else if (mode.equals("sync")) {
                    parsedArgs.mRuntimeFlags |= Zygote.MEMORY_TAG_LEVEL_SYNC;
                } else if (!mode.equals("off")) {
                    /* When we have an invalid memory tag level, keep the current level. */
                    parsedArgs.mRuntimeFlags |= Zygote.nativeCurrentTaggingLevel();
                    Slog.e(TAG, "Unknown memory tag level for the system server: \"" + mode + "\"");
                }
            } else if (Zygote.nativeSupportsTaggedPointers()) {
                /* Enable pointer tagging in the system server. Hardware support for this is present
                 * in all ARMv8 CPUs. */
                parsedArgs.mRuntimeFlags |= Zygote.MEMORY_TAG_LEVEL_TBI;
            }

            /* Enable gwp-asan on the system server with a small probability. This is the same
             * policy as applied to native processes and system apps. */
            parsedArgs.mRuntimeFlags |= Zygote.GWP_ASAN_LEVEL_LOTTERY;

            if (shouldProfileSystemServer()) {
                parsedArgs.mRuntimeFlags |= Zygote.PROFILE_SYSTEM_SERVER;
            }

            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.mUid, parsedArgs.mGid,
                    parsedArgs.mGids,
                    parsedArgs.mRuntimeFlags,
                    null,
                    parsedArgs.mPermittedCapabilities,
                    parsedArgs.mEffectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }

        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            zygoteServer.closeServerSocket();
            return handleSystemServerProcess(parsedArgs);
        }

        return null;
    }
```

&emsp; 这段代码没什么好讲的，就是准备参数(这里需要注意的是，参数中传入了com.android.server.SystemServer类，是被当作SystemServer进程启动后的入口类)，fork SystemServer进程，在SystemServer进程中关闭zygote socket，并执行handleSystemServerProcess；在Zygote进程返回null，执行runSelectLoop开启无限循环。(这里需要注意的是，SystemServer进程被孵化后，也会进入到app\_process的main函数中，最后进入到RuntimeInit.java#main方法)

### 7. ZygoteInit.java#handleSystemServerProcess

&emsp; 下面我们来看下，SystemServer进程启动之后做了什么工作。

[ZygoteInit.java#handleSystemServerProcess](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=505?gsn=handleSystemServerProcess&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.internal.os.ZygoteInit%23fe4025dd546ebe22b2c88764abb38f699f663ee364147dea1b7bfffd1868daaf)

```java
    private static Runnable handleSystemServerProcess(ZygoteArguments parsedArgs) {
        // set umask to 0077 so new files and directories will default to owner-only permissions.
        Os.umask(S_IRWXG | S_IRWXO);

        if (parsedArgs.mNiceName != null) {
            Process.setArgV0(parsedArgs.mNiceName);
        }

        final String systemServerClasspath = Os.getenv("SYSTEMSERVERCLASSPATH");
        if (systemServerClasspath != null) {
            // Capturing profiles is only supported for debug or eng builds since selinux normally
            // prevents it.
            if (shouldProfileSystemServer() && (Build.IS_USERDEBUG || Build.IS_ENG)) {
                try {
                    Log.d(TAG, "Preparing system server profile");
                    prepareSystemServerProfile(systemServerClasspath);
                } catch (Exception e) {
                    Log.wtf(TAG, "Failed to set up system server profile", e);
                }
            }
        }

        if (parsedArgs.mInvokeWith != null) {
            String[] args = parsedArgs.mRemainingArgs;
            // If we have a non-null system server class path, we'll have to duplicate the
            // existing arguments and append the classpath to it. ART will handle the classpath
            // correctly when we exec a new process.
            if (systemServerClasspath != null) {
                String[] amendedArgs = new String[args.length + 2];
                amendedArgs[0] = "-cp";
                amendedArgs[1] = systemServerClasspath;
                System.arraycopy(args, 0, amendedArgs, 2, args.length);
                args = amendedArgs;
            }

            WrapperInit.execApplication(parsedArgs.mInvokeWith,
                    parsedArgs.mNiceName, parsedArgs.mTargetSdkVersion,
                    VMRuntime.getCurrentInstructionSet(), null, args);

            throw new IllegalStateException("Unexpected return from WrapperInit.execApplication");
        } else {
            ClassLoader cl = getOrCreateSystemServerClassLoader();
            if (cl != null) {
                Thread.currentThread().setContextClassLoader(cl);
            }

            /*
             * Pass the remaining arguments to SystemServer.
             */
            return ZygoteInit.zygoteInit(parsedArgs.mTargetSdkVersion,
                    parsedArgs.mDisabledCompatChanges,
                    parsedArgs.mRemainingArgs, cl);
        }

        /* should never reach here */
    }
```

&emsp; 我们可以看到，当传入参数中的mInvokeWith为null时，会调用方法ZygoteInit.java#zygoteInit方法。

### 8. ZygoteInit.java#zygoteInit

&emsp; 那么zygoteInit又具体做了什么操作呢？

[ZygoteInit.java#zygoteInit](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/ZygoteInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=980?gsn=zygoteInit&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.internal.os.ZygoteInit%2370f40840d84e3a14d812a09c879f7f427a93ee6e100834e32525aeac4cec6376)

```java
    public static Runnable zygoteInit(int targetSdkVersion, long[] disabledCompatChanges,
            String[] argv, ClassLoader classLoader) {
        if (RuntimeInit.DEBUG) {
            Slog.d(RuntimeInit.TAG, "RuntimeInit: Starting application from zygote");
        }

        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ZygoteInit");
        RuntimeInit.redirectLogStreams();

        RuntimeInit.commonInit();
        ZygoteInit.nativeZygoteInit();
        return RuntimeInit.applicationInit(targetSdkVersion, disabledCompatChanges, argv,
                classLoader);
    }
```

&emsp; 我们可以看到，最终执行了RuntimeInit的applicationInit方法。

### 9. RuntimeInit.java#applicationInit

[RuntimeInit.java#applicationInit](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/core/java/com/android/internal/os/RuntimeInit.java;drc=b45a2ea782074944f79fc388df20b06e01f265f7;bpv=1;bpt=1;l=364?gsn=applicationInit&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.internal.os.RuntimeInit%23dcc1c5a3e3f0a9d990ae3c837bba71a5ea0ca4dee39b0c1ecc47312207d2e52d)

```java
    protected static Runnable applicationInit(int targetSdkVersion, long[] disabledCompatChanges,
            String[] argv, ClassLoader classLoader) {
        // If the application calls System.exit(), terminate the process
        // immediately without running any shutdown hooks.  It is not possible to
        // shutdown an Android application gracefully.  Among other things, the
        // Android runtime shutdown hooks close the Binder driver, which can cause
        // leftover running threads to crash before the process actually exits.
        nativeSetExitWithoutCleanup(true);

        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);
        VMRuntime.getRuntime().setDisabledCompatChanges(disabledCompatChanges);

        final Arguments args = new Arguments(argv);

        // The end of of the RuntimeInit event (see #zygoteInit).
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);

        // Remaining arguments are passed to the start class's static main
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }
```

&emsp; 该方法做的事情就比较明显了，通过反射找到传入的com.android.server.SystemServer类的main方法开始执行。

### 10. SystemServer.java#main

&emsp; 接下来我们来进入SystemServer的流程。

[SystemServer.java#main](https://cs.android.com/android/platform/superproject/+/master:frameworks/base/services/java/com/android/server/SystemServer.java;bpv=1;bpt=1;l=646?q=SystemServer.java&ss=android%2Fplatform%2Fsuperproject&gsn=main&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.server.SystemServer%23f45d603c574e54a86ba805d327c4cb0a6ddc6cfcfb92ab6d76cde7e6fae7f3b4)

```java
    public static void main String[] args) {
        new SystemServer().run();
    }

    private void run() {
        TimingsTraceAndSlog t = new TimingsTraceAndSlog();
        try {
            t.traceBegin("InitBeforeStartServices");

            // Record the process start information in sys props.
            SystemProperties.set(SYSPROP_START_COUNT, String.valueOf(mStartCount));
            SystemProperties.set(SYSPROP_START_ELAPSED, String.valueOf(mRuntimeStartElapsedTime));
            SystemProperties.set(SYSPROP_START_UPTIME, String.valueOf(mRuntimeStartUptime));

            EventLog.writeEvent(EventLogTags.SYSTEM_SERVER_START,
                    mStartCount, mRuntimeStartUptime, mRuntimeStartElapsedTime);

            //
            // Default the timezone property to GMT if not set.
            //
            String timezoneProperty = SystemProperties.get("persist.sys.timezone");
            if (!isValidTimeZoneId(timezoneProperty)) {
                Slog.w(TAG, "persist.sys.timezone is not valid (" + timezoneProperty
                        + "); setting to GMT.");
                SystemProperties.set("persist.sys.timezone", "GMT");
            }

            // If the system has "persist.sys.language" and friends set, replace them with
            // "persist.sys.locale". Note that the default locale at this point is calculated
            // using the "-Duser.locale" command line flag. That flag is usually populated by
            // AndroidRuntime using the same set of system properties, but only the system_server
            // and system apps are allowed to set them.
            //
            // NOTE: Most changes made here will need an equivalent change to
            // core/jni/AndroidRuntime.cpp
            if (!SystemProperties.get("persist.sys.language").isEmpty()) {
                final String languageTag = Locale.getDefault().toLanguageTag();

                SystemProperties.set("persist.sys.locale", languageTag);
                SystemProperties.set("persist.sys.language", "");
                SystemProperties.set("persist.sys.country", "");
                SystemProperties.set("persist.sys.localevar", "");
            }

            // The system server should never make non-oneway calls
            Binder.setWarnOnBlocking(true);
            // The system server should always load safe labels
            PackageItemInfo.forceSafeLabels();

            // Default to FULL within the system server.
            SQLiteGlobal.sDefaultSyncMode = SQLiteGlobal.SYNC_MODE_FULL;

            // Deactivate SQLiteCompatibilityWalFlags until settings provider is initialized
            SQLiteCompatibilityWalFlags.init(null);

            // Here we go!
            Slog.i(TAG, "Entered the Android system server!");
            final long uptimeMillis = SystemClock.elapsedRealtime();
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, uptimeMillis);
            if (!mRuntimeRestart) {
                FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
                        FrameworkStatsLog
                                .BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__SYSTEM_SERVER_INIT_START,
                        uptimeMillis);
            }

            // In case the runtime switched since last boot (such as when
            // the old runtime was removed in an OTA), set the system
            // property so that it is in sync. We can't do this in
            // libnativehelper's JniInvocation::Init code where we already
            // had to fallback to a different runtime because it is
            // running as root and we need to be the system user to set
            // the property. http://b/11463182
            SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

            // Mmmmmm... more memory!
            VMRuntime.getRuntime().clearGrowthLimit();

            // Some devices rely on runtime fingerprint generation, so make sure
            // we've defined it before booting further.
            Build.ensureFingerprintProperty();

            // Within the system server, it is an error to access Environment paths without
            // explicitly specifying a user.
            Environment.setUserRequired(true);

            // Within the system server, any incoming Bundles should be defused
            // to avoid throwing BadParcelableException.
            BaseBundle.setShouldDefuse(true);

            // Within the system server, when parceling exceptions, include the stack trace
            Parcel.setStackTraceParceling(true);

            // Ensure binder calls into the system always run at foreground priority.
            BinderInternal.disableBackgroundScheduling(true);

            // Increase the number of binder threads in system_server
            BinderInternal.setMaxThreads(sMaxBinderThreads);

            // Prepare the main looper thread (this thread).
            android.os.Process.setThreadPriority(
                    android.os.Process.THREAD_PRIORITY_FOREGROUND);
            android.os.Process.setCanSelfBackground(false);
            Looper.prepareMainLooper();
            Looper.getMainLooper().setSlowLogThresholdMs(
                    SLOW_DISPATCH_THRESHOLD_MS, SLOW_DELIVERY_THRESHOLD_MS);

            SystemServiceRegistry.sEnableServiceNotFoundWtf = true;

            // Initialize native services.
            System.loadLibrary("android_servers");

            // Allow heap / perf profiling.
            initZygoteChildHeapProfiling();

            // Debug builds - spawn a thread to monitor for fd leaks.
            if (Build.IS_DEBUGGABLE) {
                spawnFdLeakCheckThread();
            }

            // Check whether we failed to shut down last time we tried.
            // This call may not return.
            performPendingShutdown();

            // Initialize the system context.
            createSystemContext();

            // Call per-process mainline module initialization.
            ActivityThread.initializeMainlineModules();

            // Sets the dumper service
            ServiceManager.addService("system_server_dumper", mDumper);
            mDumper.addDumpable(this);

            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            mSystemServiceManager.setStartInfo(mRuntimeRestart,
                    mRuntimeStartElapsedTime, mRuntimeStartUptime);
            mDumper.addDumpable(mSystemServiceManager);

            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
            // Prepare the thread pool for init tasks that can be parallelized
            SystemServerInitThreadPool tp = SystemServerInitThreadPool.start();
            mDumper.addDumpable(tp);

            // Load preinstalled system fonts for system server, so that WindowManagerService, etc
            // can start using Typeface. Note that fonts are required not only for text rendering,
            // but also for some text operations (e.g. TextUtils.makeSafeForPresentation()).
            if (Typeface.ENABLE_LAZY_TYPEFACE_INITIALIZATION) {
                Typeface.loadPreinstalledSystemFontMap();
            }

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
            startBootstrapServices(t);
            startCoreServices(t);
            startOtherServices(t);
            startApexServices(t);
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            t.traceEnd(); // StartServices
        }

        StrictMode.initVmDefaults(null);

        if (!mRuntimeRestart && !isFirstBootOrUpgrade()) {
            final long uptimeMillis = SystemClock.elapsedRealtime();
            FrameworkStatsLog.write(FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME_REPORTED,
                    FrameworkStatsLog.BOOT_TIME_EVENT_ELAPSED_TIME__EVENT__SYSTEM_SERVER_READY,
                    uptimeMillis);
            final long maxUptimeMillis = 60 * 1000;
            if (uptimeMillis > maxUptimeMillis) {
                Slog.wtf(SYSTEM_SERVER_TIMING_TAG,
                        "SystemServer init took too long. uptimeMillis=" + uptimeMillis);
            }
        }

        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```

&emsp; 此时开始SystemServer进程相关的逻辑。

* Binder.setWarnOnBlocking(true); 将该进程的binder调用设置为oneway模式。Binder调用分为oneway和非oneway，oneway就是不必等待server执行完毕就返回，SystemServer是重要的系统进程，如果进行非oneway的binder调用，如果对端的应用进程卡住则会影响到整个系统。

* BinderInternal.setMaxThreads(sMaxBinderThreads); 设置SystemServer进程最大Binder线程数，此处该值为31(正常应用进程为15)。

* System.loadLibrary("android_servers"); load所需的libandroid\_servers.so共享库。(共享库的加载过程后续文章可能会描述到)

* createSystemContext; 创建SystemServer的Context

* 启动系统相关服务；
  
  - startBootStrapServices; 启动一些关键的系统服务，如Installer(用于安装apk)、AMS(管理四大组件及进程相关)、PKMS(管理APK相关)等。
  
  - startCoreServices; 启动一些重要的系统服务，如BatteryService、UsageStatsService等。
  
  - startOtherServices；启动一些其他的系统服务，这一块启动的比较多。
  
  - startApexServices; 启动mainline相关的系统服务，笔者对于apex的理解是，为了解决Android的碎片化问题，Google开始将一些模块闭源(也就是不再交给厂商维护)，然后自己通过OTA升级的方式将Apex包下发来完成对于模块的维护更新。(这点可能存在偏差，后续如有新的认识再斧正)

### 11. ActivityManagerService的启动

&emsp; 启动的系统服务较多，我们就不一一介绍，选择ActivityManagerService来看下系统服务是如何启动的。

[SystemServer.java#startBootstrapServices](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/services/java/com/android/server/SystemServer.java;bpv=1;bpt=1;l=1052?gsn=startBootstrapServices&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.server.SystemServer%2366f4c52ed4fecc0d4cd0e7d33e4b7eef7944156787c785dbc7a25f1eeac78c0c)

```java
    private void startBootstrapServices(@NonNull TimingsTraceAndSlog t) {
        ... ...
        mActivityManagerService = ActivityManagerService.Lifecycle.startService(
                mSystemServiceManager, atm);
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
        ... ...
        mActivityManagerService.setSystemProcess();
        ... ...
    }
```

[SystemServiceManager.java#startService](https://cs.android.com/android/platform/superproject/+/android-13.0.0_r1:frameworks/base/services/core/java/com/android/server/SystemServiceManager.java;bpv=1;bpt=1;l=143?gsn=startService&gs=kythe%3A%2F%2Fandroid.googlesource.com%2Fplatform%2Fsuperproject%3Flang%3Djava%3Fpath%3Dcom.android.server.SystemServiceManager%23947eb9ef1b3ccdf13762a9c86b2ba4f4256fbdf9980c46f5255f309ff433b8eb)

```java
    public SystemService startService(Stringcl    public <T extends SystemService> T startService(Class<T> serviceClass) {
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
            } catch (InstantiationException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service could not be instantiated", ex);
            } catch (IllegalAccessException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
            } catch (NoSuchMethodException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service must have a public constructor with a Context argument", ex);
            } catch (InvocationTargetException ex) {
                throw new RuntimeException("Failed to create service " + name
                        + ": service constructor threw an exception", ex);
            }

            startService(service);
            return service;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
    }
assName) {
        final Class<SystemService> serviceClass = loadClassFromLoader(className,
                this.getClass().getClassLoader());
        return startService(serviceClass);
    }
```

&emsp; 从上述代码可以看出执行了ActivityManagerService的构造方法。
