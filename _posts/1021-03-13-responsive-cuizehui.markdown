---
layout:     post
title:      "面试总结"
date:       1021-03-13 20:58:00
author:     "Nela"
header-img: "img/post-bg-rwd.jpg"
tags:
    - Nela
---

# 2021年面试题

## Java 基础
 
- 类加载 

    - 类加载双亲委托机制classloader? 
    
        双亲委任机制目的好处
        
        1. 共享功能，一些Framework层级的类一旦被顶层的ClassLoader加载过就缓存在内存里面，以后任何地方用到都不需要重新加载。 
        2. 父类加载器加载的类，不给子类进行加载,具有隔离功能，保证java/Android核心类库的纯净和安全，防止恶意加载。
       
        三种类加载器
        1. Bootstarp类加载器(该类加载器使用C++语言实现，属于虚拟机自身的一部分)加载JAVAHome下的jar包等，此过程无法引用操作
        2. Extension加载扩展包内容ext,
        3. Application应用程序类加载器,负责将系统类库加载到内存中。
        4. DexClassLoader：支持加载外部的APK、Jar或dex文件；（所有的插件化方案都是使用它来加载插件APK中的.class文件，也是动态加载的核心依据！
        
        双亲委派机制的工作流程
        ClassLoader首先从自己已经加载的类中检查是否此类已经加载(loadClass)，如果已经加载则直接返回原来已经加载的类。没加载则交给父类加载起进行检查。当没有父类后开始进行加载。
        父类没加载到调用子类加载器方法（findClass）加载。
               
       - 延伸问题

           1. 假如我们自己写了一个java.lang.String的类，我们是否可以替换调JDK本身的类？
                
                你完全可以自己写一个classLoader来加载自己写的java.lang.String类，但是你会发现也不会加载成功，具体就是因为针对java.*开头的类，jvm的实现中已经保证了必须由bootstrp来加载。
          
           2. 什么情况下打破双亲委托机制？
           
                可以通过动态加载新的类。如果你希望通过动态加载的方式，加载一个新版本的dex文件，使用里面的新类替换原有的旧类，从而修复原有类的BUG，那么你必须保证在加载新类的时候，旧类还没有被加载，因为如果已经加载过旧类，那么ClassLoader会一直优先使用旧类。
                如果旧类总是优先于新类被加载，我们也可以使用一个与加载旧类的ClassLoader没有树的继承关系的另一个ClassLoader来加载新类，因为ClassLoader只会检查其Parent有没有加载过当前要加载的类，如果两个ClassLoader没有继承关系，那么旧类和新类都能被加载。
           
           3. 类名包名相同加载不同是同一个类吗？
              
               不是
  
- 集合
    - java底层集合怎样实现？
        
        ArrayList 底层使用数组，并允许包括null在内的所有元素 索引查找
        linkedList是List接口的双向链表非同步实现，并允许包括null在内的所有元素 它的查找是分两半查找
        HashMap是基于哈希表的Map接口的非同步实现，允许使用null值和null键，但不保证映射的顺序 HashMap最多只允许一条Entry的键为Null(多条会覆盖)，但允许多条Entry的值为Null。
        HashMap实际是一种“数组+链表”数据结构。在put操作中，通过内部定义算法寻止找到数组下标，将数据直接放入此数组元素中，若通过算法得到的该数组元素已经有了元素（俗称hash冲突，链表结构出现的实际意义也就是为了解决hash冲突的问题）。将会把这个数组元素上的链表进行遍历，将新的数据放到链表末尾。
        LinkedHashMap  ashMap 的不同之处在于，LinkedHashMap 维护着一个运行于所有条目的双向链接列表 该 Entry 除了保存当前对象的引用外，还保存了其上一个元素 before 和下一个元素 after 的引用
   
    - hashMap数据结构
        - hash是什么？
            - 任意长度的输入通过hash算法等到固定长度的输出
            
        - node节点        
            - hash 
            - key
            - next
            - value
            
        - 数组（2的次方）+链表长度大于8会树化为红黑树（平衡查找树）
        - hashMap散列表数组初始长度16。put数据才创建。
        - 默认负载因子（扩容因子）0.75 扩容阈值 上一次的长度左一一位。扩容迁移，如果有链表，则高低位迁移。
        - 链表转红黑树条件 链表长度大于8& 散列表数组长度大于64。
        - hashcode返回值是keyhashCode 高16位异或低16位 按位与table16位-1得到下标
        - 哈希冲突后先查找node进行replace如果未匹配，则采用尾插法
        - 红黑树写入操作
   
- 静态变量,静态代码块,构造代码块,构造函数调用顺序

    - 优先于
    - 静态代码块其实就是给类初始化的，而构造代码块是给对象初始化的 
    
### 反射

-  如何引用FrameWork非公开API,如何HooK 系统API
 
   反射，反射方法的类加载器是系统的类加载器，自然就可以调用系统方法
   继承重新系统类 替换系统成员变量
   
   
      
-----  
  
### 设计模式

- 单例
    - 直接new ，不够精准还没getInstance就new
    - getInstance时判断。多线程操作，不能保证只new一个
    - 精准锁住对象，并且锁前锁后进行两次判空，防止获得锁后再次创建

- 动态代理

    - InvocationHandler类,实现invoke方法，绑定要代理的对象
    
    - 静态类能被代理吗？

------

## 算法

<!--- 二分查找时间复杂度 O(logn)?-->
- 无环链表查找中间节点

快指针一次前进两个结点，速度是慢指针的两倍。这样当快指针到达链表尾部时，慢指针正好到达的链表的中部

- 快排写一下，动态规划了解吗？

```java
int AdjustArray(int s[], int l, int r) //返回调整后基准数的位置
{
    int i = l, j = r;
    int x = s[l]; //s[l]即s[i]就是第一个坑
    while (i < j)
    {
        // 从右向左找小于x的数来填s[i]
        while(i < j && s[j] >= x) 
            j--;  
        if(i < j) 
        {
            s[i] = s[j]; //将s[j]填到s[i]中，s[j]就形成了一个新的坑
            i++;
        }
 
        // 从左向右找大于或等于x的数来填s[j]
        while(i < j && s[i] < x)
            i++;  
        if(i < j) 
        {
            s[j] = s[i]; //将s[i]填到s[j]中，s[i]就形成了一个新的坑
            j--;
        }
    }
    //退出时，i等于j。将x填到这个坑中。
    s[i] = x;
 
    return i;
}
```

