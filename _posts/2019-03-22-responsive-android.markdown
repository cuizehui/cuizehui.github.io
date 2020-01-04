---
layout:     post
title:      "MMS短信源码-主要功能流程分析"
subtitle:   "记录 oppo-rcs项目，MMS短信源码和RCS业务集成的一些流程和方案"
date:       2019-03-22 23:50:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Android-FrameWork
---

# MMS短信源码-主要功能流程分析


## 简介

本文主要介绍了MMS 短信会话列表（主界面）、消息列表（聊天界面）、sms短信加载/发送流程。

另外介绍，一些rcs(融合通信功能)例如群聊，集成到MMS原生短信的一些方案和思路。

## 会话列表加载

### 会话数据加载流程

```
1. ConversationList.startAsyncQuery();
2. Conversation.startQueryForAll(mQueryHandler, THREAD_LIST_QUERY_TOKEN);
3. Conversation.startQuery ->handler.startQuery(token, null, uri, ALL_THREADS_VIEW_PROJECTION, sqlCon, null, RmsDefine.Threads.PRIORITY_ORDER);
//其中handler 是AsyncQueryHandler 查询完成后触发下方回调
3. ConversationList.ThreadListQueryHandler ->onQueryComplete();
4. private void updateListContent(final boolean empty, Cursor cursor) {
5. ConversationListAdapter onBindView(); ->Conversation conv = Conversation.from(this, cursor);
```

时序图如下：

![conversation](/img/in-post/conversation.png)

### Conversation的初始化

Conversation.cacheAllThreads 通过ThreadId 将查出的会话信息全部缓存在Cache中,当调用From时会判断相同ThreadId是否在Cache中.如果没有则调用Conversation.fillFromCursor 重新加载

fillFromCursor用于将数据库对应字段映射到Conversation对象中

### Conversation渲染数据ITEM

ConversationListAdapter是这个ListView的adapter，派生于CursorAdapter实现了AbsListView.RecyclerListener（当一个view使用完，被放入 listview 回收堆时调用）； 
ConversationListItem是一个自定义的ViewGroup，派生于RelativeLayout，用于表示会话列表的每一个item；将数据映射到item上。
Conversation表示一个会话数据；Contact表示一个联系人；ContactList维护一个联系人列表； 
ConversationListAdapter中只实现了bindView和newView这两个函数，此外，它作为listView的AbsListView.RecyclerListener，还实现了onMovedToScrapHeap函数。

---

## 消息列表加载
### 消息数据加载流程

1. 初始化消息列表

```
1. ComposeMessageActivity.initMessageList  //初始化MessageListAdapter
2. ComposeMessageActivity.onStart()-> loadMessageContent(); //开始查询
3. ComposeMessageActivity.startMsgListQuery()  // 异步查询(AsyncQueryHandler)
4. mBackgroundQueryHandler.startQuery(token,null,uritmp,PROJECTION,selection,null,null); // 查询结果回调
5. BackgroundQueryHandler.onQueryComplete() 
6. mMsgListAdapter.changeCursor(cursor) //更新Adapter数据
```

### 消息列表数据更新流程
 
数据库改变触发更新,ContentObserver触发

```
step1. MessageListAdapter.onContentChanged();
step2. MessageListAdapter.onDataSetChangeListener.onContentChanged();
step3. ComposeActivity.mDataSetChangedListener.onContentChanged()
step4. ComposeMessageActivity.startMsgListQuery()
```

### 消息列表渲染流程

```
1. MessageListAdapter extends CursorAdapter 
2. MessageListAdapter.getItemViewType
3. MessageListAdapter.newView();
4. MessageListAdapter.bindView();
5. MessageListAdapter.bindView()->final MessageItem msgItem = getCachedMessageItem(type, msgId, cursor);将cursor包装成MessageItem
6. MessageListItem.bind(msgItem ); 将MessageItem渲染到MessageListItem
```

时序图如下： 

![message](/img/in-post/message.png)

**ComposeActivity**
launchMode为singleTask的时候，通过Intent跳到一个Activity,如果系统已经存在一个实例，系统就会将请求发送到这个实例上，但这个时候----------系统就不会再调用onCreate方法，而是调用onNewIntent方法。

---


## 创建新会话

当创建一个新的会话时ConversationId = 0 ;
发送消息时会当前Conversation中的联系人确认是否已经存在此会话，如果已经有则使用已有ThreadId 否则在数据库插入并返回一个新的ThreadID

```java
class  WorkingMessage{
     public synchronized long ensureThreadId() {
                if (DEBUG || DELETEDEBUG) {
                    LogTag.debug("ensureThreadId before: " + mThreadId);
                }
                if (mThreadId <= 0) {
                    mThreadId = getOrCreateThreadId(mContext, mRecipients);
                }
                if (DEBUG || DELETEDEBUG) {
                    LogTag.debug("ensureThreadId after: " + mThreadId);
                }
         
                return mThreadId;
            }
}
```


## 消息发送流程

ComposeMessageActivity 持有 workingMessage 和ThreadId对应的Conversation 
workingMessage 持有 ThreadId对应的Conversation  
WorkingMessage.MessageStatusListener 定义了message变化后通知ComposeMessageActivity的接口

```

1.ComposeMessageActivity.confirmSendMessageIfNeeded
2.ComposeMessageActivity.checkInputModeAndSendMessage
3.ComposeMessageActivity.sendMessage  //见步骤三
4.WorkingMessage.send / sendGemini 判断是否支持双卡
5.WorkingMessage.preSendSmsWorker(Conversation conv, String msgText)-> mStatusListener.onPreMessageSent(); //通知ComposeMessageActivity重置WorkingMessage
5.WorkingMessage.sendMmsWorker() // 此处根据Conversation会判断是否是新会话,返回threadId。
6.SmsMessageSender.send() //插入数据表
7.WorkingMessage->mStatusListener.onMessageSent（）是个回调，告诉ComposeMessageActivity发送完成。
8.ComposeMessageActivity.onMessageSent() ; 更新界面 调用startAsyncQuery刷新消息列表 

```

**步骤三详细代码：**

```ComposeMessage.java
private void sendMessage(final boolean bCheckEcmMode) {
  if (FeatureOption.GEMINI_SUPPORT) {
                mWorkingMessage.sendGemini(mMsgListAdapter.getCount(), mSelectedSlotId,
                        SlotId);
            } else {
                mWorkingMessage.send(mMsgListAdapter.getCount());
            }
}
```

## 群聊

1. telephoneProvider 通过改造Thread表增加会话字段rcs /groupchat字段等 区分消息是群聊还是群发还是一对一
2. 增加rms表，通过底层SDK维护并创建group表 并通过groupchat字段进行关联 
3. 改造Conversation 和workingMessage等增加对应属性
4. 通过threadId获得Conversation数据后在界面进行相应的判断和渲染（参考数据加载逻辑）
5. 主要界面逻辑在 ComposeMessageActivity 和conversationList  
6. 群详情、群管理等功能则需要新增界面。


## 参考

https://blog.csdn.net/droyon/article/details/10194591
