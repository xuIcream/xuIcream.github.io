title: Android Zygote systemServer
date: 2016-01-10 11:49:39
tags:
---

### Zygote SystemServer启动
主要参考老罗的博客．[原文地址](http://blog.csdn.net/luoshengyang/article/details/6768304)
在android系统中,所有的应用程序进程及系统服务进程SystemServer都是由Zygote而来,从名字中就可以发现android 万物来自Zygote.

Android基于Linux内核,所有进程都是init进程的子进程,都是fork出来的,Zygote也不例外,
Zygote也是由init进程创建.

system/core/rootdir/init.rc
```c
import /init.${ro.zygote}.rc
```

system/core/rootdir/init.zygote64.rc
```bash
service zygote /system/bin/app_process 
-Xzygote /system/bin --zygote --start-system-server
    class main
    socket zygote stream 660 root system
    onrestart write /sys/android_power/request_state wake
    onrestart write /sys/power/state on
    onrestart restart media
    onrestart restart netd
    onrestart restart surfaceflinger 
```

service告诉init进程创建一个名为"zygote"的进程，这个zygote进程要执行的程序是/system/bin/app_process.cpp 
(init进程的源代码位于system/core/init目录中，在init.c文件中由service_start函数解释init.rc文件中的service命令)

frameworks/base/cmds/app_process/app_main.cpp 
```c++
int main(int argc, char* const argv[])
{
    if (zygote) {
        runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
    } else if (className) {
        runtime.start("com.android.internal.os.RuntimeInit", args, zygote);
    } else {
        fprintf(stderr, "Error: no class name or --zygote supplied.\n");
        app_usage();
        LOG_ALWAYS_FATAL("app_process: no class name or --zygote supplied.");
        return 10;
    }
}
```
参数zygote 为true,调用
```c++
runtime.start("com.android.internal.os.ZygoteInit", args, zygote);
```
这里的runtime为AppRuntime继承 AndroidRuntime,start方法没有覆写,位于父类
frameworks/base/core/jni/AndroidRuntime.cpp
```c++
void AndroidRuntime::start(const char* className, const Vector<String8>& options, bool zygote)
{
    static const String8 startSystemServer("start-system-server");
    JniInvocation jni_invocation;
    jni_invocation.Init(NULL);
    JNIEnv* env;
    if (startVm(&mJavaVM, &env, zygote) != 0) {
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
    char* slashClassName = toSlashClassName(className);
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
            if (env->ExceptionCheck())
                threadExitUncaughtException(env);
        }
    }
    free(slashClassName);
}
```
start函数主要调用三个方法:
１：startVM启动虚拟机
２：startReg注册JNI方法
３：调用com.android.internal.os.ZygoteInit类的main函数

frameworks/base/core/java/com/android/internal/os/ZygoteInit.java
```java
public static void main(String argv[]) {
        try {
            registerZygoteSocket(socketName);
            gc();
            if (startSystemServer) {
                startSystemServer(abiList, socketName);
            }
            boolean disableNonCoreServices = SystemProperties.getBoolean("config.disable_noncore", false);
            boolean disableAtlas = SystemProperties.getBoolean("config.disable_atlas", false);
            if (!disableNonCoreServices && !disableAtlas && startSystemServer) {
                startAtlasService();
            }
            runSelectLoop(abiList, startSystemServer);
            closeServerSocket();
        } catch (MethodAndArgsCaller caller) {
            caller.run();
        } catch (RuntimeException ex) {
            Log.e(TAG, "Zygote died with exception", ex);
            closeServerSocket();
            throw ex;
        }
}
```

这里主要调用三个方法:
１：registerZygoteSocket 创建了一个socket接口，用来和ActivityManagerService通讯 
２：startSystemServer 启动SystemServer组件 
３：runSelectLoop 等待ActivityManagerService请求创建新的应用程序进程

主要分析startSystemServer相关流程
```java
private static boolean startSystemServer(String abiList, String socketName)
            throws MethodAndArgsCaller, RuntimeException {
        /* Hardcoded command line to start the system server */
        String args[] = {
            "--setuid=1000",
            "--setgid=1000",
            "--setgroups=1001,1002,1003,1004,1005,1006,1007,1008,1009,1010,1018,1021,1032,3001,3002,3003,3006,3007",
            "--capabilities=" + capabilities + "," + capabilities,
            "--runtime-init",
            "--nice-name=system_server",
            "com.android.server.SystemServer",
        };
        ZygoteConnection.Arguments parsedArgs = null;
        int pid;
        try {
            parsedArgs = new ZygoteConnection.Arguments(args);
            ZygoteConnection.applyDebuggerSystemProperty(parsedArgs);
            ZygoteConnection.applyInvokeWithSystemProperty(parsedArgs);
            /* Request to fork the system server process */
            pid = Zygote.forkSystemServer(
                    parsedArgs.uid, parsedArgs.gid,
                    parsedArgs.gids,
                    parsedArgs.debugFlags,
                    null,
                    parsedArgs.permittedCapabilities,
                    parsedArgs.effectiveCapabilities);
        } catch (IllegalArgumentException ex) {
            throw new RuntimeException(ex);
        }
        /* For child process */
        if (pid == 0) {
            if (hasSecondZygote(abiList)) {
                waitForSecondaryZygote(socketName);
            }

            handleSystemServerProcess(parsedArgs);
        }
        return true;
    }
```
frameworks/base/core/java/com/android/internal/os/Zygote.java
Zygote.forkSystemServer　
```java
public static int forkSystemServer(int uid, int gid, int[] gids, int debugFlags,
            int[][] rlimits, long permittedCapabilities, long effectiveCapabilities) {
        VM_HOOKS.preFork();
        int pid = nativeForkSystemServer(
                uid, gid, gids, debugFlags, rlimits, permittedCapabilities, effectiveCapabilities);
        VM_HOOKS.postForkCommon();
        return pid;
} 
```

在子进程中pid为零，所以调用handleSystemServerProcess
```java
private static void handleSystemServerProcess(
            ZygoteConnection.Arguments parsedArgs)
            throws ZygoteInit.MethodAndArgsCaller {
        RuntimeInit.zygoteInit(parsedArgs.remainingArgs);    
}
```

调用到RuntimeInit.zygoteInit
frameworks/base/core/java/com/android/internal/os/RuntimeInit.java
```java
public static final void zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        if (DEBUG) Slog.d(TAG, "RuntimeInit: Starting application from zygote");
        redirectLogStreams();
        commonInit();
        nativeZygoteInit();
        applicationInit(targetSdkVersion, argv, classLoader);
    }
```
这里主要是调用两个函数

１：nativeZygoteInit是一个Native函数，实现在AndroidRuntime nativeZygoteInit
```c++
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit();
}
```
而gCurRuntime的实际类型未AppRumtime
frameworks/base/cmds/app_process/app_main.cpp
```c++
virtual void onZygoteInit()
    {
        // Re-enable tracing now that we're no longer in Zygote.
        atrace_set_tracing_enabled(true);
        sp<ProcessState> proc = ProcessState::self();
        ALOGV("App process: starting thread pool.\n");
        proc->startThreadPool();
    }
```
ProcessState::startThreadPool启动线程池，这个线程池是用来和Binder驱动交互，
Binder进程间通信机制基础设施就准备好了

２：applicationInit
```java
private static void applicationInit(int targetSdkVersion, String[] argv, ClassLoader classLoader)
            throws ZygoteInit.MethodAndArgsCaller {
        // We want to be fairly aggressive about heap utilization, to avoid
        // holding on to a lot of memory that isn't needed.
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args;
        try {
            args = new Arguments(argv);
        } catch (IllegalArgumentException ex) {
            Slog.e(TAG, ex.getMessage());
            // let the process exit
            return;
        }

        // Remaining arguments are passed to the start class's static main
        invokeStaticMain(args.startClass, args.startArgs, classLoader);
}
```

通过invokeStaticMain走进SystemServer　main函数
frameworks/base/services/java/com/android/server/SystemServer.java
```java
public static void main(String[] args) {
        SystemProperties.set("persist.sys.dalvik.vm.lib",
                             VMRuntime.getRuntime().vmLibrary());
        // Mmmmmm... more memory!
        dalvik.system.VMRuntime.getRuntime().clearGrowthLimit();
        System.loadLibrary("android_servers");
        Slog.i(TAG, "Entered the Android system server!");
        // Initialize native services.
        nativeInit();
        // This used to be its own separate thread, but now it is
        // just the loop we run on the main thread.
        ServerThread thr = new ServerThread();
        thr.initAndLoop();
}
```
在ServerThread线程中执行一些系统关键服务的启动操作如　ActivieyManagerServices PackageManagerServices
1. 系统启动时init进程会创建Zygote进程，Zygote进程负责后续Android应用程序框架层的其它进程的创建和启动工作。

2. Zygote进程会首先创建一个SystemServer进程，SystemServer进程负责启动系统的关键服务，如包管理服务PackageManagerService和应用程序组件管理服务ActivityManagerService。

3. 当我们需要启动一个Android应用程序时，ActivityManagerService会通过Socket进程间通信机制，通知Zygote进程为这个应用程序创建一个新的进程。

