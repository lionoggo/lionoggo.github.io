title: AsyncTask设计与实现
date: 2017-4-5 19:39:39
toc: true
tags: [Android,AsyncTask]
categories: technology
description: 在从事开发的过程中,累积了很多方面的资料.之前在一次意外事故中丢失很多,现在一个不知名的网盘中找到了一些有关Android笔记,不知写于何时但还有点价值,现在重新整理下.



--------

# AsyncTask基础

AsyncTask是Android系统提供的轻量级异步任务类.在Android 1.6之前,其内部是串行任务,Android1.6之后起修改为并行任务,由于并行任务会引入并发问题,因此 Android 3.0重新提供对串行任务提供了支持.之后,AsyncTask默认是串行任务,但可以通过方法`executeOnExecutor()`指定为并行任务.

AysncTask是个抽象的泛型类,它提供了Params,Progress和Result三个泛型参数分别用来表示输入参数类型,后台任务执行进度类型以及任务的输出结果类型,如不需要可以将参数类型指定为Void.此外AsyncTask提供以下四个方法用于通知调用者任务状态:

- `onPreExecute()`:通知后台任务开始,工作在主线程.
- `doInBackground(Params...params)`:在后台线程池中执行任务.在该方法中可以通过`publishProgress(Progress...values)`来更新任务进度,它最终会导致onProgressUpdate()的调用.
- `onProgressUpdate(Progress...values)`:后台任务进度改变时被执行(即`publishProgress()`调用后),工作在主线程
- `onPostExecute(Result result)`:任务的执行结果通过该方法通知调用者,result是后台任务的返回结果,即`doInBackground()`的返回值.同样,该方法工作在主线程

除此之外,AsyncTask也提供了用来取消任务的方法,此时会回调`onCancelled()`.在使用时我们需要继承该类,并实现其中的相关方法.需要注意AsyncTask对象需要在主线程创建,其执行方法`execute()`,`executeOnExecutor()`也需在主线程调用;同一个AsyncTask对象只能被执行一次,否则会报错.

# AsyncTask原理

.AsyncTask的整体实现非常简单,其内部提供了两个线程池:SerialExecutor和自定义的Excutor分别用任务排队和真正的执行任务.除此之外,其内部实现了对任务的封装.就其设计而言,重要的是任务的封装以及SerialExecutor,整个AsyncTask建立在这两者之上.

## 任务创建

通过AsyncTask的构造过程来看其内部任务的定义.在其构造过程会首先关联主线程的Handler,用于实现线程切换,这也是方法onProgressUpdate()工作在主线程的原因.

```java
public abstract class AsyncTask<Params,Progress,Result>{
    public AsyncTask(){
        this((Looper) null);
    }
    
    public AsyncTask(){
        this(handler != null ? handler.getLooper() : null);
    }
    
    public AsyncTask(Looper callbacklooper){
       //1.关联主线程的Handler
       mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
	   //2.创建mWorker用于真正执行任务,为了能拿到任务的执行结果,WorkerRunnable实现自Callback接口而非
       //Runnable
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                //2.1 标记为任务已经被执行过,mTaskInvoked为AtomicBoolean类型
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    //2.2 将当前任务线程的优先级设置为THREAD_PRIORITY_BACKGROUND,以便
                    //影响主线程的执行效率,即不抢夺主线程的cpu资源
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //2.3 调用doInBackground()执行我们自定义的后台任务,并拿到其返回结果
                    result = doInBackground(mParams);
                    //2.4 在你能确保后续操作是会阻塞很长时间时,调用该命令将会促使Kernal释放已经
                    //挂起对象的引用,能够减少进程资源的占用
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    //2.5 任务执行期间出现异常,将其标记为被取消状态
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };
		
        // 将mWorker封装为FutreTask对象
        mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        }; 
    }
}
```

以上代码中最核心的两个对象是mWorker和mFuture.mWorker是WorkerRunnable的实例,是AsyncTask中真正用于工作的任务对象,为了能拿到任务的执行结构,WorkerRunnable实现了Callable接口而非Runnable接口,其定义如下:

```java
private static abstract class WorkerRunnable<Params, Result> implements Callable<Result> {
    Params[] mParams;
}
```

在mWorker的`call()`方法中,真正用于执行后台任务的`doInBackground()`会被调用.

mWorker对象创建完成后,会继续创建mFuture对象,mFuture是FutureTask的直接子类(匿名内部类形式)的对象,其构造方法接受mWorker作为参数.而FutureTask是java并发包中的类.其实现了RunnableFuture接口,而RunnableFuture又继承自Runnable和Future接口,因此不难猜出FutureTask既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值(当然实际上也确实如此)。最终这个mWorker会被放入到线程池中执行.

上述过程简单点说就是我们自定义的后台操作最终被封装在mFuture中.执行mFuture就是执行我们自定义的后台操作.可以说,对于AsyncTask的实现而言最重要的就是mWoker和mFuture对象的创建.关于AsyncTask中任务的实现至此已经明了,接下来需要关注AsyncTask是串行和并行的原理.

## 串行任务

