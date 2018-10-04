---
layout:     post
title:      "StatusBar添加Notifaction流程分析  "
date:       2018-01-05 18:13:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Android-FrameWork
---

##   StatusBar  Notifaction       ##

###   简介

本文介绍了系统产生一个notifaction产生的流程，提供了如何在流程中加入拦截，拦截掉第三方应用的notification.

###  StatusBar通过Notifaction初始化通知条  ###

PhoneStatusBar ->BaseStatusBar

1.  boot 引导文件启动

2.  SystemUIserVice

    ```java
        private final Class<?>[] SERVICES = new Class[] {
                com.android.systemui.tuner.TunerService.class, //定制状态栏服务
                com.android.systemui.keyguard.KeyguardViewMediator.class,//锁屏模块
                com.android.systemui.recents.Recents.class,//最近应用
                com.android.systemui.volume.VolumeUI.class,//全局音量控制
                com.android.systemui.statusbar.SystemBars.class,//系统状态栏
                com.android.systemui.usb.StorageNotification.class,//Storage存储通知
                com.android.systemui.power.PowerUI.class,//电量管理相关
                com.android.systemui.media.RingtonePlayer.class,//铃声播放
                com.android.systemui.keyboard.KeyboardUI.class,//键盘相关
        };

    ```
    这里的service只是继承了SystemUI类的普通对象而已。

    执行SystemBars.start()后，通过反射机制,最终得到BaseStatusBar对象。
    这里需要说明的是BaseStatusBar是PhoneStatusBar的父类。

3. SystemUI ->start()

