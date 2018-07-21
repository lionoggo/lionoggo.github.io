title: 从JVM内存模型到线程安全
date: 2017/02/28 00:40:38
toc  : true
tags: [JVM,JMM,线程安全]
categories: technology
description: 从存储器到JVM内存模型,从硬件结构到JVM实现,换种角度看线程安全.



----------


# 存储器层次结构
对于开发者来说,存储器的层次结构应该是非常熟悉的,大体如下:
![mage-20180709221719](https://ws1.sinaimg.cn/large/006tNc79ly1ft3yzhjyprj31c40m0n2c.jpg)

其中寄存器,L1,L2,L3都被封装在CPU芯片中,作为应用开发者而言我们很少去注意和使用它.之所以引入L1,L2,L3高速寄存器,其根本是为了解决访问运算器和内存速度不匹配.但缓存的引入也带来两个问题:

 1. 缓存命中率:缓存的数据都是主存中数据的备份,如果指令所需要的数据恰好在缓存中,我们就说缓存命中,反之,需要从主存中获取.一个好的缓存策略应该尽可能的提高命中率,如何提高却是一件非常困难的事情.
 2. 缓存一致性问题:我们知道缓存是主存数据的备份,但每个核心都有自己的缓存,当缓存中的数据和内存中的数据不一致时,应该以谁的数据为准呢,这就是所谓缓存一致性问题.



上面只是展示存储器的层次结构,现在我们来更形象的来看一下CPU芯片与内存之间联系,以Intel i5双核处理器为例:
![mage-20180709221745](https://ws4.sinaimg.cn/large/006tNc79ly1ft3yzva43oj317u0t6n1z.jpg)

通过上图我们能明显的看出各个缓存之间的联系,在随后的JVM内存模型剖析中,你同样会发现类似的结构.关于存储器层次结构到这里已经足够,毕竟我们不是专门做操作系统的,下面我们来聊聊主存,更确切的说抽象的虚拟内存.

----------


# 虚拟内存
谈起内存的时候,每个人的脑海中都会呈现出内存条的形象,在很多时候,这种实物给我们对内存最直观的理解,对于非开发者这么理解是可以接受的,但是对于从事开发开发工作的工程师而言,我们还要加深一点.

![mage-20180709221829](https://ws1.sinaimg.cn/large/006tNc79ly1ft3z0mhuiwj31120rg7oo.jpg)

从硬件的角度来看,内存就是一块有固定容量的存储体,与该硬件直接打交道的是我们的操作系统.我们知道系统的进程都是共享CPU和内存资源的,现代操作系统为了更有效的管理内存,提出了内存的抽象概念,称之为虚拟内存.换言之,我们在操作系统中所提到的内存管理谈的都是虚拟内存.虚拟内存的提出带来几个好处:

 1. 虚拟内存将主存看成是一个存储在磁盘上的地址空间的告诉缓存.应用在未运行之前,只是存储在磁盘上二进制文件,运行后,该应用才被复制到主存中.
 2. 它为每个进程提供了一致的地址空间,简化了内存管理机制.简单点来看就是每个进程都认为自己独占该主存.最简单的例子就是一栋楼被分成许多个房间,每个房间都是独立,是户主专有,每个户主都可以从零开始自助装修.另外,在未征得其他户主的同意之前,你是无法进入其他房间的.

虚拟内存的提出也改变了内存访问的方式.之前CPU访问主存的方式如下:
![mage-20180709221855](https://ws4.sinaimg.cn/large/006tNc79ly1ft3z12sx74j310w0kadha.jpg)
上图演示了CPU直接通过物理地址(假设是2)来访问主存的过程,但如果有了虚拟内存之后,整个访问过程如下:
![mage-20180709221929](https://ws1.sinaimg.cn/large/006tNc79ly1ft3z1obmsej31b40mqjtl.jpg)
 CPU给定一个虚拟地址,然后经过MMU(内存管理单元,硬件)将虚拟地址翻译成真正的物理地址,再访问主存.比如现在虚拟地址是4200经过MMU的翻译直接变成真正的物理地址2.


这里来解释下什么是虚拟内存地址.我们知道虚拟内存为每个进程提供了一个假象:每个进程都在独占地使用主存,每个进程看到的内存都是一样的,这称之为虚拟地址空间.举个例子来说,比如我们内存条是1G的,即最大地址空间$2^{10}$,这时某个进程需要4G的内存,那么操作系统可以将其映射成更大的地址空间$2^{32}$,这个地址空间就是所谓的虚拟内存地址.关于如何映射,有兴趣的可以自行学习.用一张图来抽象的表示:



![mage-20180709222000](https://ws1.sinaimg.cn/large/006tNc79ly1ft3z27imlyj31060w476e.jpg)到现在我们明白原来原来我们所谈操作系统中谈的内存其实是虚拟内存,如果你是C语言开发者,那对此的感受可能更深.既然每个进程都拥有自己的虚拟地址空间,那么它的布局是如何的呢?以Linux系统为例,来看一下它的进程空间地址的布局:
![mage-20180709222035](https://ws4.sinaimg.cn/large/006tNc79ly1ft3z2tgcdsj319m0s6q76.jpg)

到现在为止,我们终于走到了进程这一步.我们知道,每个JVM都运行在一个单独的进程当中,和普通应用不同,JVM相当于一个操作系统,它有着自己的内存模型.下面,就切入到JVM的内存模型中.

----------

# 并发模型(线程)

如果java没有多线程的支持,没有JIT的存在,那么也不会有现在JVM内存模型.为什么这么说呢?首先我们从JIT说起,JIT会追踪程序的运行过程,并对其中可能的地方进行优化,其中有一项优化和处理器的乱序执行类似,不过这里叫做指令重排.如果没有多线程,也就不会存在所谓的临界资源,如果这个前置条件不存在当然也就不会存在资源竞争这一说法了.这样一来,可能Java早已经被抛弃在历史的长河中.

尽管Java语言不像C语言能够直接操作内存,但是掌握JVM内存模型仍然非常重要.对于为什么要掌握JVM内存模型得先从Java的并发编程模型说起.

在并发模型中需要处理两个关键问题:线程之间如何通信以及线程之间如何同步.所谓的通信指的是线程之间如何交换消息,而同步则用于控制不同线程之间操作发生的相对顺序.

从实现的角度来说,并发模型一般有两种方式:基于共享内存和基于消息传递.两者实现的不同决定了通信和同步的行为的差异.在基于共享内存的并发模型中,同步是显示的,通信是隐式的;而在基于消息传递的并发模型中,通信是显式的,同步是隐式的.我们来具体解释一下.

在共享内存的并发模型中,任何线程都可以公共内存进行操作,如果不加以显示同步,那么执行顺序将是不可知的,也恰是因为哪个线程都可以对公共内存操作,所以通信是隐式的.而在基于消息传递的并发模型中,由于消息的发送一定是在接受之前,因此同步是隐式的,但是线程之间必须通过明确的发送消息来进行通信.

在最终并发模型选择方案上,java选择基于共享内存的并发模型,也就是显式同步,隐式通信.如果在编写程序时,不处理好这两个问题,那在多线程会出现各种奇怪的问题.因此,对任何Java程序员来说,熟悉JVM的内存模型是非常重要的.

----------


# JVM内存结构
对于JVM内存,主要包含两方面:JVM内存结构和JVM内存模型.两者之间的区别在于模型是一种协议,规定对特定内存或缓存的读写过程,千万不要弄混了.

很多人往往对JVM内存结构和进程的内存结构感到困惑,这里我将帮助你梳理一下.

JVM本质上也是一个程序,只不过它又有着类似操作系统的特性.当一个JVM实例开始运行时,此时在Linux进程中,其内存布局如下:
![mage-20180709222120](https://ws1.sinaimg.cn/large/006tNc79ly1ft3z3l00t3j31ag0k4ael.jpg)

JVM在进程堆空间的基础上再次进行划分,来简单看一下.此时的永生代本质上就是Java程序程序的代码区和数据区,而年轻代和老年代才是Java程序真正使用的堆区,也就是我们经常挂在嘴边的.但是此时的堆区和进程上的堆却又很大的区别:在调用C程序的malloc函数时,会引起一次系统级的调用;在使用free函数释放内存时,同样也会引起一次系统级的调用,但是JVM中堆区并非如此:JVM一次性向系统申请一块连续的内存区域,作为Java程序的堆,当Java程序使用new申请内存时,JVM会根据需要在这段内存区域中为其分配,而不需要除非一次系统级别的调用.可以看出JVM其实自行实现了一条堆内存的管理机制,这种管理方式有以下好处:

 1. 减少系统级别的调用.大部分内存申请和回首不需要触发系统函数,仅仅只在Java堆大小发生变化时才会引起系统函数的调用.相比系统级别的调用,JVM实现内存管理成本更低.
 2. 减少内存泄漏情况的发生.通过JVM接管内存管理过程,可以避免大多情况下的内存泄漏问题.

现在已经简单介绍了JVM内存结构,希望这样能帮助你打通上下.当然,为了好理解,我省略了其中一些相对不重要的点,如有兴趣可以自行学习.讲完了JVM内存结构,下一步该是什么呢?

----------


# JVM内存模型
Java采用的是基于共享内存的并发模型,使得JVM看起来非常类似现代多核处理器:在基于共享内存的多核处理器体系架构中,每个处理器都有自己的缓存,并且定期与主内存进行协调.这里的线程同样有自己的缓存(也叫工作内存),此时,JVM内存模型呈现出如下结构:
![mage-20180709222153](https://ws4.sinaimg.cn/large/006tNc79ly1ft3z465r4rj31940raq6x.jpg)

上图展示JVM的内存模型,也称之为JMM.对于JMM有以下规定:
 1. 所有的变量都存储在主内存(Main Memory)
 2. 每个线程也有用自己的工作内存(Work Memory)
 3. 工作内存中的变量是主内存变量的拷贝,线程不能直接读写主内存的变量,而只能操作自己工作内存中的变量
 4. 线程间不共享工作内存,如果线程间需要通信必须借助主内存来完成

共享变量所在的内存区域也就是共享内存,也称之为堆内存,该区域中的变量都可能被共享,即被多线程访问.说的再通俗点就是在java当中,堆内存是在线程间共享的,而局部变量,形参和异常程序参数不在堆内存,因此就不存在多线程共享的情况.

与JMM规定相对应,我们定义了以下四个原子性操作来实现变量从主内存拷贝到工作内存的过程:

 >1. read:读取主内存的变量,并将其传送到工作内存
 >2. load:把read操作从主内存得到的变量值放入到工作内存的拷贝中
 >3. store:把工作内存中的一个变量值传送到主内存当中,以便用于后面的write操作
 >4. write:把store操作从工作内存中得到的变量的值放入主内存的变量中.

可以看出,从主内存到工作内存的过程其实是要经过read和load两个操作的,反之需要经过store和write两个操作.

现在我们来看一段代码,并用结合上文谈谈下多线程安全问题:
```java
public class ThreadTest {


    public static void main(String[] args) throws InterruptedException {
        ShareVar ins = new ShareVar();
        List<Thread> threadList = new ArrayList<>();

        for (int i = 0; i < 10; i++) {
            Thread thread;
            if (i % 2 == 0) {
                thread = new Thread(new AddThread(ins));

            } else {
                thread = new Thread(new SubThread(ins));

            }

            thread.start();
            threadList.add(thread);

        }

        for (Thread thread : threadList) {
            thread.join();
        }

        System.out.println(Thread.currentThread().getId() + "   " + ins.getCount());


    }


}

class ShareVar {
    private int count;

    public void add() {
        try {
            Thread.sleep(100);//此处为了更好的体现多线程安全问题
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        count++;


    }

    public void sub() {
        count--;
    }

    public int getCount() {
        return count;
    }
}

class AddThread implements Runnable {
    private ShareVar shareVar;

    public AddThread(ShareVar shareVar) {
        this.shareVar = shareVar;
    }

    @Override
    public void run() {
        shareVar.add();
    }
}

class SubThread implements Runnable {
    private ShareVar shareVar;

    public SubThread(ShareVar shareVar) {
        this.shareVar = shareVar;
    }

    @Override
    public void run() {
        shareVar.sub();
    }
}
```
理想情况下,最后应该输出0,但是多次运行你会先可能输出-1或者-2等.为什么呢?
在创建的这10个线程中,每个线程都有自己工作内存,而这些线程又共享了ShareVar对象的count变量,当线程启动时,会经过read-load操作从主内存中拷贝该变量至自己的工作内存中,随后每个线程会在自己的工作内存中操作该变量副本,最后会将该副本重新写会到主内存,替换原先变量的值.但在多个线程中,但由于线程间无法直接通信,这就导致变量的变化不能及时的反应在线程当中,这种细微的时间差最终导致每个线程当前操作的变量值未必是最新的,这就是所谓的内存不可见性.

现在我想你已经完全明白了多线程安全问题的由来.那该怎么解决呢?最简单的方法就是让多个线程对共享对象的读写操作编程串行,也就是同一时刻只允许一个线程对共享对象进行操作.我们将这种机制成为锁机制,java中规定每个对象都有一把锁,称之为监视器(monitor),有人也叫作对象锁,同一时刻,该对象锁只能服务一个线程.

有了锁对象之后,它是怎么生效的呢?为此JMM中又定义了两个原子操作:

 >1. lock:将主内存的变量标识为一条线程独占状态
 >2. unlock:解除主内存中变量的线程独占状态

在锁对象和这两个原子操作共同作用下而成的锁机制就可以实现同步了,体现在语言层面就是synchronized关键字.上面我们也说道Java采用的是基于共享内存的并发模型,该模型典型的特征是要显式同步,也就是说在要人为的使用synchronized关键字来做同步.现在我们来改进上面的代码,只需要为add()和sub()方法添加syhcronized关键字即可,但在这之前,先来看看这两个方法对应的字节码文件:
```
  public void add();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=2, args_size=1
         0: ldc2_w        #2                  // long 100l
         3: invokestatic  #4                  // Method java/lang/Thread.sleep:(J)V
         6: goto          14
         9: astore_1
        10: aload_1
        11: invokevirtual #6                  // Method java/lang/InterruptedException.printStackTrace:()V
        14: aload_0
        15: dup
        16: getfield      #7                  // Field count:I
        19: iconst_1
        20: iadd
        21: putfield      #7                  // Field count:I
        24: return

  public void sub();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #7                  // Field count:I
         5: iconst_1
         6: isub
         7: putfield      #7                  // Field count:I
        10: return
      LineNumberTable:
        line 18: 0
        line 19: 10

```
现在我们使用synchronized来让着两个方法变得安全起来:

```java
class ShareVar {
    private int count;

    public synchronized void add() {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        count++;


    }

    public synchronized void sub() {
        count--;
    }

    public int getCount() {
        return count;
    }
}

```
此时这段代码在多线程中就会表现良好.再来看看它的字节码文件发生了什么变化:
```
  public synchronized void add();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=2, args_size=1
         0: ldc2_w        #2                  // long 100l
         3: invokestatic  #4                  // Method java/lang/Thread.sleep:(J)V
         6: goto          14
         9: astore_1
        10: aload_1
        11: invokevirtual #6                  // Method java/lang/InterruptedException.printStackTrace:()V
        14: aload_0
        15: dup
        16: getfield      #7                  // Field count:I
        19: iconst_1
        20: iadd
        21: putfield      #7                  // Field count:I
        24: return
   

  public synchronized void sub();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #7                  // Field count:I
         5: iconst_1
         6: isub
         7: putfield      #7                  // Field count:I
        10: return
      LineNumberTable:
        line 18: 0
        line 19: 10

```
通过字节码不难看出最大的变化在于方法的flags中增加了ACC_SYNCHRONIZED标识,虚拟机在遇到该标识时,会隐式的为方法添加monitorenter和monitorexit指令,这两个指令就是在JMM的lock和unlock操作上实现的.

其中monitorenter指令会获取对象的占有权,此时有以下三种可能:

 >1. 如果该对象的monitor的值0,则该线程进入该monitor,并将其值标为1,表明对象被该线程独占.
 >2. 同一个线程,如果之前已经占有该对象了,当再次进入时,需将该对象的monitor的值加1.
 >3. 如果该对象的monitor值不为0,表明该对象被其他线程独占了,此时该线程进入阻塞状态,等到该对象的monitor的值为0时,在尝试获取该对象.

而monitorexit的指令则是已占有该对象的线程在离开时,将monitor的值减1,表明该线程已经不再独占该对象.

用synchronized修饰的方法叫做同步方法,除了这种方式之外,还可以使用同步代码块的形式:
```java
package com.cd.app;

class ShareVar {
    private int count;

    public void add() {
        synchronized (this) {
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            count++;
        }


    }

    public void sub() {
        synchronized (this) {
            count--;
        }

    }

    public int getCount() {
        return count;
    }
}

```
接下来同样是看一下他的字节码,主要看add()和sub()方法:
```
 public void add();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: aload_0
         5: dup
         6: getfield      #2                  // Field count:I
         9: iconst_1
        10: iadd
        11: putfield      #2                  // Field count:I
        14: aload_1
        15: monitorexit
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit
        22: aload_2
        23: athrow
        24: return



public void sub();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter
         4: aload_0
         5: dup
         6: getfield      #2                  // Field count:I
         9: iconst_1
        10: isub
        11: putfield      #2                  // Field count:I
        14: aload_1
        15: monitorexit
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit
        22: aload_2
        23: athrow
        24: return

```
同步代码块和同步方法的实现原理是一致的,都是通过monitorenter/monitorexit指令,唯一的区别在于同步代码块中monitorenter/monitorexit是显式的加载字节码文件当中的.


上面我们通过synchronized解决了内存可见性问题,另外也可以认为凡是被synchronized修饰的方法或代码块都是原子性的,即一个变量从主内存到工作内存,再从工作内存到主内存这个过程是不可分割的.

正如我们在[ 谈乱序执行和内存屏障][1]所提到的,javac编译器和JVM为了提高性能会通过指令重排的方式来企图提高性能,但是在某些情况下我们同样需要阻止这过程,由于synchronized关键字保证了持有同一个锁的的两个同步方法/同步块只能串行进入,因此无形之中也就相当阻止了指令重排.

----



# 总结
希望这么从下往上,再从上往下的解释能让各位同学对JVM内存模型以及多线程安全问题有个更通透的理解.







[1]: http://blog.csdn.net/dd864140130/article/details/56494925