```java
void quick_sort1(int s[], int l, int r)
{
    if (l < r)
    {
        int i = AdjustArray(s, l, r);//先成挖坑填数法调整s[]
        quick_sort1(s, l, i - 1); // 递归调用 
        quick_sort1(s, i + 1, r);
    }
}
```

- 二叉树层序遍历

使用队列，父节点入队,如果无子节点,则直接出队,有子节点子节点入队后头节点出队。

```java
public Node int(){
    Que

}
```

- 反转字符串

```java
public String solve (String str) {
        // write code here
       StringBuilder stringBuilder = new StringBuilder();
            for (int i = str.length()-1; i >=0 ; i--) {
                stringBuilder.append(str.charAt(i));
            }
            return stringBuilder.toString();
    }
```

- 反转链表

我们必须在指针更改指向之前，保存修改结点的下一结点

```java
/*
public class ListNode {
    int val;
    ListNode next = null;

    ListNode(int val) {
        this.val = val;
    }
}*/

public class Solution {
    public ListNode ReverseList(ListNode head) {
        //前一个节点
        ListNode pre =null;
        //当前节点
        ListNode cur =null;
        //断开后的临时节点
        ListNode temp =null;
           
        cur = head;
        while(cur !=null){
            //保存后一个节点
            temp = cur.next ;
            //改变指针方向
            cur.next=pre;
            //pre节点后移
            pre = cur;
            //cur节点后移
            cur = temp;  
        }
        return pre;
    }
}
```

- 求二叉树第k层节点个数？

```java
   1
  3 2
1 2 4 4
public void inOrder(TreeNode node,Array<TreeNode> treeNode){

    if(node.left==null){
       return;
    }
    //左孩子有值则进入左子树
    inOrder(node.left,treeNode);
    //当前节点左孩子无值，则当前节点入队
    treeNode.add(node);
    //检查右节点
    inOrder(node.right,treeNode);
}
```

- 二叉搜索树 比左节点大，比右节点小。

采用中序遍历既可以排队（左根右）

- 1个非空数组，除了有一个元素只出现一次，其他都出现两次。 找到只出现一次的元素?

```
class test {
    //1个非空数组，除了有一个元素只出现一次，其他都出现两次。 找到只出现一次的元素
    public  static  int findOne(int[] array) {
        int key = array[0];
        for (int i = 1; i < array.length; i++) {
            key = key ^ array[i];
        }
        return key;
    }

    public static void main(String[] args) {
        int[] a =new int[]{1,3,1,5,3};
        int key = findOne(a);
        System.out.println(key);
    }

}
```

- 两个链表实现一个队列？

----

## 计算机网络

### HTTP与HTTPS的区别以及如何实现安全性?

- https如何保证安全？
  传送密钥 tls加密
  1. 服务器同时具有公钥和私钥
  2. 服务器给客户端发送公钥（通过数字证书包装防止公钥被伪造）
  3. 客户端用公钥加密对称密钥
  4. 服务端用私钥解密对称密钥
  5. 后续使用对称密钥通信

- 为什么用udp

UDP不提供可靠性，它只是把应用程序传给IP层的数据报发送出去，但是并不能保证它 们能到达目的地。
由于UDP在传输数据报前不用在客户和服务器之间建立一个连接，且没有超时重发等机制，故而传输速度很快。 
  
- tcp

TCP的目的是提供可靠的数据传输，并在相互进行通信的设备或服务之间保持一个虚拟连接。TCP在数据包接收无序、丢失或在交付期间被破坏时，负责数据恢 复。
它通过为其发送的每个数据包提供一个序号来完成此恢复。记住，较低的网络层会将每个数据包视为一个独立的单元，
因此，数据包可以沿完全不同的路径发 送，即使它们都是同一消息的组成部分。
这种路由与网络层处理分段和重新组装数据包的方式非常相似，只是级别更高而已
  

----

## 数据库

### 数据库数据迁移问题？


### 数据库事务了解吗？

- 原子性批量执行
- 要么都成功要么都失败
- 两个事务没有影响

----

## Android-性能优化

### Bitmap

- bitmap内存计算方式以及优化
    
    图片的大小不等与加载进内存的大小
    内存计算bitmap.getByteCount()
    
    压缩第一步
    宽高缩放
    1通过bitmapfactory 
    
    果将options.inJustDecodeBounds设置为true，在解码过程中就不会申请内存去创建Bitmap，返回的是一个空的Bitmap，但是可以获取图片的一些属性，例如图片宽高，图片类型等等
    
     质量压缩不会改变所占内存大小，但是方便将图片另存在一份在本地磁盘
     
    基本
    分辨率* 像素点的大小
    
    加载进不同目录，像素会进行一次转换，高分辨率会变大。
    
    只有在Res目录才会做转换，在磁盘中大小就是 分辨率* 像素点的大小
    
    减少像素点所占字节方法：
    
    ARGB_8888：
    总共32位（4byte），分别对应4个数值，数值单位为8bit位=1byte字节，分别描述透明度（1个）+RGB通道（3个）。每个字节数值范围0-255。作为Bitmap配置色彩空间的默认值。BitmapFactory加载时默认。
    public Bitmap.Config inPreferredConfig = Bitmap.Config.ARGB_8888
    RGB_565：总共16位（2byte），分别对应3个数值，5位（红）+6位（绿）+5位（蓝）分别描述RGB通道。Glide加载时默认使用，DecodeFormat类
    public static final DecodeFormat DEFAULT = PREFER_RGB_565，可以看到，RGB_565只需要ARGB_8888的一半大小，代价是没有透明度描述
    
    优化1。图片本身就要降低分辨率 通过缩放的方式，减少像素点所占大小
    
    图片外手动清理
    
    https://www.cnblogs.com/dasusu/p/9789389.html
  
- 为什么不建议在广播中启动一个线程做任务？

    onReceive方法返回后，进程和线程处于等待状态；系统任意时刻可以终止和回收该线程和进程占有资源（内存）
    可以在广播中开启服务处理。

### 内存优化
    
- 内存抖动会导致卡顿
- 尽量不要在频繁调用的方法中创建对象
- Message对象池 对象池的设计
- 使用弱引用 软引用
- 匿名内部类持有外部引用

    - 对象池复用原理

- 内存泄漏如何检测？

- jni层垃圾如何回收？

### 启动项优化 

- 3方APP如何启动优化？
    1.启动过程中减少系统调用，避免与 AMS、WMS 竞争锁。启动过程中本身 AMS 和 WMS 的工作就很多，且 AMS 和 WMS 很多操作都是带锁的，如果此时 App 再有过多的 Binder 调用与 AMS、WMS 通信，SystemServer 就会出现大量的锁等待，阻塞关键操作
  
