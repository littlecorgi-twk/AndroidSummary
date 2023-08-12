# Activity 启动流程

## Launch 请求 AMS

Launcher 进程其实就是桌面，而我们点击桌面上的 APP 图标后，就是启动根 Activity，此时会调用 `Launcher` 的 `startActivitySafely()` 方法，而他会进一步调用父类 `BaseDraggingActivity` 的 `startActivitySafely()` 方法。这个方法会将外部传入的 `intent` ，设置启动模式为 `Intent.FLAG_ACTIVITY_NEW_TASK` ，然后再调用 `startActivity()`，这个方法是基类 `Activity` 声明的，内部会调用 `startActivityForResult()`（第二个参数 `requestCode` 为-1，代表 Launcher 不需要知道 Activity 启动的结果）。

紧接着会调用 `Instrumentation` 的 `execStartActivity()` 方法，这个方法首先会调用 `ActivityManager.getService()` 获取到 AMS 的对象，接着调用他的 `startActivity` 方法。



### ActivityManager.getService() 方法实际怎么执行

```java
@UnsupportedAppUsage
public static IActivityManager getService() {
    return IActivityManagerSingleton.get();
}

private static IActivityTaskManager getTaskService() {
    return ActivityTaskManager.getService();
}

@UnsupportedAppUsage
private static final Singleton<IActivityManager> IActivityManagerSingleton =
        new Singleton<IActivityManager>() {
            @Override
            protected IActivityManager create() {
                final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                final IActivityManager am = IActivityManager.Stub.asInterface(b);
                return am;
            }
        };
```

内部会通过`IActivityManagerSingleton.get()` 来获取对象。而这个 `IActivityManagerSingleton` 其实就是一个匿名内部类，他内部会通过 ServiceManager.getService 拿到 Binder 对象，然后通过 `IActivityManager.Sub.asInterface(b)` 就能拿到这个 Binder 通信的客户端对象了。



## AMS 到 ApplicationThread

AMS的 `startActivity()` 方法实际上又会去调用 `startActivityAsUser()` 方法，同时会获取 UserId 并传进去，内部会检查权限。最后调用了 `ActivityStarter` 的 `startActivity()` ，首先会检查我们一路透传过来的 `caller` (`IApplicationThread`) 是否为 null，然后通过 AMS 的 `getRecordForAppLocked()` 获取到 callerApp 对象，接下来创建一个 `ActivityRecord` ，用于记录一个 Activity 的所有信息。最终调用 `startActivityUnchecked()` 方法，这个方法内部会通过 `setTaskFromResumeOrCreteNewTask()` 来创建一个 TaskRecord，代表一个 Activity 任务栈，最终会调用到 `ActivityStackSupervisor` 的 `realStartActivityLocked()` 方法，这个方法会拿到 ApplicationThread 的对象，本质上是一个 Binder，通过它让 AMS 和应用程序进程进行通信。



## ActivityThread 启动 Activity

ApplicationThread 是 ActivityThread 的内部类，它里面会通过 `scheduleLaunchActivty()` 方法来启动 Activity，首先将启动的参数封装成 `ActivityClientRecord` ，并向 ActivityThread 里面的 Handler `H` 发送 `LAUNCH_ACTIVITY` 消息。向 H 发消息的原因是 ApplicationThread 是一个 Binder，他运行在 Binder 线程池中，所以我们需要切换到主线程去处理剩下的逻辑。



H 的 `handleMessage()` 方法会处理这个消息，内部会通过 `performLaunchActivity()` 启动 Activity，然后将 Activity 的状态设置为 `Resume`。而 `performLaunchActivity()` 首先获取 ActivityInfo，里面存储了一些 Manifest 设置的信息，然后获取 APK 文件的描述类 LoadedApk 以及要启动的 Activity 的ComponentName（这里面保存了 Activity 的包名和类名），最终会通过类加载器类创建这个 activity。创建好了后就调用 Activity 的`attach()` 方法用于初始化 Activity 以及创建 Window 对象。最终调用 `Instrumentation.callActiviyOnCreate()` 来调用这个 Activity 的`preformCreate()` 进而调用 `onCreate()`



## 附录

- [Activity的启动过程详解（基于Android10.0）](https://juejin.cn/post/6847902222294990862#comment)
