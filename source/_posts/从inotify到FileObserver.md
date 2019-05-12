title: 从inotify机制到FileObserver原理
date: 2016/07/14 03:01:50
toc  : true
tags: [inotify,FileObserver,Android]
categories: technology
description: 有些情况下,我们难免需要监控一些文件的变化情况,这该如何实现呢?自然而然的我们会想要利用一个线程,每个一段时间便去看看文件的情况,这种方式本质上就是基于时间调度的轮训.虽然能够实现我们的需求,但是这种方式只适合文件经常变化的情况,其他情况下都非常低效,并且可能丢掉某些类型的变化,也就是说,这种方式无法实现实时的文件监控.

---

# inotify简介

那还有其他的方式么?熟悉linux的童鞋应该记得从linux kernel 2.6.13开始引入inotify机制,用于通知用户相关文件变化情况:任何一个文件发生某种变化,都会产生一个相应的文件事件.

我们不仅好奇,文件的哪些事件能够被监控,也就是说inotify支持监控文件的哪些变化呢?继续往下看.

## 可监控事件类型


目前inotify能够监控的以下文件变化事件:

|事件类型|说明|
|--------|----|
|IN_ACCESS|文件被访问|
|IN_MODIFY|文件被修改|
|IN_ATTRIB|文件属性被修改|
|IN_CLOSE_WRITE|可写文件被关闭|
|IN_CLOSE_NOWRITE|不可写文件被关闭|
|IN_CLOSE|文件被关闭,也就以上两者的集合|
|IN_OPEN|文件被打开|
|IN_MOVED_FROM|文件被移来|
|IN_MOVED_TO|文件被移走|
|IN_MOVE|文件被移动,也就是以上两者的集合|
|IN_CREATE|文件被创建|
|IN_DELETE|文件被删除|
|IN_DELETE_SELF|自删除,也就是一个可执行文件在执行时尝试删除自己|
|IN_MOVE_SELF|自移动,也就是一个可执行文件在执行时尝试移动自己|
|IN_UNMOUNT|宿主文件系统被卸载|




## API说明

不难发现,inotify对文件监控的支持是非常全面的,足以满足我们绝大部门的需求.接下来,我们对innotify的api做个简单的说明:
|方法|说明|
|----|---|
|inotify_init|用于创建一个inotify实例,返回一个指向该实例的的文件描述符|
|inotify_add_watch|添加对文件或者目录的监控,可以指定需要监控那些文件变化事件|
|inotify_rm_watch|从监控列表中移除监控文件或者目录|
|read|读取事件信息|
|close|关闭文件描述符,并会移除所有在该描述符上监控|

## 事件通知

在可监控事件中,我们已经了解inotify支持的文件事件.现在来看看这些当这些文件事件产生时,其发出通知的结构.在inotify中,事件通知结构用结构体inotify_event表示:
```
struct inotify_event
{
  int wd;               //监控目标的watch描述符
  uint32_t mask;        //事件掩码
  uint32_t cookie;      //事件同步cookie
  uint32_t len;         //name字符串的长度
  char name __flexarr;  //被监视目标的路径名
  };

```
这里需要记住一点:name字段并不是什么时候都有的:只有要监控的目标是一个目录,且产生的事件与目录内部的文件或子目录相关,且与目录本身无关时才会提供相应的name字段.
cookie用于关联被观察对象的IN_MOVED_FROM事件和IN_MOVED_TO 事件. 

## 使用流程

了解以上之后,那么该怎么用呢?要实现对文件或者目录的监控需要经过以下几个步骤:
### 1. 创建inotify实例

在应用程序中,首先需要创建inotify实例:
```c++
int fd=inotify_init(); 
```
该方法创建了一个inotify实例,并返回一个文件描述符以便能够通过这个描述符访问到inotify实例.

### 2. 添加监控
在获得inotify实例产生的文件描述符之后,我们就可以为其添加watch.另外,我们也可以使用mask(事件掩码)来设置我们想监控的事件类型.当然可以我们也可以使用IN_ALL_EVENTS监控所有事件:
```c++
int wd=inotify_add_watch(fd,path,mask)
```
***补充:***
此处的fd即inotify_init()方法返回的文件描述符.每个文件描述符都有一个排序的事件序列.path则是需要监控的文件或者目录的路径.mask则是事件掩码,它表示应用程序对哪些事件感兴趣.

文件系统产生的事件由Watch对象来管理,该方法将返回的wd就是Watch对象的句柄.


