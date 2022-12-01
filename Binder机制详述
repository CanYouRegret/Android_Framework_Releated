# Binder机制详述

## 前言

&emsp; 笔者最近在面试时被问到“说说你对Binder的了解”，这个问题其实问得比较宽泛，Binder从java层的aidl到native层的transactNative再到kernel的binder驱动，涉及到的层面较多，问题回答的深度自然也会影响到面试的结果。

&emsp; 笔者曾经研究过Binder流程的源码，但是一时不知道该从哪里回答比较合适，因此宽泛的将自己知道的描述了下，觉得有些语无伦次且不尽人意，因此决定根据源码总结下，一是加深自己对于Binder机制的理解，再者下次面试被问到相关问题时也能从容作答。

## 1. 从AIDL谈起

&emsp; Binder是Android中进程间通信的一种机制。Android基于Linux，进程间是相互隔离的，通常来说，进程间的通信使用Handler，而进程间的通信使用Binder、Socket、共享内存等方式。
&emsp; 对于上层app开发者而言，无需关注进程间通信的细节，Android系统提供了aidl，开发者只需按照相应的规则进行设计，就可以使用Binder进行通信。那么我们就先从我们可以看到的aidl相关部分来进行分析，笔者对于ActivityManagerService比较熟悉，因此就拿其最常见的startActivity方法为例来剖析流程。

&emsp; Binder调用分为Client和Server两端，Client发起请求，Server接到请求后进行实际的操作然后返回结果。对于三方应用开发者而言，如果想启动一个Activity，直接调用公共接口startActivity就可以，而该方法实质上是通过Binder调用至SystemServer从而进程activity的启动。

frameworks/base/core/java/android/app/Instrumentation.java

```java
            int result = ActivityTaskManager.getService().startActivity(whoThread,
                    who.getOpPackageName(), who.getAttributionTag(), intent,
                    intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                    target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
```

frameworks/base/core/java/android/app/ActivityTaskManager.java

```java
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

&emsp; 从上述代码我们可以看到，实质上是通过IActivityTaskManager来进行调用的，那么IActivityTaskManager是先通过ServiceManager#getService并传入相应service name拿到一个IBinder对象，然后将这个IBinder对象作为参数传入IActivityTaskManager$Stub#asInterface方法得到的。



&emsp; 此时提出第一个疑问，IActivityTaskManager是什么呢？我们在源码中搜索，可以找到frameworks/base/core/java/android/app/IActivityTaskManager.aidl这个文件。

&emsp; aidl被称为Android接口定义语言(Android Interface Definition Language)，在跨进程访问时会被用到，读者可以理解为是用来简化Binder通信相关类书写的一种方式。Binder通信相关类的实现是有模板的，为了简化开发者的开发流程，Android会在编译时自动根据aidl文件来生成相应的java文件。也就是说，使用Binder通信时，不用aidl也是可以的(读者可以参考Android系统早期时ActivityManager的Binder实现)。至于根据aidl文件生成相应java文件的过程细节，这里我们暂时不去关注。

&emsp; 那么，就IActivityTaskManager.aidl文件而言，其生成的java文件位于何处呢？ 笔者的源码环境下是在out/soong/.intermediates/frameworks/base/framework-minus-apex/android_common/dex/classes.dex里找到的，我们通过反编译这个dex文件就能看到IActivityTaskManager.java的具体内容。(笔者记得较低Android版本时会在out相关目录下直接生成java文件的，但是不重要，结果是一样的)

```java
public interface IActivityTaskManager extends IInterface {
    public static final String DESCRIPTOR = "android.app.IActivityTaskManager";
    ... ...
    int startActivity(IApplicationThread iApplicationThread, String str, String str2, Intent intent, String str3, IBinder iBinder, String str4, int i, int i2, ProfilerInfo profilerInfo, Bundle bundle) throws RemoteException;
    ... ...
    public static abstract class Stub extends Binder implements IActivityTaskManager {
        ... ...
        static final int TRANSACTION_startActivity = 1;
        ... ...
        public Stub() {
            attachInterface(this, IActivityTaskManager.DESCRIPTOR);
        }

