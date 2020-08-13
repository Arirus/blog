# View 的一些整理 Window PhoneWindow ViewRootImpl

## Window
抽象类，顶级视图的容器，其子类只有一个 PhoneWindow
```java
// activity
final void attach（...） { 
    ...    
    mWindow = new PhoneWindow(this);    
    mWindow.setWindowManager((WindowManager)context.getSystemService(Context.WINDOW_SERVICE), 
        mToken, mComponent.flattenToString(),
        (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);   
    if (mParent != null) {            
        mWindow.setContainer(mParent.getWindow());        
    }
    mWindowManager = mWindow.getWindowManager();    
    ...
}
```

在 activity attach 的时候进行创建，早于 onCreate 方法。同时设置了 WindowManager，并将 WindowManger 反向设置给 Activity。

## WindowManager
WindowManager 其实只是一个接口，其父类是 ViewManager，包含三个方法
```java
public interface ViewManager
{
    public void addView(View view, ViewGroup.LayoutParams params);
    public void updateViewLayout(View view, ViewGroup.LayoutParams params);
    public void removeView(View view);
}
```
ViewManager 才是通常情况下，我们添加视图用的。

其子类为：
```java
public final class WindowManagerImpl implements WindowManager {
    private final WindowManagerGlobal mGlobal = WindowManagerGlobal.getInstance();
    private final Context mContext;
    //创建 这个 impl 的 window 本身。
    private final Window mParentWindow;

    private IBinder mDefaultToken;

    public WindowManagerImpl(Context context) {
        this(context, null);
    }

    private WindowManagerImpl(Context context, Window parentWindow) {
        mContext = context;
        mParentWindow = parentWindow;
    }

    public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }

    public WindowManagerImpl createPresentationWindowManager(Context displayContext) {
        return new WindowManagerImpl(displayContext, mParentWindow);
    }
    ...

    @Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

    @Override
    public void updateViewLayout(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.updateViewLayout(view, params);
    }
    ...

    @Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }
    ...
}
```
内部主要是由 WindowManagerGlobal ，来进行view更新等操作。其内部就是使用了 ViewRootImpl 来直接进行 view 的相关操作。

Window 本身是无法添加视图的，都是通过 WindowManager 来进行操作的。所以创建 WindowManagerImpl，会包含一个 parentWindow。


## PhoneWindow
```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {
        installDecor();
    } 
    ...
    mLayoutInflater.inflate(layoutResID, mContentParent);
    ...
}

private void installDecor() {
    mForceDecorInstall = false;
    if (mDecor == null) {
        // 此时创建 最底层的view  decorView
        mDecor = generateDecor(-1);
        mDecor.setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        mDecor.setIsRootNamespace(true);
        if (!mInvalidatePanelMenuPosted && mInvalidatePanelMenuFeatures != 0) {
            mDecor.postOnAnimation(mInvalidatePanelMenuRunnable);
        }
    } else {
        mDecor.setWindow(this);
    }
    if (mContentParent == null) {
        //设置我们常用的 contentParent
        mContentParent = generateLayout(mDecor);

        // Set up decor part of UI to ignore fitsSystemWindows if appropriate.
        mDecor.makeOptionalFitsSystemWindows();

        final DecorContentParent decorContentParent = (DecorContentParent) mDecor.findViewById(
                R.id.decor_content_parent);

        if (decorContentParent != null) {
            mDecorContentParent = decorContentParent;
            mDecorContentParent.setWindowCallback(getCallback());
            if (mDecorContentParent.getTitle() == null) {
                mDecorContentParent.setWindowTitle(mTitle);
            }

            final int localFeatures = getLocalFeatures();
            for (int i = 0; i < FEATURE_MAX; i++) {
                if ((localFeatures & (1 << i)) != 0) {
                    mDecorContentParent.initFeature(i);
                }
            }

            mDecorContentParent.setUiOptions(mUiOptions);

            if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) != 0 ||
                    (mIconRes != 0 && !mDecorContentParent.hasIcon())) {
                mDecorContentParent.setIcon(mIconRes);
            } else if ((mResourcesSetFlags & FLAG_RESOURCE_SET_ICON) == 0 &&
                    mIconRes == 0 && !mDecorContentParent.hasIcon()) {
                mDecorContentParent.setIcon(
                        getContext().getPackageManager().getDefaultActivityIcon());
                mResourcesSetFlags |= FLAG_RESOURCE_SET_ICON_FALLBACK;
            }
            if ((mResourcesSetFlags & FLAG_RESOURCE_SET_LOGO) != 0 ||
                    (mLogoRes != 0 && !mDecorContentParent.hasLogo())) {
                mDecorContentParent.setLogo(mLogoRes);
            }

            // Invalidate if the panel menu hasn't been created before this.
            // Panel menu invalidation is deferred avoiding application onCreateOptionsMenu
            // being called in the middle of onCreate or similar.
            // A pending invalidation will typically be resolved before the posted message
            // would run normally in order to satisfy instance state restoration.
            PanelFeatureState st = getPanelState(FEATURE_OPTIONS_PANEL, false);
            if (!isDestroyed() && (st == null || st.menu == null) && !mIsStartingWindow) {
                invalidatePanelMenu(FEATURE_ACTION_BAR);
            }
        } 
        if (mDecor.getBackground() == null && mBackgroundFallbackResource != 0) {
            mDecor.setBackgroundFallback(mBackgroundFallbackResource);
        }
    }
}

```
当 Activity 第一次 setContent 时，会将底层的 DecorView 和 ContentParent 给创建出来，并且将用户自己 view 添加到 ContentParent 上面。

