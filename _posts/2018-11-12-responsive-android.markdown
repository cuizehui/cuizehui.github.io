---
layout:     post
title:      "MVP模式APP开发-总结"
subtitle:   "介绍MVP模式下APP开发的一些细节和心得"
date:       2018-12-01 22:56:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android
---

## 简介

工程的解耦合主要有两个方向

- 横向：模块化
- 纵向：分层

本文通过Demo总结并思考我这段时间MvpAPP开发的一些心得,主要介绍以下内容:

1. 工程目录结构的拆分，工具类拆分module
2. mvp模式下，BaseActivity的抽取.提供了fragment和Activity通信的特殊思路。
3. Mvp下Adapter处理
4. 最后讲了全局handler 更新UI的方式。

## Demo项目地址

[MVP_App_Frame](https://github.com/cuizehui/MVP_App_Frame)

## 项目结构

通常项目会用到已经写好的某写工具类，这里建议将其拆分为Module引入。

### 拆分toolsModule

```gradle
Module main

    implementation project(path: ':tool')
    
setting.gradle

    include  ':tool'

```

注意tool 所需要的依赖，在mainModule中需要重新引用一遍，并且sdk版本要匹配

### sdk 目录结构

通常在Main Module中可能会引入三方库和依赖等。为了目录结构的清晰和整洁可以进行以下处理。

```gradle
sourceSets {
        main {
            jniLibs.srcDirs = ['../sdk/libname']
        }
    }

dependencies {
    implementation fileTree(include: ['*.jar'], dir: '../sdk/cloud')
    implementation fileTree(include: ['*.jar'], dir: 'libs')
   }
```
将依赖库jar包，引入到主module同级目录下，sdk文件夹下的libname文件夹中。

## mvp结构 

为了减少接口文件过多，将UI接口和Presenter接口统一定义在Contract中。减少接口定义方法需要频繁写，将其抽取到base类中（通过范型等约束）。

### baseActivity

通过定义范型约定 该activity需要的presenter

```java
BaseActivity.java:

public abstract class BaseActivity<T extends BasePresenter> extends AppCompatActivity {

    protected T mPresenter;

    protected abstract T setPresenter();

    }

 @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(attachLayoutRes());
        ButterKnife.bind(this);
        mPresenter = setPresenter();
        initViews();
    }
```

### Fragment和Activity 通信

场景，一个Activity中含有多个Fragment.
那么Fragment的UI逻辑应该在Activity中处理。通常的做法是通过回调，但是如果Fragment过多那么Activity需要实现的接口过多。

这里提供一种思路：将每个Fragment共性回调（例如跳转、显示、隐藏、结束等）抽取在BaseFragment中，由Activity实现此回调方法，并通过TAG区分是那个Fragment的事件统一进行处理。


```java
BaseFragment.java

 private BaseFragmentCallBack mCallback;

    protected void finishFragment(String targetFragment, Object o) {
        if (mCallback != null) {
            mCallback.onFinishFragment(targetFragment, o);
        }
    }
    
    @Override
    public void onAttach(Context context) {
        super.onAttach(context);
        if (context instanceof BaseFragmentCallBack) {
            mCallback = (BaseFragmentCallBack) context;
        }
    }
```


```java
public class RegisterActivity extends BaseActivity<RegisterContract.Presenter> implements RegisterContract.View, BaseFragment.BaseFragmentCallBack {

 @Override
    public void onFinishFragment(String tagFragment, Object object) {
        if (TextUtils.equals(tagFragment, RegisterAccountFragment.TAG)) {
            String account = (String) object;
            showVerifyCodeFragment(account);
        } else if (TextUtils.equals(tagFragment, VerifyCodeFragment.TAG)) {
            showSetPasswordFragment();
        }
    }
}
```

## RecyclerView MVP处理

RecyclerView,通常我们的做法是将list直接放到Adapter中，这样M层就直接和V层联系到一起，没有遵循MVP模式。

对此问题做了以下处理：

1. 定义AdapterContract
2. RecyclerView 传递一个AdapterContract.Present
3. 将ViewHolder作为View层来实现我们的UI方法 实现AdapterContract.View
4. 在onBindViewHolder,将view层传递给了P层,这时我们可以根据position 将数据逐条通过View进行渲染。

```java
ContactsRecyclerViewAdapter.java

public class ContactsRecyclerViewAdapter extends RecyclerView.Adapter<ContactsRecyclerViewAdapter.ViewHolder> {

    private ContactsAdapterContract.Presenter mPresenter;

    public ContactsRecyclerViewAdapter(ContactsAdapterContract.Presenter presenter) {
        mPresenter = presenter;
        mPresenter.start();
    }
    
    ...
    
     @Override
    public void onBindViewHolder(final ViewHolder holder, int position) {
        mPresenter.onBindContractsRecyclerView(holder, position);
    }

    public class ViewHolder extends RecyclerView.ViewHolder implements ContactsAdapterContract.View {
 
     @Override
    public void renderItem(ContactsAdapterContract.Presenter.ItemDraw item) {
           ...
        }
    }
}

```

```java
ContractsAdapterPresenter.java

public class ContractsAdapterPresenter implements ContactsAdapterContract.Presenter {
   
    private ContactsWrapper mContactsWrapper;

    private void loadData() {
    ...
    }

    @Override
    public void start() {
         loadData();
    }
    @Override
    public int getDataCount() {
        return mContactsWrapper != null ? mContactsWrapper.contacts.size() : 0;
    }

    @Override
    public void onBindContractsRecyclerView(ContactsRecyclerViewAdapter.ViewHolder holder, int position) {
        holder.renderItem(getItemDraw(position));
    }

    @Override
    public ItemDraw getItemDraw(int index) {
       ....
    }
}
```

## 更新主线程UI 
1.Application在设置全局handler

```java
appdata.java

    private static Context sContext;
    private static Handler sHandler;

    /**
     * 进程已启动就调用
     *
     * @param context
     */
    public static void init(Context context) {
        sContext = context.getApplicationContext();
        sHandler = new Handler(sContext.getMainLooper());
    }

```

2. 使用handler 在子线程更新UI
```java
 new Handler(AppData.getContext().getMainLooper()).post(new Runnable() {
                    @Override
                    public void run() {
                        mView.showRegainVcode(true);
                        mView.showVcodeTime(false);
                    }
                });
            }
        }, 5000);

```

### ActivityTools

1. 封装Fragment显示隐藏移除添加
2. 封装显示dialog

[ActivityTools](https://github.com/cuizehui/MVP_App_Frame/blob/master/tool/src/main/java/cn/nela/tools/ActivityTool.java)


## 参考文章

[https://blog.csdn.net/adfghjkl/article/details/76862072](https://blog.csdn.net/adfghjkl/article/details/76862072)

[https://github.com/Rukey7/MvpApp/blob/master/app/src/main/java/com/dl7/mvp/module/base/BaseActivity.java](https://github.com/Rukey7/MvpApp/blob/master/app/src/main/java/com/dl7/mvp/module/base/BaseActivity.java)

[https://cloud.tencent.com/developer/article/1032349](https://cloud.tencent.com/developer/article/1032349)