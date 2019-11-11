---
title:        "Android中bindService的原理"
description:  "补充一下前面的Service相关，毕竟两种启动方式都很常用"
image:        "http://placehold.it/400x200"
author:       "Shie1d Shen"
date:         "2018-08-07"
---

# Android中bindService的原理

从前面我们就知道bindService这个方法其实最终使用的是`ContextImpl`里的，所以我们进去看

```java
    public boolean bindService(Intent service, ServiceConnection conn,
            int flags) {
        warnIfCallingFromSystemProcess();
        return bindServiceCommon(service, conn, flags, Process.myUserHandle());
    }

    private boolean bindServiceCommon(Intent service, ServiceConnection conn, int flags,
            UserHandle user) {
        IServiceConnection sd;
        if (conn == null) {
            throw new IllegalArgumentException("connection is null");
        }
        if (mPackageInfo != null) {
            sd = mPackageInfo.getServiceDispatcher(conn, getOuterContext(),
                    mMainThread.getHandler(), flags);
        } else {
            throw new RuntimeException("Not supported in system context");
        }
        validateServiceIntent(service);
        try {
            IBinder token = getActivityToken();
            if (token == null && (flags&BIND_AUTO_CREATE) == 0 && mPackageInfo != null
                    && mPackageInfo.getApplicationInfo().targetSdkVersion
                    < android.os.Build.VERSION_CODES.ICE_CREAM_SANDWICH) {
                flags |= BIND_WAIVE_PRIORITY;
            }
            service.prepareToLeaveProcess();
            int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(),
                service, service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, user.getIdentifier());
            if (res < 0) {
                throw new SecurityException(
                        "Not allowed to bind to service " + service);
            }
            return res != 0;
        } catch (RemoteException e) {
            return false;
        }
    }
```

这里`mPackageInfo`是`LoadedApk`的对象，`mMainThread`是`ActivityThread`的对象

```java
    public final IServiceConnection getServiceDispatcher(ServiceConnection c,
            Context context, Handler handler, int flags) {
        synchronized (mServices) {
            LoadedApk.ServiceDispatcher sd = null;
            ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher> map = mServices.get(context);
            if (map != null) {
                sd = map.get(c);
            }
            if (sd == null) {
                sd = new ServiceDispatcher(c, context, handler, flags);
                if (map == null) {
                    map = new ArrayMap<ServiceConnection, LoadedApk.ServiceDispatcher>();
                    mServices.put(context, map);
                }
                map.put(c, sd);
            } else {
                sd.validate(context, handler);
            }
            return sd.getIServiceConnection();
        }
    }
```

这部分就是把一些信息包装记录起来。再后面就是

```java
int res = ActivityManagerNative.getDefault().bindService(
                mMainThread.getApplicationThread(), getActivityToken(),
                service, service.resolveTypeIfNeeded(getContentResolver()),
                sd, flags, user.getIdentifier());
```

这行代码就不再重复去说为什么了，远程调用到AMS的bindService。

## ActivityManagerService#bindService

这里跟下去最终会调用到`ActiveServices#bindServiceLocked`

```java
    int bindServiceLocked(IApplicationThread caller, IBinder token,
            Intent service, String resolvedType,
            IServiceConnection connection, int flags, int userId) {
                ...
                AppBindRecord b = s.retrieveAppBindingLocked(service, callerApp);
                ...

                if ((flags&Context.BIND_AUTO_CREATE) != 0) {
                    s.lastActivity = SystemClock.uptimeMillis();
                    if (bringUpServiceLocked(s, service.getFlags(), callerFg, false) != null) {
                        return 0;
                    }
                }

                ...

                if (s.app != null && b.intent.received) {
                ...
                } else if (!b.intent.requested) {
                    requestServiceBindingLocked(s, b.intent, callerFg, false);
                }
            }
```

这里，首先把各种信息记录检索一遍，然后调用`bringUpServiceLocked`，这里前面的博客提到过了，就是启动Service，最终会调用到Service的onCreate。然后代码又走到这里过后，会走到下面的`requestServiceBindingLocked`

## ActiveServices#requestServiceBindingLocked

这里其实就是调用`r.app.thread.scheduleBindService(r, i.intent.getIntent(), rebind,r.app.repProcState);`这句代码，回到`ApplicationThread`。其实最后就是调用到`ActivityThread#handleBindService`