4. BaseStatusBar start():

    ```java
            public void start() {

                //注册服务

                mWindowManager = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
                mWindowManagerService = WindowManagerGlobal.getWindowManagerService();
                mDisplay = mWindowManager.getDefaultDisplay();
                mDevicePolicyManager = (DevicePolicyManager)mContext.getSystemService(
                        Context.DEVICE_POLICY_SERVICE);

                mNotificationData = new NotificationData(this);

                mAccessibilityManager = (AccessibilityManager)
                        mContext.getSystemService(Context.ACCESSIBILITY_SERVICE);

                mDreamManager = IDreamManager.Stub.asInterface(
                        ServiceManager.checkService(DreamService.DREAM_SERVICE));
                mPowerManager = (PowerManager) mContext.getSystemService(Context.POWER_SERVICE);

                mContext.getContentResolver().registerContentObserver(
                        Settings.Global.getUriFor(Settings.Global.DEVICE_PROVISIONED), true,
                        mSettingsObserver);
                mContext.getContentResolver().registerContentObserver(
                        Settings.Global.getUriFor(Settings.Global.ZEN_MODE), false,
                        mSettingsObserver);

                        //设置锁屏通知 观察者  在 子类变成 mHeadsUpObserver
                mContext.getContentResolver().registerContentObserver(
                        Settings.Secure.getUriFor(Settings.Secure.LOCK_SCREEN_SHOW_NOTIFICATIONS), false,
                        mSettingsObserver,
                        UserHandle.USER_ALL);

                if (ENABLE_LOCK_SCREEN_ALLOW_REMOTE_INPUT) {
                    mContext.getContentResolver().registerContentObserver(
                            Settings.Secure.getUriFor(Settings.Secure.LOCK_SCREEN_ALLOW_REMOTE_INPUT),
                            false,
                            mSettingsObserver,
                            UserHandle.USER_ALL);
                }
                 //锁屏通知监听者 当url 变化时 调用change 方法   于初始化有关
                mContext.getContentResolver().registerContentObserver(
                        Settings.Secure.getUriFor(Settings.Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS),
                        true,
                        mLockscreenSettingsObserver,
                        UserHandle.USER_ALL);

                mBarService = IStatusBarService.Stub.asInterface(
                        ServiceManager.getService(Context.STATUS_BAR_SERVICE));

                mRecents = getComponent(Recents.class);

                final Configuration currentConfig = mContext.getResources().getConfiguration();
                mLocale = currentConfig.locale;
                mLayoutDirection = TextUtils.getLayoutDirectionFromLocale(mLocale);
                mFontScale = currentConfig.fontScale;
                mDensity = currentConfig.densityDpi;

                mUserManager = (UserManager) mContext.getSystemService(Context.USER_SERVICE);
                mKeyguardManager = (KeyguardManager) mContext.getSystemService(Context.KEYGUARD_SERVICE);
                mLockPatternUtils = new LockPatternUtils(mContext);

                // Connect in to the status bar manager service   连接到状态栏管理器服务
                mCommandQueue = new CommandQueue(this);

                int[] switches = new int[9];
                ArrayList<IBinder> binders = new ArrayList<IBinder>();
                ArrayList<String> iconSlots = new ArrayList<>();
                ArrayList<StatusBarIcon> icons = new ArrayList<>();
                Rect fullscreenStackBounds = new Rect();
                Rect dockedStackBounds = new Rect();
                try {
                    mBarService.registerStatusBar(mCommandQueue, iconSlots, icons, switches, binders,
                            fullscreenStackBounds, dockedStackBounds);
                } catch (RemoteException ex) {
                    // If the system process isn't there we're doomed anyway.
                }
                //创建视图
                createAndAddWindows();
                //创建视图

                mSettingsObserver.onChange(false); // set up
                disable(switches[0], switches[6], false /* animate */);
                setSystemUiVisibility(switches[1], switches[7], switches[8], 0xffffffff,
                        fullscreenStackBounds, dockedStackBounds);
                topAppWindowChanged(switches[2] != 0);
                // StatusBarManagerService has a back up of IME token and it's restored here.
                setImeWindowStatus(binders.get(0), switches[3], switches[4], switches[5] != 0);

                // Set up the initial icon state
                int N = iconSlots.size();
                int viewIndex = 0;
                for (int i=0; i < N; i++) {
                    setIcon(iconSlots.get(i), icons.get(i));
                }

                // Set up the initial notification state.  设置初始通知状态
                try {
                    //设置监听 ，当发生通知产生 移除 ，更新时 会在其中调用相应方法
                    mNotificationListener.registerAsSystemService(mContext,
                            new ComponentName(mContext.getPackageName(), getClass().getCanonicalName()),
                            UserHandle.USER_ALL);
                } catch (RemoteException e) {
                    Log.e(TAG, "Unable to register notification listener", e);
                }


                if (DEBUG) {
                    Log.d(TAG, String.format(
                            "init: icons=%d disabled=0x%08x lights=0x%08x menu=0x%08x imeButton=0x%08x",
                           icons.size(),
                           switches[0],
                           switches[1],
                           switches[2],
                           switches[3]
                           ));
                }

        //注册广播
                mCurrentUserId = ActivityManager.getCurrentUser();
                setHeadsUpUser(mCurrentUserId);

                IntentFilter filter = new IntentFilter();
                filter.addAction(Intent.ACTION_USER_SWITCHED);
                filter.addAction(Intent.ACTION_USER_ADDED);
                filter.addAction(Intent.ACTION_USER_PRESENT);
                mContext.registerReceiver(mBroadcastReceiver, filter);

                IntentFilter internalFilter = new IntentFilter();
                internalFilter.addAction(WORK_CHALLENGE_UNLOCKED_NOTIFICATION_ACTION);
                internalFilter.addAction(BANNER_ACTION_CANCEL);
                internalFilter.addAction(BANNER_ACTION_SETUP);
                mContext.registerReceiver(mBroadcastReceiver, internalFilter, PERMISSION_SELF, null);

                IntentFilter allUsersFilter = new IntentFilter();
                allUsersFilter.addAction(
                        DevicePolicyManager.ACTION_DEVICE_POLICY_MANAGER_STATE_CHANGED);
                mContext.registerReceiverAsUser(mAllUsersReceiver, UserHandle.ALL, allUsersFilter,
                        null, null);
                updateCurrentProfilesCache();

                IVrManager vrManager = IVrManager.Stub.asInterface(ServiceManager.getService("vrmanager"));
                try {
                    vrManager.registerListener(mVrStateCallbacks);
                } catch (RemoteException e) {
                    Slog.e(TAG, "Failed to register VR mode state listener: " + e);
                }

             }

    ```

    由上面代码可见start()方法中主要做了如下操作：

        -    绑定Oberver  mContext.getContentResolver().registerContentObserver(
                Settings.Secure.getUriFor(Settings.Secure.LOCK_SCREEN_ALLOW_PRIVATE_NOTIFICATIONS),
                true,
                mLockscreenSettingsObserver,
                UserHandle.USER_ALL);

        -     设置监听 ，当发生通知产生 移除，更新时 会在其中调用相应方法
            mNotificationListener.registerAsSystemService(mContext,
                    new ComponentName(mContext.getPackageName(), getClass().getCanonicalName()),
                    UserHandle.USER_ALL);
        -       初始化通知视图  createAndAddWindows();

        -      注册广播 mContext.registerReceiver(mBroadcastReceiver, filter);

    ` 本次分析只分析和Notifaction 相关的部分 `