### 滑动卡顿  

    profile工具

- 原因： 程序的大多数操作都必须在16ms内完成，掉贞
- 排查方向

    1. 主线程耗时操作必须在16ms内完成，掉贞 使用TraceView工具 onBindViewHolder/onScrollChanged
            
            ```
             Debug.startMethodTracing();
            ```
    
    2. 内存抖动导致频繁GC
    3. 布局层级过多 Hierarchy Viewer排查 绿色表示OK，黄色表示其处于渲染速度比较慢的50%，红色表示渲染速度非常慢。
 
 
### Fragment优化

- ViewPager + fragment 懒加载实现方案
  1. viewPager 缓存5页
  2. ViewPager2加载Fragment使用了新的适配器FragmentStateAdapter,在新的FragmentStateAdapter中已经对Fragment的是否可见对FragmentTransaction. setMaxLifecycle(fragment, Lifecycle.State.STARTED)设置了不同的参数。所以我们可以放心的在onResume方法中进行懒加载
  3. 后台到前台如何处理？
  4. 嵌套Fragment如何处理？
  
----


###  App启动流程，从点击桌面开始？

1. Luacher IPC startActivity
2. AMS -> Socket fork 进程
3. ActivityThread.main()方法，ActivityThread随后依次调用Looper.prepareLoop()和Looper.loop()来开启消息循环。
4. attachApplication
5. ApplactionThread 注册的binder中。
6. 最终进入ActivityThread的main方法


-----  
                    
### 保活

- tcp链路保活
    
    使用定时器进行保活处理，如果保活失败重连。
    使用自己的定时器可能存在进程休眠导致保活失败。
    使用Alarm定时器可能因为对齐导致保活失败。
    注册广播启动定时器，到期后发送Intent触发定时器任务
    
    保证TCP长链接，发送pingpong 使用Alarm唤醒CPU
    
  - Android的Alarm对齐唤醒机制？  
  
- 进程保活
    
  如果进程挂了则设置进程自启动。
  bindService 短信发送广播拉起Service应用
  创建一个前台服务用于提高 app 


## 其他

- ActivityA ActivityB ActivityC 如何传递消息？
    1. Interface接口

- App 是如何沙箱化，为什么要这么做？

- 下拉状态栏是不是影响activity的生命周期，如果在onStop的时候做了网络请求，onResume的时候怎么恢复？

- 混淆后的问题如何定位？
    
- APK的编译流程？    

- 项目最大的困难，如何解决？

- SDK的技术点能给对方带来什么？

- SDK开发如何接口如何兼容老版本？需要做那些处理

- 和厂商对接遇到哪些问题，什么难以解决的问题？

    - 集成阶段 （看系统日志）
    
        权限不足导致初始化失败
        应用未加白名单取不到IMSI
        
    - 调试阶段 （解决方案看SDK日志）
    
        消息发送失败，错误兼容处理，消息重发机制
        ANR问题 （binder线程卡顿）
        Crash问题 SD未挂载导致文件失败
        登陆失败 广播上报异常导致流程失败
            
    - SDK版本适配升级    
        
        1. 沙盒版本适配
        2. 本地广播需要指定报名
        3. UI升级业务开发
        4. 接口化设计（large /Page接口底下内部处理）
        
    - 功耗问题 
        
        Alarm频繁唤醒导致功耗过高
        SDK未启动，就传输流量，
        
        
-----

# 线程并发 

## 线程同步
                     
## 锁

悲观锁适合写操作多的场景，先加锁可以保证写操作时数据正确。 synchronized/ReentrantLock

乐观锁适合读操作多的场景，不加锁的特点能够使其读操作的性能大幅提升。 AtomicInteger

### 自旋锁

自旋锁，请求锁的线程不放弃CPU的执行时间，看看持有锁的线程是否很快就会释放锁 通过CAS实现

 1. 自旋锁 CAS Compare and Swap 
 2. 自旋锁处理A B A 问题 加版本号

### synchronized

无锁 VS 偏向锁 VS 轻量级锁 VS 重量级锁 都是针对synchronized的

- 一个对象的内存布局  
        - markword 8字节的头 class 内容字节 占位符
        - markword 锁信息，GC信息，hashcode
                 
- synchronized锁定的到底是什么？
    -  synchronize实际上锁定的是对象的markword内容
        没人竞争时是偏向锁 （会在Mark Word里存储锁偏向的线程ID。在线程进入和退出同步块时不再通过CAS操作来加锁和解锁，而是检测Mark Word里是否存储着指向当前线程的偏向锁）
        有人竞争是轻量级锁CAS CAS操作尝试将对象的Mark Word更新为指向Lock 
        有多人竞争则使用重量锁（排队机制）此时等待锁的线程都会进入阻塞状态。
- 偏向锁和轻量级锁
    - 都不提供线程互斥
    - 轻量级锁主要是为了解决线程交替使用的情况，避免过早申请重量级锁
    
- 加锁基本步骤
    - 线程栈内增加锁记录
    - 修改锁markWord字段
    - 锁记录记录markWord字段       
 - synchronized可以锁重入        

- sleep和wait区别？
        - sleep 会让出CPU资源不会释放锁。时间到了会执行后续
        - wait  1.会释放锁会释放资源,2.notify后不一定能拿到CPU资源只是处于就绪状态。3.wait配合synchronize同步代码块使用

### ReentrantLock（可重入锁）

ReentrantLock里面有一个内部类Sync，Sync继承AQS（AbstractQueuedSynchronizer）

- 可重入概念
  - 单个线程执行时，重新进入同一个子程序仍是线程安全的，需要累加重入的次数，释放时也释放相同的次数

- 公平锁和非公平锁

- sync继承AQS
  
   排队队列为先进先出
   
    ```java
     public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
    ```

## 线程池

- 线程池的原理?

    corePoolSize=> 线程池里的核心线程数量
    maximumPoolSize=> 线程池里允许有的最大线程数量
    keepAliveTime=> 空闲线程存活时间
    unit=> keepAliveTime的时间单位,比如分钟,小时等
    workQueue=> 缓冲队列
    threadFactory=> 线程工厂用来创建新的线程放入线程池
    handler=> 线程池拒绝任务的处理策略,比如抛出异常等策略

    - newCachedThreadPool 线程创建，导致CPU资源消耗过多
    - newFixedThreadPool 核心线程10，多出任务加入linklockedQueue。允许请求的最大等待队列为Integer.MAX_VALUE，导致OOM原因队列中 
    - newSingleThreadExecutor 单核线程池 多出任务加入linklockedQueue。允许请求的最大等待队列为Integer.MAX_VALUE，导致OOM原因队列中 
   
    newFixedThreadPool和 newSingleThreadExecutor虽然指定了最大核心线程数，但是等待队列没限制。
    newCachedThreadPool 最大线程数，频繁创建，等待队列等于没有。
    