## ActivityThread#handleBindService

```java
private void handleBindService(BindServiceData data) {
    ...
    ActivityManagerNative.getDefault().publishService(
                                data.token, data.intent, binder);
    ...
}
```

这里又调回了AMS。

## 在AMS中

最后会调用到下面的代码，也在ActiveServices中

```java
void publishServiceLocked(ServiceRecord r, Intent intent, IBinder service) {
        final long origId = Binder.clearCallingIdentity();
        try {
            if (DEBUG_SERVICE) Slog.v(TAG, "PUBLISHING " + r
                    + " " + intent + ": " + service);
            if (r != null) {
                Intent.FilterComparison filter
                        = new Intent.FilterComparison(intent);
                IntentBindRecord b = r.bindings.get(filter);
                if (b != null && !b.received) {
                    b.binder = service;
                    b.requested = true;
                    b.received = true;
                    for (int conni=r.connections.size()-1; conni>=0; conni--) {
                        ArrayList<ConnectionRecord> clist = r.connections.valueAt(conni);
                        for (int i=0; i<clist.size(); i++) {
                            ConnectionRecord c = clist.get(i);
                            if (!filter.equals(c.binding.intent.intent)) {
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG, "Not publishing to: " + c);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG, "Bound intent: " + c.binding.intent.intent);
                                if (DEBUG_SERVICE) Slog.v(
                                        TAG, "Published intent: " + intent);
                                continue;
                            }
                            if (DEBUG_SERVICE) Slog.v(TAG, "Publishing to: " + c);
                            try {
                                c.conn.connected(r.name, service);
                            } catch (Exception e) {
                                Slog.w(TAG, "Failure sending service " + r.name +
                                      " to connection " + c.conn.asBinder() +
                                      " (in " + c.binding.client.processName + ")", e);
                            }
                        }
                    }
                }

                serviceDoneExecutingLocked(r, mDestroyingServices.contains(r), false);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

其中`c.conn.connected(r.name, service)`这个里面的conn就是之前`LoadedApk`中的

```java
        private static class InnerConnection extends IServiceConnection.Stub {
            final WeakReference<LoadedApk.ServiceDispatcher> mDispatcher;

            InnerConnection(LoadedApk.ServiceDispatcher sd) {
                mDispatcher = new WeakReference<LoadedApk.ServiceDispatcher>(sd);
            }

            public void connected(ComponentName name, IBinder service) throws RemoteException {
                LoadedApk.ServiceDispatcher sd = mDispatcher.get();
                if (sd != null) {
                    sd.connected(name, service);
                }
            }
        }
```

sd就是ServiceDispatcher

```java
        //ServiceDispatcher
        public void connected(ComponentName name, IBinder service) {
            if (mActivityThread != null) {
                mActivityThread.post(new RunConnection(name, service, 0));
            } else {
                doConnected(name, service);
            }
        }

        public void doConnected(ComponentName name, IBinder service) {
            ServiceDispatcher.ConnectionInfo old;
            ServiceDispatcher.ConnectionInfo info;

            synchronized (this) {
                if (mForgotten) {
                    // We unbound before receiving the connection; ignore
                    // any connection received.
                    return;
                }
                old = mActiveConnections.get(name);
                if (old != null && old.binder == service) {
                    // Huh, already have this one.  Oh well!
                    return;
                }

                if (service != null) {
                    // A new service is being connected... set it all up.
                    mDied = false;
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                        service.linkToDeath(info.deathMonitor, 0);
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {
                        // This service was dead before we got it...  just
                        // don't do anything with it.
                        mActiveConnections.remove(name);
                        return;
                    }

                } else {
                    // The named service is being disconnected... clean up.
                    mActiveConnections.remove(name);
                }

                if (old != null) {
                    old.binder.unlinkToDeath(old.deathMonitor, 0);
                }
            }

            // If there was an old service, it is not disconnected.
            if (old != null) {
                mConnection.onServiceDisconnected(name);
            }
            // If there is a new service, it is now connected.
            if (service != null) {
                mConnection.onServiceConnected(name, service);
            }
        }
```

这里的mConnection就是ServiceConnection。

所以与startService相比，bindService区别部分是会通过一个ServiceDispatcher来执行更多回调来传递数据，而启动Service的方式并无二致。
