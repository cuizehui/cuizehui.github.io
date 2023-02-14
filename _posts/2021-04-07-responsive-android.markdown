---
layout:     post
title:      "View的绘制流程和事件传递"
date:       2021-04-07 11:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android-Application
---

## Android事件传递流程 & Android View绘制流程

### 事件分发机制简述

Android事件分发机制
三个重要角色
1、Activity：接收Down点击事件，传递给Phonewindow和DecorView
2、ViewGroup：拦截事件，或者继续传递给子View
3、View：决定消费这个事件或者不消费从而返回给上一级
三个核心事件
1、dispatchTouchEvent()：分发点击事件，return false 事件停止往子View传递和分发
2、onTouchEvent() ： return false 是不消费事件，并让事件往父控件的方向从下往上流动。return true 是消费事件。
3、onInterceptTouchEvent()：拦截点击事件， return false 不拦截，允许事件向子View传递， return true拦截事件，不在向子View传递事件。

### ViewGroup事件传递机制

1. dispatchTouchEvent (事件分发)
2. onInterceptTouchEvent 事件拦截 (ViewGroup才有)
3. onTouchEvent

```java
    public class ViewGroup extends View {  
        /**
        *viewGroup事件传递机制
        */
         public boolean dispatchTouchEvent(MotionEvent ev) {
            boolean consume = false;
            if(onInterceptTouchEvent){
                consume = onTouchEvent();
            }else {
                consume = chile.dispatchTouchEvent();
            }
           return consume;
        }
        
        public boolean onInterceptTouchEvent(MotionEvent ev) {
            if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                    && ev.getAction() == MotionEvent.ACTION_DOWN
                    && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                    && isOnScrollbarThumb(ev.getX(), ev.getY())) {
                return true;
            }
            return false;
        }
        
        
    }
```

```java 
   public class View  {  
        /**
         *view事件传递机制
         */
         public boolean dispatchTouchEvent(MotionEvent ev) {
            boolean consume = onTouchEvent();
           return consume
        }
        
         /**
         *view事件传递机制
         */
         public boolean onTouchEvent(MotionEvent ev) {
          //其他情况
           return true;
        }
    }

```

1. 首先调用dispatch 进行事件的分发
2. 如果要拦截 那么onInterceptTouchEvent需要返回ture并且重写当前View onTouchEvent事件
3. consume返回false，则之后的action将不会接收到，如action_DOWN的时候返回了false，将不会再收到之后的Action_UP的内容
4. 如果 InterceptTouchEvent拦截 并且onTouchEvent返回false ，则之后的action将不会接收到
5. 如果没有子View去处理这个事件，即子view的onTouchEvent没有返回True，则最后还是由ViewGroup去处理这个事件，也就又执行了自己的onTouchEvent。

### OnTouchListener

```
 @Override
    public boolean onTouch(View v, MotionEvent event) {
        return false;
        //返回值决定了View的onTouchEvent会不会被调用。 false会调用,ture不会。
    }

```

onTouch调用前会自动调用onInterceptTouchEvent 如果onInterceptTouchEvent返回的false
则不会调用onTouchEvent

### 总结

1.View事件的传递以MotionEvent为例,首先ViewGroup调用dispatchTouchEvent,会判断当前ViewGroup是否interceptTouchEvent();
如果拦截则调用当前ViewGroup的OnTouchEvent()，子view不再收到事件;如果不拦截则调用child的dispatch.如果所有子View都没有处理消费,则还是会调用当前ViewGroup的onTouchEvent();
如果dispatchTouchEvent返回值是false则后续ActionUp等事件都不会收到了。

### 点击事件传递的顺序

Activity --> Window --> DecorView --> 我们的布局View
view必须依附一个Window存在 
Window实现类是 phoneWindow

### View的绘制流程
****
android的事件传递机制。如何解决滑动冲突？


### 自定义View onLayout 位置。
