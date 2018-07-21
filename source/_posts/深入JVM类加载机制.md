title: 谈JVM类加载机制
date: 2015/11/13 14:13:50
toc  : true
tags: [JVM,Classloader]
categories: technology
description: ClassLoader顾名思义就是我们所常见的类加载器,其作用就是将编译后的class文件加载内存当中.在应用启动时,JVM通过ClassLoader加载相关的类到JVM当中.在具体了解ClassLoader之前我们先来了解下JVM的类加载机制.

----

# 类加载机制

 虚拟机将class文件加载到内存，并对数据校验、转换解析和初始化，最终形成可以被虚拟机直接使用的java类型。在java中语言中类的加载、连接和初始化过程都是在程序运行期间完成的，因此在类加载时的效率相对编译型语言较低，除此之外，只有在任何一个类只有在运行期间使用到该类的时候才会将该类加到内存中。总之，java依赖于运行期间动态加载和动态链接来实现类的动态使用。其整个流程如下：

  ![image-20180721131144810](http://pbj0kpudr.bkt.clouddn.com/blog/2018-07-21-051144.png)

  其中加载、检验、准备、初始化和卸载这个五个阶段的顺序是固定的，而解析则未必。为了支持动态绑定，解析这个过程可以发生在初始化阶段之后。另外，这个过程表示的是按顺序开始，不是所谓的第一步、第二步、第三步的关系，而往往是交叉混合进行，在一个阶段中可能调用或者激活另一个过程。
  java中，对于初始化阶段，有且只有以下五种情况才会对要求类立刻“初始化”：

1. 使用new关键字实例化对象、访问或者设置一个类的静态字段（被final修饰、编译器优化时已经放入常量池的例外）、调用类方法，都会初始化该静态字段或者静态方法所在的类。
2. 初始化类的时候，如果其父类没有被初始化过，则要先触发其父类初始化。
3. 使用java.lang.reflect包的方法进行反射调用的时候，如果类没有被初始化，则要先初始化。
4. 虚拟机启动时，用户会先初始化要执行的主类（含有main）
5. jdk 1.7后，如果java.lang.invoke.MethodHandle的实例最后对应的解析结果是 REF_getStatic、REF_putStatic、REF_invokeStatic方法句柄，并且这个方法所在类没有初始化，则先初始化。

按照以上流程，我们重点来看`加载-校验-准备-解析-初始化`五个阶段。

## 加载阶段

加载过程主要完成三件事情：

 1. 通过类的全限定名来获取定义此类的二进制字节流
 2. 将这个类字节流代表的静态存储结构转为方法区的运行时数据结构
 3. 在堆中生成一个代表此类的java.lang.Class对象，作为访问方法区这些数据结构的入口。

这个过程主要就是类加载器完成。（对于HotSpot虚拟而言，Class对象较为特殊，其被放置在方法区而不是堆中）

## 校验阶段

此阶段主要确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机的自身安全。主要包括以下四个阶段：

  1. 文件格式验证：基于字节流验证，验证字节流符合当前的Class文件格式的规范，能被当前虚拟机处理。验证通过后，字节流才会进入内存的方法区进行存储。
  2. 元数据验证：基于方法区的存储结构验证，对字节码进行语义验证，确保不存在不符合java语言规范的元数据信息。
  3. 字节码验证：基于方法区的存储结构验证，通过对数据流和控制流的分析，保证被检验类的方法在运行时不会做出危害虚拟机的动作。
  4. 符号引用验证：基于方法区的存储结构验证，发生在解析阶段，确保能够将符号引用成功的解析为直接引用，其目的是确保解析动作正常执行。换句话说就是对类自身以外的信息进行匹配性校验。 

## 准备阶段

仅仅为类变量（static修饰的变量）分配内存空间并且设置该类变量的初始值（这里的初始值指的是数据类型默认的零值），这里不包含用final修饰的static，因为用final修饰的类变量在javac执行编译期间就会分配，同时要注意，这里不会为实例变量分配初始化。类变量会分配在方法区中，而实例变量会在对象实例化是随着对象一起被分配在java堆中。举例：

```java
public static int value=33;
```

这据代码的赋值过程分两次，一是上面我们提到的阶段，此时的value将会被赋值为0；而value=33这个过程发生在类构造器的`<clinit>()`方法中。

## 解析阶段

解析阶段主要是将常量池内的符号引用替换为直接引用的过程。符号引用是用一组符号来描述目标，可以是任何字面量，而直接引用则是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。通常而言一个符号引用在不同虚拟机实例翻译出来的直接引用一般不会相同。
和C之类的纯编译型语言不同，Java类文件在编译过程中只会生成class文件，并不会进行连接操作，这意味在编译阶段Java类并不知道引用类的实际地址，因此只能用“符号引用”来代表引用类。举个例子来说明，在com.sbbic.Person类中引用了com.sbbic.Animal类，在编译阶段，Person类并不知道Animal的实际内存地址，因此只能用com.sbbic.Animal来代表Animal真实的内存地址。在解析阶段，JVM可以通过解析该符号引用，来确定com.sbbic.Animal类的真实内存地址（如果该类未被加载过，则先加载）。
主要有以下四种：

 1. 类或接口的解析
 2. 字段解析
 3. 类方法解析
 4. 接口方法解析

## 初始化阶段

类加载过程的最后一步，到该阶段才真正开始执行类中定义的java代码，同样该阶段也是初始化类变量和其他资源（执行static字段和静态代码块），换句话说该阶段是执行类构造器`<clinit>()`方法的过程。
`<clinit>`()方法是由编译器自动收集类中所有的类变量的赋值动作和静态语句块（static{}）中的语句合并而成。`<clinit>()`方法和实例构造方法不同`<init>()`不同，它不需要显示的调用父类的`<clinit>()`，虚拟机会保证父类`的<clinit>(`)方法在子类的`<clinit>()`方法之前执行完成，也就是说，父类的静态语句块和静态变量优先于子类中变量赋值操作。
`<clinit>()`方法对于类或者接口来说并不是必需的，如果一个类中没有静态语句块，也没有对变量赋值的操作，那么编译器就不会为这个类生`成<clinit>(`)方法。接口中不能使用静态语句块，单仍然有变量赋值的操作，所以仍然可以生成`<clinit>(`)方法，但与类不同的执行接口`<clinit>()`方法不需要先执行父接口的`<clinit>()`方法，另外，接口的实现类在初始化时也一样不会执行接口的`<clinit>()`方法。


