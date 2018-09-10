title: OpenJDK系列(三):JVM对CAS的设计与实现
date: 2018/6/27 11:22:50
toc  : true
tags: [OpenJDK,HotSpot,CAS]
categories: technology
description: 当且仅当V的值等于A时,CAS才会通过原子方式用新值B来更新V的值,否则不会执行任何操作.无论位置V的值是否等于A,最终都会返回V原有的值.换句话说:"我认为V的值应该是A,如果是,那么就将V的值更新为B,否则不修改并告诉V的实际值是多少".

-----

# CAS简介

CAS即Compare-and-Swap的缩写,即比较并交换,它是一种实现乐观锁的技术.在CAS中包含三个操作数:

- V: 需要读写的内存位置,从java角度你可以把它当成一个变量
- A: 预期值,也就是要进行比较的值
- B: 拟写入的新值

当且仅当V的值等于A时,CAS才会通过**原子**方式用新值B来更新V的值,否则不会执行任何操作.无论位置V的值是否等于A,最终都会返回V原有的值.换句话说:"我认为V的值应该是A,如果是,那么就将V的值更新为B,否则不修改并告诉V的实际值是多少".

当多个线程使用CAS同时更新同一个变量时,只有其中一个线程能够成功更新变量的值,其他线程都将失败.和锁机制不同,失败的线程并不会被挂起,而是告知用户当前失败的情况,并由用户决定是否要再次尝试或者执行其他操作,其典型的流程如下:

![image-20180910121551030](https://i.imgur.com/P4iFBcL.png)

## 传统锁实现CAS语义

在明白CAS的语义后,我们用传统的锁机制来实现该语义.

```java
public class SimpleCAS {
    private int value;

    public int getValue() {
        return value;
    }

    // 比较并交换语义,最终都返回原有值
    public synchronized int compareAndSwap(int exectedValue, int newValue) {
        int oldValue = value;
        if (oldValue == exectedValue) {
            value = newValue;
        }
        return value;
    }

    // 比较并设置语义,返回成功与否    
    public synchronized boolean compareAndSet(int exectedValue, int newValue) {
        return exectedValue == compareAndSwap(exectedValue, newValue);
    }
}
```

在上述代码中,`compareAndSwap()`用于实现"比较并交换"的语义,在此之上我们还实现了"比较并设置"的语义.

## 使用场景

CAS典型使用模式是:首先从V中读取值A,并根据A计算出新值B,然后再通过CAS以原子方式将V中的值变成B(如果在此期间没有任何线程将V的值修改为其他值).我们借助刚才的SimpleCAS实现一个计数器,借此来说明其使用场景:

```java
// 线程安全的计数器
public class SafeCounter {
    private SimpleCAS cas;

    public SafeCounter() {
        this.cas = new SimpleCAS();
    }

    public int getValue() {
        return cas.getValue();
    }

    public int increment() {
        int value;
        int newValue;
        do {
            // 读取旧值A
            value = cas.getValue();
            // 根据A计算新值B
            newValue = value + 1;
        } while (!cas.compareAndSet(value, newValue));// 使用CAS来设置新值B
        return newValue;
    }
}

```

SafeCounter不会阻塞,如果其他线程同时更新计数器,那么会执行多次重试操作直至成功.到现在有关CAS的语义和使用已经说完,下面我们要说的是CAS在JAVA中的应用以及JVM中如何实现CAS.

# CAS实现

通过传统的锁实现的CAS语义并非JVM真正对CAS的实现,这点需要记住.JVM中能够实现CAS本质是现代CPU已经支持Compare-and-Swap指令.从Java 5.0开始,JVM中直接调用了相关指令.

## JVM对CAS的支持

有关原子性变量的操作被统一定义在atomic.hpp,并以模板方法提供,其路径为:

`/OpenJDK10/hotspot/src/share/vm/runtime/atomic.hpp`

```c++
template<typename T, typename D, typename U>
inline D Atomic::cmpxchg(T exchange_value,
                         D volatile* dest,
                         U compare_value,
                         cmpxchg_memory_order order) {
  return CmpxchgImpl<T, D, U>()(exchange_value, dest, compare_value, order);
}

template<typename T>
struct Atomic::CmpxchgImpl<
  T, T, T,
  typename EnableIf<IsIntegral<T>::value || IsRegisteredEnum<T>::value>::type>
  VALUE_OBJ_CLASS_SPEC
{
  T operator()(T exchange_value, T volatile* dest, T compare_value,
               cmpxchg_memory_order order) const {
    // Forward to the platform handler for the size of T.
    return PlatformCmpxchg<sizeof(T)>()(exchange_value,
                                        dest,
                                        compare_value,
                                        order);
  }
};
```

不同的平台PlatformCmpxchg实现不同,比如在mac平台上,其实现在

`/OpenJDK10/hotspot/src/os_cpu/bsd_x86/vm/atomic_bsd_x86.hpp`

```c++
// 1字节长度
template<typename T>
inline T Atomic::PlatformCmpxchg<1>::operator()(T exchange_value,
                                                T volatile* dest,
                                                T compare_value,
                                                cmpxchg_memory_order /* order */) const {
  STATIC_ASSERT(1 == sizeof(T));
  // 内嵌汇编代码,最终调用cmpxchgb指令实现"比较并交换"  
  __asm__ volatile (  "lock cmpxchgb %1,(%3)"
                    : "=a" (exchange_value)
                    : "q" (exchange_value), "a" (compare_value), "r" (dest)
                    : "cc", "memory");
  return exchange_value;
}
// 4字节长度
template<typename T>
inline T Atomic::PlatformCmpxchg<4>::operator()(T exchange_value,
                                                T volatile* dest,
                                                T compare_value,
                                                cmpxchg_memory_order /* order */) const {
  STATIC_ASSERT(4 == sizeof(T));
  __asm__ volatile (  "lock cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest)
                    : "cc", "memory");
  return exchange_value;
}
// 8字节长度
template<typename T>
inline T Atomic::PlatformCmpxchg<8>::operator()(T exchange_value,
                                                T volatile* dest,
                                                T compare_value,
                                                cmpxchg_memory_order /* order */) const {
  STATIC_ASSERT(8 == sizeof(T));
  __asm__ __volatile__ (  "lock cmpxchgq %1,(%3)"
                        : "=a" (exchange_value)
                        : "r" (exchange_value), "a" (compare_value), "r" (dest)
                        : "cc", "memory");
  return exchange_value;
}
```

在window_x86平台中,其实现在`/OpenJDK10/hotspot/src/os_cpu/windows_x86/vm/atomic_windows_x86.hpp`

```c++
template<>
template<typename T>
inline T Atomic::PlatformCmpxchg<1>::operator()(T exchange_value,
                                                T volatile* dest,
                                                T compare_value,
                                                cmpxchg_memory_order order) const {
  STATIC_ASSERT(1 == sizeof(T));
  // alternative for InterlockedCompareExchange
  // 内嵌汇编代码,最终调用cmpxchg指令实现"比较并交换"   
  __asm {
    mov edx, dest
    mov cl, exchange_value
    mov al, compare_value
    lock cmpxchg byte ptr [edx], cl
  }
}

template<>
template<typename T>
inline T Atomic::PlatformCmpxchg<4>::operator()(T exchange_value,
                                                T volatile* dest,
                                                T compare_value,
                                                cmpxchg_memory_order order) const {
  STATIC_ASSERT(4 == sizeof(T));
  // alternative for InterlockedCompareExchange
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    lock cmpxchg dword ptr [edx], ecx
  }
}

template<>
template<typename T>
inline T Atomic::PlatformCmpxchg<8>::operator()(T exchange_value,
                                                T volatile* dest,
                                                T compare_value,
                                                cmpxchg_memory_order order) const {
  STATIC_ASSERT(8 == sizeof(T));
  jint ex_lo  = (jint)exchange_value;
  jint ex_hi  = *( ((jint*)&exchange_value) + 1 );
  jint cmp_lo = (jint)compare_value;
  jint cmp_hi = *( ((jint*)&compare_value) + 1 );
   // 内嵌汇编代码,最终调用cmpxchg8b指令实现"比较并交换"8字节   
  __asm {
    push ebx
    push edi
    mov eax, cmp_lo
    mov edx, cmp_hi
    mov edi, dest
    mov ebx, ex_lo
    mov ecx, ex_hi
    lock cmpxchg8b qword ptr [edi]
    pop edi
    pop ebx
  }
}
```

不难发现,最终都是通过内嵌汇编代码的形式来实现对于CPU指令cmpxchg的调用,关于该指令后续单独进行说明.到目前为止,对于JVM中的CAS操作已经了解的差不多了,但在Java层又是如何使用的呢?在开始了解Java层之前,我们先来看JVM是如何向Java层暴露这些操作的.

Java层无法直接调用CPU指令,必须借助JNI,这里对CAS的调用在Java层就体现在sun.misc.Unsafe类上,UnSafe类中定义了很多Native方法:

```java
public final class Unsafe {

    private static native void registerNatives();
    static {
        registerNatives();
    }

    private Unsafe() {}

    private static final Unsafe theUnsafe = new Unsafe();
    
    public final native boolean compareAndSetObject(Object o, long offset,
                                                    Object expected,
                                                    Object x);
    
    public final native boolean compareAndSetInt(Object o, long offset,
                                                 int expected,
                                                 int x);
    .......
}

```

其对应的C++实现类是:

`/OpenJDK10/hotspot/src/share/vm/prims/unsafe.cpp`,这里的compareAndSetInt()其实就对应于unsafe.cpp中的Unsafe_CompareAndSetInt():

```c++
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) {
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *)index_oop_from_field_offset_long(p, offset);
  // 调用Atomic.cpp中的cmpxchg()
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
} UNSAFE_END
```

总结一下,sun.misc.Unsafe中Native方法的调用,最终都会通过JNI调用到unsafe.cpp中,而unsafe.cpp中的实现本质都是调用CPU的cmpxchg指令.关于cmpxchg指令将在后续单独说明.

到现在为止,JVM如何实现CAS以及如何向Java层暴露CAS操作这两个流程已经比较明了了,接下来还是要回归到Java层,来明白Java层中对CAS的支持.

## Java层对CAS的支持

在Java层面,原子变量类(java.util.concurrent.atomic中的AtomicXXX)在底层充分使用了来此JVM对CAS的支持,来实现高效的原子操作,此外,java.util.concurrent中的大多数类在实现时也是借助了这些原子变量类.以AtomicInteger为例,来了解下Atomic如何使用CAS.

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;
    // Unsafe类型的成员变量U
    private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
    private static final long VALUE = U.objectFieldOffset(AtomicInteger.class, "value");

    private volatile int value;

    public AtomicInteger(int initialValue) {
        value = initialValue;
    }

    public AtomicInteger() {
    }

    public final int get() {
        return value;
    }

    public final boolean compareAndSet(int expectedValue, int newValue) {
        return U.compareAndSetInt(this, VALUE, expectedValue, newValue);
    }
    
    ......
}

