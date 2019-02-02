

title: 深入Android辅助服务架构与设计
date: 2018-3-22 19:30:19
toc: true
tags: [AccessibilityService,ViewRootImpl,AccessibilityManagerService]
categories: technology
description: 在16年时,写过一篇关于AccessibilityService使用的文章在网上被多次提及,后续在工作中也多次涉及,在此前提下对AccessibilityService有了更深层次的理解,本文将从系统的角度分析服务服务架构.

------

首先我们需要明确在整个AccessibilityService体系中共包含三个部分,其结构基本如下:

![image-20181225174719637](https://ws1.sinaimg.cn/large/006tNbRwly1fyj4wv7zqrj317a0o6tco.jpg)

- 被监控应用端: 即我们需要监控的应用,比如微信,系统某些界面等等;
- 监控服务端: 用来实时接受来自被监控应用端的事件,并作出处理,即我们自行实现的AccessibilityService
- AccessibilityManagerService: 由于被监控应用端和监控服务端涉涉及跨进程通信,同时它们之间又是多对多的关系,因此为了需要引入中间管理器来对两端进行管理.(为了方便起见,后续我们简称为AMS,注意不要和ActivityManagerService进行混淆)

需要注意的对于被监控客户端和AMS以及监控服务端和AMS而言,都可以单独的当成C-S架构来看:对于被监控客户端和AMS: 前者是客户端Client,后者是服务端Server;对于监控服务端和AMS: 前者是客户端Client,后者是Server.

在后续的正文中,将按照 `监控服务端和AMS -> 被监控客户端和AMS`的顺序来讲述,前者以AccessibilityService注册为主题,后者以AccessibilityEvnet分为为主旨.

# AccessibilityService注册流程

在AccessibilityService注册中主要涉及AMS绑定我们自定义AccessibilityService的过程.在该阶段,当AMS检测到系统中AccessibilityService状态变化后会做出相应的处理,比如检测到某个AccessibilityService被安装到系统并被用户启用后,AMS会主动绑定AccessibilityService,这和绑定远程服务的流程一样.

## AccessibilityService架构

在开发辅助服务时,需要我们继承AccessibilityService,那AccessibilityService到底是什么呢?首先来看一张基本结构图:

![image-20181225180514185](https://ws1.sinaimg.cn/large/006tNbRwly1fyj5f0kwq7j31890u0akz.jpg)

AccessibilityService继承自Service,也就是说它就是一个标准的服务,该服务只能由AMS进行绑定.

```java
public abstract class AccessibilityService extends Service {
    
     @Override
    public final IBinder onBind(Intent intent) {
        // AMS主动绑定时,将返回IAccessibilityServiceClientWrapper实例
        return new IAccessibilityServiceClientWrapper(this, getMainLooper(), new Callbacks() {
            @Override
            public void onServiceConnected() {
                AccessibilityService.this.dispatchServiceConnected();
            }

            @Override
            public void onInterrupt() {
                AccessibilityService.this.onInterrupt();
            }

            @Override
            public void onAccessibilityEvent(AccessibilityEvent event) {
                AccessibilityService.this.onAccessibilityEvent(event);
            }

            @Override
            public void init(int connectionId, IBinder windowToken) {
                mConnectionId = connectionId;
                mWindowToken = windowToken;

                // The client may have already obtained the window manager, so
                // update the default token on whatever manager we gave them.
                final WindowManagerImpl wm = (WindowManagerImpl) getSystemService(WINDOW_SERVICE);
                wm.setDefaultToken(windowToken);
            }

       		......
                
        });
    }
    
    ......
    
}
```

不难发现AccessibilityService的`onBind()`方法中返回类型为IAccessibilityServiceClientWrapper的对象,IAccessibilityServiceClientWrapper以内部类的形式存在于AccessibilityService中,其定义如下:

```java
 public static class IAccessibilityServiceClientWrapper extends IAccessibilityServiceClient.Stub
            implements HandlerCaller.Callback {
        private static final int DO_INIT = 1;
        private static final int DO_ON_INTERRUPT = 2;
        private static final int DO_ON_ACCESSIBILITY_EVENT = 3;
        private static final int DO_ON_GESTURE = 4;
        private static final int DO_CLEAR_ACCESSIBILITY_CACHE = 5;
        private static final int DO_ON_KEY_EVENT = 6;
        private static final int DO_ON_MAGNIFICATION_CHANGED = 7;
        private static final int DO_ON_SOFT_KEYBOARD_SHOW_MODE_CHANGED = 8;
        private static final int DO_GESTURE_COMPLETE = 9;
        private static final int DO_ON_FINGERPRINT_ACTIVE_CHANGED = 10;
        private static final int DO_ON_FINGERPRINT_GESTURE = 11;
        private static final int DO_ACCESSIBILITY_BUTTON_CLICKED = 12;
        private static final int DO_ACCESSIBILITY_BUTTON_AVAILABILITY_CHANGED = 13;

        private final HandlerCaller mCaller;

        private final Callbacks mCallback;

        private int mConnectionId = AccessibilityInteractionClient.NO_ID;

        public IAccessibilityServiceClientWrapper(Context context, Looper looper,
                Callbacks callback) {
            mCallback = callback;
            mCaller = new HandlerCaller(context, looper, this, true /*asyncHandler*/);
        }

        public void init(IAccessibilityServiceConnection connection, int connectionId,
                IBinder windowToken) {
            Message message = mCaller.obtainMessageIOO(DO_INIT, connectionId,
                    connection, windowToken);
            mCaller.sendMessage(message);
        }

        public void onInterrupt() {
            Message message = mCaller.obtainMessage(DO_ON_INTERRUPT);
            mCaller.sendMessage(message);
        }

        public void onAccessibilityEvent(AccessibilityEvent event, boolean serviceWantsEvent) {
            Message message = mCaller.obtainMessageBO(
                    DO_ON_ACCESSIBILITY_EVENT, serviceWantsEvent, event);
            mCaller.sendMessage(message);
        }

		.......
		
 }
```

IAccessibilityServiceClientWrapper继承自IAccessibilityServiceClient.Stub,而IAccessibilityServiceClient用来代表AccessibilityService组件.在IAccessibilityServiceClient接口中定义了暴露给AMS可操作的方法:

```java
public interface IAccessibilityServiceClient extends android.os.IInterface {
    
    public static abstract class Stub extends android.os.Binder implements android.accessibilityservice.IAccessibilityServiceClient {
        ......
            
         private static class Proxy implements android.accessibilityservice.IAccessibilityServiceClient {
         }
        
        ......
    }
    
    public void init(android.accessibilityservice.IAccessibilityServiceConnection connection, int connectionId, android.os.IBinder windowToken) throws android.os.RemoteException;
  public void onAccessibilityEvent(android.view.accessibility.AccessibilityEvent event, boolean serviceWantsEvent) throws android.os.RemoteException;
  public void onInterrupt() throws android.os.RemoteException;
  public void onGesture(int gesture) throws android.os.RemoteException;
  public void clearAccessibilityCache() throws android.os.RemoteException;
  public void onKeyEvent(android.view.KeyEvent event, int sequence) throws android.os.RemoteException;
  public void onMagnificationChanged(android.graphics.Region region, float scale, float centerX, float centerY) throws android.os.RemoteException;
  public void onSoftKeyboardShowModeChanged(int showMode) throws android.os.RemoteException;
  public void onPerformGestureResult(int sequence, boolean completedSuccessfully) throws android.os.RemoteException;
  public void onFingerprintCapturingGesturesChanged(boolean capturing) throws android.os.RemoteException;
  public void onFingerprintGesture(int gesture) throws android.os.RemoteException;
  public void onAccessibilityButtonClicked() throws android.os.RemoteException;
  public void onAccessibilityButtonAvailabilityChanged(boolean available) throws android.os.RemoteException;

}
```

此外IAccessibilityServiceClientWrapper的构造函数中存在参数类型为Callbacks的回调接口.该回调接口最终会被AMS调用,其定义如下:

android.accessibilityservice.AccessibilityService.Callbacks

```java
    public interface Callbacks {
        void onAccessibilityEvent(AccessibilityEvent event);
        void onInterrupt();
        void onServiceConnected();
        void init(int connectionId, IBinder windowToken);
        boolean onGesture(int gestureId);
        boolean onKeyEvent(KeyEvent event);
        void onMagnificationChanged(@NonNull Region region,
                float scale, float centerX, float centerY);
        void onSoftKeyboardShowModeChanged(int showMode);
        void onPerformGestureResult(int sequence, boolean completedSuccessfully);
        void onFingerprintCapturingGesturesChanged(boolean active);
        void onFingerprintGesture(int gesture);
        void onAccessibilityButtonClicked();
        void onAccessibilityButtonAvailabilityChanged(boolean available);
    }
```

在AccessibilityService启用时,AMS会主动绑定该服务,并通过`onBind()`返回IAccessibilityServiceClientWrapper对象,当AMS需要与AccessibilityService通信时,就会回远程回调此处的Callbacks接口,以分发AccessibilityEvent到对应的AccessibilityService为例,此时AMS会远程回调Callbacks中的`onAccessibilityEvent()`:

```java
new IAccessibilityServiceClientWrapper(this, getMainLooper(), new Callbacks() {
   
            @Override
            public void onAccessibilityEvent(AccessibilityEvent event) {
                // 事件分发
                AccessibilityService.this.onAccessibilityEvent(event);
            }
    
    		......
}
```

我们发现在Callbacks的匿名实现类中,最终又调用了AccessibilityService实例中的`onAccessibilityEvent(event)`,此方法是抽象类AccessibilityService中的抽象方法,也是我们在自定义AccessibilityService时必须要重写的方法:

```java
public abstract class AccessibilityService extends Service {
	......
        
    public abstract void onAccessibilityEvent(AccessibilityEvent event);
    
    .......
}
```



## AMS绑定AccessibilityService时机

和AMS启动时机一样,AccessibilityServiceManager也是由SystemServer启动,其构造函数如下:

com.android.server.accessibility.AccessibilityManagerService#AccessibilityManagerService

```java
    public AccessibilityManagerService(Context context) {
        mContext = context;
        mPackageManager = mContext.getPackageManager();
        mPowerManager = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);
        mWindowManagerService = LocalServices.getService(WindowManagerInternal.class);
        mUserManager = (UserManager) context.getSystemService(Context.USER_SERVICE);
        mSecurityPolicy = new SecurityPolicy();
        mAppOpsManager = (AppOpsManager) context.getSystemService(Context.APP_OPS_SERVICE);
        mMainHandler = new MainHandler(mContext.getMainLooper());
        mGlobalActionPerformer = new GlobalActionPerformer(mContext, mWindowManagerService);
		// 动态注册广播接收器
        registerBroadcastReceivers();
        new AccessibilityContentObserver(mMainHandler).register(
                context.getContentResolver());
    }
```

其中`registerBroadcastReceivers()`动态注册PackageMonitor类型的广播接收器.其中PackageMonitor用于监听app安装,删除,更新以及SD卡增加/移除的广播通知,然后根据这些广播通知,来决定什么时候要主动绑定某个AccessibilityService或者和某个AccessibilityService断开.比如当某个App从系统中被删除或者被强制停止时,会分别调用`onPackageRemoved()`和`onHandleForceStop()`方法,如下所示:

```java
private void registerBroadcastReceivers() {
        PackageMonitor monitor = new PackageMonitor() {
            
            @Override
            public void onSomePackagesChanged() {
                ......
                onUserStateChangedLocked();
                ......    
            }
            
            @Override
            public void onPackageRemoved(String packageName, int uid) {
                ......
                onUserStateChangedLocked();
                ......
            }
            
            @Override
            public boolean onHandleForceStop(Intent intent, String[] packages,int uid, boolean doit) {
               ......
                onUserStateChangedLocked();
               ......  
            }
            
            ......
        }
```

通过上述代码不难看出AccessibilityManagerService通过监听系统状态变化的广播,并决定是否调用`onUserStateChangedLocked()`来更新状态,更新的时机主要涉及以下场景:

- 用户在系统设置界面,为某个APP开启辅助服务的时候
- 用户在系统设置界面,关闭某个APP辅助功能
- 接受到用户删除APP事件的时候
- 接受到某个App被强制停止

接下来来看看onUserStateChangedLocked()中到底做了什么:

```java
 private void onUserStateChangedLocked(UserState userState) {
        // TODO: Remove this hack
        mInitialized = true;
        updateLegacyCapabilitiesLocked(userState);
        updateServicesLocked(userState);
        updateAccessibilityShortcutLocked(userState);
        updateWindowsForAccessibilityCallbackLocked(userState);
        updateAccessibilityFocusBehaviorLocked(userState);
        updateFilterKeyEventsLocked(userState);
        updateTouchExplorationLocked(userState);
        updatePerformGesturesLocked(userState);
        updateDisplayDaltonizerLocked(userState);
        updateDisplayInversionLocked(userState);
        updateMagnificationLocked(userState);
        updateSoftKeyboardShowModeLocked(userState);
        scheduleUpdateFingerprintGestureHandling(userState);
        scheduleUpdateInputFilter(userState);
        scheduleUpdateClientsIfNeededLocked(userState);
        updateRelevantEventsLocked(userState);
        updateAccessibilityButtonTargetsLocked(userState);
    }
```

上述代码中会调用很多状态更新的方法,但我们目前只关心AMS什么时候主动绑定AccessibilityService.因此先看方法`updateServicesLocked()`:

```java
     private void updateServicesLocked(UserState userState) {
        Map<ComponentName, AccessibilityServiceConnection> componentNameToServiceMap =
                userState.mComponentNameToServiceMap;
        boolean isUnlockingOrUnlocked = LocalServices.getService(UserManagerInternal.class)
                    .isUserUnlockingOrUnlocked(userState.mUserId);
		// mInstalledServices表示手机中所有已经安装的AccessibilityService,
        // 每个AccessibilityService用AccessibilityServiceInfo表示
        for (int i = 0, count = userState.mInstalledServices.size(); i < count; i++) {
            AccessibilityServiceInfo installedService = userState.mInstalledServices.get(i);
            ComponentName componentName = ComponentName.unflattenFromString(
                    installedService.getId());

            AccessibilityServiceConnection service = componentNameToServiceMap.get(componentName);

            // Ignore non-encryption-aware services until user is unlocked
            if (!isUnlockingOrUnlocked && !installedService.isDirectBootAware()) {
                Slog.d(LOG_TAG, "Ignoring non-encryption-aware service " + componentName);
                continue;
            }

            // Wait for the binding if it is in process.
            if (userState.mBindingServices.contains(componentName)) {
                continue;
            }
            if (userState.mEnabledServices.contains(componentName)
                    && !mUiAutomationManager.suppressingAccessibilityServicesLocked()) {
                // 当前AccessibilityService被启用后,为其创建连接对象
                // AccessibilityServiceConnection
                if (service == null) {
                    service = new AccessibilityServiceConnection(userState, mContext, componentName,
                            installedService, sIdCounter++, mMainHandler, mLock, mSecurityPolicy,
                            this, mWindowManagerService, mGlobalActionPerformer);
                } else if (userState.mBoundServices.contains(service)) {
                    continue;
                }
                // 调用AccessibilityServiceConnection的bindLocked()方法主动连接
                // AccessibilityService
                service.bindLocked();
            } else {
                if (service != null) {
                    service.unbindLocked();
                }
            }
        }

        final int count = userState.mBoundServices.size();
        mTempIntArray.clear();
        for (int i = 0; i < count; i++) {
            final ResolveInfo resolveInfo =
                    userState.mBoundServices.get(i).mAccessibilityServiceInfo.getResolveInfo();
            if (resolveInfo != null) {
                mTempIntArray.add(resolveInfo.serviceInfo.applicationInfo.uid);
            }
        }
        // Calling out with lock held, but to a lower-level service
        final AudioManagerInternal audioManager =
                LocalServices.getService(AudioManagerInternal.class);
        if (audioManager != null) {
            audioManager.setAccessibilityServiceUids(mTempIntArray);
        }
        updateAccessibilityEnabledSetting(userState);
    }
```

这里面主要做了三件事情:

1. 遍历手机中所有已安装的辅助服务,这些服务信息被保存在`UserState.mInstalledServices`中;
2. 根据componentName,在`mEnabledServices`(mEnableServices保存了所有启动的辅助服务,即AMS绑定过的AccessibilityService)里面查找enabled状态的AccessibilityService组件,如果不存在就构造一个service.这里service类型是AccessibilityServiceConnection,在AMS中,用AccessibilityServiceConnection表示AMS和某个AccessibilityService的连接.
3. 调用service的bindLocked方法来进行真正的绑定操作

其实componentName里面保存的就是AccessibilityService的packageName和className.重点来看service的`bindLocked()`操作:

com.android.server.accessibility.AccessibilityServiceConnection#bindLocked

```java
    public void bindLocked() {
        UserState userState = mUserStateWeakReference.get();
        if (userState == null) return;
        final long identity = Binder.clearCallingIdentity();
        try {
            int flags = Context.BIND_AUTO_CREATE | Context.BIND_FOREGROUND_SERVICE_WHILE_AWAKE;
            if (userState.mBindInstantServiceAllowed) {
                flags |= Context.BIND_ALLOW_INSTANT;
            }
            // 调用bindServiceAsUser()来绑定服务,其中第二个参数this类型是ServiceConnection
            if (mService == null && mContext.bindServiceAsUser(
                    mIntent, this, flags, new UserHandle(userState.mUserId))) {
                userState.getBindingServicesLocked().add(mComponentName);
            }
        } finally {
            Binder.restoreCallingIdentity(identity);
        }
        
        
    }
```

上述代码中最终调用`bindServiceAsUser()`来绑定AccessibilityService.绑定成功后回调ServiceConnection中的`onServiceConnected()`,即AccessibilityServiceConnection中的`onServiceConnected()`:

```java
    public void onServiceConnected(ComponentName componentName, IBinder service) {
        synchronized (mLock) {
            // 此处service即AccessibilityService中onBind()方法返回的
            // IAccessibilityServiceClientWrapper对象
            if (mService != service) {
                if (mService != null) {
                    mService.unlinkToDeath(this, 0);
                }
                mService = service;
                try {
                    mService.linkToDeath(this, 0);
                } catch (RemoteException re) {
                    Slog.e(LOG_TAG, "Failed registering death link");
                    binderDied();
                    return;
                }
            }
            // mServiceInterface是AccessibilityServiceConnection类型
            mServiceInterface = IAccessibilityServiceClient.Stub.asInterface(service);
            UserState userState = mUserStateWeakReference.get();
            if (userState == null) return;
            // 调用AMS的addServiceLocked()方法将该Connection实例保存在mBoundServices
            // 成员变量中
            userState.addServiceLocked(this);
            mSystemSupport.onClientChange(false);
            // Initialize the service on the main handler after we're done setting up for
            // the new configuration (for example, initializing the input filter).
            // 主线程继续调用initializeService()方法来继续完成初始化操作
            mMainHandler.sendMessage(obtainMessage(
                    AccessibilityServiceConnection::initializeService, this));
        }
    }
```

不难看出其实每个AccessibilityServiceConnection都关联了对应AccessibilityService中返回的IAccessibilityServiceClientWrapper对象,即上述代码中的mService.然后将该AccessibilityServiceConnection保存在AMS中对应UserState的mBoundServices中:

com.android.server.accessibility.AccessibilityManagerService.UserState#addServiceLocked

```java
public class UserState {
    public final ArrayList<AccessibilityServiceConnection> mBoundServices = new ArrayList<>();

    public void addServiceLocked(AccessibilityServiceConnection serviceConnection) {
         if (!mBoundServices.contains(serviceConnection)) {
              serviceConnection.onAdded();
              mBoundServices.add(serviceConnection);
              mComponentNameToServiceMap.put(serviceConnection.mComponentName, serviceConnection);
              scheduleNotifyClientsOfServicesStateChange(this);
          }
      }
    
    ......
}
```

接下来继续调用`initializeService()`来完成初始化操作:

```java
    private void initializeService() {
        IAccessibilityServiceClient serviceInterface = null;
        synchronized (mLock) {
            UserState userState = mUserStateWeakReference.get();
            if (userState == null) return;
            Set<ComponentName> bindingServices = userState.getBindingServicesLocked();
            if (bindingServices.contains(mComponentName) || mWasConnectedAndDied) {
                bindingServices.remove(mComponentName);
                mWasConnectedAndDied = false;
                serviceInterface = mServiceInterface;
            }
        }
        if (serviceInterface == null) {
            binderDied();
            return;
        }
        try {
            // 远程调用IAccessibilityServiceClient的init()方法以便AccessibilityService所在
            // 进程能够持有当前AccessibilityServiceConnection的代理对象
            serviceInterface.init(this, mId, mOverlayWindowToken);
        } catch (RemoteException re) {
            Slog.w(LOG_TAG, "Error while setting connection for service: "
                    + serviceInterface, re);
            binderDied();
        }
    }
```

最终初始化完成后,在AMS进程一端,AMS持有远程AccessibilityService中IAccessibilityServiceClientWrapper的本地代理对象,在AMS需要和AccessibilityService通信时,就会远程回调IAccessibilityServiceClientWrapper中Callbacks接口;此外AccessibilityService也持有了AMS端中对应AccessibilityServiceConnection的本地代理对象,在AccessibilityService需要和AMS通信时便会借助该代理对象.

![image-20181225231722678](https://ws1.sinaimg.cn/large/006tNbRwly1fyjefsx7hlj31f00kgjt8.jpg)

## 小结

当AMS监听到一些系统状态变化时,最终会调用`onUserStateChangedLocked()`进行用户状态更新操作.在此期间会根据componentName,在mEnabledServices里面查找enabled状态的AccessibilityService组件,并为其生成对应AccessibilityServiceConnection对象,然后调用该对象的`bindLocked()`方法,在`bindLocked()`中会调用context的`bindServiceAsUser(mIntent...)`来绑定AccessibilityService.

![image-20181226215836704](https://ws1.sinaimg.cn/large/006tNbRwly1fykhsbwuf4j31ko0u0qi4.jpg)



# AccessibilityEvent分发

AccessibilityEvent代表将可能系统中产生的事件,该事件对象产生后会通过跨进程的方式传送给AMS,然后AMS继续通过跨进程的方式传递给对应AccessibilityService进程.

## AccessibilityEvent生成及初始化

当一个View想要对外发送AccessibilityEvent时,需要用到以下接口:

```java
public interface AccessibilityEventSource {

    public void sendAccessibilityEvent(int eventType);

    public void sendAccessibilityEventUnchecked(AccessibilityEvent event);
}
```

该接口中定义了两个方法用来发送AccessibilityEvent.Android官方希望任何一个View在开始设计时都能对残疾人友好,因此选择在View来实现该接口,这样我们在View及其子类就可以方便的支持无障碍服务了.首先来看View中关于点击操作所发出的事件的流程,即`performClick()`:

```java
public class View implements Drawable.Callback, KeyEvent.Callback,
        AccessibilityEventSource {

    AccessibilityDelegate mAccessibilityDelegate;

    public boolean performClick() {
        final boolean result;
        final ListenerInfo li = mListenerInfo;
        // 1.设置ClickListener的情况,先回调onClick()
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }
		
        // 2.向辅助服务发送CLICKED事件
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);
        notifyEnterOrExitForAutoFillIfNeeded(true);
        return result;
    }

    public void sendAccessibilityEvent(int eventType) {
        if (mAccessibilityDelegate != null) {
            // 如果已经设置了委托,则调用委托者的sendAccessibilityEvent()
            mAccessibilityDelegate.sendAccessibilityEvent(this, eventType);
        } else {
            // 没有设置委托的情况下,调用sendAccessibilityEventInternal()
            sendAccessibilityEventInternal(eventType);
        }
    }
            
    public void sendAccessibilityEventInternal(int eventType){
        // isEnabled()用来检查系统辅助功能是否开启
        if(AccessibilityManager.getInstance(mContext).isEnabled()){
            // 继续调用sendAccessibilityEventUnchecked()实现事件发送
            sendAccessibilityEventUnchecked(AccessibilityEvent.obtain(eventType))
        }       
    }
            
    public void sendAccessibilityEventUnchecked(AccessibilityEvent event) {
        // 如果已经设置了委托,则调用委托者的sendAccessibilityEventUnchecked()
        if (mAccessibilityDelegate != null) {
            mAccessibilityDelegate.sendAccessibilityEventUnchecked(this, event);
        } else {
            // 继续调用sendAccessibilityEventUncheckedInternal()实现事件发送
            sendAccessibilityEventUncheckedInternal(event);
        }
    }
                      
            
    public void sendAccessibilityEventUncheckedInternal(AccessibilityEvent event) {
        // 1.isShown()用递归检查当前view以及其父view是否可见.如果不可见了就没必要继续处理了
        if (!isShown()) {
            return;
        }
        // 2.使用有关View的事件源的信息初始化AccessibilityEvent对象
        onInitializeAccessibilityEvent(event);
        // Only a subset of accessibility events populates text content.
        if ((event.getEventType() & POPULATING_ACCESSIBILITY_EVENT_TYPES) != 0) {
            dispatchPopulateAccessibilityEvent(event);
        }
		// ViewRootImpl是ViewParent的实现类
        ViewParent parent = getParent();
        if (parent != null) {
            // 3. 请求父View发送AccessibilityEvent,这里最终调用了ViewRootImpl的
            // requestSendAccessibilityEvent()方法
            getParent().requestSendAccessibilityEvent(this, event);
        }
    }

    public void setAccessibilityDelegate(@Nullable AccessibilityDelegate delegate) {
        mAccessibilityDelegate = delegate;
    }
}
```

在上述代码中主要做了两件事:

- 事件生成: 指定要发送的AccessibilityEvent类型,并生成对应的AccessibilityEvent事件
- 事件发送: 在没有设置委托的情况下,最终调用`ViewRootImpl#requestSendAccessibilityEvent()`请求发送事件

### 事件生成

由于系统中会产生大量的事件,如果为每个事件都创建对应AccessibilityEvent对象可能会造成GC频繁发生,进而影响整体性能,因此Google采用享元模式来实现对AccessibilityEvent对象的复用.

AccessibilityEvent.obtain()根据事件类型返回一个缓存的AccessibilityEvent实例,享元模式.

```java
public class AccessibilityEvent{
    
    private static final int MAX_POOL_SIZE = 10;
    //sPool是对象池
    private static final SynchronizedPool<AccessibilityEvent> sPool =
            new SynchronizedPool<AccessibilityEvent>(MAX_POOL_SIZE);
    
    
    public static AccessibilityEvent obtain(int eventType) {
        AccessibilityEvent event = AccessibilityEvent.obtain();
        event.setEventType(eventType);
        return event;
    }
    
    public static AccessibilityEvent obtain() {
        AccessibilityEvent event = sPool.acquire();
        return (event != null) ? event : new AccessibilityEvent();
    }
    
}
```

### 事件初始化

在获取AccessibilityEvent对象之后接下来会用当前事件源的信息对AccessiblityEvent对象进行初始化操作.在没有设置mAccessibilityDelegate的情况下默认通过View.onInitializeAccessibilityEventInternal()进行初始化:

```java
public void onInitializeAccessibilityEventInternal(AccessibilityEvent event) {
        // 1.设置当前View为事件源
        event.setSource(this);
        // 2.将产生该事件所在类的类名设置为AccessibilityEvent的mClassName.View的子类中通过
    	// 复写getAccessibilityClassName()来返回事件类的类名
        event.setClassName(getAccessibilityClassName());
        // 3.该事件是由那个应用产生的
        event.setPackageName(getContext().getPackageName());
    	// 4.产生该事件的View是否在可用状态
        event.setEnabled(isEnabled());
    	// 5.该View对应的内容概要信息,可以通过android:contentDescription来设置
        event.setContentDescription(mContentDescription);

        switch (event.getEventType()) {
            case AccessibilityEvent.TYPE_VIEW_FOCUSED: {
                ArrayList<View> focusablesTempList = (mAttachInfo != null)
                        ? mAttachInfo.mTempArrayList : new ArrayList<View>();
                getRootView().addFocusables(focusablesTempList, View.FOCUS_FORWARD, FOCUSABLES_ALL);
                event.setItemCount(focusablesTempList.size());
                event.setCurrentItemIndex(focusablesTempList.indexOf(this));
                if (mAttachInfo != null) {
                    focusablesTempList.clear();
                }
            } break;
            case AccessibilityEvent.TYPE_VIEW_TEXT_SELECTION_CHANGED: {
                CharSequence text = getIterableTextForAccessibility();
                if (text != null && text.length() > 0) {
                    //设置选中字符的开始位置
                    event.setFromIndex(getAccessibilitySelectionStart());
                    //设置选中字符的结束位置
                    event.setToIndex(getAccessibilitySelectionEnd());
                    //设置选中字符的长度,为什么不把选中的文字设置过来呢?
                    event.setItemCount(text.length());
                }
            } break;
        }
    }
```

上面的代码在View中对AccessibilityEvent对象的一些公共属性进行初始化,在View的子类可以重写此方法以便根据不同View组件来添加更多的信息.比如在TextView重写了此方法,进一步添加了事件的信息:

```java
public class TextView extends View implements ViewTreeObserver.onPreDrawListener{
    
     @Override
    public void onInitializeAccessibilityEventInternal(AccessibilityEvent event) {
        super.onInitializeAccessibilityEventInternal(event);

        final boolean isPassword = hasPasswordTransformationMethod();
        event.setPassword(isPassword);

        if (event.getEventType() == AccessibilityEvent.TYPE_VIEW_TEXT_SELECTION_CHANGED) {
            event.setFromIndex(Selection.getSelectionStart(mText));
            event.setToIndex(Selection.getSelectionEnd(mText));
            event.setItemCount(mText.length());
        }
    }
    
    
}
```

## AccessibilityEvent发送到AMS

将AccessibilityEvent发送给AMS的具体操作最终由`ViewRootImpl#requestSendAccessibilityEvent`完成,具体流程如下:

```java
   @Override
    public boolean requestSendAccessibilityEvent(View child, AccessibilityEvent event) {
        if (mView == null || mStopped || mPausedForTransition) {
            return false;
        }

        // Immediately flush pending content changed event (if any) to preserve event order
        if (event.getEventType() != AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED
                && mSendWindowContentChangedAccessibilityEvent != null
                && mSendWindowContentChangedAccessibilityEvent.mSource != null) {
            mSendWindowContentChangedAccessibilityEvent.removeCallbacksAndRun();
        }

        // Intercept accessibility focus events fired by virtual nodes to keep
        // track of accessibility focus position in such nodes.
        final int eventType = event.getEventType();
        switch (eventType) {
            case AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUSED: {
                final long sourceNodeId = event.getSourceNodeId();
                final int accessibilityViewId = AccessibilityNodeInfo.getAccessibilityViewId(
                        sourceNodeId);
                View source = mView.findViewByAccessibilityId(accessibilityViewId);
                if (source != null) {
                    AccessibilityNodeProvider provider = source.getAccessibilityNodeProvider();
                    if (provider != null) {
                        final int virtualNodeId = AccessibilityNodeInfo.getVirtualDescendantId(
                                sourceNodeId);
                        final AccessibilityNodeInfo node;
                        node = provider.createAccessibilityNodeInfo(virtualNodeId);
                        setAccessibilityFocus(source, node);
                    }
                }
            } break;
            case AccessibilityEvent.TYPE_VIEW_ACCESSIBILITY_FOCUS_CLEARED: {
                final long sourceNodeId = event.getSourceNodeId();
                final int accessibilityViewId = AccessibilityNodeInfo.getAccessibilityViewId(
                        sourceNodeId);
                View source = mView.findViewByAccessibilityId(accessibilityViewId);
                if (source != null) {
                    AccessibilityNodeProvider provider = source.getAccessibilityNodeProvider();
                    if (provider != null) {
                        setAccessibilityFocus(null, null);
                    }
                }
            } break;


            case AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED: {
                handleWindowContentChangedEvent(event);
            } break;
        }
        // mAccessibilityManager是AccessibilityManager类型实例,最终调用
        // AccessibilityManager实例的sendAccessibilityEvent()来发送事件
        mAccessibilityManager.sendAccessibilityEvent(event);
        return true;
    }

```

在上述方法中首先对一些特殊类型的事件做了处理,最后调用AccessibilityManager实例的`sendAccessibilityEvent()`来向AccessibilityManagerService发送事件.

### AccessibilityManager创建

AccessibilityManager以单例的形式存在,在其构造函数中会尝试连接AMS服务,即AccessibilityManagerService.

```java
public final class AccessibilityManager {
    final Handler.Callback mCallback;
    final Handler mHandler;
    
    ...
    
    public static AccessibilityManager getInstance(Context context) {
        synchronized (sInstanceSync) {
            if (sInstance == null) {
                final int userId;
                ......
                sInstance = new AccessibilityManager(context, null, userId);
            }
        }
        return sInstance;
    }

    public AccessibilityManager(Context context, IAccessibilityManager service, int userId) {
        mCallback = new MyCallback();
        mHandler = new Handler(context.getMainLooper(), mCallback);
        mUserId = userId;
        synchronized (mLock) {
            // 尝试连接到AMS服务
            tryConnectToServiceLocked(service);
        }
    }
    
}
```

在AccessibilityManager的构造函数中,最终的的是通过`tryConnectToServiceLocked()`方法来连接AMS:

```java
private void tryConnectToServiceLocked(IAccessibilityManager service) {
        if (service == null) {
            IBinder iBinder = ServiceManager.getService(Context.ACCESSIBILITY_SERVICE);
            if (iBinder == null) {
                return;
            }
            service = IAccessibilityManager.Stub.asInterface(iBinder);
        }

        try {
            final long userStateAndRelevantEvents = service.addClient(mClient, mUserId);
            setStateLocked(IntPair.first(userStateAndRelevantEvents));
            mRelevantEventTypes = IntPair.second(userStateAndRelevantEvents);
            mService = service;
        } catch (RemoteException re) {
            Log.e(LOG_TAG, "AccessibilityManagerService is dead", re);
        }
    }
```

上述代码中首先通过ServiceManager来获取AccessibilityManagerService在本地代理对象,即IAccessibilityManager实例,在IAccessibilityManager接口中暴露了AccessibilityManagerService对外提供的方法,这是典型Binder通信过程,在系统代码中涉及多进程通信的过程基本类似.

```java
public interface IAccessibilityManager extends android.os.IInterface {
    
    public static abstract class Stub extends android.os.Binder implements android.view.accessibility.IAccessibilityManager {
        .......
    }
    
    private static class Proxy implements android.view.accessibility.IAccessibilityManager {
        .......
            
         @Override   
         public void sendAccessibilityEvent(android.view.accessibility.AccessibilityEvent uiEvent, int userId) throws android.os.RemoteException {
            .......
        }
        
        @Override
        public long addClient(android.view.accessibility.IAccessibilityManagerClient client, int userId) throws android.os.RemoteException {
            ......
        }
      
        .....
        
    }
}
```

AccessibilityManager在获取AMS的本地代理对象IAccessibilityManager后,会继续调用IAccessibilityManager的`addClient()`方法来将mClient通过跨进程的方式传递到AccessibilityManagerService中.那mClient到底是什么呢?其定义如下:

```java
public final class AccessibilityManager {
        private final IAccessibilityManagerClient.Stub mClient =
            new IAccessibilityManagerClient.Stub() {
            .......
        }
}
```

不难发现这里mClient同样是Binder对象.当AccessibilityManagerService需要通知客户端一些变化时会利用到它.

```java
public interface IAccessibilityManagerClient extends android.os.IInterface {
    
     public static abstract class Stub extends android.os.Binder implements android.view.accessibility.IAccessibilityManagerClient {
       	static final int TRANSACTION_setState = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
   		static final int TRANSACTION_notifyServicesStateChanged = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    	static final int TRANSACTION_setRelevantEventTypes = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
         
         ......
             
         private static class Proxy implements android.view.accessibility.IAccessibilityManagerClient {
        	.......
    	}    
     }
    
  public void setState(int stateFlags) throws android.os.RemoteException;
  public void notifyServicesStateChanged() throws android.os.RemoteException;
  public void setRelevantEventTypes(int eventTypes) throws android.os.RemoteException;
}
```

### AccessibilityManagerService#addClient

现在回过头来看AMS中的`addClient()`中具体做了什么事?

com.android.server.accessibility.AccessibilityManagerService#addClient

```java
    @Override
    public long addClient(IAccessibilityManagerClient callback, int userId) {
        synchronized (mLock) {
            final int resolvedUserId = mSecurityPolicy
                    .resolveCallingUserIdEnforcingPermissionsLocked(userId);
            UserState userState = getUserStateLocked(resolvedUserId);
			// 对每个请求与AccessibilityManagerService通信的客户端创建Client对象
            Client client = new Client(callback, Binder.getCallingUid(), userState);
            if (mSecurityPolicy.isCallerInteractingAcrossUsers(userId)) {
                mGlobalClients.register(callback, client);
                return IntPair.of(
                        userState.getClientState(),
                        client.mLastSentRelevantEventTypes);
            } else {
                userState.mUserClients.register(callback, client);
                return IntPair.of(
                        (resolvedUserId == mCurrentUserId) ? userState.getClientState() : 0,
                        client.mLastSentRelevantEventTypes);
            }
        }
    }
```

 对于每个客户端而言,即从AccessibilityManager传来的mClient对象,也就是该方法的callback参数,AccessibilityManagerService会将其封装为Client.

![image-20190113184139607](https://ws1.sinaimg.cn/large/006tNc79ly1fz558yimelj310s0bct9o.jpg)

### AccessibilityManager#sendAccessibilityEvent

AccessibilityManager发送事件到远程服务AccessibilityManagerService,

android.view.accessibility.AccessibilityManager#sendAccessibilityEvent

```java
    public void sendAccessibilityEvent(AccessibilityEvent event) {
        final IAccessibilityManager service;
        final int userId;
        synchronized (mLock) {
            // 1.首先获取到AccessibilityManagerService的本地代理对象
            service = getServiceLocked();
            if (service == null) {
                return;
            }
            // 2.检查Accessibility是否启用
            if (!mIsEnabled) {
                Looper myLooper = Looper.myLooper();
                if (myLooper == Looper.getMainLooper()) {
                    throw new IllegalStateException(
                            "Accessibility off. Did you forget to check that?");
                } else {
                    return;
                }
            }
            if ((event.getEventType() & mRelevantEventTypes) == 0) {
                return;
            }
            userId = mUserId;
        }
        try {
            // 3.更新当前事件的时间
            event.setEventTime(SystemClock.uptimeMillis());
            long identityToken = Binder.clearCallingIdentity();
            // 4.最终调用了AccessibilityManagerService服务的
            // sendAccessibilityEvent()方法将事件发送到AMS
            service.sendAccessibilityEvent(event, userId);
            Binder.restoreCallingIdentity(identityToken);
        } catch (RemoteException re) {
            Log.e(LOG_TAG, "Error during sending " + event + " ", re);
        } finally {
            //释放event对象,使其重新加入对象池以便重复利用
            event.recycle();
        }
    }
```

在上述代码中,首先通过`getServiceLocked()`获取AccessibilityManagerService在本地的代理对象,即IAccessibilityManager的实例,其实现如下:

```java
    private IAccessibilityManager getServiceLocked() {
        if (mService == null) {
            tryConnectToServiceLocked(null);
        }
        return mService;
    }
```

在该方法中最终还是借助`tryConnectToServiceLocked()`来连接AccessibilityManagerService.该方法返回的mService即之前IAccessibilityManager类型实例,也就是AMS在本地的代理对象,其真正的操作在`AccessibilityManagerService.sendAccessibilityEvent()`中,直接来看其实现:

```java
public class AccessibilityManagerService extends IAccessibilityManager.Stub {
    	
    ...
    
    @Override
    public void sendAccessibilityEvent(AccessibilityEvent event, int userId) {
        boolean dispatchEvent = false;

        synchronized (mLock) {
            if (event.getWindowId() ==
                AccessibilityWindowInfo.PICTURE_IN_PICTURE_ACTION_REPLACER_WINDOW_ID) {
                // The replacer window isn't shown to services. Move its events into the pip.
                AccessibilityWindowInfo pip = mSecurityPolicy.getPictureInPictureWindow();
                if (pip != null) {
                    int pipId = pip.getId();
                    event.setWindowId(pipId);
                }
            }

            final int resolvedUserId = mSecurityPolicy
                    .resolveCallingUserIdEnforcingPermissionsLocked(userId);
            // This method does nothing for a background user.
            if (resolvedUserId == mCurrentUserId) {
                if (mSecurityPolicy.canDispatchAccessibilityEventLocked(event)) {
                    mSecurityPolicy.updateActiveAndAccessibilityFocusedWindowLocked(
                            event.getWindowId(), event.getSourceNodeId(),
                            event.getEventType(), event.getAction());
                    mSecurityPolicy.updateEventSourceLocked(event);
                    dispatchEvent = true;
                }
                if (mHasInputFilter && mInputFilter != null) {
                    mMainHandler.obtainMessage(
                            MainHandler.MSG_SEND_ACCESSIBILITY_EVENT_TO_INPUT_FILTER,
                            AccessibilityEvent.obtain(event)).sendToTarget();
                }
            }
        }
		// 1.需要分发事件
        if (dispatchEvent) {
            if (event.getEventType() == AccessibilityEvent.TYPE_WINDOW_STATE_CHANGED
                    && mWindowsForAccessibilityCallback != null) {
                WindowManagerInternal wm = LocalServices.getService(WindowManagerInternal.class);
                wm.computeWindowsForAccessibility();
            }
            synchronized (mLock) {
                // 2.通知对应的AccessibilityServices
                notifyAccessibilityServicesDelayedLocked(event, false);
                notifyAccessibilityServicesDelayedLocked(event, true);
            }
        }

        if (OWN_PROCESS_ID != Binder.getCallingPid()) {
            event.recycle();
        }
    }
    
    ...
}
```

dispatchEvent为true表示需要向AccessibilityService分发事件,当需要进行事件分发时,最终会调用`notifyAccessibilityServicesDelayedLocked()`:

```java
 private void notifyAccessibilityServicesDelayedLocked(AccessibilityEvent event,
            boolean isDefault) {
        try {
            UserState state = getCurrentUserStateLocked();
            // mBoundServices保存了所有AccessibilityService与AMS的连接
            for (int i = 0, count = state.mBoundServices.size(); i < count; i++) {
                Service service = state.mBoundServices.get(i);

                if (service.mIsDefault == isDefault) {
                    if (doesServiceWantEventLocked(service, event)) {
                        service.notifyAccessibilityEvent(event, true);
                    } else if (service.mUsesAccessibilityCache
                            && (AccessibilityCache.CACHE_CRITICAL_EVENTS_MASK
                                & event.getEventType()) != 0) {
                        service.notifyAccessibilityEvent(event, false);
                    }
                }
            }
        } catch (IndexOutOfBoundsException oobe) {
           
        }
    }
```

上述方法中的mBoundServices是AMS中的成员变量,定义如下:

```java
 public final ArrayList<AccessibilityServiceConnection> mBoundServices = new ArrayList<>();
```

其中AccessibilityServiceConnection代表已经注册到AMS的AccessibilityService,也就是说mBoundServices保存了所有AccessibilityService与AMS的连接.比如我们自定义了一个辅助服务WXAccessibilityService,当该服务被启用时,AMS就会与该服务进行绑定,并生成对应的AccessibilityServiceConnection保存在mBoundServices中.(AccessibilityServiceConnection继承自AbstractAccessibilityServiceConnection)

在`notifyAccessibilityServicesDelayedLocked()`方法中会遍历所有的AccessibilityServiceConnection对象,并调用其`notifyAccessibilityEvent()`来通知事件发生变化:

`com.android.server.accessibility.AbstractAccessibilityServiceConnection#notifyAccessibilityEvent`:

```java
    public void notifyAccessibilityEvent(AccessibilityEvent event) {
        synchronized (mLock) {
            final int eventType = event.getEventType();

            final boolean serviceWantsEvent = wantsEventLocked(event);
            final boolean requiredForCacheConsistency = mUsesAccessibilityCache
                    && ((AccessibilityCache.CACHE_CRITICAL_EVENTS_MASK & eventType) != 0);
            if (!serviceWantsEvent && !requiredForCacheConsistency) {
                return;
            }

            AccessibilityEvent newEvent = AccessibilityEvent.obtain(event);
            // 1.根据event创建Message对象,后续将通过Handler进行处理
            Message message;
            if ((mNotificationTimeout > 0)
                    && (eventType != AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED)) {
                // Allow at most one pending event
                final AccessibilityEvent oldEvent = mPendingEvents.get(eventType);
                mPendingEvents.put(eventType, newEvent);
                if (oldEvent != null) {
                    mEventDispatchHandler.removeMessages(eventType);
                    oldEvent.recycle();
                }
                message = mEventDispatchHandler.obtainMessage(eventType);
            } else {
                // Send all messages, bypassing mPendingEvents
                message = mEventDispatchHandler.obtainMessage(eventType, newEvent);
            }
            message.arg1 = serviceWantsEvent ? 1 : 0;
			// 2.发送Message,最终将其切换到主线程中进行处理
            mEventDispatchHandler.sendMessageDelayed(message, mNotificationTimeout);
        }
    }
```

由于此处分发过程发生在Binder线程池,需要借助Handler将其切换到主线程中,即mEventDispatchHandler,简单来看该Handler的创建以及对Message的处理过程:

```java
abstract class AbstractAccessibilityServiceConnection extends IAccessibilityServiceConnection.Stub
        implements ServiceConnection, IBinder.DeathRecipient, KeyEventDispatcher.KeyEventFilter,
        FingerprintGestureDispatcher.FingerprintGestureClient {
    
    ......
        
    public AbstractAccessibilityServiceConnection(Context context, ComponentName componentName,
            AccessibilityServiceInfo accessibilityServiceInfo, int id, Handler mainHandler,
            Object lock, SecurityPolicy securityPolicy, SystemSupport systemSupport,
            WindowManagerInternal windowManagerInternal,
            GlobalActionPerformer globalActionPerfomer) {
  		......
        // 1.创建Handler,用于切换到主线程    
        mEventDispatchHandler = new Handler(mainHandler.getLooper()) {
            @Override
            public void handleMessage(Message message) {
                final int eventType =  message.what;
                AccessibilityEvent event = (AccessibilityEvent) message.obj;
                boolean serviceWantsEvent = message.arg1 != 0;
                // 2.事件处理
                notifyAccessibilityEventInternal(eventType, event, serviceWantsEvent);
            }
        };
       .......
    }  
      
   ......         
}
```

在handleMessage()中继续调用`notifyAccessibilityEventInternal()`来将事件分发给具体的AccessibilityService.

```java
  private void notifyAccessibilityEventInternal(
                int eventType,
                AccessibilityEvent event,
                boolean serviceWantsEvent) {
            IAccessibilityServiceClient listener;

            synchronized (mLock) {
                listener = mServiceInterface;

                if (listener == null) {
                    return;
                }
				// 1.根据eventType取出事件
                if (event == null) {
                    event = mPendingEvents.get(eventType);
                    if (event == null) {
                        return;
                    }
                    mPendingEvents.remove(eventType);
                }
                // 2.进行权限检查,检查服务是否允许检索窗口内容
                if (mSecurityPolicy.canRetrieveWindowContentLocked(this)) {
                    event.setConnectionId(mId);
                } else {
                    event.setSource((View) null);
                }
                event.setSealed(true);
            }

            try {
                // 3.回调服务服务中的onAccessibityEvent()方法
                listener.onAccessibilityEvent(event, serviceWantsEvent);
                ...
            } catch (RemoteException re) {
                Slog.e(LOG_TAG, "Error during sending " + event + " to " + listener, re);
            } finally {
                event.recycle();
            }
        }
```

## 小结

当应用界面产生AccessibilityEvent需要被发送给辅助服务时,最终会调用ViewRootImpl中的`requestSendAccessibilityEvent()`,在该方法中最终通过AccessibilityManager跨进程调用AMS的`sendAccessibilityEvent()`方法将AccessibilityEvent传递到AMS中,AMS接受到该事件后遍历所有已经注册到系统的AccessibilityService,然后再次以跨进程的方式将AccessibilityEvent发送至自定义AccessibilityService所在进程.



# 总结

相比于ActivityManagerService或者PackageManagerService而言,AccessibilityServiceManager总体设计和架构都比较简单,更能加深对Binder使用的理解,同时通过简单的源码梳理,能帮助大家更有效的学习和使用辅助服务.此文拟稿与18年初,终结于19年初,一方面是向老东家360致敬,另一方面也作为新起点的标记.









