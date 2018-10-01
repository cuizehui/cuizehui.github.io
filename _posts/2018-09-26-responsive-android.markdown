---
layout:     post
title:      "接口自动化测试(二)--socket模块"
date:       2018-09-26 12:56:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
- AndroidSDK-Test
---

# 接口自动化测试(二)--socket模块

## socket模块

### 需求

- 连接服务端
- 读数据
- 写数据
- 处理数据粘包

**由于程序轻量，采用BIO方式**

### BIO特点

1. inputStream read
2. socket.accept 
两个方法都是线程阻塞式。 
 
### 读写线程通讯

由于write和read均不能触发在主线程。
解决思路是将读写线程分开，并创建一个工作线程，统一由线程池开启。通过handler机制传递消息。
由于handler是队列管理，所以不存在消息顺序错乱等情况。

```
 Runnable mWriteTask = new Runnable() {
        @SuppressLint("HandlerLeak")
        @Override
        public void run() {
            Looper.prepare();
            mWriteHandler = new Handler() {
                @Override
                public void handleMessage(Message msg) {
                    switch (msg.what) {
                        case SERVICE_SEND_COMMAND:
                            Bundle bundle = msg.getData();
                            String data = bundle.getString("sendData");
                            Log.d("WriteTask", data.toString());
                            mTcpUtils.sentData2Server(data);
                            break;
                    }
                }
            };
            Looper.loop();
        }
    };
``` 

```
Runnable mReadTask = new Runnable() {
        @Override
        public void run() {
            mTcpUtils = new TcpUtils();
            Boolean connectResult = mTcpUtils.connectService(mHost, mPort);
            if (connectResult) {
                boolean flag = true;
                while (flag) {
                    String content = mTcpUtils.getDataFromService();
                    if (content != null) {
                        Log.d("ReadTask: ", content.toString());
                        Message message = buildMessage(SERVICE_RECEIVE_COMMAND, "ReceiveData", content.toString());
                        mWorkHandler.sendMessage(message);
                        // mWorkHandler.sendMessage();
                        // mWriteTask.sendMessage();
                    }
                }
            } else {
                Log.d(TAG, "连接失败");
            }
        }
    };
```

工作线程：

```
Runnable mWorkTask = new Runnable() {
        @SuppressLint("HandlerLeak")
        @Override
        public void run() {
            Looper.prepare();
            mWorkHandler = new Handler() {
                @Override
                public void handleMessage(Message msg) {
                    CommandUtils commandUtils = new CommandUtils();
                    switch (msg.what) {
                        case SERVICE_RECEIVE_COMMAND:
                        ....
                    }
                }
            };
            Looper.loop();
        }
    };
```
  
线程池：

```
    public void startTask(Runnable myTask) {
        mExecutor = new ThreadPoolExecutor(5, 10, 200, TimeUnit.MILLISECONDS,
                new ArrayBlockingQueue<Runnable>(5));
        mExecutor.execute(myTask);
    }
```  
### 判读连接是否断开
 
**方案：**  轮询,发送一段数据，根据返回值，判断socket是否断开。
 
### 粘包处理  

因为，数据是以流的形式传递的，所以不存在数据"包"。那么包就是我们自行进行装包和拆包的。

#### 处理数据粘包的几种方案

1. 可在头部约定数据包大小关流 
2. 约定有效数据的结尾字段


#### 根据 /r/n 处理数据粘包

假设我们发送的命令以 /r/n 结尾为一个命令。

1.阻塞读流
2.定义缓冲区，将读到的数据放入缓冲区
3.判断缓冲区有无一个完整命令如果有则依次return

这里我们简单处理/r/n 就是readLine

同样在发包时增加 /r/n 并替换所有数据包内/r/n为 空格

```
public class TcpUtils {

    private final int CONNECT_TIME_OUT = 30000;

    public final  String TAG=TcpUtils.class.getSimpleName();

    private Socket client;

    private InputStream mInputStream;
    private InputStreamReader mInputStreamReader;
    private BufferedReader mBufferedReader;

    public Boolean connectService(final String adress, final int port) {
        try {
            client = new Socket(adress, port);
            client.setSoTimeout(CONNECT_TIME_OUT);
            if(isConnected()){
                initReader();
                return true;
            }else {
                return false;
            }

        } catch (IOException e) {
            e.printStackTrace();
            return false;
        }
    }

    private void initReader() throws IOException {
        mInputStream = client.getInputStream();
        mInputStreamReader = new InputStreamReader(mInputStream);
        mBufferedReader = new BufferedReader(mInputStreamReader);
    }

    //判断是否连接
    public boolean isConnected(){
        try{
            client.sendUrgentData(0xFF);
            return true;
        }catch(Exception e){
            return false;
        }
    }


    //阻塞方法
    public String getDataFromService() {
        String content = null;
        try {
            if (mInputStream != null) {
                content = mBufferedReader.readLine();
            } else {
                Log.d(TAG, "InputStream  获取失败");
            }
        } catch (IOException e) {
            e.printStackTrace();
            return content;
        }
        return content;
    }

    public void sentData2Server(String data) {
        if(TextUtils.isEmpty(data)){
            return;
        }
         data=data.replaceAll("\r\n","");
        try {
            OutputStream outputStream = client.getOutputStream();
            if (outputStream != null) {
                BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(outputStream));
                data = data + "\r\n";
                writer.write(data);
                writer.flush();
                outputStream.flush();
            } else {

            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

```

## NIO
在编写之前，了解到NIO相关技术。
使用新的方法处理输入输出， 和input outPut 的区别在于，提供了map 方法，如果传统是面向流，那么新io 则是面向块。
nio 最关键的部分是由阻塞读流，改为读状态，而读取状态不是真正的占用CPU。只有读流写流是占用cpu的而且时间很短。所以定义NIO为非阻塞。


## 参考文章

[https://www.cnblogs.com/QG-whz/p/5537447.html](https://www.cnblogs.com/QG-whz/p/5537447.html)
[https://www.jianshu.com/p/3ff7065bb38b](自定义BufferReader)
[http://guguzhang.iteye.com/blog/2253218](http://guguzhang.iteye.com/blog/2253218)