- 线程池的几个核心参数，超过线程池个数了如何处理？

    - AbortPolicy  线程池队列满了丢掉这个任务并且抛出RejectedExecutionException异常。 如果是比较关键的业务，推荐使用此拒绝策略，这样子在系统不能承载更大的并发量的时候，能够及时的通过异常发现。
    - DiscardPolicy 如果线程池队列满了丢掉这个任务并且抛出RejectedExecutionException异常。  建议是一些无关紧要的业务采用此策略。
    - DiscardOldestPolicy 如果队列满了，会将最早进入队列的任务删掉腾出空间，再尝试加入队列。 是否要采用此种拒绝策略，还得根据实际业务是否允许丢弃老任务来认真衡量。
    - CallerRunsPolicy 如果添加到线程池失败，由调用线程处理该任务
   
自定义拒绝策略 
 
```
public class MyRejectPolicy implements RejectedExecutionHandler{
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        //Sender是我的Runnable类，里面有message字段
        if (r instanceof Sender) {
            Sender sender = (Sender) r;
            //直接打印
            System.out.println(sender.getMessage());
        }
    }
}
```    

https://zhuanlan.zhihu.com/p/112527671

## ConcurrentHashMap为什么线程安全

ConcurrentHashMap 是如何保证线程安全的？

- 为什么快？
- put时内部使用CAS处理
- get操作全程不需要加锁;
- Node的元素val和指针next是用volatile修饰的

## CountDownLatch源码原理

- 分段下载一个文件如何处理？ 

    分段下载处理 具体细节 CountDownLatch
    https://juejin.im/post/6844904013440221198
 
CountDownLatch基于AQS实现，volatile变量state维持倒数状态，多线程共享变量可见。

CountDownLatch通过构造函数初始化传入参数实际为AQS的state变量赋值，维持计数器倒数状态

当主线程调用await()方法时，当前线程会被阻塞，当state不为0时进入AQS阻塞队列等待。

其他线程调用countDown()时，state值原子性递减，当state值为0的时候，唤醒所有调用await()方法阻塞的线程

## volatile

1. 保持线程可见行
   
    - 解决A线程修改了B线程不知道的情况，每次都从内存中读一边

2. 防止指令重排序
    
    - 解决CPU乱序执行问题（乱序执行无联系指令，提高效率）
    - 使用内存屏障方式防止指令重排
    
## ThreadLocal    

- 因为每个 Thread 内有自己的实例副本，且该副本只能由当前 Thread 使用。这是也是 ThreadLocal 命名的由来
- ThreadLocal 适用于变量在线程间隔离且在方法间共享的场景
- 每个线程持有一个 Map 并维护了 ThreadLocal 对象与具体实例的映射，该 Map 由于只被持有它的线程访问，故不存在线程安全以及锁的问题
- ThreadLocalMap 的 Entry 对 ThreadLocal 的引用为弱引用，避免了 ThreadLocal 对象无法被回收的问题

## 如何实现一个简单的死锁？

## 如何按照顺序依次输出1A2B3C4D

```
    Array 1 = 123456
    Array 2 = "abcdef" ;
    CountDownLatch countDown =new CountDownLatch(1);
    
    Object o = lock()l
    Thread 1 =new Thread(){
     sync(lock){ 
         for (String 1:1){ 
            countDownLatch.countDown();
            print 1;
            o.notify();
            o.wait();
         }
          o.notify();
     }
    }
    
    Thread 2 =new Thread(){
     sync(lock){ 
         for (String 2:2){
            countDown.await();
            print 2;
            o.notify();
            o.wait();   
         }
        o.notify();
     }
    }
 ```
 
wait 和notify 无法精确叫醒

Lock和condition可以实现

```
    Lock lock =new ReentrantLock();
    Condition condition1=lock.newCondition();
    Condition condition2=lock.newCondition();
    
     Thread 1 =new Thread(){
      lock.lock();
         for (String 1:1){ 
            print 1;
            condition2.signal();
            condition1.wait();

         }
          condition2.signal();
    }
    
    Thread 2 =new Thread(){
     lock.lock();
     for (String 1:1){ 
            print 1;
            condition1.signal();
            condition2.wait();
         } 
          condition1.signal();
    }
```

##  控制线程数量 

- 如何判断当前应用线程数量？
 
---

# 开源

## EventBus 

- EventBus
    发布-订阅模式。

    subscriptionsByEventType： 一个Map
    key： 事件类型，如本例 MsgEvent.class
    value： 一个按照订阅方法优先级排序的订阅者的列表集合
    1. register时将方法添加到EventBus管理
    3. 发布时根据Event类型寻找订阅者
    2. 根据注解拿到方法(处理线程所在)
    3. 获取事件 


- 不同注解的调用线程？

POSTING（在调用post所在的线程执行回调）：不需要poster来调度，直接运行。
MAIN（在UI线程回调）：如果post所在线程为UI线程则直接执行，否则则通过mainThreadPoster来调度。
BACKGROUND（在Backgroud线程回调）：如果post所在线程为非UI线程则直接执行，否则则通过backgroundPoster来调度。
ASYNC（交给线程池来管理）：直接通过asyncPoster调度。

- 如何知道当前线程是什么线程呢？
   
  通过比较当前线程的Looper是否是MainLooper判断当前是否是主线程。
  
- 如何实现线程切换？

    - 子线程切主线程
        通过mainThreadPoster调度。通过主线程的handler将事件投递到主线程执行,设置主线程执行10秒没排空消息则退出重启HandlerMessage
    - 主线程切子线程
        通过BackgroundPoster调度，线程池执行。
    - 异步任务无论怎样都会在子线程执行，适合网络请求。不会判断当前线程主/子。

- 粘性事件post时加入单独队列，注册时会检查粘性事件的存在。    


## Glide

### LRUCache原理？ 

LinkedHashMap ：

hashMap+ 双向链表 实现时间复杂度为O1
1. hashMap解决取数据O1问题
2. 双线链表解决数据后移到尾节点问题
3. 满了删调头节点
4. 使用数组计数可以实现无法达到O1


### LiveData

ViewModel

---

## JobScheduler 怎么实现的？

对于满足网络、电量、时间等一定预定条件而触发的任务，那么jobScheduler便是绝佳选择。
JobScheduler主要用于在未来某个时间下满足一定条件时触发执行某项任务的情况，那么可以创建一个JobService的子类，重写其onStartJob()方法来实现这个功能。

