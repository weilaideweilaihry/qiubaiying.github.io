---
layout:     post
title:      理解Android的Window
subtitle:   以Android8.0理解Android的Window
date:       2018-08-03
author:     hry
header-img: img/post-bg-android.jpg
catalog: true
tags:
    - Window
---

### 一、Window的三种类型
1. APPLICATION_WINDOW : represent normal application windows.(应用类Window，层级范围1-99)
2. SUB_WINDOW : a panel on top of an application window.  These windows appear on top of their attached window.(子Window，层架范围1000-1999)
3. SYSTEM_WINDOW :system-specific window types.  These are not normally created by applications.(系统Window，层级范围2000-2999)

### 二、Window的内部机制
#### 2.1 Winddow的添加过程

WindoqManager的真正实现是WindowManagerImpl，addView方法代码如下

	@Override
    public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
        applyDefaultToken(params);
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
    }

addView真正的实现是在WindowManagerGlobal的addView方法，代码如下

	public void addView(View view, ViewGroup.LayoutParams params,
            Display display, Window parentWindow) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (display == null) {
            throw new IllegalArgumentException("display must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
        if (parentWindow != null) {
            parentWindow.adjustLayoutParamsForSubWindow(wparams);
        } else {
            // If there's no parent, then hardware acceleration for this view is
            // set from the application's hardware acceleration setting.
            final Context context = view.getContext();
            if (context != null
                    && (context.getApplicationInfo().flags
                            & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
                wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            }
        }

        ViewRootImpl root;
        View panelParentView = null;

        synchronized (mLock) {
            // Start watching for system property changes.
            if (mSystemPropertyUpdater == null) {
                mSystemPropertyUpdater = new Runnable() {
                    @Override public void run() {
                        synchronized (mLock) {
                            for (int i = mRoots.size() - 1; i >= 0; --i) {
                                mRoots.get(i).loadSystemProperties();
                            }
                        }
                    }
                };
                SystemProperties.addChangeCallback(mSystemPropertyUpdater);
            }

            int index = findViewLocked(view, false);
            if (index >= 0) {
                if (mDyingViews.contains(view)) {
                    // Don't wait for MSG_DIE to make it's way through root's queue.
                    mRoots.get(index).doDie();
                } else {
                    throw new IllegalStateException("View " + view
                            + " has already been added to the window manager.");
                }
                // The previous removeView() had not completed executing. Now it has.
            }

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

            // do this last because it fires off messages to start doing things
            try {
                root.setView(view, wparams, panelParentView);
            } catch (RuntimeException e) {
                // BadTokenException or InvalidDisplayException, clean up.
                if (index >= 0) {
                    removeViewLocked(index, true);
                }
                throw e;
            }
        }
    }

这边主要是先检查参数等代码，然后构建ViewRootImpl，将布局参数设置给View，分别存储ViewRootImpl，View，LayoutParams到mRoots，mViews，mParams列表数据，最后通过ViewRootImpl的setView方法将View显示到窗口上。

***

ViewRootImpl是Framework层与Native层的通信桥梁，它的构造函数如下：

 	public ViewRootImpl(Context context, Display display) {
        mContext = context;

	//1 获取IWindowSession，用来和WindowManagerService建立连接
        mWindowSession = WindowManagerGlobal.getWindowSession();

        mDisplay = display;
        mBasePackageName = context.getBasePackageName();
	
	//2 保存当前线程，更新UI的线程只能是在创建ViewRootImpl时的线程，否则会抛出异常
        mThread = Thread.currentThread();

        mLocation = new WindowLeaked(null);
        mLocation.fillInStackTrace();
        mWidth = -1;
        mHeight = -1;
        mDirty = new Rect();
        mTempRect = new Rect();
        mVisRect = new Rect();
        mWinFrame = new Rect();
        mWindow = new W(this);
        mTargetSdkVersion = context.getApplicationInfo().targetSdkVersion;
        mViewVisibility = View.GONE;
        mTransparentRegion = new Region();
        mPreviousTransparentRegion = new Region();
        mFirst = true; // true for the first time the view is added
        mAdded = false;
        mAttachInfo = new View.AttachInfo(mWindowSession, mWindow, display, this, mHandler, this,
                context);
        mAccessibilityManager = AccessibilityManager.getInstance(context);
        mAccessibilityManager.addAccessibilityStateChangeListener(
                mAccessibilityInteractionConnectionManager, mHandler);
        mHighContrastTextManager = new HighContrastTextManager();
        mAccessibilityManager.addHighTextContrastStateChangeListener(
                mHighContrastTextManager, mHandler);
        mViewConfiguration = ViewConfiguration.get(context);
        mDensity = context.getResources().getDisplayMetrics().densityDpi;
        mNoncompatDensity = context.getResources().getDisplayMetrics().noncompatDensityDpi;
        mFallbackEventHandler = new PhoneFallbackEventHandler(context);
        mChoreographer = Choreographer.getInstance();
        mDisplayManager = (DisplayManager)context.getSystemService(Context.DISPLAY_SERVICE);

        if (!sCompatibilityDone) {
            sAlwaysAssignFocus = true;

            sCompatibilityDone = true;
        }

        loadSystemProperties();
    }

1处 获取IWindowSession的WindowManagerGlobal类的getWindowSession方法代码如下： 
	
	public static IWindowSession getWindowSession() {
        synchronized (WindowManagerGlobal.class) {
            if (sWindowSession == null) {
                try {
	
	//1 InputMethodManager 整个输入法框架（IMF）架构的中央系统API，
	//它仲裁应用程序和当前输入方法之间的交互。
                    InputMethodManager imm = InputMethodManager.getInstance();
		
	//2 获取WIndowManagerService
                    IWindowManager windowManager = getWindowManagerService();

	//3 与WIndowManagerService建立一个Session
                    sWindowSession = windowManager.openSession(
                            new IWindowSessionCallback.Stub() {
                                @Override
                                public void onAnimatorScaleChanged(float scale) {
                                    ValueAnimator.setDurationScale(scale);
                                }
                            },
                            imm.getClient(), imm.getInputContext());
                } catch (RemoteException e) {
                    throw e.rethrowFromSystemServer();
                }
            }
            return sWindowSession;
        }
    }
	
	其中上面2处的获取WIndowManagerService代码如下：
		 
	
		public static IWindowManager getWindowManagerService() {
	        synchronized (WindowManagerGlobal.class) {
	            if (sWindowManagerService == null) {
	                sWindowManagerService = IWindowManager.Stub.asInterface(
	                        ServiceManager.getService("window"));
	                try {
	                    if (sWindowManagerService != null) {
	                        ValueAnimator.setDurationScale(
	                                sWindowManagerService.getCurrentAnimatorScale());
	                    }
	                } catch (RemoteException e) {
	                    throw e.rethrowFromSystemServer();
	                }
	            }
	            return sWindowManagerService;
	        }
	    }
	

***

回到ViewRootImpl的setView方法，代码如下
	
	    /**
     * We have one child
     */
    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
        synchronized (this) {
            if (mView == null) {
                ...

                // Schedule the first layout -before- adding to the window
                // manager, to make sure we do the relayout before receiving
                // any other events from the system.

	//1 请求布局
                requestLayout();

                if ((mWindowAttributes.inputFeatures
                        & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
                    mInputChannel = new InputChannel();
                }
                mForceDecorViewVisibility = (mWindowAttributes.privateFlags
                        & PRIVATE_FLAG_FORCE_DECOR_VIEW_VISIBILITY) != 0;
                try {
                    mOrigWindowType = mWindowAttributes.type;
                    mAttachInfo.mRecomputeGlobalAttributes = true;
                    collectViewAttributes();

	//2 向wms发起显示的请求
                    res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(),
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mInputChannel);

                } catch (RemoteException e) {
                    mAdded = false;
                    mView = null;
                    mAttachInfo.mRootView = null;
                    mInputChannel = null;
                    mFallbackEventHandler.setView(null);
                    unscheduleTraversals();
                    setAccessibilityFocus(null, null);
                    throw new RuntimeException("Adding window failed", e);
                } finally {
                    if (restore) {
                        attrs.restore();
                    }
                }
               ...
                
            }
        }
    }

1处代码requestLayout方法如下：
	
	@Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }

	void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            if (!mUnbufferedInputDispatch) {
                scheduleConsumeBatchedInput();
            }
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
mChoreographer是Choreographer类，关于编舞者Choreographer可以看这篇文章[Android系统的编舞者Choreographer](https://blog.csdn.net/stven_king/article/details/80098845),最后会执行performTraversals()方法。

	private void performTraversals() {
	
	...
	performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
	...
	performLayout(lp, mWidth, mHeight);
	...
	performDraw();
	...	
	}

#### 2.2 Winddow的删除过程

WindowManagerImpl

	@Override
    public void removeView(View view) {
        mGlobal.removeView(view, false);
    }

    @Override
    public void removeViewImmediate(View view) {
        mGlobal.removeView(view, true);
    }

WindowManagerGlobal的removeView方法如下，

	public void removeView(View view, boolean immediate) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }

        synchronized (mLock) {

	//1 从mViews找到待删除的View的索引
            int index = findViewLocked(view, true);

            View curView = mRoots.get(index).getView();

	//2 删除
            removeViewLocked(index, immediate);
            if (curView == view) {
                return;
            }

            throw new IllegalStateException("Calling with view " + view
                    + " but the ViewAncestor is attached to " + curView);
        }
    }

