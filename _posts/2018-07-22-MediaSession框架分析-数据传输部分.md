---
layout:     post
title:      MediaSession框架分析-数据传输部分
subtitle:   大部分以Android8.0分析MediaSession框架的数据传输部分
date:       2018-07-22
author:     hry
header-img: img/post-bg-android.jpg
catalog: true
tags:
    - MediaSession
---

### 一、简单使用
1. [官方文档](https://developer.android.google.cn/guide/topics/media-apps/audio-app/building-an-audio-app)
2. [第三方介绍](https://juejin.im/post/5aa0e18851882577b45e91df)
3. [官方通用音乐播放器示例](https://github.com/googlesamples/android-UniversalMusicPlayer)

### 二、Activity绑定Service的过程

1. Activity或者Fragment创建MediaBrowserCompat对象并调用connect();

	    MediaBrowserCompat mMediaBrowser = new MediaBrowserCompat(this,
	    new ComponentName(this, MediaPlaybackService.class),
	    mConnectionCallbacks,
	    bundle); // optional Bundle
	    mMediaBrowser.connect();

2. MediaBrowser,真正调用对象

	      final Intent intent = new Intent(MediaBrowserService.SERVICE_INTERFACE);
	                intent.setComponent(mServiceComponent);
	
	                mServiceConnection = new MediaServiceConnection();
	
	                boolean bound = false;
	                try {
	                    bound = mContext.bindService(intent, mServiceConnection,
	                            Context.BIND_AUTO_CREATE);
	                } catch (Exception ex) {
	                    Log.e(TAG, "Failed binding to service " + mServiceComponent);
	                }
	
3. MediaBrowserService，Service真正回调的对象，返回ServiceBinder对象

	    @Override
	    public void onCreate() {
	        super.onCreate();
	        mBinder = new ServiceBinder();
	    }
	
	    @Override
	    public IBinder onBind(Intent intent) {
	        if (SERVICE_INTERFACE.equals(intent.getAction())) {
	            return mBinder;
	        }
	        return null;
	    }

4. MediaBrowser．MediaServiceConnection，bindService成功后，调用mServiceBinder.connect()请求连接
   					
	     // Save their binder
	    mServiceBinder = IMediaBrowserService.Stub.asInterface(binder);
	    
	    // We make a new mServiceCallbacks each time we connect so that we can drop
	    // responses from previous connections.
	    mServiceCallbacks = getNewServiceCallbacks();
	    mState = CONNECT_STATE_CONNECTING;
	    
	    // Call connect, which is async. When we get a response from that we will
	    // say that we're connected.
	    try {
	    if (DBG) {
	    Log.d(TAG, "ServiceCallbacks.onConnect...");
	    dump();
	    }
	    mServiceBinder.connect(mContext.getPackageName(), mRootHints,
	    mServiceCallbacks);
	    } 



5. MediaBrowserService.ServiceBinder,会回调Service的onGetRoot()方法，如果为null，连接失败

    	@Override
        public void connect(final String pkg, final Bundle rootHints,
                final IMediaBrowserServiceCallbacks callbacks) {

            final int uid = Binder.getCallingUid();
            if (!isValidPackage(pkg, uid)) {
                throw new IllegalArgumentException("Package/uid mismatch: uid=" + uid
                        + " package=" + pkg);
            }

            mHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        final IBinder b = callbacks.asBinder();

                        // Clear out the old subscriptions. We are getting new ones.
                        mConnections.remove(b);

                        final ConnectionRecord connection = new ConnectionRecord();
                        connection.pkg = pkg;
                        connection.rootHints = rootHints;
                        connection.callbacks = callbacks;

                        connection.root = MediaBrowserService.this.onGetRoot(pkg, uid, rootHints);

                        // If they didn't return something, don't allow this client.
                        if (connection.root == null) {
                            Log.i(TAG, "No root for client " + pkg + " from service "
                                    + getClass().getName());
                            try {
                                callbacks.onConnectFailed();
                            } catch (RemoteException ex) {
                                Log.w(TAG, "Calling onConnectFailed() failed. Ignoring. "
                                        + "pkg=" + pkg);
                            }
                        } else {
                            try {
                                mConnections.put(b, connection);
                                if (mSession != null) {
                                    callbacks.onConnect(connection.root.getRootId(),
                                            mSession, connection.root.getExtras());
                                }
                            } catch (RemoteException ex) {
                                Log.w(TAG, "Calling onConnect() failed. Dropping client. "
                                        + "pkg=" + pkg);
                                mConnections.remove(b);
                            }
                        }
                    }
                });
        }