使用

```
 JobInfo.Builder builder = new JobInfo.Builder(SCHEDULE_ID, new ComponentName(JApplication.sContext, RcsSchedulerService.class));
        //设置最多延迟多久后执行，单位毫秒
        builder.setMinimumLatency(sNextScheduledTime);
        //设置最多延迟多久后执行，单位毫秒
        builder.setOverrideDeadline(sNextScheduledTime);
        PersistableBundle bundle = new PersistableBundle(1);
        bundle.putInt(RcsLoginManager.SCHEDULE_NUM, ++sScheduleNum);
        builder.setExtras(bundle);
        JobInfo ji = builder.build();
        JobScheduler js = (JobScheduler) JApplication.sContext.getSystemService(Context.JOB_SCHEDULER_SERVICE);
        js.schedule(ji);
```

```
 JobInfo jobInfo = new JobInfo.Builder(123, jobService) //任务Id等于123
         .setMinimumLatency(5000)// 任务最少延迟时间 
         .setOverrideDeadline(60000)// 任务deadline，当到期没达到指定条件也会开始执行 
         .setRequiredNetworkType(JobInfo.NETWORK_TYPE_UNMETERED)// 网络条件，默认值NETWORK_TYPE_NONE
         .setRequiresCharging(true)// 是否充电 
         .setRequiresDeviceIdle(false)// 设备是否空闲
         .setPersisted(true) //设备重启后是否继续执行
         .setBackoffCriteria(3000，JobInfo.BACKOFF_POLICY_LINEAR) //设置退避/重试策略
         .build();  
 scheduler.schedule(jobInfo);
```

使用
```

public class RcsSchedulerService extends JobService {

    private final static String TAG = RcsSchedulerService.class.getSimpleName();

    @Override
    public boolean onStartJob(JobParameters jobParameters) {
        Intent intent = new Intent(RcsUtils.ACTION_SCHEDULE_LOGIN);
        intent.setPackage(getPackageName());
        intent.putExtra(RcsLoginManager.SCHEDULE_NUM, jobParameters.getExtras().getInt(RcsLoginManager.SCHEDULE_NUM));
        RcsLoginReceiver.deal(this, intent);
        return false;
    }

    @Override
    public boolean onStopJob(JobParameters jobParameters) {
        return false;
    }
}

```

```
  // IJobScheduler implementation
        @Override
        public int schedule(JobInfo job) throws RemoteException {
            if (DEBUG) {
                Slog.d(TAG, "Scheduling job: " + job.toString());
            }
            final int pid = Binder.getCallingPid();
            final int uid = Binder.getCallingUid();
            final int userId = UserHandle.getUserId(uid);

            enforceValidJobRequest(uid, job);
            if (job.isPersisted()) {
                if (!canPersistJobs(pid, uid)) {
                    throw new IllegalArgumentException("Error: requested job be persisted without"
                            + " holding RECEIVE_BOOT_COMPLETED permission.");
                }
            }

            validateJobFlags(job, uid);

            long ident = Binder.clearCallingIdentity();
            try {
                return JobSchedulerService.this.scheduleAsPackage(job, null, uid, null, userId,
                        null);
            } finally {
                Binder.restoreCallingIdentity(ident);
            }
        }

 ```
 
JobServiceContext：作用是负责与JobService进行Binder通信已经管理job的生命周期，而关键桥梁是IJobService.Stub，两者关系是JobService是JobServiceContext的Binder服务端。执行流程简单如下：

JobScheduler框架将通过bindService()方式来启动该服务。因此，用户必须在应用程序中创建一个JobService的子类，并实现其onStartJob()等回调方法，以及在


#### 使用方式

先获取JobScheduler调度器的代理对象，要理解这个过程，那么就需要先看看JobSchedulerService的启动过程；
创建继承于JobService的对象，见小节3.2；
创建JobInfo对象，采用builder模式；
调用schedule()来调度任务，见小节3.3。
    
    
    ConnectivityController	注册监听网络连接状态的广播
    TimeController	注册监听job时间到期的广播
    IdleController	注册监听屏幕亮/灭,dream进入/退出,状态改变的广播
    BatteryController	注册监听电池是否充电,电量状态的广播
    AppIdleController	监听app是否空闲
   
  5个StateController很重要,会根据时机来触发 

#### 为什么使用JobScheduler
  
- 低功耗优化减少CPU的频繁唤醒
      
#### 和Alarm有什么区别    
 
AlarmManager+Broadcast

#### WorkManager

WorkManager能依据设备的情况，选择不同的执行方案。在API Level 23+，通过JobScheduler来完成任务，而在API Level 23以下的设备中，通过AlarmManager和Broadcast Receivers组合完成任务。但无论采用哪种方案，任务最终都是交由Executor来完成。


## 当前问题
1. 定时登陆不应该使用  JobScheduler因为是跨进程IPC调用影响性能
2. Alarm无法唤醒CPU导致消息发送失败，建议使用JobScheduler.定时任务不准，但CPU可保证处于活跃状态。并可以监听电量网络等改变。


# java垃圾回收 和 JMM内存模型

## jVM内存模型
    
- 栈
    - 每个线程会分配一块栈内存
    - 栈帧跳出内存销毁
    - 栈帧四部分（方法出口,局部变量表,操作数栈，动态链接）
    - 栈内对象存的是在堆中的地址值  

- 方法区    
    - 类元信息class类
    - 静态变量  同样是和产生联系                        

- 程序计数器    
    - 作用多线程知道执行到哪行

- 本地方法
    - native方法内存   

- 堆                    

- 如何产生栈内存溢出？

    - 递归
    - 栈帧过深
                           
##  垃圾回收机制
    
https://blog.csdn.net/weixin_30682043/article/details/113051854

- 如何识别什么是垃圾，有那些算法。
    1. 从GCRoot根（栈内变量，静态变量 本地方法栈）判断对象引用是否可达
    2. 引用计数存在一团垃圾的情况出现了根可达算法

-  弱引用,强引用,软引用？

    - 强引用当对象无指向时，会被GC
    
    - 软引用当内存不足的时候会被回收 会被回收当作缓存
    
    - 弱引用GC回收时，弱引用来引用短生命周期对象ThreadLocal()
        - 线程类的成员变量,ThreadLocal线程本地对象
        - ThreadLocal作用 ThreadLocal中包含map,当map的key是ThreadLocal对象，key使用的是弱引用
        - 避免当ThreadLocal=null时 map仍指向ThreadLocal导致泄漏。但是value仍存在内存泄漏情况。
        - ThreadLocal可以存大对象吗？
    - 虚引用 当一个虚引用对象被回收时,信息会被通知到虚引用队列中,虚引用get找不到对象。
    
        