        public static IActivityTaskManager asInterface(IBinder obj) {
            if (obj == null) {
                return null;
            }
            IInterface iin = obj.queryLocalInterface(IActivityTaskManager.DESCRIPTOR);
            if (iin != null && (iin instanceof IActivityTaskManager)) {
                return (IActivityTaskManager) iin;
            }
            return new Proxy(obj);
        }

        @Override // android.os.IInterface
        public IBinder asBinder() {
            return this;
        }

        public static String getDefaultTransactionName(int transactionCode) {
            switch (transactionCode) {
                case 1:
                    return "startActivity";
                    ... ...
        public String getTransactionName(int transactionCode) {
            return getDefaultTransactionName(transactionCode);
        }

        @Override // android.os.Binder
        public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
            if (code >= 1 && code <= 16777215) {
                data.enforceInterface(IActivityTaskManager.DESCRIPTOR);
            }
            switch (code) {
                case 1598968902:
                    reply.writeString(IActivityTaskManager.DESCRIPTOR);
                    return true;
                default:
                    switch (code) {
                        case 1:
                            IApplicationThread _arg0 = IApplicationThread.Stub.asInterface(data.readStrongBinder());
                            String _arg1 = data.readString();
                            String _arg2 = data.readString();
                            Intent _arg3 = (Intent) data.readTypedObject(Intent.CREATOR);
                            String _arg4 = data.readString();
                            IBinder _arg5 = data.readStrongBinder();
                            String _arg6 = data.readString();
                            int _arg7 = data.readInt();
                            int _arg8 = data.readInt();
                            ProfilerInfo _arg9 = (ProfilerInfo) data.readTypedObject(ProfilerInfo.CREATOR);
                            Bundle _arg10 = (Bundle) data.readTypedObject(Bundle.CREATOR);
                            data.enforceNoDataAvail();
                            int _result = startActivity(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5, _arg6, _arg7, _arg8, _arg9, _arg10);
                            reply.writeNoException();
                            reply.writeInt(_result);
                            return true;
                            ... ...
        }
    
        public static class Proxy implements IActivityTaskManager {
            private IBinder mRemote;

            Proxy(IBinder remote) {
                this.mRemote = remote;
            }

            @Override // android.os.IInterface
            public IBinder asBinder() {
                return this.mRemote;
            }

            public String getInterfaceDescriptor() {
                return IActivityTaskManager.DESCRIPTOR;
            }

            @Override // android.app.IActivityTaskManager
            public int startActivity(IApplicationThread caller, String callingPackage, String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo, String resultWho, int requestCode, int flags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
                Parcel _data = Parcel.obtain();
                Parcel _reply = Parcel.obtain();
                try {
                    _data.writeInterfaceToken(IActivityTaskManager.DESCRIPTOR);
                    _data.writeStrongInterface(caller);
                } catch (Throwable th) {
                    th = th;
                }
                try {
                    _data.writeString(callingPackage);
                } catch (Throwable th2) {
                    th = th2;
                    _reply.recycle();
                    _data.recycle();
                    throw th;
                }
                try {
                    _data.writeString(callingFeatureId);
                    try {
                        _data.writeTypedObject(intent, 0);
                    } catch (Throwable th3) {
                        th = th3;
                        _reply.recycle();
                        _data.recycle();
                        throw th;
                    }
                    try {
                        _data.writeString(resolvedType);
                    } catch (Throwable th4) {
                        th = th4;
                        _reply.recycle();
                        _data.recycle();
                        throw th;
                    }
                    try {
                        _data.writeStrongBinder(resultTo);
                    } catch (Throwable th5) {
                        th = th5;
                        _reply.recycle();
                        _data.recycle();
                        throw th;
                    }
                } catch (Throwable th6) {
                    th = th6;
                    _reply.recycle();
                    _data.recycle();
                    throw th;
                }
                try {
                    _data.writeString(resultWho);
                    try {
                        _data.writeInt(requestCode);
                    } catch (Throwable th7) {
                        th = th7;
                        _reply.recycle();
                        _data.recycle();
                        throw th;
                    }
                    try {
                        _data.writeInt(flags);
                    } catch (Throwable th8) {
                        th = th8;
                        _reply.recycle();
                        _data.recycle();
                        throw th;
                    }
                    try {
                        _data.writeTypedObject(profilerInfo, 0);
                        try {
                            _data.writeTypedObject(options, 0);
                            try {
                                this.mRemote.transact(1, _data, _reply, 0);
                                _reply.readException();
                                int _result = _reply.readInt();
                                _reply.recycle();
                                _data.recycle();
                                return _result;
                            } catch (Throwable th9) {
                                th = th9;
                                _reply.recycle();
                                _data.recycle();
                                throw th;
                            }
                        } catch (Throwable th10) {
                            th = th10;
                        }
                    } catch (Throwable th11) {
                        th = th11;
                        _reply.recycle();
                        _data.recycle();
                        throw th;
                    }
                } catch (Throwable th12) {
                    th = th12;
                    _reply.recycle();
                    _data.recycle();
                    throw th;
                }
            }
            ... ...
        }
    }
}
```

&emsp; 限于篇幅，笔者只是将相关重要的类及方法贴了出来。从上面的代码我们可以看到:

* IActivityTaskManager extends IInterface
* IActivityTaskManager$Stub extends Binder implements IActivityTaskManager
* IActivityTaskManager$Stub$Proxy implements IActivityTaskManager

&emsp; IActivityTaskManager是一个Interface，其方法实质上是被IActivityTaskManager$Stub$Proxy实现的。这里涉及到java层的几个类概念：

* IInterface; 其代表远程的Server端具备什么能力，具体的就指的是aidl里的各个接口。其内的asBinder方法可以将IInterface转换成与其相关的IBinder对象。
* Binder; Binder类代表的是Binder实体，因其实现了IBinder而具备了跨进程传输的能力。
* IBinder; IBinder是一个Interface，其代表了一种跨进程传输的能力，只要一个类实现了IBinder，就可以将这个类的对象跨进程进行传输，在跨进程传输时，驱动会识别IBinder类型的数据，从而完成不同进程Binder本地对象和Binder代理对象的转换。
* BinderProxy; 表示远程进程Binder对象的本地代理，该类也实现了IBinder，因此具备跨进程传输的能力。其实我们从ServiceManager.getService拿到的IBinder就是BinderProxy。读者可自行在系统中打印日志进行确认
* Stub; Stub是IActivityTaskManager的一个静态内部抽象类，其继承了Binder，说明其是一个Binder本地对象；其实现了IActivityTaskManager，说明其具备远程Server承诺给Client的能力，从ActivityTaskManagerService.java的代码来看，Server端的ActivityTaskManagerService继承了IActivityTaskManager.Stub。Stub类的asInterface方法中，会先调用queryLocalInterface方法来查看是否存在本地Binder对象，如果存在的话则属于同进程调用，也就不需要通过ServiceManager.getService拿到的BinderProxy；相反如果不存在本地Binder对象，说明属于跨进程调用，则会将BinderProxy对象传入构造Proxy。
* Proxy; Proxy只是一个代理类，其方法的真正调用是将相关参数序列化，并通过传入transact code调用BinderProxy(mRemote)的transact方法。

&emsp; 通过上面的分析，java层的binder调用流程已经清晰了。当跨进程通信时，通过ServiceManager拿到远程实体Binder的本地代理BinderProxy，然后通过封装参数以及Transact code调用BinderProxy的transact方法；进而远端Server受到请求后调用Stub类(也就是Binder实体，此例中指ActivityTaskManagerService)的onTransact方法进行实际的操作然后返回。 java层的Binder调用流程还是清晰易懂的，因为具体的传输转换细节被native和dirver封装了，接下来我们继续深入传输转换的细节。

## 2. transact调用到了哪里？

&emsp; 由上一节我们知道，跨进程Binder通信的请求，实际上是调用了本地代理对象BinderProxy的transact方法。从BinderProxy.java的源码我们可以看到，实质上是调用了transactNative的jni方法。

frameworks/base/core/java/android/os/BinderProxy.java

```java
    public native boolean transactNative(int code, Parcel data, Parcel reply,
            int flags) throws RemoteException;
```

frameworks/base/core/jni/android_util_Binder.cpp

```cpp
static jboolean android_os_BinderProxy_transact(JNIEnv* env, jobject obj,
        jint code, jobject dataObj, jobject replyObj, jint flags) // throws RemoteException
{
    if (dataObj == NULL) {
        jniThrowNullPointerException(env, NULL);
        return JNI_FALSE;
    }

    Parcel* data = parcelForJavaObject(env, dataObj);
    if (data == NULL) {
        return JNI_FALSE;
    }
    Parcel* reply = parcelForJavaObject(env, replyObj);
    if (reply == NULL && replyObj != NULL) {
        return JNI_FALSE;
    }

    IBinder* target = getBPNativeData(env, obj)->mObject.get();
    if (target == NULL) {
        jniThrowException(env, "java/lang/IllegalStateException", "Binder has been finalized!");
        return JNI_FALSE;
    }

    ALOGV("Java code calling transact on %p in Java object %p with code %" PRId32 "\n",
            target, obj, code);


    bool time_binder_calls;
    int64_t start_millis;
    if (kEnableBinderSample) {
        // Only log the binder call duration for things on the Java-level main thread.
        // But if we don't
        time_binder_calls = should_time_binder_calls();

        if (time_binder_calls) {
            start_millis = uptimeMillis();
        }
    }

    //printf("Transact from Java code to %p sending: ", target); data->print();
    status_t err = target->transact(code, *data, reply, flags);
    //if (reply) printf("Transact from Java code to %p received: ", target); reply->print();

    if (kEnableBinderSample) {
        if (time_binder_calls) {
            conditionally_log_binder_call(start_millis, target, code);
        }
    }

    if (err == NO_ERROR) {
        return JNI_TRUE;
    } else if (err == UNKNOWN_TRANSACTION) {
        return JNI_FALSE;
    }

    signalExceptionForError(env, obj, err, true /*canThrowRemoteException*/, data->dataSize());
    return JNI_FALSE;
}
```

&emsp; 我们可以看到，native层通过getBPNativeData将BinderProxy的java对象转化成native层对应的BpBinder对象。这里需要说明一下，Binder相关java类对象在native有相对应的native类对象，如BinderProxy对应BpBinder，Binder对应BBinder(同样的BBinder和BpBinder都实现native层的IBinder接口)。那么java层的类对象是如何与native对应连接起来的呢？通过源码我们可以找到答案。

frameworks/base/core/java/android/os/BinderProxy.java

```java
    private BinderProxy(long nativeData) {
        mNativeData = nativeData;
    }
```

frameworks/base/core/jni/android_util_Binder.cpp

```cpp
BinderProxyNativeData* getBPNativeData(JNIEnv* env, jobject obj) {
    return (BinderProxyNativeData *) env->GetLongField(obj, gBinderProxyOffsets.mNativeData);
}

```

&emsp; 以BinderProxy为例，在构造BinderProxy时传入的nativeData被赋值给mNativeData变量；而native层将BinderProxy的java object转换为native object时，是通过反射java类BinderProxy的mNativeData变量。由此可见对于本地Binder代理，java和native层是通过mNativeData对应起来的。同样的，对于Binder实体，java和native层是通过mObject变量对应起来的。



&emsp; 然后我们继续往下追踪transact的后续流程，也就是会执行BpBinder的transact方法。

frameworks/native/libs/binder/BpBinder.cpp

```cpp
status_t BpBinder::transact(
    uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags)
{
    // Once a binder has died, it will never come back to life.
    if (mAlive) {
        bool privateVendor = flags & FLAG_PRIVATE_VENDOR;
        // don't send userspace flags to the kernel
        flags = flags & ~FLAG_PRIVATE_VENDOR;

        // user transactions require a given stability level
        if (code >= FIRST_CALL_TRANSACTION && code <= LAST_CALL_TRANSACTION) {
            using android::internal::Stability;

            int16_t stability = Stability::getRepr(this);
            Stability::Level required = privateVendor ? Stability::VENDOR
                : Stability::getLocalLevel();

            if (CC_UNLIKELY(!Stability::check(stability, required))) {
                ALOGE("Cannot do a user transaction on a %s binder (%s) in a %s context.",
                      Stability::levelString(stability).c_str(),
                      String8(getInterfaceDescriptor()).c_str(),
                      Stability::levelString(required).c_str());
                return BAD_TYPE;
            }
        }

        status_t status;
        if (CC_UNLIKELY(isRpcBinder())) {
            status = rpcSession()->transact(sp<IBinder>::fromExisting(this), code, data, reply,
                                            flags);
        } else {
            status = IPCThreadState::self()->transact(binderHandle(), code, data, reply, flags);
        }
        if (data.dataSize() > LOG_TRANSACTIONS_OVER_SIZE) {
            Mutex::Autolock _l(mLock);
            ALOGW("Large outgoing transaction of %zu bytes, interface descriptor %s, code %d",
                  data.dataSize(),
                  mDescriptorCache.size() ? String8(mDescriptorCache).c_str()
                                          : "<uncached descriptor>",
                  code);
        }

        if (status == DEAD_OBJECT) mAlive = 0;

        return status;
    }

    return DEAD_OBJECT;
}
```

&emsp; 从BpBinder的transact内容我们可以看到，有RPC和IPC的分支差异，IPC为进程间通信，RPC为远程通信。我们此刻分析的场景为进程间通信，RPC相关逻辑我们暂且不管(笔者认为RPC用于设备间通信)。忽略代码的相关细节，我们只关注主干部分，最后实质上调用到了IPCThreadState::self()->transact，此时传入的参数为binderHandle、code、data、reply、flags。后四个参数是我们从java层传递下来的，那么binderHandle又是什么东西？ 这里我们可以理解为每个Binder的句柄，用来标识一个Binder对象，具体是在构造BpBinder/BBinder的时候传入的，后续我们会分析这个句柄的具体用途。

&emsp; 接下来我们继续IPCThreadState::transact方法的分析。

frameworks/native/libs/binder/IPCThreadState.cpp

```cpp
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    LOG_ALWAYS_FATAL_IF(data.isForRpc(), "Parcel constructed for RPC, but being used with binder.");

    status_t err;

    flags |= TF_ACCEPT_FDS;

    IF_LOG_TRANSACTIONS() {
        TextOutput::Bundle _b(alog);
        alog << "BC_TRANSACTION thr " << (void*)pthread_self() << " / hand "
            << handle << " / code " << TypeCode(code) << ": "
            << indent << data << dedent << endl;
    }

    LOG_ONEWAY(">>>> SEND from pid %d uid %d %s", getpid(), getuid(),
        (flags & TF_ONE_WAY) == 0 ? "READ REPLY" : "ONE WAY");
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);