6. Service.onGetRoot()，要允许客户端连接到您的服务并浏览其媒体内容，onGetRoot（）必须返回一个非空BrowserRoot，它是一个表示您的内容层次结构的根ID。

	    @Override
		public BrowserRoot onGetRoot(String clientPackageName, int clientUid,
	    Bundle rootHints) {
	
	    // (Optional) Control the level of access for the specified package name.
	    // You'll need to write your own logic to do this.
	    if (allowBrowsing(clientPackageName, clientUid)) {
	        // Returns a root ID that clients can use with onLoadChildren() to retrieve
	        // the content hierarchy.
	        return new BrowserRoot(MY_MEDIA_ROOT_ID, null);
	    } else {
	        // Clients can connect, but this BrowserRoot is an empty hierachy
	        // so onLoadChildren returns nothing. This disables the ability to browse for content.
	        return new BrowserRoot(MY_EMPTY_MEDIA_ROOT_ID, null);
	    }
		}


### 三、获取数据的过程

1. Activity或者Fragment,连接service成功后
    
		mMediaBrowser.unsubscribe(parentId);
	  	mBrowser.subscribe(parentId, options, callback)


2. MediaBrowser 调用mServiceBinder.addSubscription(parentId, callback.mToken, options,mServiceCallbacks);
    
	    private void subscribeInternal(String parentId, Bundle options, SubscriptionCallback callback) {
	        // Check arguments.
	        if (TextUtils.isEmpty(parentId)) {
	            throw new IllegalArgumentException("parentId cannot be empty.");
	        }
	        if (callback == null) {
	            throw new IllegalArgumentException("callback cannot be null");
	        }
	        // Update or create the subscription.
	        Subscription sub = mSubscriptions.get(parentId);
	        if (sub == null) {
	            sub = new Subscription();
	            mSubscriptions.put(parentId, sub);
	        }
	        sub.putCallback(options, callback);
	
	        // If we are connected, tell the service that we are watching. If we aren't connected,
	        // the service will be told when we connect.
	        if (isConnected()) {
	            try {
	                if (options == null) {
	                    mServiceBinder.addSubscriptionDeprecated(parentId, mServiceCallbacks);
	                }
	                mServiceBinder.addSubscription(parentId, callback.mToken, options,
	                        mServiceCallbacks);
	            } catch (RemoteException ex) {
	                // Process is crashing. We will disconnect, and upon reconnect we will
	                // automatically reregister. So nothing to do here.
	                Log.d(TAG, "addSubscription failed with RemoteException parentId=" + parentId);
	            }
	        }
	    }
 
