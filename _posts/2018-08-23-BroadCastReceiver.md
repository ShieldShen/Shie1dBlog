---
title:        "Android中的BroadcastReceiver"
description:  "介绍一下BroadcastReceiver"
image:        "http://placehold.it/400x200"
author:       "Shie1d Shen"
date:         "2018-08-23"
---

# Android中的BroadcastReceiver

广播是Android四大组件中最能体现松耦合概念的一个组件，它的使用由两步组成

1. 注册广播`Context#registerReceiver`
2. 发送广播`Context#sendBroadcast`

## registerReceiver

Context中的实现直接看ContextImpl

```java
    private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
            return ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
        } catch (RemoteException e) {
            return null;
        }
    }
```

其中，ReceiverDispatcher跟ServiceDispatcher类似，不再去研究，后面又是远程调用到AMS中，我们直接去看AMS中的代码

### AMS#registerReceiver

```java
    public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
        enforceNotIsolatedCaller("registerReceiver");
        int callingUid;
        int callingPid;
        synchronized(this) {
            ProcessRecord callerApp = null;
            if (caller != null) {
                callerApp = getRecordForAppLocked(caller);
                if (callerApp == null) {
                    throw new SecurityException(
                            "Unable to find app for caller " + caller
                            + " (pid=" + Binder.getCallingPid()
                            + ") when registering receiver " + receiver);
                }
                if (callerApp.info.uid != Process.SYSTEM_UID &&
                        !callerApp.pkgList.containsKey(callerPackage) &&
                        !"android".equals(callerPackage)) {
                    throw new SecurityException("Given caller package " + callerPackage
                            + " is not running in process " + callerApp);
                }
                callingUid = callerApp.info.uid;
                callingPid = callerApp.pid;
            } else {
                callerPackage = null;
                callingUid = Binder.getCallingUid();
                callingPid = Binder.getCallingPid();
            }

            userId = this.handleIncomingUser(callingPid, callingUid, userId,
                    true, true, "registerReceiver", callerPackage);

            List allSticky = null;

            // Look for any matching sticky broadcasts...
            Iterator actions = filter.actionsIterator();
            if (actions != null) {
                while (actions.hasNext()) {
                    String action = (String)actions.next();
                    allSticky = getStickiesLocked(action, filter, allSticky,
                            UserHandle.USER_ALL);
                    allSticky = getStickiesLocked(action, filter, allSticky,
                            UserHandle.getUserId(callingUid));
                }
            } else {
                allSticky = getStickiesLocked(null, filter, allSticky,
                        UserHandle.USER_ALL);
                allSticky = getStickiesLocked(null, filter, allSticky,
                        UserHandle.getUserId(callingUid));
            }

            // The first sticky in the list is returned directly back to
            // the client.
            Intent sticky = allSticky != null ? (Intent)allSticky.get(0) : null;

            if (DEBUG_BROADCAST) Slog.v(TAG, "Register receiver " + filter
                    + ": " + sticky);

            if (receiver == null) {
                return sticky;
            }

            ReceiverList rl
                = (ReceiverList)mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                    }
                    rl.linkedToDeath = true;
                }
                mRegisteredReceivers.put(receiver.asBinder(), rl);
            } else if (rl.uid != callingUid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for uid " + callingUid
                        + " was previously registered for uid " + rl.uid);
            } else if (rl.pid != callingPid) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for pid " + callingPid
                        + " was previously registered for pid " + rl.pid);
            } else if (rl.userId != userId) {
                throw new IllegalArgumentException(
                        "Receiver requested to register for user " + userId
                        + " was previously registered for user " + rl.userId);
            }
            BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
                    permission, callingUid, userId);
            rl.add(bf);
            if (!bf.debugCheck()) {
                Slog.w(TAG, "==> For Dynamic broadast");
            }
            mReceiverResolver.addFilter(bf);

            // Enqueue broadcasts for all existing stickies that match
            // this filter.
            if (allSticky != null) {
                ArrayList receivers = new ArrayList();
                receivers.add(bf);

                int N = allSticky.size();
                for (int i=0; i<N; i++) {
                    Intent intent = (Intent)allSticky.get(i);
                    BroadcastQueue queue = broadcastQueueForIntent(intent);
                    BroadcastRecord r = new BroadcastRecord(queue, intent, null,
                            null, -1, -1, null, null, AppOpsManager.OP_NONE, receivers, null, 0,
                            null, null, false, true, true, -1);
                    queue.enqueueParallelBroadcastLocked(r);
                    queue.scheduleBroadcastsLocked();
                }
            }

            return sticky;
        }
    }
```

这里主要是把这个Receiver以及各种信息保存起来，如果有stickyIntent则直接触发

## sendBroadcast

这里也是由ContextImpl远程调用到AMS#broadcastIntent()中

### AMS#broadcastIntent()

再往里走一步到`broadcastIntentLocked`