此时 Window 和 DecorView 还没有产生联系。这两步都是创建并设置 decorView 相关。

## ActivityThread.handleResumeActivity
```java
public void handleResumeActivity(IBinder token, boolean finalStateRequest, boolean isForward,String reason) {
    //这里面其实已经调到了 activity.performResume(r.startsNotResumed, reason)，因此此时 onResume 其实已经执行结束了。
    final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason);

    final Activity a = r.activity;

    if (r.window == null && !a.mFinished && willBeVisible) {
        r.window = r.activity.getWindow();
        View decor = r.window.getDecorView();
        decor.setVisibility(View.INVISIBLE);
        ViewManager wm = a.getWindowManager();
        WindowManager.LayoutParams l = r.window.getAttributes();
        a.mDecor = decor;
        ...
        if (a.mVisibleFromClient) {
            if (!a.mWindowAdded) {
                a.mWindowAdded = true;
                wm.addView(decor, l);
            }
        }

    }
}
```

在 handleResumeActivity 方法中，在onResume之后，会正式把 DecorView 和 Window 想关联起来。

## WindowManagerGlobal.addView
之前我们分析了，WindowManager 其实是 WindowManagerImpl，因此直接看 WindowManagerImpl 的 addView 方法
```java
public void addView(View view, ViewGroup.LayoutParams params, Display display, Window parentWindow) {
    ...

    ViewRootImpl root;
    View panelParentView = null;
    ...
    synchronized (mLock) {
        ...

        // If this is a panel window, then find the window it is being
        // attached to for future reference.
        if (wparams.type >= WindowManager.LayoutParams.FIRST_SUB_WINDOW &&
                wparams.type <= WindowManager.LayoutParams.LAST_SUB_WINDOW) {
            final int count = mViews.size();
            for (int i = 0; i < count; i++) {
                if (mRoots.get(i).mWindow.asBinder() == wparams.token) {
                    panelParentView = mViews.get(i);
                }
            }
        }

        root = new ViewRootImpl(view.getContext(), display);

        view.setLayoutParams(wparams);

        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

        ...
        root.setView(view, wparams, panelParentView);
        ...
    }
}
```

到这里，添加之后，会有一个ViewRootImpl生成，来真正复制添加一个decor。WindowManagerGlobal 这里其实只是吧相应的 decor 和 ViewRootImpl 保存了起来。

因此一个 Decor 和 ViewRootImpl 应该是一一对应的。因此 decor 和 ViewRootImpl 就这样的绑定了起来。