- 垃圾回收器的算法知道吗？
     
    1. 标记清除 （缺点碎片化严重）
    2. copy算法一分为二只用一半。 将有用拷贝到一半，另一半擦除 （缺点占用空间,优点可以获得连续内存）
    3. 标记压缩,标记完顶上有用的，边清除边填补整理 （缺点效率较低）
 
- 垃圾收集发生的区域？ 
    
     程序计数器、虚拟机栈、本地方法栈三个区域随线程共存亡，栈中的每一个栈帧分配多少内存基本上在类结构确定下来时就已知
     而 Java 堆和方法区这两个区域则有显著的不确定性
   
- JVM的回收机制和分代策略？
       
    - 针对年轻代的回收方式？
        
        1. 年轻代采用copy算法
        2. 年轻代分成3个区，伊甸园区和两个幸存区
        3. 伊甸园区挑出不被回收的,放到其中一个幸存区
        4. 原有的幸存区复制到另一个幸存区
        
         Survivor 区相当于是 Eden 区和 Old 区的一个缓冲，类似于我们交通灯中的黄灯。
        Survivor 又分为2个区，一个是 From 区，一个是 To 区。
        每次执行 Minor GC，会将 Eden 区和 From 存活的对象放到 Survivor 的 To 区
        （如果 To 区不够，则直接进入 Old 区）。
                  
        
        如果Survivor 有2个区域，所以每次 Minor GC，会将之前 Eden 区和 From 区中的存活对象复制到 To 区域。
        第二次 Minor GC 时，From 与 To 职责兑换，这时候会将 Eden 区和 To 区中的存活对象再复制到 From 区域，
        以此反复。这种复制算法保证了S1中来自S0和Eden两部分的存活对象占用连续的内存空间，避免了碎片化的发生。
        
        当Survivor放满后放到老年代。

- 回收方法区？

    废弃的常量和不再使用的类型。 只要某个常量不再被引用，就会被清理。而判定一个类型是否属于“不再被使用的类”的条件就比较苛刻了
    类加载器已经被回收。所有实例都已经被回收。无法在任何地方通过反射访问该类的方法。     

- 本地方法区的内存怎么回收？JNI如何调Java 有什么要注意的吗？
    
    想要自动回收，必须依赖GC机制。但仅仅依靠现有的GC机制还不够。我们还需要考虑以下两点：

    1. 如何在native内存增长过多的时候 自动 触发GC？
    2. 如何在GC回收Java对象时 同步回收 native资源？
    
    8.0以后 使用NativeAllocationRegistry类
    
    NativeAllocationRegistry -内部使用虚引用得知对象被GC的时机，在GC前执行额外的回收工作
    因此新版本的NativeAllocationRegistry连同GC一起做了调整，使得进程在native内存增长过多的时候可以自动触发GC
    NativeAllocationRegistry，native资源虽然可以回收，但仍然有些缺陷。譬如被设计成牵线木偶的Java类所占空间很小，但其间接引用的native资源占用很大。因此就会导致Java堆的增长很慢，而native堆的增长很快。在某些场景下，Java堆的增长还没有达到下一次GC触发的水位，而native堆中的垃圾已经堆积成山。由程序主动调用System.gc()
    
    8.0之前使用Object#finalize(）函数触发native回收
    
-  Java 中垃圾回收机制中如何判断对象需要回收？
    
    
## Java内存模型

内存共享 内存隔离 线程同步

## 控制线程数量 – 线程池

---

# 线程间通信

###  handler 线程通信
       
- handler 
  
    - handler postDelayed如何实现？
    
       1. 子线程Handler sendMessage
       2. MessageQueue # enqueueMessage 按执行时间先后顺序加入到消息队列
       3. Thread 调用 Looper # loop从MessageQueue # next 取出Message
       3. Looper 调用 Message.target.dispatchMessage(msg);
       4. Hanlder.handleMessage 回到Handler所在线程 ,将系统时间增加延迟假如到MessageQue队列

    - 一个线程有几个Looper，looper什么时候开始循环？
    
      1. 主线程Looper在ActivityThread # main 中初始化.
      2. ActivityThread 分别执行 Looper.prepareMainLooper();   Looper.loop();
      3. Looper.prepare 内部Thread会ThreadLocal（HashMap）setLooper线程绑定,保证一个线程只有一个
    
    - Handler为什么会导致内存泄漏?如何处理？
    
        因为内部类持有外部对象引用。MessageQueue持有Message，message持有Handler,handler持有Activity等外部资源。
        可将handler设置为静态类,以弱引用方式持有外部引用,或者在Destory时清除handler队列中的message
    
    - HandlerThread如何保证 handler在绑定Looper前完成looper创建工作? (多线程)
        
        执行Run方法时加锁,getLooper时如无looper则处于wait状态.等待Run方法Looper创建完毕后释放锁后,再返回Looper。
        
    - messageQueue是什么结构 为什么Looper在主线程循环不会导致ANR？
    
       message链表结构,Looper从消息队列中for循环取message ， Loopefor循环执行 MessageQueue.next时会执行如下函数使线程等待状态(底层使用管道技术控制休眠和唤醒)。
       
       ```         
                   nativePollOnce(ptr, nextPollTimeoutMillis);
       ```
       
       直到下一个Message事件加入到MessageQueue时会唤醒Looper.
       
    - ActivityThread 类的handler主要是处理binder线程和主线程的通信任务。
    
    - Looper & Handler & message 三者的关系？
    
      Thread关联Looper hanler关联looper
      handler把Message投递给Looper,Looper循环读MessageQueue消息  
    
    - Looper和ThreadLocal
    
        - Looper.getMainLooper();
            从上面可以看出来getMainLooper取得是sMainLooper，sMainLooper是static的。而sMainLooper是在prepareMainLooper中调用 sMainLooper = myLooper();
            本质获取当前线程的looper又因为prepareMainLooper是在主线程ActivityThread中调用的所以是主线程通过静态变量sMainLooper保存的.
        - Looper.myLooper();
             取的是ThreadLocal的looper
     
    - IO多路复用
        - 本质是监听文件变化 投递message改变文件唤醒。         

- handler屏障消息

### handlerThread

### 讲解一下HandlerThread

handlerThread内部启动的一个looper并且关联了一个 Handler.
我们可以通过Thread.getLooper绑定我们的handler.
通过handler触发工作线程handlerThread的任务。

