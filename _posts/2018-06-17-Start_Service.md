---
title:        "Android是如何开启一个Service的"
description:  "简单介绍下Service的启动过程"
image:        "http://placehold.it/400x200"
author:       "Shie1d Shen"
date:         "2018-06-17"
---

# Android是如何开启一个Service的

## 从Context#startService(Intent)开始

如果我们要开启一个Service服务，就需要调用到`Context#startService(Intent)`这个方法，而这个方法往里走，会走到使用`ActivityManagerServiceProxy#startService()`

```java
//ActivityManagerServiceProxy#startService()
public ComponentName startService(IApplicationThread caller, Intent service,
            String resolvedType, int userId) throws RemoteException
    {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        service.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeInt(userId);
        mRemote.transact(START_SERVICE_TRANSACTION, data, reply, 0);
        reply.readException();
        ComponentName res = ComponentName.readFromParcel(reply);
        data.recycle();
        reply.recycle();
        return res;
    }
```

这个方法中的`mRemote`就是使用`ServiceManager.getService("activity")`获取的Binder对象。所以，这个跨进程的调用最后会走进`ActivityManagerService#startService()`中。

这里，我们经过了**第一次**的跨进程通信

## ActivityManagerService#startService()

### 启动一个没有进程存在的Service

`ActivityManagerService`中会有一些诸如`ActiveServices`这些类的对象的参加，这里就不仔细去说各个方法的详细调用以及具体实现。

最终，这个方法会调用到`ActivityManagerService#startProcessLocked()`中，这个方法里面会有

```java
Process.ProcessStartResult startResult = Process.start("android.app.ActivityThread",
                    app.processName, uid, uid, gids, debugFlags, mountExternal,
                    app.info.targetSdkVersion, app.info.seinfo, null);
```

这个方法会去创建一个新的进程，并加载`android.app.ActivityThread`这个类，调用它的`main`函数。

以上这些都是发生在`ActivityManagerService`所在进程中，直到通过启动一个新的进程并加载`android.app.ActivityThread`开启了想要启动的Service所在的进程

### 启动一个已有进程且存在的Service

这个步骤里会走到`ActiveServices#sendServiceArgsLocked()`

```java
r.app.thread.scheduleServiceArgs(r, si.taskRemoved, si.id, flags, si.intent);
```

会调用这个方法，进入到Service所在进程中，然后最终调用到`Service#onStartCommand()`

## ActivityThread#main()

在这个方法中会调用`attach()`

```java
  private void attach(boolean system) {
        ...
        if (!system) {
            ...
            IActivityManager mgr = ActivityManagerNative.getDefault();
            try {
                mgr.attachApplication(mAppThread);
            } catch (RemoteException ex) {
                // Ignore
            }
        } else {
            ...
        }
        ...
  }
```

在这个方法中，又跨进程到了`ActivityManagerService#attachApplication()`中。

这是**第二次**跨进程调用

## ActivityManagerService#attachApplication()

这个里面会去跨进程调用到`ApplicationThread#bindApplication`等方法，并最终走到`realStartServiceLocked()`这个方法

```java
private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
       ...
        try {
            ...
            app.thread.scheduleCreateService(r, r.serviceInfo, //这里的Thread就是之前传过来的ApplicationThread的代理
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            r.postNotification();
            created = true;
        } finally {
            if (!created) {
                app.services.remove(r);
                r.app = null;
                scheduleServiceRestartLocked(r, false);
            }
        }

        ...

        sendServiceArgsLocked(r, execInFg, true);

        ...
    }
```

这里又会跨进程调用`ApplicationThread#scheduleCreateService()`，这也是**第三次**跨进程调用，回到了Service所在的进程

## ApplicationThread#scheduleCreateService() `ApplicationThread这个类存在于ActivityThread中`

这个过程中通过Handler调用到`handleCreateService()`

```java
private void handleCreateService(CreateServiceData data) {
    ...
     Service service = null;
        try {
            java.lang.ClassLoader cl = packageInfo.getClassLoader();
            service = (Service) cl.loadClass(data.info.name).newInstance();
        } catch (Exception e) {
            ...
        }
    ...
    service.onCreate();
    ...
}
```

在这个方法中，加载Service，并调用onCreate()。

至此，一个Service的启动过程就到这里，总结一下

1. 请求启动Service的进程（S1）向ActivityManagerService发出消息
2. ActivityManagerService检索是否存在这个Service进程
   * 已存在，告诉会Service所在进程（S2） END
   * 不存在，创建一个新的进程，加载ActivityThread
  
3. 新的进程ActivityThread运行成功后，会通知到ActivityManagerService创建完毕
4. ActivityManagerService接收到通知后，查找需要启动的Service，然后通知ActivityThread
5. ActivityThread接收通知，创建相应Service
