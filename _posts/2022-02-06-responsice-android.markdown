---
layout:     post
title:      "SQL-CursorLoader"
date:       2022-02-05 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---

## 简介

CursorLoader通常和LoaderManger、ContentProvider一起使用，简化数据库操作。本文介绍如何使用CursorLoader进行多表查询数据库操作,并将结果加在cursor结果集增加一列。以及介绍contactProvider刷新数据时遇到的小问题。

## CursorLoader 使用多表查询方式

 cursorLoader封装了contentProvider的查询操作,创建cursorLoader入参分为几个部分

- DataContentProvider.Data_URI ： ContactProvider的URI

- PROJECTION ： 要查询的字段

- Columns.COLUMNS_A + "=1" ：查询条件

```java
    loader = new BoundCursorLoader(bindingId, mContext,
                                DataContentProvider.Data_URI,
                                PROJECTION,
                                Columns.COLUMNS_A + "=1", null, null)
```


通常我们的查询条件比较复杂,可能涉及到多个表查询的复杂查询语句,这时要把查询条件塞到PROJECTION字段里。此时的Cursor则会多出现一列 。

例如如下代码，查询A表时，又查询了B表，使用 as 作为查询结果 用查询结果作为 ADDCLUME 字段的值,为Cursor 增加了一个数据库表中不存在的字段, 并将结果新增了一列ADDCLUME

```java
    public static final String[] PROJECTION = {
            TABLE_A_COLUMN.COLUMN_A,
            TABLE_A_COLUMN.COLUMN_B,
            "(SELECT count(" + DatabaseHelper.TABLE_B + '.' + TABLE_B_COLUMN._ID + ")"
                    + " FROM " + DatabaseHelper.TABLE_B
                    + " LEFT JOIN " + DatabaseHelper.TABLE_A + " ON (" + DatabaseHelper.TABLE_A + "." + TABLE_A_COLUMN._ID + "=" + DatabaseHelper.TABLE_B + '.' + TABLE_B_COLUMN.CONVERSATION_ID + ")"
                    + " WHERE "
                    + TABLE_B_COLUMN.READ + " = 0 AND " + TABLE_A_COLUMN.ENTERPRISE_FLAG + "=1 
                    ") AS " + TABLE_A_COLUMN.ADDCLUME,
    };
```


## LoaderManager 使用

 LoaderManger封装了CursorLoader创建销毁和执行,并且封装了回调接口

 LoaderManager封装了异步操作,创建Loader后,数据查询结果通过onLoadFinished返回，到此便获得了cursor数据

```java
    private class MyLoaderManager implements LoaderManager.LoaderCallbacks<Cursor> {
            @Override
            public Loader<Cursor> onCreateLoader(final int id, final Bundle args) {
             switch (id) {
                case LOADER_FLAG_A:
                   loader = new BoundCursorLoader(bindingId, mContext,
                            DataContentProvider.Data_URI,
                            PROJECTION,
                            SELECTION,
                            null,       
                            SORT_ORDER);
                   break;
                }
                return loader;
            }
    
            @Override
            public void onLoadFinished(final Loader<Cursor> generic, final Cursor data) {
                 switch (loader.getId()) {
               
                case LOADER_FLAG_A:
                    //todo notify data loader
                    break;
                default:
                    Assert.fail("Unknown loader id");
                    break;
            }
            }
    
            @Override
            public void onLoaderReset(final Loader<Cursor> generic) {
                 switch (loader.getId()) {
               
                case LOADER_FLAG_A:
                     //todo notify data change
                    break;
              
            }
            }
        }
    
```


## 通知刷新方式

CursorLoader内部已经封装了对URI的监听，因此只需要在数据发生改变时调用contactProvider的通知方法即可

```java
public class DataContentProvider extends ContentProvider {

  final ContentResolver cr = Factory.get().getApplicationContext().getContentResolver();
        cr.notifyChange(Data_URI, null);

}
```

这里需要注意的是在后台无法通知刷新数据，frameWork框架做了限制，需要加上此Flag才能在后台通知

```java
public class DataContentProvider extends ContentProvider {
    cr.notifyChange(Data_URI, null, NOTIFY_NO_DELAY | ContentResolver.NOTIFY_SYNC_TO_NETWORK);
}
```


## cursorLoader 分页

1，cursor一次性加载所有数据不会延迟吗？