Android 3.0之后,使用AsyncTask的execute()执行任务时,默认是串行操作,即任务一个接一个的执行,下面我们通过代码的来揭示AsyncTask是如何实现串行任务的.正常情况下,在创建AsyncTask实例后,我们会通过以下方法执行任务:

```java
 @MainThread
 public final AsyncTask<Params, Progress, Result> execute(Params... params) {
      return executeOnExecutor(sDefaultExecutor, params);
  }

```

在该方法中调用`executeOnExecutor(Executor exec,Params... params)`来继续执行操作:

```java
@MainThread
    public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
            Params... params) {
        //1. 检查任务状态
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
		
        //2.正常情况下,将任务状态设置为RUNNING
        mStatus = Status.RUNNING;
		//3.后台任务执行前调用.由于executeOnExecutor()在主线程被调用,因此onPreExecute()也在主线程
        onPreExecute();
		//4.将我们传入给的params参数赋值给mWorker的mParam成员变量.
        mWorker.mParams = params;
        //5.提交任务到线程池中开始执行
        exec.execute(mFuture);

        return this;
    }
```

需要注意,一个AsyncTask对象执行过execute()方法后,其任务状态会被设置为RUNNING状态.当AsyncTask对象的当前状态为RUNNING或者FINISHED时,再次调用其execute()方法将会抛出异常.这也是之前我们说同一个AsyncTask实例只能执行一次的原因.

通过上述代码我们可以看到任务对象mFuture最终会被线程执行器exec执行.此时exec被赋值为sDefaultExecutor,而sDefaultExecutor的定义如下:

```java
public abstract class AsyncTask<Params,Progress,Result>{
    ...
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
    ...
        
    private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
    
    ...
}
```

### SerialExecutor

SerialExecutor是AsyncTask中自定义的执行器,其内部引入了ArrayDeque用于保存任务队列:

```java
    private static class SerialExecutor implements Executor {
        // mTask用于保存任务
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            // 1.保存新添加进来的任务到mTasks
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        // 当前任务执行完成后,通过scheduleNext()安排执行下个任务
                        scheduleNext();
                    }
                }
            });
            // 2.当前没有任务在执行,通过通过scheduleNext()安排执行下个任务
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            // 懂ArrayDeque的队首取出新任务
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }

```

SerialExecutor中存在ArrayDeque类型的成员变量mTasks,其被用来保存任务队列.

> ArrayDeque是基于循环数组实现的双端队列,其`offer()`操作用来在队尾添加元素,而`poll()`用来取出并删除队首的元素.

当SerialExecutor实例的`execute()`执行时首先会通过`offer()`保存新任务到mTasks中,保存完成后如果当前没用正在执行的任务就通过`scheduleNext()`安排执行下个任务.此外在当前任务执行完成后,也会通过`scheduleNext()`来继续执行下个任务.`scheduleNext()`中做的唯一一件事就是从mTasks中取出一个任务,然后交给THEAD_POOL_EXECUTOR线程池执行.小结

简单总结下新加入的任务首先会通过SerialExecutor保存在双端队列的队尾,然后从队首取出任务交给THREAD_POOL_EXECUTOR线程池来执行.虽然THREAD_POOL_EXECUTOR支持多个任务同时执行,但是由于SerialExecutor只会在一个任务执行完成后才会从双端队列中取出新任务并放到线程池中执行,因此此时任务的执行都是串行的.

## 线程池定义

THREAD_POOL_EXECUTOR是AsyncTask中自定义的线程池对象,核心线程数根据实际情况设置,最少2个,最多不超过4个.最大线程数是CPU核数*2+1,默认非核心线程存活时间为30秒,其完整定义如下:

```java
public abstract class AsyncTask<Params, Progress, Result>{
   private static final int CPU_COUNT = Runtime.getRuntime().availableProcessors();
   private static final int CORE_POOL_SIZE = Math.max(2, Math.min(CPU_COUNT - 1, 4));
   private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
   private static final int KEEP_ALIVE_SECONDS = 30;

   private static final ThreadFactory sThreadFactory = new ThreadFactory() {
       private final AtomicInteger mCount = new AtomicInteger(1);

       public Thread newThread(Runnable r) {
           return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
       }
    };
    
   static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }
    
    ...
}
  
```



## 并行任务

上面我们提到AsyncTask存在线程池THREAD_POOL_EXECUTOR,只不过由于之前每次提交一个任务到该线程池中导致任务无法并行.在此情况下要想实现任务并行操作就非常简单了,只需要将多个任务直接提交给该线程池执行即可.AsyncTask的中提供的`executeOnExecutor()`就是如此,在使用时直接调用该方法并指定参数exec为AsyncTask.THREAD_POOL_EXECUTOR任务执行器即可.(当然我们这里也可以自定义新的Executor)

```java
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



## 总结

到现在关于AsyncTask的分析就结束了,从实际使用来说AsyncTask在设计上略有不错,使用起来仍然比较麻烦,且受限于一些场景.在Android发展初期,AsyncTask备受青睐,现在随着整个Android开发生态的发展,AsyncTask越来越少用了.但通过AsyncTask我们仍然能重温有关线程池设计/线程使用的哪些知识点.

![image-20181010204003807](https://i.imgur.com/1eIGFDJ.png)