### 3. 等待事件与循环处理
在为inotify实例添加watch之后,接下来就是等待事件了.为了能不断的处理事件,我们将其放在循环体当中.
在循环中,通过read()方法可以一次获得多个事件.在没有事件产生时,read()被阻塞,一旦有事件产生,那么我们就可以读取事件到的我们设置的事件数组中,然后对事件数组进行处理,其简单代码如下:
```	c++
//事件数组,自定义设置,这里我们设置为128
char event_buf[128];

while(true){
     int num_bytes=read(fd,event_buf,len);
     //处理事件
     handleEvent(event_buf);
     //....省略....
	}
	
```
***补充***
event_buf是一个事件数组,用于接受文件变化所产生的事件(inotify_event).len则指定了要读的长度.通常来说,len大于事件数组的大小,很多时候,我们也会直接取事件数组的大小来作为len.


### 4.停止监控
当需要停止监控的时候,需要为文件描述符删除watch:
```c++
int r=inotify_rm_watch(fd,wd);
```
此处的fd也是在创建inotify时返回的文件描述符,wd则是上面提到watch对象的句柄.

到现在,我们已经对inotity有了初步的理解,感兴趣的童鞋可以自行研究.我们的重点还是Android中FileObserver的实现.接下来,我们真正的开始了解FileObserver的实现.


# FileObserver实现原理

我们知道Android 1.5时对应的linux内核已经是2.6.26,因此完全可以在Android上利用inotify机制来实现对文件的监控.Google很显然意识到了这一点,并且帮我们在inotify的基础上进行封装---FileObserver,以实现监听文件访问,创建,修改,删除等操作

接下来,来看一下FileObserver如何借助inotify实现文件监控的.
## 监控线程初始化

在FileObserver中存在一个静态内部类ObserverThread,该线程类是实现了文件监控的过程:
```java
public abstract class FileObserver {
    //可监控的事件
    public static final int ALL_EVENTS = ACCESS | MODIFY | ATTRIB | CLOSE_WRITE
            | CLOSE_NOWRITE | OPEN | MOVED_FROM | MOVED_TO | DELETE | CREATE
            | DELETE_SELF | MOVE_SELF;

    private static class ObserverThread extends Thread {
        //....省略多行代码....
    }

    private static ObserverThread s_observerThread;

    static {
        s_observerThread = new ObserverThread();
        s_observerThread.start();
    }

       //....省略多行代码....

}


```
不难发现发现FileObserver通过静态代码块的方式构造了s_observerThread对象,我们来看一下其构造过程:
```c++
     public ObserverThread() {
            super("FileObserver");
            m_fd = init();
        }
```
这里又调用natvie方法`init()`.既然这样,我们就在深入一下,看看init()方法的实现(现在,是不是发现我们自己编译源码的好处了?)该方法的实现在: /frameworks/base/core/jni/android_util_FileObserver.cpp

```c++
static jint android_os_fileobserver_init(JNIEnv* env, jobject object)
{
#if defined(__linux__)
    return (jint)inotify_init();
#else
    return -1;
#endif
}

```

其实现非常简单，就是调用inotify中的inotify_init()来创建一个inotify实例。回到FileObserver中来看s_ObserverThread的启动：
```java
public void run() {
            observe(m_fd);
        }

```
这里同样是调用natvie方法`observe(int fd)`：
```c++
static void android_os_fileobserver_observe(JNIEnv* env, jobject object, jint fd)
{
#if defined(__linux__)
	
	//设置事件数组
    char event_buf[512];
    struct inotify_event* event;

    //循环处理读到的事件
    while (1)
    {
        int event_pos = 0;
        //读取事件
        int num_bytes = read(fd, event_buf, sizeof(event_buf));

        if (num_bytes < (int)sizeof(*event))
        {
            if (errno == EINTR)
                continue;

            ALOGE("***** ERROR! android_os_fileobserver_observe() got a short event!");
            return;
        }

		//处理事件数组
        while (num_bytes >= (int)sizeof(*event))
        {
            int event_size;
            event = (struct inotify_event *)(event_buf + event_pos);

            jstring path = NULL;

            if (event->len > 0)
            {
                path = env->NewStringUTF(event->name);
            }
           
           //调用ObserverThread中的onEvent方法 
            env->CallVoidMethod(object, method_onEvent, event->wd, event->mask, path);
            if (env->ExceptionCheck()) {
                env->ExceptionDescribe();
                env->ExceptionClear();
            }
            if (path != NULL)
            {
                env->DeleteLocalRef(path);
            }

            event_size = sizeof(*event) + event->len;
            num_bytes -= event_size;
            event_pos += event_size;
        }
    }

#endif
}
```
不难看出,此处的循环主要就是从inotity中取出事件,然后回调ObserverThread中的`onEvent()`方法.现在,回到ObserverThread中的`onEvent()`方法中:
```java
public void onEvent(int wfd, int mask, String path) {
            // look up our observer, fixing up the map if necessary...
            FileObserver observer = null;

            synchronized (m_observers) {
                //根据wfd找出FileObserver对象
                WeakReference weak = m_observers.get(wfd);
                if (weak != null) {  // can happen with lots of events from a dead wfd
                    observer = (FileObserver) weak.get();
                    if (observer == null) {//observer已经被回收时,从m_observers中删除该对象
                        m_observers.remove(wfd);
                    }
                }
            }

            // ...then call out to the observer without the sync lock held
            if (observer != null) {
                try {
                    //回调给FileObserver中的onEvent方法进行处理
                    observer.onEvent(mask, path);
                } catch (Throwable throwable) {
                    Log.wtf(LOG_TAG, "Unhandled exception in FileObserver " + observer, throwable);
                }
            }
        }

```
FileObserver中的`onEvent()`为抽象方法,也就是要求你继承FileObserver,并实现该方法,在其中做相关的操作.
到现在为止我们已经明白了ObserverThread如何被启动,以及如何获取inotify中的事件,并回调给上层进行处理.

