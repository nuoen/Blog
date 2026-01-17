android so load的前世今生

# 1. 入口程序 和 linker64
## 入口：app_process64(Zygote/Runtime)
### app_process64是什么
* 一个ELF可执行文件
* 路径通常是`/system/bin/app_process64`
* 作用：Android java世界的统一入口程序
* 它可以以**不同”模式“**运行：
  * Zygote模式
  * 普通App Runtime模式
  * system_server模式（历史上）
### Zygote是什么？
* 一个长期存在的进程，由init.rc启动
aosp路径：`system/core/rootdir/init.zygote64.rc`
命令：`service zygote /system/bin/app_process64 -Xzygote /system/bin --zygote --start-system-server --socket-name=zygote
    class main ...`
* 这个进程
  * 是用app_process64启动的
  * 启动参数告诉它：”你现在是Zygote“ 
* 进程名通常是：`zygote64`
* 它的职责是：
  * 预加载 framework/ART ,建好了大量 class / dex / so
  * 监听socket
  * fork()出新的App进程,利用 Copy-On-Write,子进程几乎“零成本继承”这些内存
Zygot = 运行在”zygote模式“下的app_process64实例
### 普通App进程是什么
  * 由Zygote fork出来的
  * 仍然是app_process64的代码
* 但它不再是Zygote，而是：
  * 进入ActivityThread.main()
  * 运行某个具体App
* 进程名会变成：
  * com.xx.yy

## linker64:bionic/liner
### 是什么
* ELF可执行文件
*  Zygote被启动时，通过PT_INTERP段加载和执行
* 负责加载所有 .so（libc.so、libdl.so、你的 app so）
* 设备路径：`/apex/com.android.runtime/bin/linker64`
* aosp路径:`bionic/liner`
# Zygote 加载 linker64
## 内核决定用linker64
当 init 通过 execve() 启动 app_process64（用于 Zygote 模式）时，内核在加载 app_process64 这个 ELF 可执行文件的过程中，读取其 PT_INTERP 段，从而先加载并执行 linker64 作为动态解释器。
ELF 中的 interpreter 是指：
当一个 ELF“动态可执行文件”被 execve() 启动时，
内核要先加载并执行的那个“动态链接器程序”。
## linker64自举 call链
bionic/linker/arch/arm64/begin.S
```c
ENTRY(_start)
  // Force unwinds to end in this function.
  .cfi_undefined x30

  mov x0, sp
  bl __linker_init

  // __linker_init returns the address of the entry point in the main image.
  br x0
END(_start)
```
bionic/linker/linker_main.cpp
```cpp
extern "C" ElfW(Addr) __linker_init(void* raw_args) {
    ...
    return __linker_init_post_relocation(args, tmp_linker_so);
}

static ElfW(Addr) __attribute__((noinline))
__linker_init_post_relocation(KernelArgumentBlock& args, soinfo& tmp_linker_so){
    ...
    tmp_linker_so.call_constructors();
    ...
}
```
bionic/linker/linker_soinfo.cpp
```cpp
void soinfo::call_constructors() {
  // DT_INIT should be called before DT_INIT_ARRAY if both are present.
  call_function("DT_INIT", init_func_, get_realpath());
  call_array("DT_INIT_ARRAY", init_array_, init_array_count_, false, get_realpath());
}
```
关键点
* 主程序依赖加载：解析 DT_NEEDED → mmap → relocate
* 执行 constructors：.init / .init_array

