---
layout: single
related: false
title:  Android 屏幕旋转流程
date:   2020-06-01 23:52:00
categories: android
tags: display
toc: true
---

> Android支持横屏和竖屏，用户可以选择锁定(rotation lock)也可以选择让传感器来自动转屏。而转屏时为了使用户体验更流畅，会对屏幕截屏，然后使用截屏的图来做转屏动画，直到转屏动作结束。

<!--more-->

# 1. 调试方法

## 1.1. 打开debug log开关

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerDebugConfig.java
    static final String TAG_WM = "WindowManager";
    static final boolean DEBUG_ORIENTATION = false;  //true
    static final boolean DEBUG_APP_ORIENTATION = false;  //true
```

`adb logcat -v threadtime|grep -Ei "rotation|ActivityTaskManager|WindowOrientationListener"`

## 1.2. Settings设置开启/关闭自动旋转屏幕

是否要自动转屏是在Setting中设置的。为了监听Setting中的改动，系统启动时，`PhoneWindowManager`的init()函数中创建了SettingsObserver对象。

它的observe()方法会监听`Settings.System.USER_ROTATION`的值（Android Q中此处没有这个property了）。

```cpp
  public void init(Context context, IWindowManager windowManager,
            WindowManagerFuncs windowManagerFuncs) {
                ...
                mSettingsObserver = new SettingsObserver(mHandler);
                mSettingsObserver.observe();
        ...
```

当用户在Setting中设置自动转屏后，会触发以下流程：

1. `public boolean onPreferenceTreeClick(Preference preference)`：packages/apps/Settings/src/com/android/settings/accessibility/AccessibilitySettings.java
2. `handleLockScreenRotationPreferenceClick()`：被调用
3. `setRotationLockForAccessibility(Context context, final boolean enabled)`

```cpp
//frameworks/base/core/java/com/android/internal/view/RotationPolicy.java
    public static void setRotationLockForAccessibility(Context context, final boolean enabled) {
        Settings.System.putIntForUser(context.getContentResolver(),
                Settings.System.HIDE_ROTATION_LOCK_TOGGLE_FOR_ACCESSIBILITY, enabled ? 1 : 0,
                        UserHandle.USER_CURRENT);

        setRotationLock(enabled, NATURAL_ROTATION);
    }
```

4. `setRotationLockForAccessibility`

```cpp
//frameworks/base/core/java/com/android/internal/view/RotationPolicy.java
    public static void setRotationLockForAccessibility(Context context, final boolean enabled) {
        Settings.System.putIntForUser(context.getContentResolver(),
                Settings.System.HIDE_ROTATION_LOCK_TOGGLE_FOR_ACCESSIBILITY, enabled ? 1 : 0,
                        UserHandle.USER_CURRENT);

        setRotationLock(enabled, NATURAL_ROTATION);
    }
```

5. `setRotationLock(final boolean enabled, final int rotation)`调用`wm.freezeRotation`或者`wm.thawRotation`

```cpp
//frameworks/base/core/java/com/android/internal/view/RotationPolicy.java
    private static void setRotationLock(final boolean enabled, final int rotation) {
        AsyncTask.execute(new Runnable() {
            @Override
            public void run() {
                try {
                    IWindowManager wm = WindowManagerGlobal.getWindowManagerService();
                    if (enabled) {
                        wm.freezeRotation(rotation);
                    } else {
                        wm.thawRotation();
                    }
                } catch (RemoteException exc) {
                    Log.w(TAG, "Unable to save auto-rotate setting");
                }
            }
        });
    }
```

6. `thawRotation()`，此处在Android Q中有变化。

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public void thawRotation() {
        if (!checkCallingPermission(android.Manifest.permission.SET_ORIENTATION,
                "thawRotation()")) {
            throw new SecurityException("Requires SET_ORIENTATION permission");
        }

        if (DEBUG_ORIENTATION) Slog.v(TAG_WM, "thawRotation: mRotation="
                + getDefaultDisplayRotation());

        long origId = Binder.clearCallingIdentity();
        try {
            mPolicy.setUserRotationMode(WindowManagerPolicy.USER_ROTATION_FREE,
                    777); // rot not used   free
        } finally {
            Binder.restoreCallingIdentity(origId);
        }

        updateRotationUnchecked(false, false);  //查看下面第十步分析
    }
```

而`freezeRotation函数`，只是调用PhoneWindowManager的setUserRotationMode的参数不一样，这里是Locked，而thawRotation传下去的参数是free。

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
   @Override
    public void freezeRotation(int rotation) {
        // TODO(multi-display): Track which display is rotated.
        if (!checkCallingPermission(android.Manifest.permission.SET_ORIENTATION,
                "freezeRotation()")) {
            throw new SecurityException("Requires SET_ORIENTATION permission");
        }
        if (rotation < -1 || rotation > Surface.ROTATION_270) {
            throw new IllegalArgumentException("Rotation argument must be -1 or a valid "
                    + "rotation constant.");
        }

        final int defaultDisplayRotation = getDefaultDisplayRotation();
        if (DEBUG_ORIENTATION) Slog.v(TAG_WM, "freezeRotation: mRotation="
                + defaultDisplayRotation);

        long origId = Binder.clearCallingIdentity();
        try {
            mPolicy.setUserRotationMode(WindowManagerPolicy.USER_ROTATION_LOCKED,
                    rotation == -1 ? defaultDisplayRotation : rotation);    //lock
        } finally {
            Binder.restoreCallingIdentity(origId);
        }

        updateRotationUnchecked(false, false);
    }
```



7. `setUserRotationMode(int mode, int rot)`：此处是设置property（即Settings数据库），然后会触发到上面初始化的`mSettingsObserver`对象的`onChange`函数。

触发监听`SettingsObserver.onChange()`， 其中主要调用了updateSettings()和updateRotation()两个函数。

简单地说，主要的工作是根据需要监听传感器数据，据此判断是否要转屏。如果需要就是**对configuration的各种更新**。过程中会冻结屏幕，同时截屏并以此作为转屏动画。另外还需要将新configuration传给AMS，广播该事件给需要的模块，同时App也会被调度来响应变更。

```cpp
//frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
    // User rotation: to be used when all else fails in assigning an orientation to the device
    @Override
    public void setUserRotationMode(int mode, int rot) {
        ContentResolver res = mContext.getContentResolver();

        // mUserRotationMode and mUserRotation will be assigned by the content observer
        if (mode == WindowManagerPolicy.USER_ROTATION_LOCKED) {
            Settings.System.putIntForUser(res,
                    Settings.System.USER_ROTATION,
                    rot,
                    UserHandle.USER_CURRENT);
            Settings.System.putIntForUser(res,
                    Settings.System.ACCELEROMETER_ROTATION,
                    0,
                    UserHandle.USER_CURRENT);
        } else {
            Settings.System.putIntForUser(res,
                    Settings.System.ACCELEROMETER_ROTATION,
                    1,
                    UserHandle.USER_CURRENT);
        }
    }
......
       @Override public void onChange(boolean selfChange) {
            updateSettings();    //以下8,9步骤
            updateRotation(false);  //以下10-步骤
        }
......
public void updateSettings() {
        ContentResolver resolver = mContext.getContentResolver();
        boolean updateRotation = false;
        synchronized (mLock) {
            ...
            int userRotationMode = Settings.System.getIntForUser(resolver,
                    Settings.System.ACCELEROMETER_ROTATION, 0, UserHandle.USER_CURRENT) != 0 ?
                            WindowManagerPolicy.USER_ROTATION_FREE :
                                    WindowManagerPolicy.USER_ROTATION_LOCKED;
            if (mUserRotationMode != userRotationMode) {
                mUserRotationMode = userRotationMode;
                updateRotation = true;
                updateOrientationListenerLp();  //传感器相关操作
            }
```

第一个函数`updateSettings()`如它的名字主要更新设置信息。

如果UserRotation（朝向信息，如Surface.ROTATION_0）和UserRotationMode（USER_ROTATION_FREE vs. USER_ROTATION_LOCKED）有更新，就设置标记updateRotation为true，表示接下去需要更新rotation相关信息。

此外，如果UserRotationMode的配置有变，由于需要传感器信息的配合，还需调用updateOrientationListenerLp()来设置或取消监听传感器。

这里假设设置为自动旋转，那么PhoneWindowManager会通过MyOrientationListener来监听传感器信息。MyOrientationListener是WindowOrientationListener的继承类。它的enable()函数中调用SensorManager提供的registerListener()接口来设置Sensor信息的listener。


8. `updateOrientationListenerLp()`：作用是enable和disable传感器

其中的`mOrientationListener.enable`和`mOrientationListener.disable`是注册传感器回调和去除传感器回调。

```cpp 
void updateOrientationListenerLp() {
        if (!mOrientationListener.canDetectOrientation()) {
            // If sensor is turned off or nonexistent for some reason
            return;
        }

        if (localLOGV) Slog.v(TAG, "mScreenOnEarly=" + mScreenOnEarly
                + ", mAwake=" + mAwake + ", mCurrentAppOrientation=" + mCurrentAppOrientation
                + ", mOrientationSensorEnabled=" + mOrientationSensorEnabled
                + ", mKeyguardDrawComplete=" + mKeyguardDrawComplete
                + ", mWindowManagerDrawComplete=" + mWindowManagerDrawComplete);

        boolean disable = true;
        if (mScreenOnEarly && mAwake && ((mKeyguardDrawComplete && mWindowManagerDrawComplete))) {
            if (needSensorRunningLp()) {
                disable = false;
                //enable listener if not already enabled 启动传感器监听！！
                if (!mOrientationSensorEnabled) {
                    mOrientationListener.enable(true /* clearCurrentRotation */);
                    if(localLOGV) Slog.v(TAG, "Enabling listeners");
                    mOrientationSensorEnabled = true;
                }
            }
        }
        //check if sensors need to be disabled
        if (disable && mOrientationSensorEnabled) {
            mOrientationListener.disable();   //关闭传感器
            if(localLOGV) Slog.v(TAG, "Disabling listeners");
            mOrientationSensorEnabled = false;
        }
    }
```

9. mOrientationListener是MyOrientationListener对象，而MyOrientationListener类继承父类WindowOrientationListener，从而会调用父类的`enable`函数。

该函数中会调用registerListener向SensorManager注册一个监听。

```cpp
//frameworks/base/services/core/java/com/android/server/policy/WindowOrientationListener.java
 public void enable(boolean clearCurrentRotation) {
        synchronized (mLock) {
            if (mSensor == null) {
                Slog.w(TAG, "Cannot detect sensors. Not enabled");
                return;
            }
            if (mEnabled) {
                return;
            }
            if (LOG) {
                Slog.d(TAG, "WindowOrientationListener enabled clearCurrentRotation="
                        + clearCurrentRotation);
            }
            mOrientationJudge.resetLocked(clearCurrentRotation);
            if (mSensor.getType() == Sensor.TYPE_ACCELEROMETER) {
                mSensorManager.registerListener(
                        mOrientationJudge, mSensor, mRate, DEFAULT_BATCH_LATENCY, mHandler);   //mOrientationJudge的回调
            } else {
                mSensorManager.registerListener(mOrientationJudge, mSensor, mRate, mHandler);
            }
            mEnabled = true;
        }
    }
```

registerListener()的具体实现在frameworks/base/core/java/android/hardware/SensorManager.java中。

然后调用SystemSensorManager.java的registerListenerImpl()，其中会创建`SensorEventQueue`对象（基类为BaseEventQueue），它是传感器事件的队列，记录需要监听哪些传感器信息。

`SensorEventQueue queue = mSensorListeners.get(listener);`

同时它也负责与SensorService的连接和通信，可以说是SensorEventListener与SensorService间的桥梁。

SensorEventListener和SensorEventQueue之间是1:1的关系，它们的映射关系保存在成员mSensorListeners中。如果这里注册的SensorEventListener还没有相应的SensorEventQueue，则新建一个，然后通过addSensor()方法将要关注的传感器进行注册。这个过程中addSensor()调用了enableSensor()，它最终是通过SensorService的enableDisable()方法来完成注册工作的。

这样，SensorService就开始监听该Sensor，当底层有传感器数据来时，SensorService主线程中会调用相应SensorEventConnection的sendEvents()将之发给对应的Client。

前面初始化SensorEventQueue时会创建Receiver，它是一个Looper的回调对象，在Client端收到从SensorService来的数据后被回调。

当有数据收到时Receiver的handleEvent()被调用，继而通过JNI调用到SystemSensorManager::dispatchSensorEvent()。

**接着就调到了`WindowOrientationListener的onSensorChanged()函数`。该函数计算是否需要转屏。如果需要转屏，将计算结果传给onProposedRotationChanged()。**

比如以下函数的日志打印，在旋转手机，传感器会触发屏幕旋转打印这部分log：

```java frameworks/base/services/core/java/com/android/server/policy/WindowOrientationListener.java
       @Override
        public void onSensorChanged(SensorEvent event) {
            ......
            // Tell the listener.
            if (proposedRotation != oldProposedRotation && proposedRotation >= 0) {
                if (LOG) {
                    Slog.v(TAG, "Proposed rotation changed!  proposedRotation=" + proposedRotation
                            + ", oldProposedRotation=" + oldProposedRotation);
                }
                onProposedRotationChanged(proposedRotation);
            }
        }

```

***

10. 另一处`updateRotation(false)`函数会调用到WMS.java，然后调用到`updateRotationUnchecked`函数。

最终在该函数中调用`rotationChanged = displayContent.updateRotationUnchecked();`

***

# 2. 屏幕旋转

假设现在用户转了屏幕，期望转屏事件发生。如上面第九步的代码，`onProposedRotationChanged()`被调用。

最后就调用其run函数，run函数先会提升性能（cpu频率），然后调用了updateRotation，这个函数一样就到WMS的updateRotationUnchecked函数。

```java
//frameworks/base/services/core/java/com/android/server/policy/PhoneWindowManager.java
class MyOrientationListener extends WindowOrientationListener {
...
        private class UpdateRunnable implements Runnable {
            private final int mRotation;
            UpdateRunnable(int rotation) {
                mRotation = rotation;
            }

            @Override
            public void run() {
                // send interaction hint to improve redraw performance
                mPowerManagerInternal.powerHint(PowerHint.INTERACTION, 0);
                if (isRotationChoicePossible(mCurrentAppOrientation)) {
                    final boolean isValid = isValidRotationChoice(mCurrentAppOrientation,
                            mRotation);
                    sendProposedRotationChangeToStatusBarInternal(mRotation, isValid);
                } else {
                    updateRotation(false);
                }
            }
        }

        @Override
        public void onProposedRotationChanged(int rotation) {
            if (localLOGV) Slog.v(TAG, "onProposedRotationChanged, rotation=" + rotation);
            Runnable r = mRunnableCache.get(rotation, null);
            if (r == null){
                r = new UpdateRunnable(rotation);
                mRunnableCache.put(rotation, r);
            }
            mHandler.post(r);  //发送了一个消息
        }
    }
```

updateRotation()中主要是执行两个函数：updateRotationUnchecked()（`displayContent.updateRotationUnchecked()`）和sendNewConfiguration()。前者执行转屏动作，包含转屏动画
等。后者使AMS获取当前新的configuration，并且广播该事件给所有相应的listener。

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
 @Override
 public void updateRotation(boolean alwaysSendConfiguration, boolean forceRelayout) {
        updateRotationUnchecked(alwaysSendConfiguration, forceRelayout);
}

 //上面settings中设置自动旋转屏幕也会调用到（thawRotation和freezeRotation函数）
 private void updateRotationUnchecked(boolean alwaysSendConfiguration, boolean forceRelayout) {
        if(DEBUG_ORIENTATION) Slog.v(TAG_WM, "updateRotationUnchecked:"
                + " alwaysSendConfiguration=" + alwaysSendConfiguration
                + " forceRelayout=" + forceRelayout);
                ...
        try {
            // TODO(multi-display): Update rotation for different displays separately.
            final boolean rotationChanged;
            final int displayId;
            synchronized (mWindowMap) {
                final DisplayContent displayContent = getDefaultDisplayContentLocked();
                Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "updateRotation: display");
                //Step 1
                rotationChanged = displayContent.updateRotationUnchecked();
                ...
            }

            if (rotationChanged || alwaysSendConfiguration) {
                Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "updateRotation: sendNewConfiguration");
                //Step 2
                sendNewConfiguration(displayId);
                Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
    }
```

**Note:**: 其它途径可能会触发转屏，比如应用请求转屏。例如需要横屏的游戏（通过frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java的`updateOrientationFromAppTokensLocked()`方法）。

***

## 2.1. updateRotationUnchecked函数

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
boolean updateRotationUnchecked(boolean forceUpdate) {
        final int oldRotation = mRotation;
        final int lastOrientation = mLastOrientation;
        final boolean oldAltOrientation = mAltOrientation;
        //先调用PhoneWindowManager的rotationForOrientationLw函数来获取rotation，然后与之前的mRotation对比是否有变化
        //没有变化直接返回false。有变化将mRotation重新赋值
        //函数rotationForOrientationLw作用：获取sensor的rotation，然后计算返回我们需要的rotation
        final int rotation = mService.mPolicy.rotationForOrientationLw(lastOrientation, oldRotation,
                isDefaultDisplay);
        if (DEBUG_ORIENTATION) Slog.v(TAG_WM, "Computed rotation=" + rotation + " for display id="
                + mDisplayId + " based on lastOrientation=" + lastOrientation
                + " and oldRotation=" + oldRotation);
...
        if (oldRotation == rotation && oldAltOrientation == altOrientation) {  //没有变化
            // No change.
            return false;
        }
        ...
        mRotation = rotation;   //有变化则赋值
        mAltOrientation = altOrientation; 
        ...
        updateDisplayAndOrientation(getConfiguration().uiMode);
        ...
        mService.mDisplayManagerInternal.performTraversal(getPendingTransaction());
        ...
}
```

### 2.1.1. updateDisplayAndOrientation函数

还会调用到updateDisplayAndOrientation函数，会把`各种数据更新下放到DisplayInfo中`，最后调用了DisplayManagerService的`setDisplayInfoOverrideFromWindowManager`函数。

```java
//frameworks/base/services/core/java/com/android/server/wm/DisplayContent.java
    private DisplayInfo updateDisplayAndOrientation(int uiMode) {
        // Use the effective "visual" dimensions based on current rotation
        final boolean rotated = (mRotation == ROTATION_90 || mRotation == ROTATION_270);
        final int realdw = rotated ? mBaseDisplayHeight : mBaseDisplayWidth;
        final int realdh = rotated ? mBaseDisplayWidth : mBaseDisplayHeight;
        int dw = realdw;
        int dh = realdh;
        ...
                final int appWidth = mService.mPolicy.getNonDecorDisplayWidth(dw, dh, mRotation, uiMode,
                mDisplayId, displayCutout);
        final int appHeight = mService.mPolicy.getNonDecorDisplayHeight(dw, dh, mRotation, uiMode,
                mDisplayId, displayCutout);
        mDisplayInfo.rotation = mRotation;
        mDisplayInfo.logicalWidth = dw;
        mDisplayInfo.logicalHeight = dh;
        mDisplayInfo.logicalDensityDpi = mBaseDisplayDensity;
        mDisplayInfo.appWidth = appWidth;
        mDisplayInfo.appHeight = appHeight;
        if (isDefaultDisplay) {
            mDisplayInfo.getLogicalMetrics(mRealDisplayMetrics,
                    CompatibilityInfo.DEFAULT_COMPATIBILITY_INFO, null);
        }
        mDisplayInfo.displayCutout = displayCutout.isEmpty() ? null : displayCutout;
        mDisplayInfo.getAppMetrics(mDisplayMetrics);
        if (mDisplayScalingDisabled) {
            mDisplayInfo.flags |= Display.FLAG_SCALING_DISABLED;
        } else {
            mDisplayInfo.flags &= ~Display.FLAG_SCALING_DISABLED;
        }
        ...
        mService.mDisplayManagerInternal.setDisplayInfoOverrideFromWindowManager(mDisplayId,
            overrideDisplayInfo);
        ...    
    }
```

setDisplayInfoOverrideFromWindowManager会调用setDisplayInfoOverrideFromWindowManagerInternal，然后调用`display.setDisplayInfoOverrideFromWindowManagerLocked(info)`函数，最后到LogicalDisplay的setDisplayInfoOverrideFromWindowManagerLocked函数中，把DisplayInfo数据放到了mOverrideDisplayInfo中。

```java
//frameworks/base/services/core/java/com/android/server/display/LogicalDisplay.java
   public boolean setDisplayInfoOverrideFromWindowManagerLocked(DisplayInfo info) {
        if (info != null) {
            if (mOverrideDisplayInfo == null) {
                mOverrideDisplayInfo = new DisplayInfo(info);
                mInfo = null;
                return true;
            }
            if (!mOverrideDisplayInfo.equals(info)) {
                mOverrideDisplayInfo.copyFrom(info); //拷贝到mOverrideDisplayInfo中
                mInfo = null;
                return true;
            }
        } else if (mOverrideDisplayInfo != null) {
            mOverrideDisplayInfo = null;
            mInfo = null;
            return true;
        }
        return false;
    }
```

### 2.1.2. performTraversal处理显示Layer的大小宽高尺寸

调用到DisplayManagerService.java中，然后调用performTraversalInternal函数。

```java
//frameworks/base/services/core/java/com/android/server/display/DisplayManagerService.java
    void performTraversalInternal(SurfaceControl.Transaction t) {
        synchronized (mSyncRoot) {
            if (!mPendingTraversal) {
                return;
            }
            mPendingTraversal = false;
            //遍历所有的Device
            performTraversalLocked(t);
        }

        // List is self-synchronized copy-on-write.
        for (DisplayTransactionListener listener : mDisplayTransactionListeners) {
            listener.onDisplayTransaction();
        }
    }

  private void performTraversalLocked(SurfaceControl.Transaction t) {
        // Clear all viewports before configuring displays so that we can keep
        // track of which ones we have configured.
        clearViewportsLocked();

        // 遍历所有的Device
        final int count = mDisplayDevices.size();
        for (int i = 0; i < count; i++) {
            DisplayDevice device = mDisplayDevices.get(i);
            //step 1：找到那个LogicalDisplay 然后调用其configureDisplayInTransactionLocked函数（看上面的将参数赋值到mOverrideDisplayInfo中）
            configureDisplayLocked(t, device);
            //step 2:调用了各个Device的performTraversalInTransactionLocked，而普通的Device的为空
            device.performTraversalLocked(t);
        }

        // Tell the input system about these new viewports.
        if (mInputManagerInternal != null) {
            mHandler.sendEmptyMessage(MSG_UPDATE_VIEWPORT);
        }
    }

    private void configureDisplayLocked(SurfaceControl.Transaction t, DisplayDevice device) {
        ...
        //设置长宽，旋转角度等
        display.configureDisplayLocked(t, device, info.state == Display.STATE_OFF);
        ...
    }
```

**configureDisplayLocked函数的这部分代码就是设置layer的显示大小，例如viewport，通过Dump SF可以查看layer。**

```java
//frameworks/base/services/core/java/com/android/server/display/LogicalDisplay.java
 //这部分代码就是设置layer的显示大小，例如viewport，通过Dump SF可以查看layer
 public void configureDisplayLocked(SurfaceControl.Transaction t,
            DisplayDevice device,
            boolean isBlanked) {
                ...
        //step 1. 获取mInfo的数据，而mOverrideDisplayInfo如有数据就要copy到mInfo中去
        final DisplayInfo displayInfo = getDisplayInfoLocked(); 
        final DisplayDeviceInfo displayDeviceInfo = device.getDisplayDeviceInfoLocked();

        // Set the viewport.
        // This is the area of the logical display that we intend to show on the
        // display device.  For now, it is always the full size of the logical display.
        mTempLayerStackRect.set(0, 0, displayInfo.logicalWidth, displayInfo.logicalHeight);
        ...
        mTempDisplayRect.left += mDisplayOffsetX;
        mTempDisplayRect.right += mDisplayOffsetX;
        mTempDisplayRect.top += mDisplayOffsetY;
        mTempDisplayRect.bottom += mDisplayOffsetY;
        //step 2
        device.setProjectionLocked(t, orientation, mTempLayerStackRect, mTempDisplayRect);
    }

 public DisplayInfo getDisplayInfoLocked() {
        if (mInfo == null) {
            mInfo = new DisplayInfo();
            mInfo.copyFrom(mBaseDisplayInfo);
            if (mOverrideDisplayInfo != null) {
                mInfo.appWidth = mOverrideDisplayInfo.appWidth;
                mInfo.appHeight = mOverrideDisplayInfo.appHeight;
                mInfo.smallestNominalAppWidth = mOverrideDisplayInfo.smallestNominalAppWidth;
                mInfo.smallestNominalAppHeight = mOverrideDisplayInfo.smallestNominalAppHeight;
                mInfo.largestNominalAppWidth = mOverrideDisplayInfo.largestNominalAppWidth;
                mInfo.largestNominalAppHeight = mOverrideDisplayInfo.largestNominalAppHeight;
                mInfo.logicalWidth = mOverrideDisplayInfo.logicalWidth;
                mInfo.logicalHeight = mOverrideDisplayInfo.logicalHeight;
                mInfo.overscanLeft = mOverrideDisplayInfo.overscanLeft;
                mInfo.overscanTop = mOverrideDisplayInfo.overscanTop;
                mInfo.overscanRight = mOverrideDisplayInfo.overscanRight;
                mInfo.overscanBottom = mOverrideDisplayInfo.overscanBottom;
                mInfo.rotation = mOverrideDisplayInfo.rotation;
                mInfo.displayCutout = mOverrideDisplayInfo.displayCutout;
                mInfo.logicalDensityDpi = mOverrideDisplayInfo.logicalDensityDpi;
                mInfo.physicalXDpi = mOverrideDisplayInfo.physicalXDpi;
                mInfo.physicalYDpi = mOverrideDisplayInfo.physicalYDpi;
            }
        }
        return mInfo;
    }
```

setProjectionLocked会调用SurfaceControl的SurfaceControl函数。然后在SurfaceControl中调用nativeSetDisplayProjection函数，通过JNI调用到Native层。

```java
//frameworks/base/services/core/java/com/android/server/display/DisplayDevice.java
  public final void setProjectionLocked(SurfaceControl.Transaction t, int orientation,
            Rect layerStackRect, Rect displayRect) {
...
            t.setDisplayProjection(mDisplayToken,
                    orientation, layerStackRect, displayRect);
        }
    }
```

此时Java层的`updateRotationUnchecked`函数分析完。

***

# 3. sendNewConfiguration函数

从上面的updateRotation()函数中看到，除了调用updateRotationUnchecked()（即`displayContent.updateRotationUnchecked()`），还会调用sendNewConfiguration()。

```java
//frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
void sendNewConfiguration(int displayId) {
        try {
            final boolean configUpdated = mActivityManager.updateDisplayOverrideConfiguration(
                    null /* values */, displayId);
...
            }
        } catch (RemoteException e) {
        }
    }
```

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java
@Override
    public boolean updateDisplayOverrideConfiguration(Configuration values, int displayId) {
        enforceCallingPermission(CHANGE_CONFIGURATION, "updateDisplayOverrideConfiguration()");

        synchronized (this) {
            ...
            if (values == null && mWindowManager != null) {
                // sentinel: fetch the current configuration from the window manager
                //Step 1 获取一些配置信息
                values = mWindowManager.computeNewConfiguration(displayId);
            }


            final long origId = Binder.clearCallingIdentity();
            try {
                if (values != null) {
                    Settings.System.clearConfiguration(values);
                }
                //Step 2
                updateDisplayOverrideConfigurationLocked(values, null /* starting */,
                        false /* deferResume */, displayId, mTmpUpdateConfigurationResult);
                return mTmpUpdateConfigurationResult.changes != 0;
            } finally {
                Binder.restoreCallingIdentity(origId);
            }
        }
    }

  private boolean updateDisplayOverrideConfigurationLocked(Configuration values,
            ActivityRecord starting, boolean deferResume, int displayId,
            UpdateConfigurationResult result) {
 ...
        try {
            if (values != null) {
                if (displayId == DEFAULT_DISPLAY) {
                    //Step 1:调用
                    changes = updateGlobalConfigurationLocked(values, false /* initLocale */,
                            false /* persistent */, UserHandle.USER_NULL /* userId */, deferResume);
                } else {
                    //
                    changes = performDisplayOverrideConfigUpdate(values, deferResume, displayId);
                }
            }
            //Step 2
            kept = ensureConfigAndVisibilityAfterUpdate(starting, changes);
        } finally {
            if (mWindowManager != null) {
                mWindowManager.continueSurfaceLayout();
            }
        }

        if (result != null) {
            result.changes = changes;
            result.activityRelaunched = !kept;
        }
        return kept;
    }

private int updateGlobalConfigurationLocked(@NonNull Configuration values, boolean initLocale,
            boolean persistent, int userId, boolean deferResume) {
        //把Configuration数据保存在mTempConfig
        mTempConfig.setTo(getGlobalConfiguration());
        final int changes = mTempConfig.updateFrom(values);
        if (changes == 0) {
            performDisplayOverrideConfigUpdate(values, deferResume, DEFAULT_DISPLAY);
            return 0;
        }
        //会发送ACTION_CONFIGURATION_CHANGED广播，然后获取当前最上面活动的Activity，调用ActivityStack的ensureActivityConfigurationLocked函数和ActivityStackSupervisor的ensureActivitiesVisibleLocked函数。
        Intent intent = new Intent(Intent.ACTION_CONFIGURATION_CHANGED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY | Intent.FLAG_RECEIVER_REPLACE_PENDING
                | Intent.FLAG_RECEIVER_FOREGROUND
                | Intent.FLAG_RECEIVER_VISIBLE_TO_INSTANT_APPS);
        broadcastIntentLocked(null, null, intent, null, null, 0, null, null, null,
                OP_NONE, null, false, false, MY_PID, SYSTEM_UID,
                UserHandle.USER_ALL);
......
    }

     private int performDisplayOverrideConfigUpdate(Configuration values, boolean deferResume,
            int displayId) {
        mTempConfig.setTo(mStackSupervisor.getDisplayOverrideConfiguration(displayId));
        //把Configuration数据保存在mTempConfig
        final int changes = mTempConfig.updateFrom(values);
        ...

    }

    private boolean ensureConfigAndVisibilityAfterUpdate(ActivityRecord starting, int changes) {
        boolean kept = true;
        final ActivityStack mainStack = mStackSupervisor.getFocusedStack();
        // mainStack is null during startup.
        if (mainStack != null) {
            if (changes != 0 && starting == null) {
                // If the configuration changed, and the caller is not already
                // in the process of starting an activity, then find the top
                // activity to check if its configuration needs to change.
                starting = mainStack.topRunningActivityLocked();
            }
            //先后调用两个函数
            if (starting != null) {
                kept = starting.ensureActivityConfiguration(changes,
                        false /* preserveWindow */);
                // And we need to make sure at this point that all other activities
                // are made visible with the correct configuration.
                mStackSupervisor.ensureActivitiesVisibleLocked(starting, changes,
                        !PRESERVE_WINDOWS);
            }
        }

        return kept;
    }
```

***

# 4. 应用强制设置屏幕方向

之前提过，其它途径可能会触发转屏，比如应用请求转屏。例如需要横屏的游戏（通过frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java的`updateOrientationFromAppTokensLocked()`方法）。

首先调用AMS的`setRequestedOrientation`函数，然后调用到ActivityRecord的setRequestedOrientation函数。

```java
//frameworks/base/services/core/java/com/android/server/am/ActivityRecord.java
void setRequestedOrientation(int requestedOrientation) {
        final int displayId = getDisplayId();
        final Configuration displayConfig =
                mStackSupervisor.getDisplayOverrideConfiguration(displayId);
        //Step 1
        final Configuration config = mWindowContainerController.setOrientation(requestedOrientation,
                displayId, displayConfig, mayFreezeScreenLocked(app));
        if (config != null) {
            frozenBeforeDestroy = true;
            //Step 2：当返回false，就是现在的状态要改变（比如重启Activity）
            //然后就调用ActivityStackSupervisor的resumeTopActivitiesLocked函数来启动最上面的Activity。
            if (!service.updateDisplayOverrideConfigurationLocked(config, this,
                    false /* deferResume */, displayId)) {
                mStackSupervisor.resumeFocusedStackTopActivityLocked();
            }
        }
        service.mTaskChangeNotificationController.notifyActivityRequestedOrientationChanged(
                task.taskId, requestedOrientation);
    }
```

其中调用到`frameworks/base/services/core/java/com/android/server/wm/AppWindowContainerController.java`的`mWindowContainerController.setOrientation`函数。

```java
//frameworks/base/services/core/java/com/android/server/wm/AppWindowContainerController.java
public Configuration setOrientation(int requestedOrientation, int displayId,
            Configuration displayConfig, boolean freezeScreenIfNeeded) {
        synchronized(mWindowMap) {
            if (mContainer == null) {
                Slog.w(TAG_WM,
                        "Attempted to set orientation of non-existing app token: " + mToken);
                return null;
            }

            mContainer.setOrientation(requestedOrientation);

            final IBinder binder = freezeScreenIfNeeded ? mToken.asBinder() : null;
            //调用WMS的该函数旋转屏幕！！
            return mService.updateOrientationFromAppTokens(displayConfig, binder, displayId);
        }
    }
```

该函数调用到WMS的`updateOrientationFromAppTokensLocked`函数。这个函数先调用另一个updateOrientationFromAppTokensLocked函数，根据这个函数的返回值，返回true代表要旋转，就调用computeNewConfigurationLocked计算Configuration返回。

```java
//WMS.java
private Configuration updateOrientationFromAppTokensLocked(Configuration currentConfig,
            IBinder freezeThisOneIfNeeded, int displayId, boolean forceUpdate) {
        if (!mDisplayReady) {
            return null;
        }
        Configuration config = null;

        if (updateOrientationFromAppTokensLocked(displayId, forceUpdate)) {
            // If we changed the orientation but mOrientationChangeComplete is already true,
            // we used seamless rotation, and we don't need to freeze the screen.
            if (freezeThisOneIfNeeded != null && !mRoot.mOrientationChangeComplete) {
                final AppWindowToken atoken = mRoot.getAppWindowToken(freezeThisOneIfNeeded);
                if (atoken != null) {
                    atoken.startFreezingScreen();
                }
            }
            config = computeNewConfigurationLocked(displayId);

        } else if (currentConfig != null) {
           ......
            }
        }

        return config;
    }

boolean updateOrientationFromAppTokensLocked(int displayId, boolean forceUpdate) {
        long ident = Binder.clearCallingIdentity();
        try {
            final DisplayContent dc = mRoot.getDisplayContent(displayId);
            //获取上次强制设置的方向
            final int req = dc.getOrientation();
            //如果和上次设置的方向不同
            if (req != dc.getLastOrientation() || forceUpdate) {
                dc.setLastOrientation(req);
                //send a message to Policy indicating orientation change to take
                //action like disabling/enabling sensors etc.,
                // TODO(multi-display): Implement policy for secondary displays.
                if (dc.isDefaultDisplay) {
                    mPolicy.setCurrentOrientationLw(req);
                }
                return dc.updateRotationUnchecked(forceUpdate);
            }
            return false;
        } finally {
            Binder.restoreCallingIdentity(ident);
        }
    }
```

# 5. 应用Activity强制设置方向

1. Activity：

如果要强制设置一个Activity的横竖屏可以通过Manifest去设置，跟Activity相关的信息都会保存在ActivityInfo当中。

```s
android:screenOrientation=["unspecified" | "user" | "behind" |
                                     "landscape" | "portrait" |
                                     "reverseLandscape" | "reversePortrait" |
                                     "sensorLandscape" | "sensorPortrait" |
                                     "sensor" | "fullSensor" | "nosensor"]
```

2. Window

如果是要强制设置一个Window的横竖屏可以通过`LayoutParams.screenOrientation`来设置。在通过`WindowManager.addView的时候把对应的LayoutParams`传递给WMS。

```s
WindowManager.LayoutParams.screenOrientation
```

# 6. 参考

+ 参考：https://blog.csdn.net/jinzhuojun/article/details/50085491  
+ 参考：https://blog.csdn.net/kc58236582/article/details/53741445