5.  其中的 mBroadcastReceiver 这个广播接受者 其中处理了notifaction 的事情：

    ```java
            private final BroadcastReceiver mBroadcastReceiver = new BroadcastReceiver() {
                @Override
                public void onReceive(Context context, Intent intent) {
                    String action = intent.getAction();
                    if (Intent.ACTION_USER_SWITCHED.equals(action)) {
                       .......
                    } else if (Intent.ACTION_USER_ADDED.equals(action)) {
                        updateCurrentProfilesCache();
                    } else if (Intent.ACTION_USER_PRESENT.equals(action)) {
                        .......
                    } else if (BANNER_ACTION_CANCEL.equals(action) || BANNER_ACTION_SETUP.equals(action)) {
                        NotificationManager noMan = (NotificationManager)
                                mContext.getSystemService(Context.NOTIFICATION_SERVICE);
                        noMan.cancel(R.id.notification_hidden);

                        Settings.Secure.putInt(mContext.getContentResolver(),
                                Settings.Secure.SHOW_NOTE_ABOUT_NOTIFICATION_HIDING, 0);
                        if (BANNER_ACTION_SETUP.equals(action)) {
                            animateCollapsePanels(CommandQueue.FLAG_EXCLUDE_RECENTS_PANEL,
                                    true /* force */);
                            mContext.startActivity(new Intent(Settings.ACTION_APP_NOTIFICATION_REDACTION)
                                    .addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                            );
                        }
                    } else if (WORK_CHALLENGE_UNLOCKED_NOTIFICATION_ACTION.equals(action)) {
                    .........
                }
                };
    ```


