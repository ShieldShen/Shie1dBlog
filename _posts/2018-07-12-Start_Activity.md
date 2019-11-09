---
title:        "Android是如何开启一个Activity的"
description:  "简单介绍下Activity的启动过程"
image:        "http://placehold.it/400x200"
author:       "Shie1d Shen"
date:         "2018-07-12"
---

# Android是如何开启一个Activity的

## 从Context#startService(Intent)开始

同Service的启动一样，这里讨论Activity从一个新的进程启动以及本进程启动两个情况

无论哪种情况，都是从`Context#startActivity(Intent)`这个方法开始，这个方法调用了`ContextImpl#startActivity()`,再转交给`Instrumentation#execStartActivity()`方法

```java
 public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ...
            int result = ActivityManagerNative.getDefault()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, null, options);
            checkStartActivityResult(result, intent);
        ...
    }
```

跨进程调用`ActivityManagerService#startActivity()`

## ActivityManagerService#startActivity()

再AMS中，Service有个`ActiveServices`类做管理，Activity由`ActivityStack`管理。

这个方法最后会转发到`ActivityStack#resumeTopActivityLocked()`中

```java
final boolean resumeTopActivityLocked(ActivityRecord prev, Bundle options) {
    ...
if (mResumedActivity != null) {
            pausing = true;
            startPausingLocked(userLeaving, false);
            if (DEBUG_STATES) Slog.d(TAG, "resumeTopActivityLocked: Pausing " + mResumedActivity);
        }
        if (pausing) {
            return true;
        }
        ...
         mStackSupervisor.startSpecificActivityLocked(next, true, false);
        ...

}
```

这里因为发起启动的Activity还在，所以要先去将他pause住，因此走入`startPausingLocked(userLeaving, false);`中。

## startPausingLocked(userLeaving, false)

```java
prev.app.thread.schedulePauseActivity(prev.appToken,
        prev.finishing,userLeaving, prev.configChangeFlags);
```

在`startPausingLocked`中，最终调用的是上面的代码，也就是跨进程调用发起进程的`ApplicationThread#schedulePauseActivity()`

## ApplicationThread#schedulePauseActivity()

其实这个调用最后起到的效果就是保存当前信息，然后再通过`ActivityManagerNative.getDefault().activityPaused(token);`告诉AMS准备好了

## ActivityManagerService#activityPaused()

这里又会调用到`ActivityStack#resumeTopActivityLocked()`,但是这次因为发起的前一个Activity已经进入Pause状态，所以会走到`mStackSupervisor.startSpecificActivityLocked(next, true, false);`方法中

```java
void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {
        // Is this activity's application already running?
        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        r.task.stack.setLaunchTime(r);

        if (app != null && app.thread != null) {
            try {
                if ((r.info.flags&ActivityInfo.FLAG_MULTIPROCESS) == 0
                        || !"android".equals(r.info.packageName)) {
                    // Don't add this if it is a platform component that is marked
                    // to run in multiple processes, because this is actually
                    // part of the framework so doesn't make sense to track as a
                    // separate apk in the process.
                    app.addPackage(r.info.packageName, mService.mProcessStats);
                }
                realStartActivityLocked(r, app, andResume, checkConfig);
                return;
            } catch (RemoteException e) {
                Slog.w(TAG, "Exception when starting activity "
                        + r.intent.getComponent().flattenToShortString(), e);
            }

            // If a dead object exception was thrown -- fall through to
            // restart the application.
        }

        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
```

在这个方法中，已有进程会直接调用`realStartActivityLocked`，而尚未创建进程的，会先去创建进程`mService.startProcessLocked`

## ActivityManagerService#startProcessLocked()

简单来说就是创建新的进程，加载`ActivityThread`然后会调用到`ActivityManagerProxy#attachApplication()`，跨进程调用AMS#attachApplication(),而AMS中在准备好Application后，会去ActivityStack中检索最上层的Activity也就是我们要唤醒的Activity，
然后同样会走入`ActivityStackSupervisor#realStartActivityLocked`方法中。

## ActivityStackSupervisor#realStartActivityLocked()

```java
app.thread.scheduleLaunchActivity(new Intent(r.intent), r.appToken,
                    System.identityHashCode(r), r.info,
                    new Configuration(mService.mConfiguration), r.compat,
                    app.repProcState, r.icicle, results, newIntents, !andResume,
                    mService.isNextTransitionForward(), profileFile, profileFd,
                    profileAutoStop);
```

在`realStartActivityLocked`中，又会调用到刚刚创建的进程的`ApplicationThread#scheduleLaunchActivity`

## ApplicationThread#scheduleLaunchActivity

这里就是加载Activity的对象，调用onCreate,onStart,onResume等。
到这里就大概Activity的创建过程。