# 启动一个新的APP(Zygote fork-> ActivityThread)
这一步主要在 Java framework + Zygote native glue
### systemService 发起启动 （frameworks/base）
打开activity入口 frameworks/base/core/java/android/app/ContextImpl.java
```java
    @Override
    public void startActivity(Intent intent, Bundle options) {
        mMainThread.getInstrumentation().execStartActivity(
                getOuterContext(), mMainThread.getApplicationThread(), null,
                (Activity) null, intent, -1, applyLaunchDisplayIfNeeded(options));
    }
```
frameworks/base/core/java/android/app/ActivityThread.java
frameworks/base/core/java/android/app/Instrumentation.java
```java
    @UnsupportedAppUsage
    public ActivityResult execStartActivity(
        ...
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getOpPackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
        }
        ...
```
把请求交给system_server里的ATMS
frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
```java
    private int startActivityAsUser(IApplicationThread caller, String callingPackage,
            @Nullable String callingFeatureId, Intent intent, String resolvedType,
            IBinder resultTo, String resultWho, int requestCode, int startFlags,
            ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser){
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
                .setActivityOptions(opts)
                .setUserId(userId)
                .execute();   
            }
```
frameworks/base/services/core/java/com/android/server/wm/ActivityStartController.java
frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java
```java
int execute() {
  ...
    try {
        res = resolveToHeavyWeightSwitcherIfNeeded();
        if (res != START_SUCCESS) {
            return res;
        }

        res = executeRequest(mRequest);
    } finally {
        Binder.restoreCallingIdentity(origId);
        mRequest.logMessage.append(" result code=").append(res);
        Slog.i(TAG, mRequest.logMessage.toString());
        mRequest.logMessage.setLength(0);
    }
  ...
}
  private int executeRequest(Request request) {
            //日志中的START u0
            //为什么userId一般都是0,
            if (err == ActivityManager.START_SUCCESS) {
            request.logMessage.append("START u").append(userId).append(" {")
                    .append(intent.toShortString(true, true, true, false))
                    .append("} with ").append(launchModeToString(launchMode))
                    .append(" from uid ").append(callingUid);
            if (callingPackage != null) {
                request.logMessage.append(" (").append(callingPackage).append(")");
            }
            if (callingUid != realCallingUid
                    && realCallingUid != Request.DEFAULT_REAL_CALLING_UID) {
                request.logMessage.append(" (realCallingUid=").append(realCallingUid).append(")");
            }
        }
        ...
        //把这次启动“实体化”为一个 ActivityRecord
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

                ...
                //最终开始
                  mLastStartActivityResult = startActivityUnchecked(r, sourceRecord, voiceSession,
                request.voiceInteractor, startFlags, checkedOptions,
                inTask, inTaskFragment, balVerdict, intentGrants, realCallingUid,
                transition, isIndependent);
  }
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
        int startFlags, ActivityOptions options, Task inTask,
        TaskFragment inTaskFragment,
        BalVerdict balVerdict,
        NeededUriGrants intentGrants, int realCallingUid, Transition transition,
        boolean isIndependentLaunch) {
        ...
        try {
            mService.deferWindowLayout();
            r.mTransitionController.collect(r);
            try {
                Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "startActivityInner");
                result = startActivityInner(r, sourceRecord, voiceSession, voiceInteractor,
                        startFlags, options, inTask, inTaskFragment, balVerdict,
                        intentGrants, realCallingUid);
            } catch (Exception ex) {
                Slog.e(TAG, "Exception on startActivityInner", ex);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
                startedActivityRootTask = handleStartResult(r, options, result, isIndependentLaunch,
                        remoteTransition, transition);
            }
        } finally {
            mService.continueWindowLayout();
        }
        ...
        }
int startActivityInner(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, ActivityOptions options, Task inTask,
            TaskFragment inTaskFragment, BalVerdict balVerdict,
            NeededUriGrants intentGrants, int realCallingUid) {
              ...
                mRootWindowContainer.resumeFocusedTasksTopActivities(
                        mTargetRootTask, mStartActivity, mOptions, mTransientLaunch);
              ...
            }
```
frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java
```java
boolean resumeFocusedTasksTopActivities(
            Task targetRootTask, ActivityRecord target, ActivityOptions targetOptions,
            boolean deferPause) {
            if (targetRootTask != null && (targetRootTask.isTopRootTaskInDisplayArea()
                || getTopDisplayFocusedRootTask() == targetRootTask)) {
            result = targetRootTask.resumeTopActivityUncheckedLocked(target, targetOptions,
                    deferPause);
        }
            }
        
```
frameworks/base/services/core/java/com/android/server/wm/Task.java
```java
boolean resumeTopActivityUncheckedLocked(ActivityRecord prev, ActivityOptions options,boolean deferPause) {
          if (isLeafTask()) {
                if (isFocusableAndVisible()) {
                    someActivityResumed = resumeTopActivityInnerLocked(prev, options, deferPause);
                }
                }
    }


private boolean resumeTopActivityInnerLocked(ActivityRecord prev, ActivityOptions options,
            boolean deferPause) {
        final TaskFragment topFragment = topActivity.getTaskFragment();
        resumed[0] |= topFragment.resumeTopActivity(prev, options, deferPause);
            }
```
frameworks/base/services/core/java/com/android/server/wm/TaskFragment.java
```java
final boolean resumeTopActivity(ActivityRecord prev, ActivityOptions options,boolean skipPause) {
  ActivityRecord next = topRunningActivity(true /* focusableOnly */);
  if (next.attachedToProcess()) {

  }else{
    mTaskSupervisor.startSpecificActivity(next, true, true);
  }
}
```
frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java
```java
final ActivityTaskManagerService mService;
void startSpecificActivity(ActivityRecord r, boolean andResume, boolean checkConfig) {
          // Is this activity's application already running?
  final WindowProcessController wpc =
    mService.getProcessController(r.processName, r.info.applicationInfo.uid);
  if (wpc != null && wpc.hasThread()) {
    realStartActivityLocked(r, wpc, andResume, checkConfig);
  }
  mService.startProcessAsync(r, knownToBeDead, isTop,
    isTop ? HostingRecord.HOSTING_TYPE_TOP_ACTIVITY
    : HostingRecord.HOSTING_TYPE_ACTIVITY);
}

```
frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java
```java
void startProcessAsync(ActivityRecord activity, boolean knownToBeDead, boolean isTop,String hostingType) {
            // Post message to start process to avoid possible deadlock of calling into AMS with the
            // ATMS lock held.
            final Message m = PooledLambda.obtainMessage(ActivityManagerInternal::startProcess,
                    mAmInternal, activity.processName, activity.info.applicationInfo, knownToBeDead,
                    isTop, hostingType, activity.intent.getComponent());
            mH.sendMessage(m);
}
  //mAmInternel是AactivityManagerService的代理
    public void onActivityManagerInternalAdded() {
        synchronized (mGlobalLock) {
            mAmInternal = LocalServices.getService(ActivityManagerInternal.class);
            mUgmInternal = LocalServices.getService(UriGrantsManagerInternal.class);
        }
    }

    }
```
frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
```java
    private void start() {
        mBatteryStatsService.publish();
        mAppOpsService.publish();
        mProcessStats.publish();
        Slog.d("AppOps", "AppOpsService published");
        LocalServices.addService(ActivityManagerInternal.class, mInternal);
        LocalManagerRegistry.addManager(ActivityManagerLocal.class,
                (ActivityManagerLocal) mInternal);
        mActivityTaskManager.onActivityManagerInternalAdded();
        mPendingIntentController.onActivityManagerInternalAdded();
        mAppProfiler.onActivityManagerInternalAdded();
        CriticalEventLog.init();
    }

    public final class LocalService extends ActivityManagerInternal
            implements ActivityManagerLocal {

        public void startProcess(String processName, ApplicationInfo info, boolean knownToBeDead,
                boolean isTop, String hostingType, ComponentName hostingName){
                    synchronized (ActivityManagerService.this) {
                    HostingRecord hostingRecord =
                            new HostingRecord(hostingType, hostingName, isTop);
                    ProcessRecord rec = getProcessRecordLocked(processName, info.uid);
                    ProcessRecord app = startProcessLocked(processName, info, knownToBeDead,
                            0 /* intentFlags */, hostingRecord,
                            ZYGOTE_POLICY_FLAG_LATENCY_SENSITIVE, false /* allowWhileBooting */,
                            false /* isolated */);
                }
                }
            }

    final ProcessRecord startProcessLocked(String processName,
            ApplicationInfo info, boolean knownToBeDead, int intentFlags,
            HostingRecord hostingRecord, int zygotePolicyFlags, boolean allowWhileBooting,
            boolean isolated) {
        return mProcessList.startProcessLocked(processName, info, knownToBeDead, intentFlags,
                hostingRecord, zygotePolicyFlags,allowWhileBooting, isolated, 0 /* isolatedUid */,
                false /* isSdkSandbox */, 0 /* sdkSandboxClientAppUid */,
                null /* sdkSandboxClientAppPackage */,
                null /* ABI override */, null /* entryPoint */,
                null /* entryPointArgs */, null /* crashHandler */);
```