6.   mNotificationListener 此监听为分析的重点，这里负责新notification 产生后视图的创建和销毁

    ```java
         private final NotificationListenerService mNotificationListener =
                    new NotificationListenerService() {
                @Override
                public void onListenerConnected() {
                    if (DEBUG) Log.d(TAG, "onListenerConnected");
                    final StatusBarNotification[] notifications = getActiveNotifications();
                    final RankingMap currentRanking = getCurrentRanking();
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            for (StatusBarNotification sbn : notifications) {
                                android.util.Log.d("czh","init  notification ");
                                addNotification(sbn, currentRanking, null /* oldEntry */);
                            }
                        }
                    });
                }

                @Override
                public void onNotificationPosted(final StatusBarNotification sbn,
                        final RankingMap rankingMap) {
                    /// M: Enable this log for unusual case debug.
                    /*if (DEBUG)*/ Log.d(TAG, "onNotificationPosted: " + sbn);
                    if (sbn != null) {
                        //redmine115450 duxinyun modify powsave music 20180103 begin
                        String packageName = sbn.getPackageName();
                        int open = Settings.System.getInt(mContext.getContentResolver(), Settings.System.POWER_SAVE_SWITCH, 0);
                        if(open==1 && !"com.android.mms".equalsIgnoreCase(packageName)&&
                            !"com.android.calculator2".equalsIgnoreCase(packageName)&&
                            !"com.android.deskclock".equalsIgnoreCase(packageName)&&
                            !"com.android.soundrecorder".equalsIgnoreCase(packageName)&&
                            !"com.android.dialer".equalsIgnoreCase(packageName)){
                                removeNotification(sbn.getKey(), rankingMap);
                                return;
                        }
                        //redmine115450 duxinyun modify powsave music 20180103 end
                        mHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                processForRemoteInput(sbn.getNotification());
                                String key = sbn.getKey();
                                mKeysKeptForRemoteInput.remove(key);
                                boolean isUpdate = mNotificationData.get(key) != null;
                                // In case we don't allow child notifications, we ignore children of
                                // notifications that have a summary, since we're not going to show them
                                // anyway. This is true also when the summary is canceled,
                                // because children are automatically canceled by NoMan in that case.
                                if (!ENABLE_CHILD_NOTIFICATIONS
                                    && mGroupManager.isChildInGroupWithSummary(sbn)) {
                                    if (DEBUG) {
                                        Log.d(TAG, "Ignoring group child due to existing summary: " + sbn);
                                    }

                                    // Remove existing notification to avoid stale data.
                                    if (isUpdate) {
                                        removeNotification(key, rankingMap);
                                    } else {
                                        mNotificationData.updateRanking(rankingMap);
                                    }
                                    return;
                                }
                                if (isUpdate) {
                                    updateNotification(sbn, rankingMap);
                                } else {
                                    addNotification(sbn, rankingMap, null /* oldEntry */);
                                }
                            }
                        });
                    }
                }

                @Override
                public void onNotificationRemoved(StatusBarNotification sbn,
                        final RankingMap rankingMap) {
                    /// M: Enable this log for unusual case debug.
                    /*if (DEBUG)*/ Log.d(TAG, "onNotificationRemoved: " + sbn);
                    if (sbn != null) {
                        final String key = sbn.getKey();
                        mHandler.post(new Runnable() {
                            @Override
                            public void run() {
                                removeNotification(key, rankingMap);
                            }
                        });
                    }
                }

                @Override
                public void onNotificationRankingUpdate(final RankingMap rankingMap) {
                    if (DEBUG) Log.d(TAG, "onRankingUpdate");
                    if (rankingMap != null) {
                    mHandler.post(new Runnable() {
                        @Override
                        public void run() {
                            updateNotificationRanking(rankingMap);
                        }
                    });
                }                            }

            };

    ```

    本质上：NotificationListenerService extends Service  此为远程服务
    通过绑定此服务在底层对notification进行删除和添加，然后回调上面的方法,当一个新的通知产生后则先调用onNotificationPosted()
    然后调用BaseStatusBar 的addNotification()，而addNotification()由其子类PhoneStatusBar 实现：

7. addNotification() 代码如下; 
    ```java
          public void addNotification(StatusBarNotification notification, RankingMap ranking,
                    Entry oldEntry) {
                if (DEBUG) Log.d(TAG, "addNotification key=" + notification.getKey());
                /// M: [ALPS02738355] fix foreground service flag_hide_notification issue. @{
                if (notification != null && notification.getNotification() != null &&
                       (notification.getNotification().flags & Notification.FLAG_HIDE_NOTIFICATION) != 0) {
                    Log.d(TAG, "Will not add the notification.flags contains FLAG_HIDE_NOTIFICATION");
                    return;
                }
                /// @}
                mNotificationData.updateRanking(ranking);
                Entry shadeEntry = createNotificationViews(notification);
                if (shadeEntry == null) {
                    return;
                }
                boolean isHeadsUped = shouldPeek(shadeEntry);
                if (isHeadsUped) {
                    mHeadsUpManager.showNotification(shadeEntry);
                    // Mark as seen immediately
                    setNotificationShown(notification);
                }

                if (!isHeadsUped && notification.getNotification().fullScreenIntent != null) {
                    if (shouldSuppressFullScreenIntent(notification.getKey())) {
                        if (DEBUG) {
                            Log.d(TAG, "No Fullscreen intent: suppressed by DND: " + notification.getKey());
                        }
                    } else if (mNotificationData.getImportance(notification.getKey())
                            < NotificationListenerService.Ranking.IMPORTANCE_MAX) {
                        if (DEBUG) {
                            Log.d(TAG, "No Fullscreen intent: not important enough: "
                                    - notification.getKey());
                        }
                    } else {
                        // Stop screensaver if the notification has a full-screen intent.
                        // (like an incoming phone call)
                        awakenDreams();

                        // not immersive & a full-screen alert should be shown
                        if (DEBUG)
                            Log.d(TAG, "Notification has fullScreenIntent; sending fullScreenIntent");
                        try {
                            EventLog.writeEvent(EventLogTags.SYSUI_FULLSCREEN_NOTIFICATION,
                                    notification.getKey());
                            notification.getNotification().fullScreenIntent.send();
                            shadeEntry.notifyFullScreenIntentLaunched();
                            MetricsLogger.count(mContext, "note_fullscreen", 1);
                        } catch (PendingIntent.CanceledException e) {
                        }
                    }
                }
                //cuizehui
                if(notification.getKey().equalsIgnoreCase("0|com.android.systemui|2131886133|null|10033"))
                {
                android.util.Log.d("czh","not allow "+notification.getKey());
                }
                else{
                android.util.Log.d("czh","equels  allow"+notification.getKey());
                }
                //
                addNotificationViews(shadeEntry, ranking);
                
                // Recalculate the position of the sliding windows and the titles.
                setAreThereNotifications();
            }
    ```