    if (err != NO_ERROR) {
        if (reply) reply->setError(err);
        return (mLastError = err);
    }

    if ((flags & TF_ONE_WAY) == 0) {
        if (UNLIKELY(mCallRestriction != ProcessState::CallRestriction::NONE)) {
            if (mCallRestriction == ProcessState::CallRestriction::ERROR_IF_NOT_ONEWAY) {
                ALOGE("Process making non-oneway call (code: %u) but is restricted.", code);
                CallStack::logStack("non-oneway call", CallStack::getCurrent(10).get(),
                    ANDROID_LOG_ERROR);
            } else /* FATAL_IF_NOT_ONEWAY */ {
                LOG_ALWAYS_FATAL("Process may not make non-oneway calls (code: %u).", code);
            }
        }

        #if 0
        if (code == 4) { // relayout
            ALOGI(">>>>>> CALLING transaction 4");
        } else {
            ALOGI(">>>>>> CALLING transaction %d", code);
        }
        #endif
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
        #if 0
        if (code == 4) { // relayout
            ALOGI("<<<<<< RETURNING transaction 4");
        } else {
            ALOGI("<<<<<< RETURNING transaction %d", code);
        }
        #endif

        IF_LOG_TRANSACTIONS() {
            TextOutput::Bundle _b(alog);
            alog << "BR_REPLY thr " << (void*)pthread_self() << " / hand "
                << handle << ": ";
            if (reply) alog << indent << *reply << dedent << endl;
            else alog << "(none requested)" << endl;
        }
    } else {
        err = waitForResponse(nullptr, nullptr);
    }