```

通过上述代码不难看出,AtomicInteger本质就是CAS在Java层的应用.在明白CAS原理后,对AtomicInteger并不会感到难以理解.可以说正是有了CPU对Compare-and-Swap的支持,才使得Java在并发上有了突飞猛进的提升.另外在不支持Compare-and-Swap的平台上,JVM将使用自旋锁来代替.

# CAS缺陷

ABA问题是CAS中是一种常见的问题:在于并发环境下,当第一个线程执行CAS(V,A,B)操作,在已经获取到当前变量V，但还没将其修改为新值B前，其他线程在其期间连续修改了两次变量V的值，使得该值又恢复为旧值.在这种情况下,我们无法判断这个变量是否已被修改过.其流程如下:

![image-20180910153730654](https://i.imgur.com/GyRDyLY.png)

在大多数情况下,ABA问题并不会影响最终的计算结果,但如果需要避免ABA问题该怎么办呢?从上述流程看出导致ABA的问题个根源是在时间线推进过程中没有为每次修改记录版本号导致.解决该问题只需要在每次修改时记录下其当前时间戳作为版本号就可以避免.在Java中,AtomicStampedReference原子类实现原理就是如此:设置值时要求对象值以及时间戳都必须满足期望值才能写入成功.(可以理解为git中的commit-id,先将A修改为B,再修改成A,最终内容虽然没变,但通过版本commit-id,我们仍然可以知道中间发生了变化)

除了ABA问题外,如果自旋CAS长时间失败会给CPU带来很大的开销,在并发激烈的时候该问题尤其明显.另外,CAS使用场景比较单一,只能用于保证一个共享变量的原子操作.