```

public class HandlerThread extends Thread {
    int mPriority;
    int mTid = -1;
    Looper mLooper;
    private @Nullable Handler mHandler;

    public HandlerThread(String name) {
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }
    
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }
    
    protected void onLooperPrepared() {
    }

    @Override
    public void run() {
        mTid = Process.myTid();
        Looper.prepare();
        synchronized (this) {
            mLooper = Looper.myLooper();
            notifyAll();
        }
        Process.setThreadPriority(mPriority);
        onLooperPrepared();
        Looper.loop();
        mTid = -1;
    }
    
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }
        
        // If the thread has been started, wait until the looper has been created.
        synchronized (this) {
            while (isAlive() && mLooper == null) {
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    @NonNull
    public Handler getThreadHandler() {
        if (mHandler == null) {
            mHandler = new Handler(getLooper());
        }
        return mHandler;
    }

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    public int getThreadId() {
        return mTid;
    }
}

```

quitSafely可以等事件执行完销毁Looper,但是调用方法后都不再接收新的Message事件。     

### AsyncTask源码分析

AsyncTask.execute
```
@MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        if (mStatus != Status.PENDING) {
            switch (mStatus) {
                case RUNNING:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task is already running.");
                case FINISHED:
                    throw new IllegalStateException("Cannot execute task:"
                            + " the task has already been executed "
                            + "(a task can be executed only once)");
            }
        }

        mStatus = Status.RUNNING;

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);

        return this;
    }
```

实际调用的是 Executor.execute.
是SerialExecutor的一个实例，而且它是个静态变量。也就是说，一个进程里面所有AsyncTask对象都共享同一个SerialExecutor对象。

```
 private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

```

异步任务r被放到了ArrayDeque对象mTasks中，然后通过scheduleNext()来从mTasks里面得到一个任务去一个后台线程执行。
在一个异步任务执行后，再次调用scheduleNext来执行下一个任务（run函数）。


THREAD_POOL_EXECUTOR 是一个线程池看一下这个线程池的配置

```
  // We keep only a single pool thread around all the time.
    // We let the pool grow to a fairly large number of threads if necessary,
    // but let them time out quickly. In the unlikely case that we run out of threads,
    // we fall back to a simple unbounded-queue executor.
    // This combination ensures that:
    // 1. We normally keep few threads (1) around.
    // 2. We queue only after launching a significantly larger, but still bounded, set of threads.
    // 3. We keep the total number of threads bounded, but still allow an unbounded set
    //    of tasks to be queued.
    private static final int CORE_POOL_SIZE = 1;
    private static final int MAXIMUM_POOL_SIZE = 20;
    private static final int BACKUP_POOL_SIZE = 5;
    private static final int KEEP_ALIVE_SECONDS = 3;

 @Deprecated
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(), sThreadFactory);
        threadPoolExecutor.setRejectedExecutionHandler(sRunOnSerialPolicy);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
```

- 为什么线程池的配置是那样的?
API 30 
串行执行所以核心数是1。

- 串行执行,为什么用线程池队列管理?

避免线程创建和销毁

### 如何并行执行多个AsyncTask对象

```
   executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR);
```

----

# 进程间通信

## AIDL封装了什么

1.其中的stub类封装了远端ontransact的调用过程,stub子类proxy封装了客户端拿到远端binder进行transact的调用过程

- Stub类的asInterface做了什么？

1.asInterface(android.os.IBinder obj) 用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端位于同一进程，那么此方法返回的 就是服务端的Stub对象本身，否则返回的是系统封装后的Stub.proxy对象。

- Stub类的onTransact做了什么？

Stub就是一个Binder类，服务端会实现此类中的接口,当发成方法调用需要走onTransact过程，后调用服务端实现的接口。

- AIDL三个参数有理解过吗/传递自定义类型参数的修饰符in，out，inout的区别？

    1. AIDL支持传输八种基本数据类型和实现Parcelabel的引用类型
    2. 对于非基本数据类型(传递自定义对象一定要设置定向TAG)，也不是String和CharSequence类型的，需要有方向指示，包括in、out和inout，in表示由客户端设置，out表示由服务端设置，inout是两者均可设置
    3. 如果你对自定义类型使用了in或者inout标识符的话;你必须再给自定义类实现readFromParcel()

  in 为定向 tag 的话表现为服务端将会接收到一个那个对象的完整数据，但是客户端的那个对象不会因为服务端对传参的修改而发生变动；
  out 的话表现为服务端将会接收到那个对象的的空对象，但是在服务端对接收到的空对象有任何修改之后客户端将会同步变动；
  inout 为定向 tag 的情况下，服务端将会接收到客户端传来对象的完整信息，并且客户端将会同步服务端对该对象的任何变动。
  主要是通过,data_ reply_ 实现操作 序列化的两个回调函数实现的

```java 

{
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          if ((bookBean!=null)) {
            _data.writeInt(1);
            bookBean.writeToParcel(_data, 0);
          }
          else {
            _data.writeInt(0);
          }
          boolean _status = mRemote.transact(Stub.TRANSACTION_addBookInOut, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().addBookInOut(bookBean);
            return;
          }
          _reply.readException();
          if ((0!=_reply.readInt())) {
            bookBean.readFromParcel(_reply);
          }
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
      }
```

https://www.jianshu.com/p/0eda666085a1

- 如何指定AIDL为异步调用？
  放到非 UI 线程

- 默认情况下AIDL的调用过程是同步还是异步？
  默认情况下AIDL调用过程是同步的，例如A进程请求与B进程通信，A会等到B海枯石烂的，如果A为主线程调用的话，那么B如果执行时间过程很可能就直接ANR了，
  并且注意B那边是很多进程都可以调用的，所以要注意同步数据，并且B那边被调用执行的时候都是在子线程（binder线程）
  ，如果有**回调**的话，那么也是在子线程，所有A在获取B那边的**回调数据后**如果要更新ui要注意不能在子线程更新ui

- AIDL 优点和缺点

    1. 是通信的约定,严格意义的约定的接口
    2. 容易出现连接断开
    
### AIDL异步接口的使用

1. 接口方法名增加oneWay,无返回值
2. 什么情况适合使用oneway,只需要传递数据不在意返回结果或者是使用回调的方式

```
 public void onWay() throws RemoteException {
                Parcel _data = Parcel.obtain();

                try {
                    _data.writeInterfaceToken("com.juphoon.service.rcs.IRcsService");
                    this.mRemote.transact(27, _data, (Parcel)null, 1);
                } finally {
                    _data.recycle();
                }

            }
```

对于客户端是异步的，但是对于服务端还是同步执行。