2处removeViewLocked方法代码如下

	private void removeViewLocked(int index, boolean immediate) {
        ViewRootImpl root = mRoots.get(index);
        View view = root.getView();

        if (view != null) {
            InputMethodManager imm = InputMethodManager.getInstance();
            if (imm != null) {
                imm.windowDismissed(mViews.get(index).getWindowToken());
            }
        }
        boolean deferred = root.die(immediate);
        if (view != null) {
            view.assignParent(null);
            if (deferred) {
                mDyingViews.add(view);
            }
        }
    }


具体的删除操作由ViewRootImpl的die方法完成，代码如下

	 /**
     * @param immediate True, do now if not in traversal. False, put on queue and do later.
     * @return True, request has been queued. False, request has been completed.
     */
    boolean die(boolean immediate) {
        // Make sure we do execute immediately if we are in the middle of a traversal or the damage
        // done by dispatchDetachedFromWindow will cause havoc on return.
        if (immediate && !mIsInTraversal) {
            doDie();
            return false;
        }

        if (!mIsDrawing) {
            destroyHardwareRenderer();
        } else {
            Log.e(mTag, "Attempting to destroy the window while drawing!\n" +
                    "  window=" + this + ", title=" + mWindowAttributes.getTitle());
        }
        mHandler.sendEmptyMessage(MSG_DIE);
        return true;
    }
