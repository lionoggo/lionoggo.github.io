title: StringBuilder & StringBuffer设计与实现
date: 2015-12-1 11:29:39
toc: true
tags: [String,StringBuffer,StringBuilder]
categories: technology
description: 和String不同,StringBuilder和StringBuffer都是基于数组扩容来实现,其核心代码在其父类AbstractStringBuilder中.可以说StringBuilder和StringBuffer的实现原理是一样的,唯一的不同的是StringBuffer的大多数方法都是用synchronize修饰.



-----

#  AbstractStringBuilder

`StringBuilder`与`StringBuffer`是两个常用的操作字符串的类,两者最大的区别在于`StringBuilder`是线程不安全的，而`StringBuffer`是线程安全的,此外前者是JDK1.5加入的.后者在JDK1.0就有了.

和String不同,StringBuilder和StringBuffer都是基于数组扩容来实现,其核心代码在其父类AbstractStringBuilder中.可以说StringBuilder和StringBuffer的实现原理是一样的,唯一的不同的是StringBuffer的大多数方法都是用synchronize修饰,即线程安全.直接看下其类图更明了些:

 ![image-20181023093009963](https://i.imgur.com/5dT87rr.png)

`AbstractStringBuilder`内部用一个`char[]`数组保存字符串，可以在构造的时候指定初始容量方法,其构造方法定义如下:

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    char[] value;	// 用于保存字符串
    int count;
    
    AbstractStringBuilder() {
    }

	// 在创建时可以为其指定初始化容量.
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];
    }
    
}
```



## 扩容算法

正如之前我们提到StringBuilder和StringBuffer最核心的地方是数据扩容,现在我们来看一下其数组扩容的原理.由于StringBuilder和StringBuffer都是AbstractStringBuilder的子类,其扩容原理一致,因此扩容算法在AbstrStringBuilder中实现.数组扩容算法主要由`ensureCapacityInternal()`来实现:

```java
  private void ensureCapacityInternal(int minimumCapacity) {
        //minimumCapcity为当前所需容量大小,value.length表示当前数组容量大小
        if (minimumCapacity - value.length > 0) {
            //数组扩容,先计算扩容之后的容量,再将原数组内容拷贝
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));
        }
    }
```

`newCapacity()`用于计算扩容之后数组的大小:

```java
 private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int newCapacity = (value.length << 1) + 2;
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }
```

不难看出扩容算法是首先把容量扩大为原来的两倍,然后加2.如果扩容之后容量仍小于指定容量,那么就把新的容量设置为指定容量minCapacity.接下来对新的容量进行判断,主要是对溢出情况的判断,因此可能会抛出OutMemoryError.



## 字符串追加

append()是我们最常用的方法,用于实现字符串追加.append()有很多的重载方法,用于实现不同类型的数据的追加.常见用于追加String类型的方法定义如下:

```java
  public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        //1. 数组扩容
        ensureCapacityInternal(count + len);
        //2. 将str拷贝到value数组中
        str.getChars(0, len, value, count);
        //3. 修改count
        count += len;
        return this;
    }
```

所有的追加操作实现比较类似,首先进行扩容操作,其次进行字符串的拷贝,最后进行字符数中字符个数的修改.需要注意如果`str`是`null`,则会调用`appendNull()`方法。这个方法其实是追加了`'n'`、`'u'`、`'l'`、`'l'`这几个字符。如果不是`null`,则首先进行扩容操作,然后调用`String`的`getChars()`方法将`str`追加到`value`末尾.最后返回对象本身，所以`append()`可以连续调用.

## 字符串截取

在AbstractStringBuilder中提供了`subString()`方法用于字符串的截取操作,其最终都通过以下方法实现:

```java
    public String substring(int start, int end) {
        if (start < 0)
            throw new StringIndexOutOfBoundsException(start);
        if (end > count)
            throw new StringIndexOutOfBoundsException(end);
        if (start > end)
            throw new StringIndexOutOfBoundsException(end - start);
        return new String(value, start, end - start);
    }
```

不难发现截取操作返回的是一个新的String对象,在生成该String对象的时候内部需要进行数组拷贝操作,相对耗时,因此在操作"长"字符串时需要格外注意.

# StringBuilder

由于AbstractStringBuilder实现字符串操作和核心操作,因此StringBuilder只需要继承它,并调用父类的大部分操作即可.首先我们来看其构造函数:

```java
    public StringBuilder() {
        super(16);
    }

    public StringBuilder(int capacity) {
        super(capacity);
    }

    public StringBuilder(String str) {
        super(str.length() + 16);
        append(str);
    }
```

可以看出StringBuilder的默认的容量大小为16,当然也允许我们指定初始化容量.在某些情况下我们可能直接用String类型初始化它,此时的初始化容量大小是:String类型的长度+16,比如初始化字符串是"Hello",那么此时默认的容量大小为5+16.



## 字符串追加

StringBuilder的append()方法直接调用了父类AbstractStringBuilder中的方法,如`append(String str)`

```java
@Override
    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }
```

需要注意,对于StringBuilder而言,不要进行没必要的toString()方法调用,因为每次调用toString()时,都new了一个新的String对象.因此频繁调用会增加gc的工作量,影响性能.

```java
 @Override
    public String toString() {
        // Create a copy, don't share the array
        return new String(value, 0, count);
    }
```

# StringBuffer

和StringBuilder实现类似,只不过考虑到线程同步的问题,在StringBuffer中很多方法使用synchronized修饰,以`append(String str)`为例:

```java
  public synchronized StringBuffer append(String str) {
        toStringCache = null;
        super.append(str);
        return this;
    }
```

## 字符串数组缓存

和StringBuilder不同的是StringBuffer中多了toStingCache属性,其定义如下:

```java
 public final class StringBuffer extends AbstractStringBuilder implements java.io.Serializable, CharSequence {
	
	private transient char[] toStringCache;

}
```

toStringCache用于缓存最后一次调用toString时的字符数组.一旦StringBuffer发生修改时(比如append时),该字段会被重新置为null,其实现如下:

```java
   @Override
    public synchronized String toString() {
        if (toStringCache == null) {
            toStringCache = Arrays.copyOfRange(value, 0, count);
        }
        return new String(toStringCache, true);
    }
```

在toString()中除了会初始化toStringCache对象,返回的String对象也和StringBuider的不一样:构造出来的String对象并没有实际复制字符串,只是把value指定了构造参数.这样可以节省复制元素的时间.

```java
    String(char[] value, boolean share) {
        // assert share : "unshared not supported";
        this.value = value;
    }
```

# 总结

- `StringBuilder`和`StringBuffer`都是可变字符串，前者线程不安全，后者线程安全。
- `StringBuilder`和`StringBuffer`的大部分方法均调用父类`AbstractStringBuilder`的实现。其扩容机制首先是把容量变为原来容量的2倍加2。最大容量是`Integer.MAX_VALUE`，也就是`0x7fffffff`。
- `StringBuilder`和`StringBuffer`的默认容量都是16，事先预先估计好字符串的大小避免扩容带来的时间消耗。

需要注意,在jdk 8之前,字符串常量被保存在PermGen中,而从jdk 1.8开始,JVM采用Metaspace来代替了PermGen,字符串常量也因此被转移到了堆区.