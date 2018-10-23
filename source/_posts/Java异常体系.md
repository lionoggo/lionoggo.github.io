title: Java异常体系
date: 2015-1-17 22:13:11
toc: true
tags: [JVM,Java,Exception,Error]
categories: technology
description: 任何程序都追求正确有效的运行,除了保证我们代码尽可能的少出错之外,我们还要考虑如何有效的处理异常,一个良好的异常框架对于系统来说是至关重要的.最近在给公司写采集框架的时候系统的总结下,收获颇多,特此记录相关的理论.

------

# 异常体系

异常是指由于各种不期而至的情况,导致程序中断运行的一种指令流,如:文件找不到,非法参数,网络超时等.为了保证正序正常运行,在设计程序时必须考虑到各种异常情况,并正确的对异常进行处理.异常也是一种对象,java当中定义了许多异常类,并且定义了java.lang.Throwable作为所有异常类的基类.

Java语言设计者将异常划分为两类:Error和Exception,其体系结构大致如下图所示：

![image-20181023103106457](https://i.imgur.com/HHbCeed.png)

Throwable有两个重要的子类:Exception和Error.Error是程序中无法处理的错误,表示运行应用程序中出现了严重的错误.此类错误一般表示代码运行时JVM出现问题.通常有Virtual MachineError,NoClassDefFoundError,OutOfMemoryError等,这些错误是不可查的,通常是非代码性错误.Exception则是程序本身可以捕获并且可以处理的异常.我们重点来看下Throwable类中常用的方法:

- e.getCause():返回抛出异常的原因

- e.getMessage():返回异常信息

- e.printStackTrace():发生异常时，跟踪堆栈信息并输出

## 运行时异常与编译异常

Exception这种异常又分为两类: 运行时异常和编译异常。

- 运行时异常: 程序运行期间抛出的异常.RuntimeException类及其子类表示JVM在运行期间可能出现的错误.比如`NullPointerException`等.此类异常属于不可查异常,一般是由程序逻辑错误引起的,在程序中可以选择捕获处理.
- 编译异常: 在编译阶段抛出的异常.Exception中除RuntimeException及其子类之外的异常.如果程序中出现此类异常,如说IOException,必须对该异常进行处理,否则编译不通过.在程序中,通常不会自定义该类异常,而是直接使用系统提供的异常类.

## 可查异常与不可查异常

java中的异常其中一共就分为两类:checked execption和unchecked exception.

- checked exception: 编译器要求必须处理的异常,除RuntimeException及其子类外.其他的Exception异常都属于可查异常.编译器会检查此类异常,也就是说当编译器检查到应用中的某处可能会此类异常时，将会提示你处理本异常:要么使用try-catch捕获,要么使用throws语句抛出,否则编译不通过.

- unchecked exception: 编译器不会进行检查并且不要求必须处理的异常,也就说当程序中出现此类异常时,即使我们没有try-catch捕获它,也没有使用throws抛出该异常,编译也会正常通过.该类异常包括RuntimeException及其子类和Error.

# 异常处理流程

 在java应用中，当一个方法出现错误而引发异常时，虚拟机会将该异常类型以及异常出现时的程序状态信息封装为异常对象对象,JVM采用不同的处理方式:

- 运行异常将由系统自动抛出,应用本身可以捕获该异常或者继续往外层抛出
- 对于方法中产生的Error,一旦发生JVM会自行处理
- 对于所有的可查异常,必须进行捕获或者往外抛出交给上层处理.

## 捕获异常

### try-catch

```java
try {
	......
     //可能产生的异常的代码区，也称为监控区
	......
    }catch (ExceptionType1 e) {
     //捕获并处理try抛出异常类型为ExceptionType1的异常
    }catch(ExceptionType2 e) {
     //捕获并处理try抛出异常类型为ExceptionType2的异常
    }
```

监控区一旦发生异常,虚拟机根据当前运行时的信息创建异常对象，并将该异常对象抛出监控区，同时虚拟机根据该异常对象依次匹配catch子句.若匹配成功(抛出的异常对象的类型和catch子句的异常类的类型或者是该异常类的子类的类型一致),则运行其中catch代码块中的异常处理代码.一旦处理结束那就意味着整个try-catch结束.对于含有多个catch子句的情况,一旦其中一个catch子句与抛出的异常对象类型一致时,其他catch子句将不再有匹配异常对象的机会.

### try-catch-finally

```java
  try {
      .......
       //可能产生的异常的代码区
	  .......
   }catch (ExceptionType1 e) {
       //捕获并处理try抛出异常类型为ExceptionType1的异常
   }catch (ExceptionType2 e){
       //捕获并处理try抛出异常类型为ExceptionType2的异常
   }finally{
       //无论是出现异常，finally块中的代码都将被执行
   }
```

try-catch-finally代码块的执行顺序按照以下方式进行:

1. try没有捕获异常时,try代码块中的语句依次被执行,跳过catch.如果存在finally则执行finally代码块,否则执行后续代码.

2. try捕获到异常时,如果没有与之匹配的catch子句,则该异常交给JVM处理.如果存在finally,则其中的代码仍然被执行,但是finally之后的代码不会被执行.

3. try捕获到异常时,如果存在与之匹配的catch,则跳到该catch代码块执行处理.如果存在finally则执行finally代码块.执行完finally代码块之后继续执行后续代码;否则直接执行后续代码.需要注意,try代码块出现异常之后的代码不会被执行,如下图所示:

   ![image-20181023105901877](https://i.imgur.com/SfVcmte.png)

### 小结

- try代码块:用于捕获异常.其后可以接零个或者多个catch块.如果没有catch块,后必须跟finally块来完成资源释放等操作.另外建议不要在finally中使用return.不用尝试通过catch来控制代码流程.
- catch代码块:用于捕获异常,并在其中处理异常.
- finally代码块:无论是否捕获异常,finally代码总会被执行.如果try代码块或者catch代码块中有return语句时,finally代码块将在方法返回前被执行.注意以下几种情况，finally代码块不会被执行
  1. 在前边的代码中使用
  2. 程序所在的线程死亡或者(系统突然关机)
  3. 如果在finally代码块中的操作又产生异常,则该finally代码块不能完全执行结束,同时该异常会覆盖之前抛出的异常

## 抛出异常

### throws

如果一个方法可能抛出异常,但是没有能力处理该异常或者需要通过该异常向上层汇报处理结果,可以在方法声明时使用throws来抛出异常,这就相当于计算机硬件发生损坏,但是计算机本身无法处理,就将该异常交给维修人员来处理,

```java
public returnType methodName(params) throws Exception1,Exception2......{ }
```

其中Exception1,Exception2…代表该方法可能抛出的异常.该方法的调用者需要对其进行处理,当然也可以继续往外抛出.

### throw

在方法内,可以用throw来抛出一个Throwable类型的异常.JVM一旦执行到throw语句，将退出该方法栈,也就是throw后面的代码将不会被执行了.需要注意我们只能抛出Throwable和其子类类型的对象。

```java
public returnType methodName (params) {
    ......
        throw new ExceptionType;
    ......
} 
```

# 异常转译

异常转义就是将一种类型的异常转成另一种类型的异常.之所以要进行转译,是为了更准确的描述异常.就我个人而言,我更喜欢称之为异常类型转换.在实际应用中,为了构建自己的日志系统,经常需要把系统的一些异常信息描述成我们统一的异常信息,此时就可以使用异常转译.异常转译针对所有Throwable类的子类而言,其子类型都可以相互转换.

在下面的代码中,我们自定义了MyException异常类,然后我们在遇到IOException类型的异常将其转为MyException类型的异常:

```java
class MyException extends Exception {

   public MyException(String msg, Throwable e) {
      super(msg, e);
   }
}

public class Demo {

   public static void main(String[] args)throws MyException {
      Filefile =new File("H:/test.txt");
      if (file.exists())
          try {
             file.createNewFile();
          }catch (IOException e) {
             throw new MyException("文件创建失败！", e);
          }
   }
}
```

# 异常链

在异常转译的过程中,我们希望在新类型的异常对象中保存原始类型异常对象的信息,即当程序捕获了一个底层的异常,而在catch处理异常的时候选择将该异常封装在一个新类型异常对象中,并将新异常对象继续抛给上层,此时最原始的异常就会逐层传递,形成一个由低到高的异常链.由于异常链每次都需要就将原始的异常对象封装为新的异常对象,消耗大量资源,因此在实际应用中需要特别谨慎.现在（jdk 1.4之后）所有的Throwable的子类构造中都可以接受一个cause对象,这个cause也就是原始的异常对象.下面是一个不错的例子：

```java
// 高层
class HighLevelExceptionextends Exception{

   public HighLevelException(Throwable cause) {
      super(cause);
   }
}
// 中层
class MiddleLevelExceptionextends Exception{

   public MiddleLevelException(Throwable cause) {
      super(cause);
   }
}
// 底层
class LowLevelExceptionextends Exception{
}

public class TestException {

   public void highLevelAccess() throws HighLevelException{
      try {
          middleLevelAccess();
      }catch (Exception e) {
          throw new HighLevelException(e);
      }
   }

   public void middleLevelAccess() throws MiddleLevelException{
      try {
          lowLevelAccess();
      }catch (Exception e) {
          throw new MiddleLevelException(e);
      }
   }

   public void lowLevelAccess() throws LowLevelException {
      throw new LowLevelException();
   }

   public static void main(String[] args) {
      /*
       * lowlevelAccess()将异常对象抛给middleLevelAccess()，而
       * middleLevelAccess()又将异常对象抛给highLevelAccess(),
       *也就是底层的异常对象一层层传递给高层。最终在在高层可以获得底层的异常对象。
       */
      try {
          new TestException().highLevelAccess();
      }catch (HighLevelException e) {
          e.printStackTrace();
          System.out.println(e.getCause());
      }
   }
}
```

# 异常汇总

此部分可以api文档中进行查阅,这里仅做参考.

| 异常类型                        | 含义                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| IllegalAccessError              | 违法访问错误.当一个应用试图访问,修改某个类的域或者调用其方法,但是又违反域或方法的可见性声明,则抛出该异常 |
| InstantiationError              | 实例化错误.当一个应用试图通过Java的new操作符构造一个抽象类或者接口时抛出该异常. |
| OutOfMemoryError                | 内存不足错误.当可用内存不足以让Java虚拟机分配给一个对象时抛出该错误. |
| StackOverflowError              | 堆栈溢出错误.当一个应用递归调用的层次太深而导致堆栈溢出或者陷入死循环时抛出该错误. |
| ClassCastException              | 强制类型转换异常.假设有类A和B(A不是B的父类或子类),O是A的实例,那么当强制将O构造为类B的实例时抛出该异常. |
| ClassNotFoundException          | 找不到类异常.当应用试图根据字符串形式的类名构造类,而在遍历CLASSPAH之后找不到对应名称的class文件时.抛出该异常. |
| ArithmeticException             | 算术条件异常.譬如:整数除零等                                 |
| ArrayIndexOutOfBoundsException  | 数组索引越界异常,当对数组的索引值为负数或大于等于数组大小时抛出 |
| IndexOutOfBoundsException       | 索引越界异常.当访问某个序列的索引值小于0或大于等于序列大小时,抛出该异常. |
| NoSuchFieldException            | 属性不存在异常,当访问某个类的不存在的属性时抛出该异常        |
| NoSuchMethodException           | 方法不存在异常.当访问某个类的不存在的方法时抛出该异常        |
| NullPointerException            | 空指针异常,当应用试图在要求使用对象的地方使用了null时抛出该异常. |
| NumberFormatException           | 数字格式异常.当试图将一个String转换为指定的数字类型.而该字符串确不满足数字类型要求的格式时,抛出该异常. |
| StringIndexOutOfBoundsException | 字符串索引越界异常.当使用索引值访问某个字符串中的字符,而该索引值小于0或大于等于序列大小时,抛出该异常. |
| llegalAccessException           | 违法的访问异常.当应用试图通过反射方式创建某个类的实例,访问该类属性,调用该类方法,而当时又无法访问类 |
| InterruptedException            | 线程被中断异常.当某个线程处于长时间的等待,休眠或其他暂停状态,而此时其他的线程通过Thread的interrupt方法终止该线程时抛出该异常 |
| IllegalStateException           | 违法的状态异常.当在Java环境和应用尚未处于某个方法的合法调用状态,而调用了该方法时,抛出该异常. |
| UnsatisfiedLinkError            | 链接库链接错误.当Java虚拟机未找到某个类的声明为native方法的本机语言定义时抛出. |
|                                 |                                                              |