# 类加载器

把类加载阶段的“通过一个类的全限定名来获取描述此类的二进制字节流”这个动作交给虚拟机之外的类加载器来完成。这样的好处在于，我们可以自行实现类加载器来加载其他格式的类，只要是二进制字节流就行，这就大大增强了加载器灵活性。

通常系统自带的类加载器分为三种：

- 启动类加载器
- 扩展类加载器
- 应用类加载器

**启动类加载器**(Bootstrap ClassLoader):由C/C++实现,负责加载`<JAVA_HOME>\jre\lib`目录下或者是-Xbootclasspath所指定路径下目录以及系统属性sun.boot.class.path制定的目录中特定名称的jar包到虚拟机内存中。在JVM启动时,通过Bootstrap ClassLoader加载rt.jar,并初始化sun.misc.Launcher从而创建Extension ClassLoader和Application ClassLoader的实例.

需要注意的是,Bootstrap ClassLoader智慧加载特定名称的类库,比如rt.jar.这意味我们自定义的jar扔到`<JAVA_HOME>\jre\lib`也不会被加载.我们可以通过以下代码,查看Bootstrap ClassLoader到底初始化了那些类库:

 ```java
  URL[] urLs = Launcher.getBootstrapClassPath().getURLs();
       for (URL urL : urLs) {
           System.out.println(urL.toExternalForm());
       }
 ```

**扩展类加载器**（Extension Classloader）：只有一个实例,由sun.misc.Launcher$ExtClassLoader实现,负责加载`<JAVA_HOME>\lib\ext`目录下或是被系统属性java.ext.dirs所指定路径目录下的所有类库。

**应用类加载器**(Application ClassLoader):只有一个实例,由sun.misc.Launcher$AppClassLoader实现,负责加载系统环境变量ClassPath或者系统属性java.class.path制定目录下的所有类库，如果应用程序中没有定义自己的加载器，则该加载器也就是默认的类加载器.该加载器可以通过以下获取：