    return err;
}
```

&emsp; 通过泛读上述代码，我们大概能看出几个可能重要的点，如writeTransactionData、ONE_WAY、waitForResponse，下面我们逐一来进行介绍和分析。

### 1). writeTransactionData

frameworks/native/libs/binder/IPCThreadState.cpp

```cpp
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    binder_transaction_data tr;

    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));

    return NO_ERROR;
}
```



* 我们可以看到writeTransactionData时传入了BC_TRANSACTION这个参数，BC就是Binder Command Protocol，BC_xxxxx相关的变量是由用户态发往Binder driver的命令，也就是说该参数是由kernel去进行相应处理的。 对应的相应BR_XXXXX(BR是Binder Return Protocol)是由kernel发往用户态的，由用户态进行相应的处理。
* 阅读writeTransactionData的代码，其实就是将参数封装到binder_transaction_data的结构体中，然后再将其写入到mOut里。mIn和mOut是Parcel类型的对象。

### 2). ONE_WAY

&emsp; oneway在aidl中用来声明binder调用的类型。binder调用分为oneway和非oneway，非oneway的需要等待调用结果的返回，相应的如果对端server的执行卡住，那么就会一直等待；而oneway调用则无需等待结果，不会阻塞。 可能有的读者会遇到，明明是oneway的binder调用，但是一直卡住没有返回，这个时候就需要检查下server的binder线程是不是因为某些原因已经被占满了，无法接收新的binder调用。

### 3). waitForResponse

frameworks/native/libs/binder/IPCThreadState.cpp

```cpp
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
    uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        err = mIn.errorCheck();
        if (err < NO_ERROR) break;
        if (mIn.dataAvail() == 0) continue;

        cmd = (uint32_t)mIn.readInt32();

        IF_LOG_COMMANDS() {
            alog << "Processing waitForResponse Command: "
                << getReturnString(cmd) << endl;
        }

        switch (cmd) {
        case BR_ONEWAY_SPAM_SUSPECT:
            ALOGE("Process seems to be sending too many oneway calls.");
            CallStack::logStack("oneway spamming", CallStack::getCurrent().get(),
                    ANDROID_LOG_ERROR);
            [[fallthrough]];
        case BR_TRANSACTION_COMPLETE:
            if (!reply && !acquireResult) goto finish;
            break;

        case BR_DEAD_REPLY:
            err = DEAD_OBJECT;
            goto finish;

        case BR_FAILED_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_FROZEN_REPLY:
            err = FAILED_TRANSACTION;
            goto finish;

        case BR_ACQUIRE_RESULT:
            {
                ALOG_ASSERT(acquireResult != NULL, "Unexpected brACQUIRE_RESULT");
                const int32_t result = mIn.readInt32();
                if (!acquireResult) continue;
                *acquireResult = result ? NO_ERROR : INVALID_OPERATION;
            }
            goto finish;

        case BR_REPLY:
            {
                binder_transaction_data tr;
                err = mIn.read(&tr, sizeof(tr));
                ALOG_ASSERT(err == NO_ERROR, "Not enough command data for brREPLY");
                if (err != NO_ERROR) goto finish;

                if (reply) {
                    if ((tr.flags & TF_STATUS_CODE) == 0) {
                        reply->ipcSetDataReference(
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t),
                            freeBuffer);
                    } else {
                        err = *reinterpret_cast<const status_t*>(tr.data.ptr.buffer);
                        freeBuffer(nullptr,
                            reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                            tr.data_size,
                            reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                            tr.offsets_size/sizeof(binder_size_t));
                    }
                } else {
                    freeBuffer(nullptr,
                        reinterpret_cast<const uint8_t*>(tr.data.ptr.buffer),
                        tr.data_size,
                        reinterpret_cast<const binder_size_t*>(tr.data.ptr.offsets),
                        tr.offsets_size/sizeof(binder_size_t));
                    continue;
                }
            }
            goto finish;

        default:
            err = executeCommand(cmd);
            if (err != NO_ERROR) goto finish;
            break;
        }
    }

finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}
```

&emsp; 我们从waitForResponse的代码可以看到主要做了两件事，talkWithDirver、根据mIn中读取的命令做相应的处理，下面我们分别来看下。

* talkWithDriver

```cpp
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    if (mProcess->mDriverFD < 0) {
        return -EBADF;
    }

    binder_write_read bwr;

    // Is the read buffer empty?
    const bool needRead = mIn.dataPosition() >= mIn.dataSize();

    // We don't want to write anything if we are still reading
    // from data left in the input buffer and the caller
    // has requested to read the next data.
    const size_t outAvail = (!doReceive || needRead) ? mOut.dataSize() : 0;

    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    // This is what we'll read.
    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }

    IF_LOG_COMMANDS() {
        TextOutput::Bundle _b(alog);
        if (outAvail != 0) {
            alog << "Sending commands to driver: " << indent;
            const void* cmds = (const void*)bwr.write_buffer;
            const void* end = ((const uint8_t*)cmds)+bwr.write_size;
            alog << HexDump(cmds, bwr.write_size) << endl;
            while (cmds < end) cmds = printCommand(alog, cmds);
            alog << dedent;
        }
        alog << "Size of receive buffer: " << bwr.read_size
            << ", needRead: " << needRead << ", doReceive: " << doReceive << endl;
    }

    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        IF_LOG_COMMANDS() {
            alog << "About to read/write, write size = " << mOut.dataSize() << endl;
        }
