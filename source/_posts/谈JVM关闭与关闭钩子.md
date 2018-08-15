---
title: 谈JVM关闭与关闭钩子
date: 2015-10-15 17:13:05
toc: true
tags: JVM
categories: technology
description: 了解虚拟机Runtime及Shutdown Hook的原理.

---

# JVM关闭

通常而言，对于JVM的关闭我们很少去关注，但是了解JVM的关闭能帮我们在JVM关闭时做一些合理的事情。首先JVM的关闭方式可以分为三种：

- 正常关闭：当最后一个非守护线程结束或者调用了System.exit或者通过其他特定平台的方法关闭（发送SIGINT，SIGTERM信号等）
- 强制关闭：通过调用Runtime.halt方法或者是在操作系统中直接kill(发送SIGKILL信号)掉JVM进程
- 异常关闭：运行中遇到RuntimeException异常等。

在某些情况下，我们需要在JVM关闭时做些扫尾的工作，比如删除临时文件、停止日志服务以及内存数据写到磁盘等，为此JVM提供了关闭钩子（shutdown hooks）来做这些事情。另外特别注意的是：如果JVM因**异常**关闭，那么子线程（Hook本质上也是子线程）将不会停止。但在JVM被强行关闭时，这些线程都会被强行结束。

在了解关闭钩子之前我们首先认识**Runtime**。Runtime封装Java应用运行时的环境。通过Runtime实例，使得应用程序和其运行环境相连接。Runtime是在应用启动期间自动建立，应用程序不能够创建Runtime，但是我们可以通过`Runtime.getRuntime()`来获得当前应用的Runtime对象引用，通过该引用我们可以获得当前运行环境的相关信息，比如空闲内存、最大内存以及为当前虚拟机添加关闭钩子（`addShutdownHook（）`），执行指定命令（`exec（）`）等等。

# 关闭钩子（shutdown hook）

关闭钩子本质上是一个线程（也称为Hook线程），用来监听JVM的关闭。通过使用Runtime的addShutdownHook(Thread hook)可以向JVM注册一个关闭钩子。Hook线程在JVM **正常关闭**才会执行，在强制关闭时不会执行。

对于一个JVM中注册的多个关闭钩子它们将会并发执行，所以JVM并不能保证它的执行顺行。当所有的Hook线程执行完毕后，如果此时runFinalizersOnExit为true，那么JVM将先运行终结器，然后停止。Hook线程会延迟JVM的关闭时间，这就要求在编写钩子过程中必须要尽可能的减少Hook线程的执行时间。另外由于多个钩子是并发执行的，那么很可能因为代码不当导致出现竞态条件或死锁等问题，为了避免该问题，强烈建议在一个钩子中执行一系列操作。

另外在使用关闭钩子还要注意以下几点:

 1. 不能在钩子调用`System.exit()`，否则卡住JVM的关闭过程，但是可以调用`Runtime.halt()`。
 2. 不能再钩子中再进行钩子的添加和删掉操作，否则将会抛出IllegalStateException。
 3. 在`System.exit()`之后添加的钩子无效。
 4. 当JVM收到SIGTERM命令（比如操作系统在关闭时）后，如果钩子线程在一定时间没有完成，那么Hook线程可能在执行过程中被终止。
 5. Hool线程中同样会抛出异常，如果抛出异常又不处理，那么钩子的执行序列就会被停止。

## 源码解析

现在我们来通过源码来简单的分析一下shutdown hook的实现原理。简单的说，Shutdown类充当触发器，负责触发关闭钩子，而ApplicationShutdownHooks负责管理所有注册到JVM的关闭钩子。

### Shutdown源码分析