3. MediaBrowserService，最后会调用Service的onLoadChildren方法，Result的sendResult方法又会回调onResultSent方法，最后结果回调回去

	     /**
	     * Save the subscription and if it is a new subscription send the results.
	     */
	    private void addSubscription(String id, ConnectionRecord connection, IBinder token,
	            Bundle options) {
	        // Save the subscription
	        List<Pair<IBinder, Bundle>> callbackList = connection.subscriptions.get(id);
	        if (callbackList == null) {
	            callbackList = new ArrayList<>();
	        }
	        for (Pair<IBinder, Bundle> callback : callbackList) {
	            if (token == callback.first
	                    && MediaBrowserUtils.areSameOptions(options, callback.second)) {
	                return;
	            }
	        }
	        callbackList.add(new Pair<>(token, options));
	        connection.subscriptions.put(id, callbackList);
	        // send the results
	        performLoadChildren(id, connection, options);
	    }
	
	        /**
	     * Call onLoadChildren and then send the results back to the connection.
	     * <p>
	     * Callers must make sure that this connection is still connected.
	     */
	    private void performLoadChildren(final String parentId, final ConnectionRecord connection,
	            final Bundle options) {
	        final Result<List<MediaBrowser.MediaItem>> result
	                = new Result<List<MediaBrowser.MediaItem>>(parentId) {
	            @Override
	            void onResultSent(List<MediaBrowser.MediaItem> list, @ResultFlags int flag) {
	                if (mConnections.get(connection.callbacks.asBinder()) != connection) {
	                    if (DBG) {
	                        Log.d(TAG, "Not sending onLoadChildren result for connection that has"
	                                + " been disconnected. pkg=" + connection.pkg + " id=" + parentId);
	                    }
	                    return;
	                }
	
	                List<MediaBrowser.MediaItem> filteredList =
	                        (flag & RESULT_FLAG_OPTION_NOT_HANDLED) != 0
	                        ? applyOptions(list, options) : list;
	                final ParceledListSlice<MediaBrowser.MediaItem> pls =
	                        filteredList == null ? null : new ParceledListSlice<>(filteredList);
	                try {
	                    connection.callbacks.onLoadChildrenWithOptions(parentId, pls, options);
	                } catch (RemoteException ex) {
	                    // The other side is in the process of crashing.
	                    Log.w(TAG, "Calling onLoadChildren() failed for id=" + parentId
	                            + " package=" + connection.pkg);
	                }
	            }
	        };
	
	        mCurConnection = connection;
	        if (options == null) {
	            onLoadChildren(parentId, result);
	        } else {
	            onLoadChildren(parentId, result, options);
	        }
	        mCurConnection = null;
	
	        if (!result.isDone()) {
	            throw new IllegalStateException("onLoadChildren must call detach() or sendResult()"
	                    + " before returning for package=" + connection.pkg + " id=" + parentId);
	        }
	    }

4. MediaBrowser，回调给主工程

	        private final void onLoadChildren(final IMediaBrowserServiceCallbacks callback,
	            final String parentId, final ParceledListSlice list, final Bundle options) {
	        mHandler.post(new Runnable() {
	            @Override
	            public void run() {
	                // Check that there hasn't been a disconnect or a different
	                // ServiceConnection.
	                if (!isCurrent(callback, "onLoadChildren")) {
	                    return;
	                }
	
	                if (DBG) {
	                    Log.d(TAG, "onLoadChildren for " + mServiceComponent + " id=" + parentId);
	                }
	
	                // Check that the subscription is still subscribed.
	                final Subscription subscription = mSubscriptions.get(parentId);
	                if (subscription != null) {
	                    // Tell the app.
	                    SubscriptionCallback subscriptionCallback = subscription.getCallback(options);
	                    if (subscriptionCallback != null) {
	                        List<MediaItem> data = list == null ? null : list.getList();
	                        if (options == null) {
	                            if (data == null) {
	                                subscriptionCallback.onError(parentId);
	                            } else {
	                                subscriptionCallback.onChildrenLoaded(parentId, data);
	                            }
	                        } else {
	                            if (data == null) {
	                                subscriptionCallback.onError(parentId, options);
	                            } else {
	                                subscriptionCallback.onChildrenLoaded(parentId, data, options);
	                            }
	                        }
	                        return;
	                    }
	                }
	                if (DBG) {
	                    Log.d(TAG, "onLoadChildren for id that isn't subscribed id=" + parentId);
	                }
	            }
	        });
	    }

    