## Binder 

- binder
    
    - binder的内存映射是如何实现的？
    
    用户空间的一块内存区域映射到内核空间。是基于C/S架构的，即service端产生Binder服务。通过serviceManager注册至系统内核. client端通过serviceManager获取Binder服务（实际是代理对象）完成通信
    1. Binder驱动在内核空间创建一块数据接受缓冲区物理(1M - 8K) binder的物理页由binder驱动负责分配
    2. 在内核空间开辟一块内核缓冲区虚拟,将数据接受缓冲区和内核缓冲区做映射。 数据接受缓冲区和接受进程用户空间虚拟做映射
    3. 发送方将数据复制到内核中由于存在映射，也就相当于把数据复制到接收方用户空间。
    
    https://xiaozhuanlan.com/topic/7903248561
    
    - binder客户端和Binder服务端的通信流程
   
      1. 客户端通过data reply 数据包调用transcat方法.
      2. 客户端调用的transcat()函数会被挂起(同步调用会后线程会挂起，异步调用不会挂起)，直到对应的onTransact()函数在服务端执行完成 
      3. 由binder驱动到binder服务端     未必正确:找方法是根据方法IDcode,（所以jar包增加AIDL接口都放在最下面兼容老版本）
      4. 服务端调用实现的onTransact()和其他处理远程调用的函数必须是线程安全的
      5. Binder线程池最多有16个线程，也就是每个进程可以并发的处理16个远程调用
      
    - 代码细节
    
        1. 服务端创建binder 实现onTransact 方法。
        2. 客户端通过ServiceManager（反射获取 ）获得binder 调用transact方法 
        3. 如果实现双向通讯，客户端创建新的Binder,可通过data_writeStrongBinder，将本地binder注册传递给服务binder完成双向通信 
        4. 检测远端binder挂调 通过IBinder.DeathRecipient
   
    - 还有那些其他IPC方式？Binder相比与其他IPC有什么优势？
    
       管道（两次拷贝） 共享内存（不安全）socket  Binder(内存映射)
       1.  管道（两次拷贝)sokect 安全性依赖上层协议
       2. 用户空间和用户空间无法直接数据传输,可通过内核空间中转。 即用户空间A->内核空间->用户空间B 
       3. 共享内存,死锁等问题 安全性依赖上层协议 
       4. Binder(内存映射)校验id
   
               
## 如何跨进程传递大数据

- 利用Binder与匿名共享内存, 通过Binder传递匿名共享内存的文件描述符

openFile

```java
 @Override
    public ParcelFileDescriptor openFile(Uri uri, String mode) throws FileNotFoundException {
        switch (sURLMatcher.match(uri)) {
            case RMS_LOG_FILE:
                String filename = uri.getPathSegments().get(1);
                if (TextUtils.isEmpty(filename)) {
                    return null;
                }
                File file = new File(RmsDefine.RMS_FILE_PATH, filename);
                if (file.exists()) {
                    return ParcelFileDescriptor.open(file,
                            ParcelFileDescriptor.MODE_READ_ONLY);
                } else {
                    return null;
                }
            default:
                Log.e("RmsProvider","Unknown Uri");
                break;
        }
        return null;
    }
```

- MemoryFile是Android提供大数据传输的封装类

1. 应用层使用匿名共享内存的方法，关键点就是文件描述符（FileDescriptor）的传递

2. Binder驱动通过当前进程的fd找到对应的文件，然后为目标进程新建fd，并传递给目标进程，核心就是把进程A中的fd转化成进程B中的fd

## Serializable 与 Parcelable区别

- 序列化的实现的两个方法？

- 序列化有什么区别吗？
    
    Serializable 使用 I/O 读写存储在硬盘上，而 Parcelable 是直接 在内存中读写。
    很明显，内存的读写速度通常大于 IO 读写，所以在 Android 中传递数据优先选择 Parcelable。
    
    Serializable 会使用反射，序列化和反序列化过程需要大量 I/O 操作， Parcelable 自已实现封送和解封（marshalled &unmarshalled）操作不需要用反射，数据也存放在 Native 内存中，效率要快很多。

## Android跨进程同步FileLock,进程间通信如何保证同步?

AIDL共享内存可以使用

FileLock 

共享锁: 共享读操作，但只能一个写（读可以同时，但写不能）。共享锁防止其他正在运行的程序获得重复的独占锁，但是允许他们获得重复的共享锁。
独占锁: 只有一个读或一个写（读和写都不能同时）。独占锁防止其他程序获得任何类型的锁

2.有两种方式获得文件锁，FileChannel的lock和tryLock，用lock会阻塞当前线程，直到获取到锁，用tryLock会尝试获取，如果获取失败则返回null，不会阻塞线程。

3.FileLock释放的条件是：自己调用release/close或者所使用的FileChannel调用close或者是JVM终止运行

```java 

public class FileLockTest {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		FileChannel channel = null;
		FileLock lock = null;
		try {
			FileOutputStream raf = new FileOutputStream("logfile.txt");
			channel = raf.getChannel();
			lock = channel.lock();
			Thread.sleep(10000);
			lock.close();
			System.err.println(System.currentTimeMillis() + "  release lock");
		} catch (FileNotFoundException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (IOException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		} catch (InterruptedException e) {
			// TODO Auto-generated catch block
			e.printStackTrace();
		}

	}

}
```

```java 
public class LockTest2 {

	public static void main(String[] args) {
		// TODO Auto-generated method stub
		new Thread(new Runnable() {

			@Override
			public void run() {
				// TODO Auto-generated method stub
				FileChannel channel = null;
				FileLock lock = null;
				try {
					FileOutputStream raf = new FileOutputStream("logfile.txt");
					channel = raf.getChannel();
					lock = channel.lock();
					if (lock.isValid()) {
						//any thing you want
						System.out.println(System.currentTimeMillis() + "  ok");
					} else {
						System.out.println("no ok");
					}

				} catch (FileNotFoundException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				} catch (IOException e) {
					// TODO Auto-generated catch block
					e.printStackTrace();
				}
			}
		}).start();

	}

}
```

### 进程之间如何同步,同步方案

- ContentProvider提供了进程同步方案

1. onCreate方法运行在主线程,并且先于Application启动
2. insert、query、update、delete.call ,openFile方法是运行在子线程的,支持binder线程池，可避免主线程ANR问题
3. 只能保证进程间的互斥，无法保证线程安全，因此还是要处理线程安全。
4. 数据传输可靠性高，极大避免连接断开和DeadObject问题
5. ContentProvider是懒加载的

### Binder数据传输通信效率,优化问题。


