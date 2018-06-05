---
layout:     post
title:      "Android消息推送和提醒"
date:       2018-05-12 12:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---

# Android消息推送和提醒（Push/Calender）

## 简介

本文介绍两种app未在前台的情况下，如何进行有效的消息通知的方法。

第一种方式调用系统日历api，插入日历事件在指定时间进行提示。

第二种方式通过集成google推送,fireBase.唤醒app，或者提示一个notification.

## 使用Push
### 集成firebaseCloudMessage
1. google play 11.8+
2. 科学上网：[https://github.com/getlantern/forum/issues/833]()
3. 谷歌上网助手 Chrome插件
4. 手机科学上网：lantern :[https://github.com/getlantern/forum]()
4. 倒入依赖的jar 如出现：找不到com.google.android.gms.internal.zzbck的类文件，怎注意版本匹配问题
5. 校验本地是否有Google service的方法，需要:play-services-gcm:11.4.2 这个库
	
 	```
	 dependencies {
    compile 'com.google.firebase:firebase-core:11.4.2'
    compile 'com.google.firebase:firebase-messaging:11.4.2'
    compile "com.google.android.gms:play-services-gcm:11.4.2"
}
apply plugin: 'com.google.gms.google-services'

	```
	
### 客户端流程

1. 生成app token 上传至服务器

	```
	    FirebaseApp.initializeApp(context.getApplicationContext());
	    String token = FirebaseInstanceId.getInstance().getToken();
	```
2. 生成某些模版数据上传至服务器
3. 将注册的applacation key 生成的service ID 和sendID 给到服务端，使服务端可以和fcm服务器通信
4. 定义通讯的消息协议（上行xmpp等并不常用）
5. 接受消息onMessageReceived 

### 服务器
1. 需要apk 提供的serviceID  需要sendID
2. 接收apk token和其他的 app端参数
3. 和firebase 服务器协议交互



## 使用系统Calendar 

### 实现步骤

contentprovider提供了日历接口

1. 插入一个新的日历账户
2. 插入日历事件
3. 根据日历事件对应的eventId 插入日历事件的提醒
4. 根据eventID 删除日历事件

### 具体接口代码

权限：

```
  <uses-permission android:name="android.permission.WRITE_CALENDAR" />  
```

#### 插入日历账户

```
public class CalendarManager {
    /**
     * 插入一条新的日历账户
     *
     * @return
     */
    public String insertAccount(Context context) {
        ContentResolver cr = context.getContentResolver();
        Uri uri = CalendarContract.Calendars.CONTENT_URI;
        StringBuilder sb = new StringBuilder();
        Uri accountUri = null;

        String accountName = "CocahTalk";

        ContentValues values = new ContentValues();
        values.put(CalendarContract.Calendars.ACCOUNT_NAME, accountName);
        //在添加账户时，如果账户类型不存在系统中，则可能该新增记录会被标记为脏数据而被删除，设置为ACCOUNT_TYPE_LOCAL可以保证在不存在账户类型时，该新增数据不会被删除；
        values.put(CalendarContract.Calendars.ACCOUNT_TYPE, CalendarContract.ACCOUNT_TYPE_LOCAL);
        values.put(CalendarContract.Calendars.NAME, "CocahTalk");
        values.put(CalendarContract.Calendars.CALENDAR_DISPLAY_NAME, "CocahTalk");
        values.put(CalendarContract.Calendars.CALENDAR_COLOR, 0XFF5555);
        values.put(CalendarContract.Calendars.CALENDAR_ACCESS_LEVEL, CalendarContract.Calendars.CAL_ACCESS_OWNER);
        values.put(CalendarContract.Calendars.VISIBLE, 1);
        values.put(CalendarContract.Calendars.CALENDAR_TIME_ZONE, TimeZone.getDefault().getID());
        values.put(CalendarContract.Calendars.CAN_MODIFY_TIME_ZONE, 1);
        values.put(CalendarContract.Calendars.SYNC_EVENTS, 1);
        values.put(CalendarContract.Calendars.OWNER_ACCOUNT, "CocahTalk Account");
        values.put(CalendarContract.Calendars.CAN_ORGANIZER_RESPOND, 1);
        values.put(CalendarContract.Calendars.MAX_REMINDERS, 8);
        values.put(CalendarContract.Calendars.ALLOWED_REMINDERS, "0,1,4");
        values.put(CalendarContract.Calendars.ALLOWED_AVAILABILITY, "0,1,2");
        values.put(CalendarContract.Calendars.ALLOWED_ATTENDEE_TYPES, "0,1,2,3");
//        values.put(CalendarContract.Calendars.IS_PRIMARY, 1);
//        values.put(CalendarContract.Calendars.DIRTY, 1);
        //修改或添加ACCOUNT_NAME只能由syn-adapter调用，对uri设置CalendarContract.CALLER_IS_SYNCADAPTER为true，即标记当前操作为syn_adapter操作；
        //再设置CalendarContract.CALLER_IS_SYNCADAPTER为true时，必须带上参数ACCOUNT_NAME和ACCOUNT_TYPE，至于这两个参数的值，可以随意填，不影响最终结果；
        uri = uri
                .buildUpon()
                .appendQueryParameter(CalendarContract.CALLER_IS_SYNCADAPTER, "true")
                .appendQueryParameter(CalendarContract.Calendars.ACCOUNT_NAME, "Test")
                .appendQueryParameter(CalendarContract.Calendars.ACCOUNT_TYPE, CalendarContract.ACCOUNT_TYPE_LOCAL)
                .build();

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (PackageManager.PERMISSION_GRANTED == context.checkSelfPermission("android.permission.WRITE_CALENDAR")) {
                accountUri = cr.insert(uri, values);
            } else {
                return "请授予编辑日历的权限";
            }
        } else {
            accountUri = cr.insert(uri, values);
        }
        // get the event ID that is the last element in the Uri
        long accountID = Long.parseLong(accountUri.getLastPathSegment());

        sb.append("新增日历账户成功！\n");
        sb.append("accountID:" + accountID).append("\n");
        sb.append("accountName:" + accountName).append("\n");
        sb.append("\n");
        return sb.toString();
    }

```
#### 删除日历账户
```
 /**
     * 删除日历账户
     *
     * @return
     */
    public String deleteAccount(Context context) {
        ContentResolver cr = context.getContentResolver();
        Uri uri = CalendarContract.Calendars.CONTENT_URI;
        StringBuilder sb = new StringBuilder();
        int deletedCount = 0;
        String selection = "((" + CalendarContract.Calendars.ACCOUNT_NAME + " = ?) AND ("
                + CalendarContract.Calendars.ACCOUNT_TYPE + " = ?))";
        String[] selectionArgs = new String[]{"CoachTalk", CalendarContract.ACCOUNT_TYPE_LOCAL};
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (PackageManager.PERMISSION_GRANTED == context.checkSelfPermission("android.permission.WRITE_CALENDAR")) {
                deletedCount = cr.delete(uri, selection, selectionArgs);
            } else {
                return "请授予编辑日历的权限";
            }
        } else {
            deletedCount = cr.delete(uri, selection, selectionArgs);
        }
        sb.append("删除日历账户成功！\n");
        sb.append("删除账户数:" + deletedCount).append("\n");
        sb.append("\n");
        return sb.toString();
    }
```

#### 生成日历事件

```
    /**
     * @param beginYear
     * @param beginMonth
     * @param beginDay
     * @param beginHour
     * @param beginMinute
     * @param endYear
     * @param endMonth
     * @param endDay
     * @param endHour
     * @param endMinute
     * @return 返回一个时间提供给CreateContentResolver
     */
    public long[] createEvent(int beginYear, int beginMonth, int beginDay, int beginHour, int beginMinute, int endYear, int endMonth, int endDay, int endHour, int endMinute) {
        long[] calenderMillis = new long[2];
        calenderMillis[0] = 0;
        calenderMillis[1] = 0;
        Calendar beginTime = Calendar.getInstance();
        beginTime.set(beginYear, beginMonth, beginDay, beginHour, beginMinute);
        calenderMillis[0] = beginTime.getTimeInMillis();
        Calendar endTime = Calendar.getInstance();
        endTime.set(endYear, endMonth, endDay, endHour, endMinute);
        calenderMillis[1] = endTime.getTimeInMillis();

        return calenderMillis;
    }

    /**
     * @param context
     * @param calID          日历事件id
     * @param calenderMillis 时间段数组
     * @param title          事件主题
     * @param description    事件描述
     * @param location       事件地点
     * @param organizer      组织者
     * @param eventtitle
     * @return
     */
    public ContentValues CreateContentResolver(Context context, long calID, long[] calenderMillis, String title, String description, String location, String organizer, String eventtitle) {
        ContentValues values = new ContentValues();
          /*
        以下是针对插入一个新的事件的一些规则：
        1.  必须包含CALENDAR_ID和DTSTART字段
        2.  必须包含EVENT_TIMEZONE字段。使用getAvailableIDs()方法获得系统已安装的时区ID列表。注意如果通过INSTERT类型Intent对象来插入事件，那么这个规则不适用，因为在INSERT对象的场景中会提供一个默认的时区；
        3.  对于非重复发生的事件，必须包含DTEND字段；
        4.  对重复发生的事件，必须包含一个附加了RRULE或RDATE字段的DURATIION字段。注意，如果通过INSERT类型的Intent对象来插入一个事件，这个规则不适用。因为在这个Intent对象的应用场景中，你能够把RRULE、DTSTART和DTEND字段联合在一起使用，并且Calendar应用程序能够自动的把它转换成一个持续的时间。
         */
        values.put(CalendarContract.Events.CALENDAR_ID, calID);
        values.put(CalendarContract.Events.DTSTART, calenderMillis[0]);
        values.put(CalendarContract.Events.DTEND, calenderMillis[1]);
        values.put(CalendarContract.Events.TITLE, title);
        values.put(CalendarContract.Events.DESCRIPTION, description);
        values.put(CalendarContract.Events.EVENT_LOCATION, location);
        values.put(CalendarContract.Events.EVENT_TIMEZONE, TimeZone.getDefault().getID());
        values.put(CalendarContract.Events.ACCESS_LEVEL, CalendarContract.Events.ACCESS_DEFAULT);
        values.put(CalendarContract.Events.ORGANIZER, organizer);
        values.put(CalendarContract.Events.STATUS, 1);
        values.put(CalendarContract.Events.HAS_ALARM, 1);
        values.put(CalendarContract.Events.HAS_ATTENDEE_DATA, 1);
        StringBuilder sb = new StringBuilder();
        sb.append("添加日历事件成功！\n");
        sb.append("calID:" + calID).append("\n");
        sb.append("title:" + title).append("\n");
        sb.append("\n");
        return values;
    }

    /**
     * @param values  日历事件
     * @param context
     * @return 此次日历事件Uri
     */
    public long insertEvent(ContentValues values, Context context) {
        ContentResolver cr = context.getContentResolver();
        Uri uri = CalendarContract.Events.CONTENT_URI;

        Uri eventUri = null;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (PackageManager.PERMISSION_GRANTED == context.checkSelfPermission("android.permission.WRITE_CALENDAR")) {
                eventUri = cr.insert(uri, values);
            } else {
                eventUri = null;
            }
        } else {
            eventUri = cr.insert(uri, values);
        }
        long eventID = Long.parseLong(eventUri.getLastPathSegment());

        return eventID;
    }
    
```

#### 生成日历事件提醒

```
    /***
     *
     * @param context
     *
     * @param remindertime 提前多久提醒
     */
    public void createReminder(Context context, long eventID, int remindertime) {
        // get the event ID that is the last element in the Uri
        ContentResolver cr = context.getContentResolver();
        // 事件提醒的设定
        ContentValues reminderValues = new ContentValues();
        reminderValues.put(CalendarContract.Reminders.EVENT_ID, eventID);
        //提前10分钟
        reminderValues.put(CalendarContract.Reminders.MINUTES, remindertime);// 提前10分钟有提醒
        //闹钟提醒
        reminderValues.put(CalendarContract.Reminders.METHOD, CalendarContract.Reminders.METHOD_ALERT);// 提醒方式

        if (ActivityCompat.checkSelfPermission(context, Manifest.permission.WRITE_CALENDAR) != PackageManager.PERMISSION_GRANTED) {
            // TODO: Consider calling
            //    ActivityCompat#requestPermissions
            // here to request the missing permissions, and then overriding
            //   public void onRequestPermissionsResult(int requestCode, String[] permissions,
            //                                          int[] grantResults)
            // to handle the case where the user grants the permission. See the documentation
            // for ActivityCompat#requestPermissions for more details.
            return;
        }
        Uri reminderUri = cr.insert(CalendarContract.Reminders.CONTENT_URI, reminderValues);

    }
```
 
#### 删除日历事件

```
    /**
     * 删除指定ID的日历事件及其提醒
     *
     * @param eventId
     * @return
     */
    public String deleteEvent(Context context, long eventId) {
        ContentResolver cr = context.getContentResolver();
        Uri uri = CalendarContract.Events.CONTENT_URI;
        StringBuilder sb = new StringBuilder();
        int deletedCount = 0;

        String selection = "(" + CalendarContract.Events._ID + " = ?)";
        String[] selectionArgs = new String[]{String.valueOf(eventId)};

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (PackageManager.PERMISSION_GRANTED == context.checkSelfPermission("android.permission.WRITE_CALENDAR")) {
                deletedCount = cr.delete(uri, selection, selectionArgs);
            } else {
                return "请授予编辑日历的权限";
            }
        } else {
            deletedCount = cr.delete(uri, selection, selectionArgs);
        }

        String reminderSelection = "(" + CalendarContract.Reminders.EVENT_ID + " = ?)";
        String[] reminderSelectionArgs = new String[]{String.valueOf(eventId)};

        int deletedReminderCount = cr.delete(CalendarContract.Reminders.CONTENT_URI, reminderSelection, reminderSelectionArgs);

        sb.append("删除日历事件成功！\n");
        sb.append("删除事件数:" + deletedCount).append("\n");
        sb.append("删除事件提醒数:" + deletedReminderCount).append("\n");
        sb.append("\n");

        return sb.toString();
    }
}
```
#### 查询日历事件

```
 public ArrayList queryEvent(Context context, long calendarID) {
        ArrayList arrayList = new ArrayList();
        String[] EVENT_PROJECTION = new String[]{
                CalendarContract.Events.CALENDAR_ID,                           // 0
                CalendarContract.Events.TITLE,                  // 1
                CalendarContract.Events.DESCRIPTION,          // 2
                CalendarContract.Events.EVENT_LOCATION,                  //3
                CalendarContract.Events.DISPLAY_COLOR,              //4
                CalendarContract.Events.STATUS,           //5
                CalendarContract.Events.DTSTART,                         //6
                CalendarContract.Events.DTEND,              //7
                CalendarContract.Events.DURATION,                     //8
                CalendarContract.Events.EVENT_TIMEZONE,                   //9
                CalendarContract.Events.EVENT_END_TIMEZONE,           //10
                CalendarContract.Events.ALL_DAY,            //11
                CalendarContract.Events.ACCESS_LEVEL,                   //12
                CalendarContract.Events.AVAILABILITY,               //13
                CalendarContract.Events.HAS_ALARM,            //14
                CalendarContract.Events.RRULE,          //15
                CalendarContract.Events.RDATE,                       //16
                CalendarContract.Events.HAS_ATTENDEE_DATA,                  // 17
                CalendarContract.Events.LAST_DATE,                  // 18
                CalendarContract.Events.ORGANIZER,                  // 19
                CalendarContract.Events.IS_ORGANIZER,                  // 20
                CalendarContract.Events._ID                         //21
        };
        // Run query
        Cursor cur = null;
        ContentResolver cr = context.getContentResolver();
        Uri uri = CalendarContract.Events.CONTENT_URI;
        StringBuilder sb = new StringBuilder();
        String selection = "(" + CalendarContract.Events.CALENDAR_ID + " = ?)";
        String[] selectionArgs = new String[]{String.valueOf(calendarID)};
//        String selection = null;
//        String[] selectionArgs = null;
        // Submit the query and get a Cursor object back.
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (PackageManager.PERMISSION_GRANTED == context.checkSelfPermission("android.permission.READ_CALENDAR")) {
                cur = cr.query(uri, EVENT_PROJECTION, selection, selectionArgs, null);
            } else {
                return null;
                //return "请授予读取日历的权限";
            }
        } else {
            cur = cr.query(uri, EVENT_PROJECTION, selection, selectionArgs, null);
        }

        // Use the cursor to step through the returned records
        while (cur.moveToNext()) {
            long id = 0;
            long calID = 0;
            String title = null;
            String description = null;
            String eventLocation = null;
            int displayColor = 0;
            int status = 0;
            long start = 0;
            long end = 0;
            String duration = null;
            String eventTimeZone = null;
            String eventEndTimeZone = null;
            int allDay = 0;
            int accessLevel = 0;
            int availability = 0;
            int hasAlarm = 0;
            String rrule = null;
            String rdate = null;
            int hasAttendeeData = 0;
            int lastDate;
            String organizer;
            String isOrganizer;
            // Get the field values
            id = cur.getLong(21);
            calID = cur.getLong(0);
            title = cur.getString(1);
            description = cur.getString(2);
            eventLocation = cur.getString(3);
            displayColor = cur.getInt(4);
            status = cur.getInt(5);
            start = cur.getLong(6);
            end = cur.getLong(7);
            duration = cur.getString(8);
            eventTimeZone = cur.getString(9);
            eventEndTimeZone = cur.getString(10);
            allDay = cur.getInt(11);
            accessLevel = cur.getInt(12);
            availability = cur.getInt(13);
            hasAlarm = cur.getInt(14);
            rrule = cur.getString(15);
            rdate = cur.getString(16);
            hasAttendeeData = cur.getInt(17);
            lastDate = cur.getInt(18);
            organizer = cur.getString(19);
            isOrganizer = cur.getString(20);
            sb.append("id:" + id).append("\n");
            sb.append("calID:" + calID).append("\n");
            sb.append("title:" + title).append("\n");
            sb.append("description:" + description).append("\n");
            sb.append("eventLocation:" + eventLocation).append("\n");
            sb.append("displayColor:" + displayColor).append("\n");
            sb.append("status:" + status).append("\n");
            sb.append("start:" + start).append("\n");
            sb.append("end:" + end).append("\n");
            sb.append("duration:" + duration).append("\n");
            sb.append("eventTimeZone:" + eventTimeZone).append("\n");
            sb.append("eventEndTimeZone:" + eventEndTimeZone).append("\n");
            sb.append("allDay:" + allDay).append("\n");
            sb.append("accessLevel:" + accessLevel).append("\n");
            sb.append("availability:" + availability).append("\n");
            sb.append("hasAlarm:" + hasAlarm).append("\n");
            sb.append("rrule:" + rrule).append("\n");
            sb.append("rdate:" + rdate).append("\n");
            sb.append("hasAttendeeData:" + hasAttendeeData).append("\n");
            sb.append("lastDate:" + lastDate).append("\n");
            sb.append("organizer:" + organizer).append("\n");
            sb.append("isOrganizer:" + isOrganizer).append("\n");
            sb.append("------------------\n");
            String[] REMINDER_PROJECTION = new String[]{
                    CalendarContract.Reminders._ID,                           // 0
                    CalendarContract.Reminders.EVENT_ID,                // 1
                    CalendarContract.Reminders.MINUTES,                  //2
                    CalendarContract.Reminders.METHOD,                  //3
            };
            String reminderSelection = "(" + CalendarContract.Reminders.EVENT_ID + " = ?)";
            String[] reminderSelectionArgs = new String[]{String.valueOf(id)};
            Cursor reminderCur = cr.query(CalendarContract.Reminders.CONTENT_URI, REMINDER_PROJECTION, reminderSelection, reminderSelectionArgs, null);
            while (reminderCur.moveToNext()) {
                long reminderId = 0;
                long reminderEventID = 0;
                int reminderMinute = 0;
                int reminderMethod = 0;
                reminderId = reminderCur.getLong(0);
                reminderEventID = reminderCur.getLong(1);
                reminderMinute = reminderCur.getInt(2);
                reminderMethod = reminderCur.getInt(3);
                arrayList.add(reminderEventID);
                sb.append("reminderId:" + reminderId).append("\n");
                sb.append("reminderEventID:" + reminderEventID).append("\n");
                sb.append("reminderMinute:" + reminderMinute).append("\n");
                sb.append("reminderMethod:" + reminderMethod).append("\n");
                sb.append("\n");
            }
        }
        cur.close();
        return arrayList;
    }
```
#### 删除所有日历事件
```
    public void deleteAll(Context context) {
        ArrayList arrayList = this.queryEvent(context, 3);
        int ei;
        for (ei = 0; ei < arrayList.size(); ei++) {
            long eventId = (long) arrayList.get(ei);
            this.deleteEvent(context, eventId);
        }
    }

```

#### 接口调用

````
    @OnClick(R.id.add_event)
    public void addEvent() {
        CalendarManager calendarManager = new CalendarManager();
        long[] calenderMillis = calendarManager.createEvent(2018, 4, 9, 15, 42, 2018, 4, 9, 17, 30);
        ContentValues contentValues = calendarManager.CreateContentResolver(mHomeStudentActivity, 3, calenderMillis, "title", "description", "location", "organizer", "eventtitle");
        long eventID = calendarManager.insertEvent(contentValues, mHomeStudentActivity);
        calendarManager.createReminder(mHomeStudentActivity, eventID, 3);
    }

    @OnClick(R.id.delete_event)
    public void deleteEvent() {
        CalendarManager calendarManager = new CalendarManager();
        Log.d(calendarManager.deleteEvent(mHomeStudentActivity, 22), "sdfsdf");
    }

    @OnClick(R.id.add_calander_account)
    public void addAccount() {
        CalendarManager calendarManager = new CalendarManager();
        calendarManager.insertAccount(mHomeStudentActivity);
    }
````

其中eventID可以根据mobileAPI 传入的参数字组装 存入sharePrefences

### 参考
[https://firebase.google.com/docs/android/setup]()
[https://www.jianshu.com/p/5d1982dd588b]()
[https://blog.csdn.net/newhope1106/article/details/54709916]()