`java.lang.ClassLoader.getSystemClassLoader`

以上三种是我们经常认识最多,除此之外还包括**线程上下文类加载器**(Thread Context ClassLoader)和**自定义类加载器**.

每个线程都有一个类加载器(jdk 1.2后引入),称之为Thread Context ClassLoader,即线程上下文类加载器,如果线程创建时没有设置,则默认从父线程中继承一个,如果在应用全局内都没有设置,则所有线程上下文类加载器就是Application ClassLoader.可通过Thread.currentThread().setContextClassLoader(ClassLoader)来设置,通过Thread.currentThread().getContextClassLoader()来获取.

我们来想想线程上下文加载器有什么用的?该类加载器容许父类加载器通过子类加载器加载所需要的类库,也就是打破了我们下文所说的双亲委派模型.这有什么好处呢?利用线程上下文加载器,我们能够实现所有的代码热替换,热部署,Android中的热更新原理也是借鉴如此的.

至于自定义加载器就更简单了,JVM运行我们通过自定义的ClassLoader加载相关的类库.


## 双亲委派模型

当一个类加载器收到一个类加载的请求，它首先会将该请求委派给父类加载器去加载，每一个层次的类加载器都是如此，因此所有的类加载请求最终都应该被传入到顶层的启动类加载器(Bootstrap ClassLoader)中，只有当父类加载器反馈无法完成这个列的加载请求时（它的搜索范围内不存在这个类），子类加载器才尝试加载。其层次结构示意图如下：