```java
class Shutdown {

    /* 定义了三个状态码，用来表示关闭状态，也是System.exit(state)中的state的取值范围 */
    private static final int RUNNING = 0;
    private static final int HOOKS = 1;
    private static final int FINALIZERS = 2;
    private static int state = RUNNING;//初始化为运行状态

    //在JVM退出前是否要运行终结器？
    private static boolean runFinalizersOnExit = false;

    // 三种预定义的钩子类型，如下：
    // (0) Console restore hook
    // (1) Application hooks  （ApplicationShutdownHooks中添加的就是该类型）
    // (2) DeleteOnExit hook
    private static final int MAX_SYSTEM_HOOKS = 10;
    private static final Runnable[] hooks = new Runnable[MAX_SYSTEM_HOOKS];

    // the index of the currently running shutdown hook to the hooks array
    private static int currentRunningHook = 0;

    /* The preceding static fields are protected by this lock */
    private static class Lock { };
    private static Object lock = new Lock();

    /* Lock object for the native halt method */
    private static Object haltLock = new Lock();

    /* Invoked by Runtime.runFinalizersOnExit */
    static void setRunFinalizersOnExit(boolean run) {
        synchronized (lock) {
            runFinalizersOnExit = run;
        }
    }

    //添加钩子，每种类型的钩子只能添加一次。ApplicationShutdownHooks类用于管理应用程序添加的钩子，通过static{}在初始化阶段添加。
    static void add(int slot, boolean registerShutdownInProgress, Runnable hook) {
        synchronized (lock) {
            if (hooks[slot] != null)//每个钩子只能被添加一次
                throw new InternalError("Shutdown hook at slot " + slot + " already registered");

            if (!registerShutdownInProgress) {
                if (state > RUNNING)
                    throw new IllegalStateException("Shutdown in progress");
            } else {
                if (state > HOOKS || (state == HOOKS && slot <= currentRunningHook))
                    throw new IllegalStateException("Shutdown in progress");
            }

            hooks[slot] = hook;
        }
    }

    //遍历所有已注册到JVM的钩子，并执行
    private static void runHooks() {
        for (int i=0; i < MAX_SYSTEM_HOOKS; i++) {
            try {
                Runnable hook;
                synchronized (lock) {
                    // acquire the lock to make sure the hook registered during
                    // shutdown is visible here.
                    currentRunningHook = i;
                    hook = hooks[i];
                }
                if (hook != null) hook.run();
            } catch(Throwable t) {
                if (t instanceof ThreadDeath) {
                    ThreadDeath td = (ThreadDeath)t;
                    throw td;
                }
            }
        }
    }

    static void halt(int status) {
        synchronized (haltLock) {
            halt0(status);
        }
    }

    static native void halt0(int status);


    private static native void runAllFinalizers();

     //关闭钩子的序列，调用Shutdown.exit()或者Shutdown.shutdown()最终会调用该方法
    private static void sequence() {
        synchronized (lock) {
            /* Guard against the possibility of a daemon thread invoking exit
             * after DestroyJavaVM initiates the shutdown sequence
             */
            if (state != HOOKS) return;
        }
        runHooks();
        boolean rfoe;
        synchronized (lock) {
            state = FINALIZERS;
            rfoe = runFinalizersOnExit;
        }
        if (rfoe) runAllFinalizers();
    }

    static void exit(int status) {
        boolean runMoreFinalizers = false;
        synchronized (lock) {
            if (status != 0) runFinalizersOnExit = false;
            switch (state) {
            case RUNNING:       /* Initiate shutdown */
                state = HOOKS;
                break;
            case HOOKS:         /* Stall and halt */
                break;
            case FINALIZERS:
                if (status != 0) {
                    /* Halt immediately on nonzero status */
                    halt(status);
                } else {
                    /* Compatibility with old behavior:
                     * Run more finalizers and then halt
                     */
                    runMoreFinalizers = runFinalizersOnExit;
                }
                break;
            }
        }
        if (runMoreFinalizers) {
            runAllFinalizers();
            halt(status);
        }
        synchronized (Shutdown.class) {
            /* Synchronize on the class object, causing any other thread
             * that attempts to initiate shutdown to stall indefinitely
             */
            sequence();
            halt(status);
        }
    }

    static void shutdown() {
        synchronized (lock) {
            switch (state) {
            case RUNNING:       /* Initiate shutdown */
                state = HOOKS;
                break;
            case HOOKS:         /* Stall and then return */
            case FINALIZERS:
                break;
            }
        }
        synchronized (Shutdown.class) {
            sequence();
        }
    }

}


```

## ApplicationShutdownHooks源码分析

接着再来看一下ApplicationShutdownHooks类,该类用来管理和维护用户等级的钩子，也就是Shutdown提到的Application hooks。该类中通过使用一个Map性质的IdentityHashMap来保持用户的关闭钩子。