8. 调用 addNotificationViews(); //添加条目.此方法在BaseStatusBar 中实现：

    ```java
          protected void addNotificationViews(Entry entry, RankingMap ranking) {
                if (entry == null) {
                    return;
                }

                android.util.Log.d("czh","add_notificationview");
                
                // Add the expanded view and icon.
                mNotificationData.add(entry, ranking);
                updateNotifications();
            }

    ```

9. 调用子类PhoneStatusBar:

    ```java
            protected void updateNotifications() {
                mNotificationData.filterAndSort();

                updateNotificationShade();
                mIconController.updateNotificationIcons(mNotificationData);
            }

    ```

10. updateNotificationShade()：
    ```java
            private void updateNotificationShade() {
                if (mStackScroller == null) return;

                // Do not modify the notifications during collapse.
                if (isCollapsing()) {
                    addPostCollapseAction(new Runnable() {
                        @Override
                        public void run() {
                            updateNotificationShade();
                        }
                    });
                    return;
                }

                ArrayList<Entry> activeNotifications = mNotificationData.getActiveNotifications();
                ArrayList<ExpandableNotificationRow> toShow = new ArrayList<>(activeNotifications.size());
                final int N = activeNotifications.size();
                for (int i=0; i<N; i++) {
                    Entry ent = activeNotifications.get(i);
                    int vis = ent.notification.getNotification().visibility;

                    // Display public version of the notification if we need to redact.
                    final boolean hideSensitive =
                            !userAllowsPrivateNotificationsInPublic(ent.notification.getUserId());
                    boolean sensitiveNote = vis == Notification.VISIBILITY_PRIVATE;
                    boolean sensitivePackage = packageHasVisibilityOverride(ent.notification.getKey());
                    boolean sensitive = (sensitiveNote && hideSensitive) || sensitivePackage;
                    boolean showingPublic = sensitive && isLockscreenPublicMode();
                    if (showingPublic) {
                        updatePublicContentView(ent, ent.notification);
                    }
                    ent.row.setSensitive(sensitive, hideSensitive);
                    if (ent.autoRedacted && ent.legacy) {
                        // TODO: Also fade this? Or, maybe easier (and better), provide a dark redacted form
                        // for legacy auto redacted notifications.
                        if (showingPublic) {
                            ent.row.setShowingLegacyBackground(false);
                        } else {
                            ent.row.setShowingLegacyBackground(true);
                        }
                    }
                    if (mGroupManager.isChildInGroupWithSummary(ent.row.getStatusBarNotification())) {
                        ExpandableNotificationRow summary = mGroupManager.getGroupSummary(
                                ent.row.getStatusBarNotification());
                        List<ExpandableNotificationRow> orderedChildren =
                                mTmpChildOrderMap.get(summary);
                        if (orderedChildren == null) {
                            orderedChildren = new ArrayList<>();
                            mTmpChildOrderMap.put(summary, orderedChildren);
                        }
                        orderedChildren.add(ent.row);
                    } else {
                        toShow.add(ent.row);
                    }

                }
    ```
    
    至此一个notification 则展示完全 ,可以发现最终显示在锁屏界面的通知栏是 ：ExpandableNotificationRow 


###  应用  ###

可以分析当我们需要，拦截掉某些notification 不让其显示则可以在源头：onNotificationPosted 进行包名的过滤和拦截
也可以在 addNotification() 限制其view 的添加和显示 addNotificationViews()不让其调用此方法


### 待分析 ###

源头 ：NotificationListenerService

删除一条notification的操作：

- public void removeNotification(String key, RankingMap ranking);
- updateRowStates->addNotification();  :    show Row