## 启动监控

上面提到m_observers表,该表维护着已经注册的FileObserver对象.接下来,我们就就来看看FileObserver中的`startWatching()`方法,该方法注册FileObserver对象,也是启动监控的过程:
```java
public void startWatching() {
        if (m_descriptor < 0) {
            m_descriptor = s_observerThread.startWatching(m_path, m_mask, this);
        }
    }
```
具体的注册操作委托给s_observerThread中的`startWatching()`:
```java
public int startWatching(String path, int mask, FileObserver observer) {
            //调用native方法startWatching,并得到一个watch对象的句柄
            int wfd = startWatching(m_fd, path, mask);

            Integer i = new Integer(wfd);
            if (wfd >= 0) {
                synchronized (m_observers) {
                    //将watch对象句柄和当前FileObserver关联
                    m_observers.put(i, new WeakReference(observer));
                }
            }

            return i;
        }

```
该方法中同样调用了native方法,其具体实现是:
```c++
static jint android_os_fileobserver_startWatching(JNIEnv* env, jobject object, jint fd, jstring pathString, jint mask)
{
    int res = -1;
#if defined(__linux__)
    if (fd >= 0)
    {
        const char* path = env->GetStringUTFChars(pathString, NULL);

        res = inotify_add_watch(fd, path, mask);

        env->ReleaseStringUTFChars(pathString, path);
    }
#endif
    return res;
}
```
不难看出,这里通过inotify的`inotify_add_watch()`为上面生成的inotify对象添加watch对象,并将watch对象的句柄返回给ObserverThread.

## 停止监控

到现在我们已经了解了如何注册watch句柄到FileObserver对象.有了注册的过程,当然少不了反注册的过程.同样,FileObserver为我们提供了`stopWatching()`来实现反注册,即停止监控的过程:
```java
 public void stopWatching() {
        if (m_descriptor >= 0) {//已经注册过的才能反注册
            s_observerThread.stopWatching(m_descriptor);
            m_descriptor = -1;
        }
    }

```
具体的实现也是交给了s_observerThread的`stopWatching()`方法:
```java
public void stopWatching(int descriptor) {
            stopWatching(m_fd, descriptor);
        }

```
接着委托给了natvie方法:
```c++
static void android_os_fileobserver_stopWatching(JNIEnv* env, jobject object, jint fd, jint wfd)
{
#if defined(__linux__)
    inotify_rm_watch((int)fd, (uint32_t)wfd);
#endif
}
```
这里的实现非常简单,就是调用inotify_rm_watch方法来解除inotify实例和watch实例的关系.

到现在为止我们已经弄明白了FileObserver的实现原理,为了方便理解,我们用一张简单的图来描述整个过程:

![notify](https://i.imgur.com/sdTuQlt.jpg)



## 使用说明

想必你已经了解了FileObserver的实现原理,接下里我们来看看如何使用.要想实现文件监控,我们只需要继承FileObserver类,并在`onEvent()`处理相关事件即可,简单的用代码演示一下:
```java
public class SDCardObserver extends FileObserver {
    public SDCardObserver(String path) {
        super(path);
    }

    @Override
    public void onEvent(int i, String s) {
        switch (i) {
            case FileObserver.ALL_EVENTS:
                //全部事件
                break;
            case FileObserver.CREATE:
                //文件被创建
                break;
            case FileObserver.DELETE:
                //文件被删除
                break;
            case FileObserver.MODIFY:
                //文件被修改
                break;
        }
    }
}

```

## 注意事项

这里我们需要注意两个问题:
1. 谨慎的选择你要处理的事件,否则可能造成死循环.
2. 当不再需要监听时,请记得停止监控
3. 需要注意FileObserver对象被垃圾回收的情况,从上面的原理中我们知道,该对象被回收后将不会再触发事件.


