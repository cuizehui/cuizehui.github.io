---
layout:     post
title:      "MVP模式Adapter分离失败案例-总结"
subtitle:   "通过两种案例对比Adapter 在MVP模式下的处理方法，并分析此案例失败的原因"
date:       2018-12-03 22:56:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- Android
---

## 简介

本文介绍了Adapter在MVP模式中的使用方式
首先Adapter主要是用作数据展示的作用，通常在Fragment或者Activity中嵌套使用。

## 尝试

我在参考了一些资料后做出如下尝试，这里以联系人列表为例子。

1. adapter接收P层，viewHolder作为V层,通过onBindViewHolder绑定在一起
2. 数据保存在P层并监听数据源的更新
3. 在AdapterContract定义要渲染数据的数据格式.按条目渲染Item.
4. 将Item的事件控制在P层
5. 将ListView的显示等交给它所在父控件。

代码如下：

```java

BaseAdapterPresenter.java

public interface BaseAdapterPresenter<T> {


    void start();

    /**
     * 加载数据
     */
    void loadData();

    /**
     * 获取数据个数
     */
    int getDataCount();

    /**
     * 销毁
     */
    void onDestroy();

    /**
     * 获得渲染对象
     *
     * @param index
     * @return
     */
    T getItemDraw(int index);
}


```
```java
BaseAdapterView.java

public interface BaseAdapterView<T> {

    void onDataUpdate();

    /**
     * 渲染的Item
     *
     * @param itemDraw
     * @param position
     */
    void renderItem(T itemDraw, int position);
}

```

```java
ContactsAdapterContract.java

public class ContactsAdapterContract {
    /**
     * 渲染数据封装对象
     */
    public static class ContactItemDraw {
        public boolean showSection;
        public String section;
        public String header;
        public String name;
        public String number;
        public boolean showCloud;
    }

    public interface View extends BaseAdapterView<ContactItemDraw> {

        /***
         *  掉起xx界面
         */
        void showXXActivity(String args);
    }

    public interface Presenter extends BaseAdapterPresenter<ContactItemDraw> {
        /**
         * 跳转至显示联系人详情界面
         *
         * @param mContext
         * @param position
         */
        void startContactsDetailsActivity(Context mContext, int position);


        void onBindContractsRecyclerView(ContactsAdapterContract.View view, int position);

        /**
         * 其他事件
         *
        */
        
        ....
    }
}
```

```java

public class ContractsAdapterPresenter implements ContactsAdapterContract.Presenter {

    private ContactsSync.ContactsWrapper mContactsWrapper;

    private ContactsAdapterContract.View mView;

    @Subscribe(threadMode = ThreadMode.MAIN)
    public void onContactEvent(ContactEvent event) {
        switch (event.type) {
            case ContactEvent.EVENT_READ_ANDROID_CONTACTS:
                loadData();
                break;
            default:
                break;
        }
    }

    @Override
    public void loadData() {
        mContactsWrapper = ContactsSync.getContactsWrapper();
        //todo 数据改变通知上层
    }

    @Override
    public void start() {
        EventBus.getDefault().register(this);
        loadData();
    }

    @Override
    public void onDestroy() {
        EventBus.getDefault().unregister(this);
    }

    @Override
    public ContactsAdapterContract.ContactItemDraw getItemDraw(int index) {
        ContactsAdapterContract.ContactItemDraw item = new ContactsAdapterContract.ContactItemDraw();
        ContactTool.AndroidContact androidContact = mContactsWrapper.contacts.get(index);
            item.header = androidContact.photoUri;
            item.name = androidContact.name;
            item.number = androidContact.phone;
            item.showCloud = false;
            return item;
        }
        return null;
    }

    @Override
    public void startContactsDetailsActivity(Context mContext, int position) {
        ContactsDetailsActivity.launch(mContext, getItemDraw(position).number);
    }

    @Override
    public void onBindContractsRecyclerView(ContactsAdapterContract.View view, int position) {
        mView = view;
        mView.renderItem(getItemDraw(position), position);
    }

    @Override
    public int getDataCount() {
        return mContactsWrapper != null ? mContactsWrapper.contacts.size() : 0;
    }

}

```


```java
public class ContactsRecyclerViewAdapter extends RecyclerView.Adapter<ContactsRecyclerViewAdapter.ViewHolder> {

    private ContactsAdapterContract.Presenter mPresenter;
    private Context mContext;

    public ContactsRecyclerViewAdapter(Context context, ContactsAdapterContract.Presenter presenter) {
        mContext = context;
        mPresenter = presenter;
        mPresenter.start();
    }

    @Override
    public ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        View view = LayoutInflater.from(parent.getContext())
                .inflate(R.layout.view_contact, parent, false);
        return new ViewHolder(view);
    }

    @Override
    public void onBindViewHolder(final ViewHolder holder, int position) {
        mPresenter.onBindContractsRecyclerView(holder, position);
    }

    @Override
    public int getItemCount() {
        return mPresenter.getDataCount();
    }

    public class ViewHolder extends RecyclerView.ViewHolder implements ContactsAdapterContract.View {

        @BindView(R.id.txt_contact_item_section)
        TextView mSection;
        @BindView(R.id.txt_contact_item_name)
        TextView mName;
        @BindView(R.id.txt_contact_item_number)
        TextView mNumber;
        @BindView(R.id.img_cloud)
        ImageView mCloud;
        @BindView(R.id.img_avatar)
        ImageView mAvatar;
        @BindView(R.id.txt_avatar_name)
        TextView mAvatarName;

        private int mCurrentPosition;

        public ViewHolder(View view) {
            super(view);
            ButterKnife.bind(this, view);
        }

     
        @OnClick(R.id.view_avatar)
        void onClickHead(View view) {
            mPresenter.startContactsDetailsActivity(mContext, mCurrentPosition);
        }

        @OnClick(R.id.view_contact_item)
        void onClickCallItem(View view) {
            mPresenter.call(mCurrentPosition);
        }

        @Override
        public void onDataUpdate() {

        }

        @Override
        public void renderItem(ContactsAdapterContract.ContactItemDraw item, int position) {
            mCurrentPosition = position;
            if (item != null) {
                mSection.setVisibility(item.showSection ? View.VISIBLE : View.GONE);
                mSection.setText(item.section);
                mName.setText(item.name);
                mNumber.setText(item.number);
                mCloud.setVisibility(item.showCloud ? View.VISIBLE : View.INVISIBLE);
                if (item.header != null) {
                    mAvatarName.setVisibility(View.INVISIBLE);
                } else {
                    mAvatarName.setVisibility(View.VISIBLE);
                    if (item.name.length() >= 2) {
                        mAvatarName.setText(item.name.substring(item.name.length() - 2, item.name.length()));
                    } else {
                        mAvatarName.setText(item.name);
                    }
                    mAvatar.setImageDrawable(CircleDrawables.getDefault());
                }
            }
        }
    }
}

```

## 分析

经过大佬指点，做出以下结论：

1. 将adapter和宿主界面拆分开会导致接口类爆炸。
2. adapter功能简单，主要功能就是获取数据和展示数据，抽取后接口不多。
3. 事件交给宿主界面处理,将list和宿主作为整体即可，没有必要进行拆分。
4. 宿主V层控制Adapter的P层，逻辑错误。
5. 不利于平台逻辑代码移植

## 最终处理方案

将数据层封装到宿主界面的P层，由adapter 自己拿数据渲染。