不管immediate是什么值，最后都会调doDie方法，只是异步删除时，ViewRootImpl的Handler会处理此消息并调用doDie方法，该方法代码如下
	
	void doDie() {
        checkThread();
        if (LOCAL_LOGV) Log.v(mTag, "DIE in " + this + " of " + mSurface);
        synchronized (this) {
            if (mRemoved) {
                return;
            }
            mRemoved = true;
            if (mAdded) {

	//1  这个方法会调用mWindowSession.remove(mWindow)去删除Window,以及调用
	//View的dispatchDetachedFromWindow()方法	
                dispatchDetachedFromWindow();
            }

            if (mAdded && !mFirst) {
                destroyHardwareRenderer();

                if (mView != null) {
                    int viewVisibility = mView.getVisibility();
                    boolean viewVisibilityChanged = mViewVisibility != viewVisibility;
                    if (mWindowAttributesChanged || viewVisibilityChanged) {
                        // If layout params have been changed, first give them
                        // to the window manager to make sure it has the correct
                        // animation info.
                        try {
                            if ((relayoutWindow(mWindowAttributes, viewVisibility, false)
                                    & WindowManagerGlobal.RELAYOUT_RES_FIRST_TIME) != 0) {
                                mWindowSession.finishDrawing(mWindow);
                            }
                        } catch (RemoteException e) {
                        }
                    }

                    mSurface.release();
                }
            }

            mAdded = false;
        }
	//2 将Window关联的对象在mRoots，mParams，mViews列表中删除
        WindowManagerGlobal.getInstance().doRemoveView(this);
    }

#### 2.3 Winddow的更新过程

	public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
        if (view == null) {
            throw new IllegalArgumentException("view must not be null");
        }
        if (!(params instanceof WindowManager.LayoutParams)) {
            throw new IllegalArgumentException("Params must be WindowManager.LayoutParams");
        }

        final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;

        view.setLayoutParams(wparams);

        synchronized (mLock) {
	//1 更新新值
            int index = findViewLocked(view, true);
            ViewRootImpl root = mRoots.get(index);
            mParams.remove(index);
            mParams.add(index, wparams);

	//2 ViewRootImpl的setLayoutParams方法内部会调用scheduleTraversals()方法,最后重新布局，
	//走performTraversals()方法（具体这个方法见上面添加过程那部分）
            root.setLayoutParams(wparams, false);
        }
    }

#### 参考
Android开发艺术探索，Android源码设计模式