#if defined(__ANDROID__)
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD < 0) {
            err = -EBADF;
        }
        IF_LOG_COMMANDS() {
            alog << "Finished read/write, write size = " << mOut.dataSize() << endl;
        }
    } while (err == -EINTR);

    IF_LOG_COMMANDS() {
        alog << "Our err: " << (void*)(intptr_t)err << ", write consumed: "
            << bwr.write_consumed << " (of " << mOut.dataSize()
                        << "), read consumed: " << bwr.read_consumed << endl;
    }

    if (err >= NO_ERROR) {
        if (bwr.write_consumed > 0) {
            if (bwr.write_consumed < mOut.dataSize())
                LOG_ALWAYS_FATAL("Driver did not consume write buffer. "
                                 "err: %s consumed: %zu of %zu",
                                 statusToString(err).c_str(),
                                 (size_t)bwr.write_consumed,
                                 mOut.dataSize());
            else {
                mOut.setDataSize(0);
                processPostWriteDerefs();
            }
        }
        if (bwr.read_consumed > 0) {
            mIn.setDataSize(bwr.read_consumed);
            mIn.setDataPosition(0);
        }
        IF_LOG_COMMANDS() {
            TextOutput::Bundle _b(alog);
            alog << "Remaining data size: " << mOut.dataSize() << endl;
            alog << "Received commands from driver: " << indent;
            const void* cmds = mIn.data();
            const void* end = mIn.data() + mIn.dataSize();
            alog << HexDump(cmds, mIn.dataSize()) << endl;
            while (cmds < end) cmds = printReturnCommand(alog, cmds);
            alog << dedent;
        }
        return NO_ERROR;
    }

    return err;
}
```

&emsp; talkWithDriver顾名思义就是去跟binder驱动交互，这里我们注意binder_write_read这个结构体，它是用户态与kernel数据交互的载体，可以看到，会将mOut里的内容放到binder_write_read的write_buffer中，而将mIn相关的地址放入read_buffer供kernel填充。到这里我们已经明确了，mOut用来存储用户态发往kernel的参数，而mIn用来存储kernel返回给用户态的结果。

&emsp; 而具体的交互是通过系统调用ioctl实现的，指定相应的Binder fd以及需要执行的操作(BINDER_WRITE_READ)以及携带的数据binder_write_read。此时由用户态进入到内核态



* 根据mIn中读取的命令做相应的处理。我们可以看到此时处理的命令为BR_XXXX，也就是说kernel已经将返回结果填充至mIn中，用户态此时根据不同的命令作处理，这点也跟我们上面分析的一致。(如BR_REPLY其实就是从mIn中读取数据放至binder_transaction_data结构体中，而后再将结果填充至reply)



&emsp; 其实分析到这里，BinderProxy代理在用户态发起binder调用的流程已经清晰了，接下来是跟kernel binder驱动的交互。