frameworks/base/services/core/java/com/android/server/am/ProcessList.java
```java
    boolean startProcessLocked(ProcessRecord app, HostingRecord hostingRecord,
            int zygotePolicyFlags, boolean disableHiddenApiChecks, boolean disableTestApiChecks,
            String abiOverride) {
                final Process.ProcessStartResult startResult = startProcess(hostingRecord,
                        entryPoint, app,
                        uid, gids, runtimeFlags, zygotePolicyFlags, mountExternal, seInfo,
                        requiredAbi, instructionSet, invokeWith, startUptime);
            }

    private Process.ProcessStartResult startProcess(HostingRecord hostingRecord, String entryPoint,ProcessRecord app, 
      int uid, int[] gids, int runtimeFlags, int zygotePolicyFlags,
      int mountExternal, String seInfo, String requiredAbi, String instructionSet,
      String invokeWith, long startTime) {
            //某些 WebView 渲染相关进程用 webview zygote（更快/隔离特性）。
            if (hostingRecord.usesWebviewZygote()) {
                startResult = startWebView(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, null, app.info.packageName,
                        app.getDisabledCompatChanges(),
                        new String[]{PROC_START_SEQ_IDENT + app.getStartSeq()});
            } else if (hostingRecord.usesAppZygote()) {
                //app 自己的 zygote（子 zygote）：用于隔离、加速、或特定架构（比如 SDK sandbox 也会用到相关机制）。
                final AppZygote appZygote = createAppZygoteForProcessIfNeeded(app);

                // We can't isolate app data and storage data as parent zygote already did that.
                startResult = appZygote.startProcess(entryPoint,
                        app.processName, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, app.info.packageName, isTopApp,
                        app.getDisabledCompatChanges(), pkgDataInfoMap,
                        allowlistedAppDataInfoMap,
                        new String[]{PROC_START_SEQ_IDENT + app.getStartSeq()});
            } else {
                regularZygote = true;
                //它内部会调用 ZygoteProcess.start(...) → 通过 socket 请求 zygote fork。
                startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, runtimeFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith, app.info.packageName, zygotePolicyFlags,
                        isTopApp, app.getDisabledCompatChanges(), pkgDataInfoMap,
                        allowlistedAppDataInfoMap, bindMountAppsData, bindMountAppStorageDirs,
                        bindOverrideSysprops,
                        new String[]{PROC_START_SEQ_IDENT + app.getStartSeq()});
                // By now the process group should have been created by zygote.
                app.mProcessGroupCreated = true;
            }  
            }
```
frameworks/base/core/java/android/os/Process.java
```java
    public static ProcessStartResult start(@NonNull final String processClass,
                                           @Nullable final String niceName,
                                           int uid, int gid, @Nullable int[] gids,
                                           int runtimeFlags,
                                           int mountExternal,
                                           int targetSdkVersion,
                                           @Nullable String seInfo,
                                           @NonNull String abi,
                                           @Nullable String instructionSet,
                                           @Nullable String appDataDir,
                                           @Nullable String invokeWith,
                                           @Nullable String packageName,
                                           int zygotePolicyFlags,
                                           boolean isTopApp,
                                           @Nullable long[] disabledCompatChanges,
                                           @Nullable Map<String, Pair<String, Long>>
                                                   pkgDataInfoMap,
                                           @Nullable Map<String, Pair<String, Long>>
                                                   whitelistedDataInfoMap,
                                           boolean bindMountAppsData,
                                           boolean bindMountAppStorageDirs,
                                           boolean bindMountSystemOverrides,
                                           @Nullable String[] zygoteArgs) {
        return ZYGOTE_PROCESS.start(processClass, niceName, uid, gid, gids,
                    runtimeFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, packageName,
                    zygotePolicyFlags, isTopApp, disabledCompatChanges,
                    pkgDataInfoMap, whitelistedDataInfoMap, bindMountAppsData,
                    bindMountAppStorageDirs, bindMountSystemOverrides, zygoteArgs);
    }
```
frameworks/base/core/java/android/os/ZygoteProcess.java
```java
    public final Process.ProcessStartResult start(){
      startViaZygote()
    }

    private Process.ProcessStartResult startViaZygote(){
      argsForZygote.add...
      synchronized(mLock) {
        // The USAP pool can not be used if the application will not use the systems graphics
        // driver.  If that driver is requested use the Zygote application start path.
        return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi),
                                          zygotePolicyFlags,
                                          argsForZygote);
      }
    }
    private Process.ProcessStartResult zygoteSendArgsAndGetResult(){
        //优先尝试 USAP（Unspecialized App Process）池
        if (shouldAttemptUsapLaunch(zygotePolicyFlags, args)) {
            try {
                return attemptUsapSendArgsAndGetResult(zygoteState, msgStr);
            } catch (IOException ex) {
            }
        }
        return attemptZygoteSendArgsAndGetResult(zygoteState, msgStr);
    }

    private Process.ProcessStartResult attemptZygoteSendArgsAndGetResult(){
      //取通信通道
      final BufferedWriter zygoteWriter = zygoteState.mZygoteOutputWriter;
final DataInputStream zygoteInputStream = zygoteState.mZygoteInputStream;
      //发送请求（写入 + flush）
      zygoteWriter.write(msgStr);
      zygoteWriter.flush();
      //读完整结果，避免“脏字节”污染下一次启动
      ProcessStartResult result = new ProcessStartResult();
      result.pid = zygoteInputStream.readInt();
      result.usingWrapper = zygoteInputStream.readBoolean();
      //pid < 0 → fork 失败
      /**
       * zygote 端约定：成功：返回子进程 pid（>0）失败：返回 -1（或负数）
       * */
      if (result.pid < 0) {
      throw new ZygoteStartFailedEx("fork() failed");
      }
    }
```
关键点：
* AMS/ProcessList 通过 ZygoteProcess 去和 zygote socket 通信

### Zygote侧接收命令并fork（frameworks/base + app_main）
frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
frameworks/base/core/java/com/android/internal/os/Zygote.java
native 层：
frameworks/base/core/jni/com_android_internal_os_Zygote.cpp
关键点：
* Zygote socket 收到参数（uid/gid、seinfo、niceName、classpath…）
* 调用 native ForkAndSpecialize → 最终 fork()
* 子进程返回后进入 ActivityThread.main()（应用主线程）