![image-20180721131241091](http://pbj0kpudr.bkt.clouddn.com/blog/2018-07-21-051241.png)


不难发现,该种加载流程的好处在于：

 1. 可以避免重复加载，父类已经加载了，子类就不需要再次加载
 2. 更加安全，很好的解决了各个类加载器的基础类的统一问题，如果不使用该种方式，那么用户可以随意定义类加载器来加载核心api，会带来相关隐患。

接下来,我们看看双亲委派模型是如何实现的：
```java
 protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先先检查该类已经被加载过了
            Class c = findLoadedClass(name);
            if (c == null) {//该类没有加载过，交给父类加载
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {//交给父类加载
                        c = parent.loadClass(name, false);
                    } else {//父类不存在，则交给启动类加载器加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                   //父类加载器抛出异常，无法完成类加载请求
                }

                if (c == null) {//
                    long t1 = System.nanoTime();
                    //父类加载器无法完成类加载请求时，调用自身的findClass方法来完成类加载
                    c = findClass(name);
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

```

这里有些童鞋会问,JVM怎么知道一个某个类加载器的父加载器呢?如果你有此疑问,请重新再看一遍.


## 类加载器的特点

 1. 运行任何一个程序时，总是由Application Loader开始加载指定的类。
 2. 一个类在收到加载类请求时，总是先交给其父类尝试加载。
 3. Bootstrap Loader是最顶级的类加载器，其父加载器为null。


## 类加载的方式

 1. 通过命令行启动应用时由JVM初始化加载含有main()方法的主类。
 2. 通过Class.forName()方法动态加载，会默认执行初始化块（static{}），但是Class.forName(name,initialize,loader)中的initialze可指定是否要执行初始化块。
 3. 通过ClassLoader.loadClass()方法动态加载，不会执行初始化块。


## 自定义类加载器

1、遵守双亲委派模型：继承ClassLoader，重写findClass()方法。
2、破坏双亲委派模型：继承ClassLoader,重写loadClass()方法。
通常我们推荐采用第一种方法自定义类加载器，最大程度上的遵守双亲委派模型。
自定义类加载的目的是想要手动控制类的加载,那除了通过自定义的类加载器来手动加载类这种方式,还有其他的方式么?
利用现成的类加载器进行加载:
```java
1. 利用当前类加载器
Class.forName();

2. 通过系统类加载器
Classloader.getSystemClassLoader().loadClass();

3. 通过上下文类加载器
Thread.currentThread().getContextClassLoader().loadClass();
```

利用URLClassLoader进行加载:
```java
URLClassLoader loader=new URLClassLoader();
loader.loadClass();
```

**类加载实例演示：**
命令行下执行HelloWorld.java
```java
public class HelloWorld{
    public static void main(String[] args){
        System.out.println("Hello world");
    }
}
```

该段代码大体经过了一下步骤:

 1. 寻找jre目录，寻找jvm.dll，并初始化JVM.
 2. 产生一个Bootstrap ClassLoader；
 3. Bootstrap ClassLoader加载器会加载他指定路径下的java核心api，并且生成Extended ClassLoader加载器的实例，然后Extended ClassLoader会加载指定路径下的扩展java api，并将其父设置为Bootstrap ClassLoader。
 4. Bootstrap ClassLoader生成Application ClassLoader，并将其父Loader设置为Extended ClassLoader。
 5. 最后由AppClass ClassLoader加载classpath目录下定义的类——HelloWorld类。

我们上面谈到 Extended ClassLoader和Application ClassLoader是通过Launcher来创建,现在我们再看看源代码:

```java
 public Launcher() {
        Launcher.ExtClassLoader var1;
        try {
	        //实例化ExtClassLoader
            var1 = Launcher.ExtClassLoader.getExtClassLoader();
        } catch (IOException var10) {
            throw new InternalError("Could not create extension class loader", var10);
        }

        try {
            //实例化AppClassLoader
            this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);
        } catch (IOException var9) {
            throw new InternalError("Could not create application class loader", var9);
        }
		//主线程设置默认的Context ClassLoader为AppClassLoader.
		//因此在主线程中创建的子线程的Context ClassLoader 也是AppClassLoader
        Thread.currentThread().setContextClassLoader(this.loader);
        String var2 = System.getProperty("java.security.manager");
        if(var2 != null) {
            SecurityManager var3 = null;
            if(!"".equals(var2) && !"default".equals(var2)) {
                try {
                    var3 = (SecurityManager)this.loader.loadClass(var2).newInstance();
                } catch (IllegalAccessException var5) {
                    ;
                } catch (InstantiationException var6) {
                    ;
                } catch (ClassNotFoundException var7) {
                    ;
                } catch (ClassCastException var8) {
                    ;
                }
            } else {
                var3 = new SecurityManager();
            }

            if(var3 == null) {
                throw new InternalError("Could not create SecurityManager: " + var2);
            }

            System.setSecurityManager(var3);
        }

    }
```



## 注意
在明确上述内容之后，来思考几个问题:
1. 我们知道ClassLoader通过一个类的全限定名来获取二进制流,那么如果我们需要通过自定义类加载其来加载一个Jar包的时候,难道要自己遍历jar中的类,然后依次通过ClassLoader进行加载吗?或者说我们怎么来加载一个jar包呢?
2. 如果一个类引用的其他的类,那么这个其他的类由谁来加载?
3. 既然类可以由不同的加载器加载,那么如何确定两个类如何是同一个类?

我们来依次解答这两个问题:
对于动态加载jar而言,JVM默认会使用第一次加载该jar中指定类的类加载器作为默认的ClassLoader.假设我们现在存在名为sbbic的jar包,该包中存在ClassA和ClassB这两个类(ClassA中没有引用ClassB).现在我们通过自定义的ClassLoaderA来加载在ClassA这个类,那么此时此时ClassLoaderA就成为sbbic.jar中其他类的默认类加载器.也就是,ClassB也默认会通过ClassLoaderA去加载.

那么如果ClassA中引用了ClassB呢?当类加载器在加载ClassA的时候,发现引用了ClassB,此时类加载如果检测到ClassB还没有被加载,则先回去加载.当ClassB加载完成后,继续回来加载ClassA.换句话说,类会通过自身对应的来加载其加载其他引用的类.

JVM规定,对于任何一个类,都需要由加载它的类加载器和这个类本身一同确立在java虚拟机中的唯一性,通俗点就是说,在jvm中判断两个类是否是同一个类取决于类加载和类本身,也就是同一个类加载器加载的同一份Class文件生成的Class对象才是相同的,类加载器不同,那么这两个类一定不相同.




[1]: http://img.blog.csdn.net/20151022154227640