```java
class ApplicationShutdownHooks {
    /* 存放用户级的管理钩子 */
    private static IdentityHashMap<Thread, Thread> hooks;
    static {
        try {
            //初始化阶段，首先向Shutdown中添加一个用户级别的线程，该线程中将会执行`runHooks()`方法.后续在Shutdown中通过该线程的引用来执行该线程run()方法。
            Shutdown.add(1 /* shutdown hook invocation order */,
                false /* not registered if shutdown in progress */,
                new Runnable() {
                    public void run() {
                        runHooks();
                    }
                }
            );
            hooks = new IdentityHashMap<>();
        } catch (IllegalStateException e) {
            // application shutdown hooks cannot be added if
            // shutdown is in progress.
            hooks = null;
        }
    }

    static synchronized void add(Thread hook) {
        if(hooks == null)
            throw new IllegalStateException("Shutdown in progress");

        if (hook.isAlive())
            throw new IllegalArgumentException("Hook already running");

        if (hooks.containsKey(hook))
            throw new IllegalArgumentException("Hook previously registered");

        hooks.put(hook, hook);
    }
    
    static synchronized boolean remove(Thread hook) {
        if(hooks == null)
            throw new IllegalStateException("Shutdown in progress");

        if (hook == null)
            throw new NullPointerException();

        return hooks.remove(hook) != null;
    }

    //该方法将提供给钩子类型的线程调用。
    static void runHooks() {
        Collection<Thread> threads;
        synchronized(ApplicationShutdownHooks.class) {
            threads = hooks.keySet();
            hooks = null;
        }

        for (Thread hook : threads) {
            hook.start();
        }
        for (Thread hook : threads) {
            try {
                hook.join();
            } catch (InterruptedException x) { }
        }
    }
}

```

这里需要注意几点：

1. 在初始化阶段，当static{}执行时，`runHooks()`并未执行。
2. 在`runHooks()`中通过定义局部的threads来转移hooks，转移成功后，hooks将被清空，这样就保证了，关闭钩子的逻辑只能执行一次。例如你在代码中多处调用System.exit()方法，但关闭钩子的执行逻辑只能运行一次。随后启动所有用户Hook线程，再通过调用每个join()的来使主线程等待子线程执行完毕，这也就保证了在正常情况下关闭钩子会在应用最终关闭前完成。
3. 任何一个应用都会默认存在一个Appliction Hook，该Hook被Shutdown触发后将会引起所有已注册到JVM的ShutdownHook的执行。

用一张简单的图来描述该过程，如下：

```sequence
System->Runtime: exit()
Runtime->Shutdown: exit()
Shutdown->Shutdown:exit()
Note right of Shutdown: 执行sequence()，进而执行该类\nrunHooks(),runHooks()中通过执行\nif (hook != null) hook.run()最终引起\nApplicationShutdownHooks.runHooks();
Note Left of ApplicationShutdownHooks:runHooks()
```

------

# 综合应用

   明白了Runtime和Shutdown Hook原理之后，也需要知道其使用场景,一般有以下三种使用:1.内存管理;2.执行命令;3.临时文件清理
 ## 内存管理
 在某些情况下，我们需要根据当前内存的使用情况，人为的调用`System.gc()`来尝试回收堆内存中失效的对象。此时就可以用到Runtime中的`totalMemory()`、`freeMemory()`等方法。示例如下：

```java
    public void autoClean(){
        Runtime rt=Runtime.getRumtime();
        if((rt.totalMemory()-rt.freeMemory())/(double)rt.maxMemory()>0.90){
            //内存利用率到90%以上时，执行相关清理工作
        }
    }
```

## 执行命令

在某些情况下,需要执行JVM外部的命令,比如打开Window平台中的记事本应用:

```java
    class Test { 
        public static void main(String args[]){ 
                Runtime r = Runtime.getRuntime(); 
                Process p = null; 
                try{ 
                        p = r.exec("notepad"); 
                } catch (Exception e) { 
                        
                } 
        } 
}
```

注意：通过`exec（）`方式执行命令时，该命令在单独的进程（Process）中。

## 临时文件清理

在应用运行期间,我们往往会创建很多临时文件.一个比较好的时机就是在虚拟机关闭的时候删除这些临时文件:

```java
    public class test {


	public static void main(String[] args) {
		try {
			Thread.sleep(20000);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		
		Runtime.getRuntime().addShutdownHook(new Thread(new Runnable() {
			public void run() {
				System.out.println("auto clean temporary file");
			}
		}));
	}
	
}

```