```java
private final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle map, String requiredPermission, int appOp,
            boolean ordered, boolean sticky, int callingPid, int callingUid,
            int userId) {
        intent = new Intent(intent);
        ...
        if (intent.getComponent() == null) {
            registeredReceivers = mReceiverResolver.queryIntent(intent,
                    resolvedType, false, userId);
        }

        final boolean replacePending =
                (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;

        int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        if (!ordered && NR > 0) {
            // If we are not serializing this broadcast, then send the
            // registered receivers separately so they don't wait for the
            // components to be launched.
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType, requiredPermission,
                    appOp, registeredReceivers, resultTo, resultCode, resultData, map,
                    ordered, sticky, false, userId);
            if (DEBUG_BROADCAST) Slog.v(
                    TAG, "Enqueueing parallel broadcast " + r);
            final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
            if (!replaced) {
                queue.enqueueParallelBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
            registeredReceivers = null;
            NR = 0;
        }

        ...
    }

```

这里去掉了很多代码，我们只看重要的

`registeredReceivers = mReceiverResolver.queryIntent(intent,resolvedType, false, userId);`这行代码，我们不去深究它的原理，这个是取出与发送Action相应的Receiver并按照优先级排序给我们

`final BroadcastQueue queue = broadcastQueueForIntent(intent);`先简单介绍一下BroadcastQueue，AMS中存在两个BroadcastQueue，一个Foreground，一个Background。这行代码根据intent的flag找到相应的BroadcastQueue。

`queue.scheduleBroadcastsLocked();`这行代码就是发送广播内容，我们往里看

```java
    private final void processNextBroadcast(boolean fromMsg) {
        synchronized(this) {
            BroadcastRecord r;

            ......

            if (fromMsg) {
                mBroadcastsScheduled = false;
            }

            // First, deliver any non-serialized broadcasts right away.
            while (mParallelBroadcasts.size() > 0) {
                r = mParallelBroadcasts.remove(0);
                ......
                final int N = r.receivers.size();
                ......
                for (int i=0; i<N; i++) {
                    Object target = r.receivers.get(i);
                    ......

                    deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
                }
                addBroadcastToHistoryLocked(r);
                ......
            }
            ......

        }
    }
```

这里其实除了主要代码以外还有诸如BroadcastReceiver的有效时间限制，发起时间算起后的20s(foreground)到120s(background)。

而`deliverToRegisteredReceiverLocked`里面又会调用到`performReceiveLocked`

```java
    private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
        // Send the intent to the receiver asynchronously using one-way binder calls.
        if (app != null && app.thread != null) {
            // If we have an app thread, do the call through that so it is
            // correctly ordered with other one-way calls.
            app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                    data, extras, ordered, sticky, sendingUser, app.repProcState);
        } else {
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    }
```

这就是转给ActivityThread处理

```java
//ActivityThread
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
            updateProcessState(processState, false);
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }
```

这个Receiver就在ReceiverDispatcher中

```java
    static final class ReceiverDispatcher {

        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }
            public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                LoadedApk.ReceiverDispatcher rd = mDispatcher.get();
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = intent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Receiving broadcast " + intent.getAction() + " seq=" + seq
                            + " to " + (rd != null ? rd.mReceiver : null));
                }
                if (rd != null) {
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser);
                } else {
                    // The activity manager dispatched a broadcast to a registered
                    // receiver in this process, but before it could be delivered the
                    // receiver was unregistered.  Acknowledge the broadcast on its
                    // behalf so that the system's broadcast sequence can continue.
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing broadcast to unregistered receiver");
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    try {
                        if (extras != null) {
                            extras.setAllowFds(false);
                        }
                        mgr.finishReceiver(this, resultCode, data, extras, false);
                    } catch (RemoteException e) {
                        Slog.w(ActivityThread.TAG, "Couldn't finish broadcast to unregistered receiver");
                    }
                }
            }
        }
```

又走到`rd.performReceive`

```java
public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            if (ActivityThread.DEBUG_BROADCAST) {
                int seq = intent.getIntExtra("seq", -1);
                Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction() + " seq=" + seq
                        + " to " + mReceiver);
            }
            Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
            if (!mActivityThread.post(args)) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing sync broadcast to " + mReceiver);
                    args.sendFinished(mgr);
                }
            }
        }
```

因为Args继承自Runnable，所以`mActivityThread.post(args)`执行都就是

```java
    public void run() {
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered;

                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = mCurIntent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Dispatching broadcast " + mCurIntent.getAction()
                            + " seq=" + seq + " to " + mReceiver);
                    Slog.i(ActivityThread.TAG, "  mRegistered=" + mRegistered
                            + " mOrderedHint=" + ordered);
                }

                final IActivityManager mgr = ActivityManagerNative.getDefault();
                final Intent intent = mCurIntent;
                mCurIntent = null;

                if (receiver == null || mForgotten) {
                    if (mRegistered && ordered) {
                        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                "Finishing null broadcast to " + mReceiver);
                        sendFinished(mgr);
                    }
                    return;
                }

                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveReg");
                try {
                    ClassLoader cl =  mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    setExtrasClassLoader(cl);
                    receiver.setPendingResult(this);
                    receiver.onReceive(mContext, intent);
                } catch (Exception e) {
                    if (mRegistered && ordered) {
                        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                "Finishing failed broadcast to " + mReceiver);
                        sendFinished(mgr);
                    }
                    if (mInstrumentation == null ||
                            !mInstrumentation.onException(mReceiver, e)) {
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                        throw new RuntimeException(
                            "Error receiving broadcast " + intent
                            + " in " + mReceiver, e);
                    }
                }

                if (receiver.getPendingResult() != null) {
                    finish();
                }
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
        }
```

这里就是onReceive的地方
