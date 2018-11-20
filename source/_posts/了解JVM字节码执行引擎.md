---
title: 谈JVM字节码执行引擎
date: 2015-10-30 18:39:39
toc: true
tags: JVM
categories: technology
description: 我们都知道，在当前的Java中（1.0）之后，编译器讲源代码转成字节码，那么字节码如何被执行的呢？这就涉及到了JVM的字节码执行引擎，执行引擎负责具体的代码调用及执行过程.

---



就目前而言，所有的执行引擎的基本一致：

1. 输入：字节码文件
2. 处理：字节码解析
3. 输出：执行结果。

物理机的执行引擎是由硬件实现的，和物理机的执行过程不同的是虚拟机的执行引擎由于自己实现的。


# 运行时候的栈结构

每一个线程都有一个栈,也就是前文中提到的虚拟机栈，栈中的基本元素我们称之为栈帧。栈帧是用于支持虚拟机进行方法调用和方法执行的数据结构。每个栈帧都包括了一下几部分：局部变量表、操作数栈、动态连接、方法的返回地址 和一些额外的附加信息。

栈帧中需要多大的局部变量表和多深的操作数栈在编译代码的过程中已经完全确定，并写入到方法表的Code属性中。在活动的线程中，位于当前栈顶的栈帧才是有效的，称之为当前帧，与这个栈帧相关联的方法称为当前方法。

执行引擎运行的所有字节码指令只针对当前栈帧进行操作。需要注意的是一个栈中能容纳的栈帧是受限，过深的方法调用可能会导致StackOverFlowError，当然，我们可以认为设置栈的大小。其模型示意图大体如下：

![image-20180815144510852](https://ws2.sinaimg.cn/large/006tNbRwly1fxekc64aj8j30mo0h0mxz.jpg)



针对上面的栈结构，我们重点解释一下局部变量表，操作栈，指令计数器几个概念：

## 局部变量表

是变量值的存储空间，由方法参数和方法内部定义的局部变量组成，其容量用Slot作为最小单位。在编译期间，就在方法的Code属性的max_locals数据项中确定了该方法所需要分配的局部变量表的最大容量。由于局部变量表是建立在线程的栈上，是线程的私有数据，因此不存在数据安全问题。

在方法执行时，虚拟机通过使用局部变量表完成参数值到参数变量列表的传递过程。如果是实例方法，那局部变量表第0位索引的Slot存储的是方法所属对象实例的引用，因此在方法内可以通过关键字this来访问到这个隐含的参数。其余的参数按照参数表顺序排列，参数表分配完毕之后，再根据方法体内定义的变量的顺序和作用域分配。

我们知道类变量表有两次初始化的机会，第一次是在“准备阶段”，执行系统初始化，对类变量设置零值，另一次则是在“初始化”阶段，赋予程序员在代码中定义的初始值。和类变量初始化不同的是，局部变量表不存在系统初始化的过程，这意味着一旦定义了局部变量则必须人为的初始化，否则无法使用。举例说明：

```java
public void test(){
    call(2,3);
    ...
    call2(2,3);
}

public void call(int i,int j){
    int b=2;
      ...
}

public static void call2(int i,int j){
    int b=2;
    ...
}
```

为了方便起见，假设以上两段代码在同一个类中。这时call()所对应的栈帧中的局部变量表大体如下：

![image-20180815144627847](https://ws3.sinaimg.cn/large/006tNbRwly1fxekc71hsdj306203ht8k.jpg)

而call2()所对应的栈帧的局部变量表大体如下：
![image-20180815144659903](https://ws3.sinaimg.cn/large/006tNbRwly1fxekkcrwaej3068034glg.jpg)



## 操作数栈

后入先出栈，由字节码指令往栈中存数据和取数据，栈中的任何一个元素都是可以任意的Java数据类型。和局部变量类似，操作数栈的最大深度也在编译的时候写入到Code属性的max_stacks数据项中。

当一个方法刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数中写入和提取内容，也就是出栈/入栈操作。

操作数栈中元素的数据类型必须与字节码指令的序列严格匹配，这由编译器在编译器期间进行验证，同时在类加载过程中的类检验阶段的数据流分析阶段要再次验证。另外我们说Java虚拟机的解释引擎是基于栈的执行引擎，其中的栈指的就是操作数栈。


## 动态连接

每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用，持有该引用是为了支持方法调用过程中的动态连接。

## 方法返回地址

存放调用调用该方法的pc计数器的值。当一个方法开始之后，只有两种方式可以退出这个方法：

- 1、执行引擎遇到任意一个方法返回的字节码指令，也就是所谓的正常完成出口。
- 2、在方法执行的过程中遇到了异常，并且这个异常没有在方法内进行处理，也就是只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，这种方式成为异常完成出口。正常完成出口和异常完成出口的区别在于：通过异常完成出口退出的不会给他的上层调用者产生任何的返回值。

无论通过哪种方式退出，在方法退出后都返回到该方法被调用的位置，方法正常退出时，调用者的pc计数器的值作为返回地址，而通过异常退出的，返回地址是要通过异常处理器表来确定，栈帧中一般不会保存这部分信息。本质上，方法的退出就是当前栈帧出栈的过程。


# 方法调用

方法调用的主要任务就是确定被调用方法的版本（即调用哪一个方法），该过程不涉及方法具体的运行过程。按照调用方式共分为两类：

1. 解析调用是静态的过程，在编译期间就完全确定目标方法。
2. 分派调用即可能是静态，也可能是动态的，根据分派标准可以分为单分派和多分派。两两组合有形成了静态单分派、静态多分派、动态单分派、动态多分派

## 解析

在Class文件中，所有方法调用中的目标方法都是常量池中的符号引用，在类加载的解析阶段，会将一部分符号引用转为直接引用，也就是在编译阶段就能够确定唯一的目标方法，这类方法的调用成为解析调用。此类方法主要包括静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可访问，因此决定了他们都不可能通过继承或者别的方式重写该方法，符合这两类的方法主要有以下几种：静态方法、私有方法、实例构造器、父类方法。虚拟机中提供了以下几条方法调用指令：

1. invokestatic：调用静态方法，解析阶段确定唯一方法版本
2. invokespecial：调用`<init>`方法、私有及父类方法，解析阶段确定唯一方法版本
3. invokevirtual：调用所有虚方法
4. invokeinterface：调用接口方法
5. invokedynamic：动态解析出需要调用的方法，然后执行

前四条指令固化在虚拟机内部，方法的调用执行不可认为干预，而invokedynamic指令则支持由用户确定方法版本。其中invokestatic指令和invokespecial指令调用的方法称为非虚方法，其余的（final修饰的除外）称为虚方法。

## 分派

分派调用更多的体现在多态上。  

1. 静态分派：所有依赖静态类型来定位方法执行版本的分派成为静态分派，发生在编译阶段，典型应用是方法**重载**。
2. 动态分派：在运行期间根据实际类型来确定方法执行版本的分派成为动态分派，发生在程序运行期间，典型的应用是方法的**重写**。
3. 单分派：根据一个宗量 对目标方法进行选择。
4. 多分派：根据多于一个宗量对目标方法进行选择。

### JVM实现动态分派

动态分派在Java中被大量使用，使用频率及其高，如果在每次动态分派的过程中都要重新在类的方法元数据中搜索合适的目标的话就可能影响到执行效率，因此JVM在类的方法区中建立虚方法表（virtual method table）来提高性能。

每个类中都有一个虚方法表，表中存放着各个方法的实际入口。如果某个方法在子类中没有被重写，那子类的虚方法表中该方法的地址入口和父类该方法的地址入口一样，即子类的方法入口指向父类的方法入口。如果子类重写父类的方法，那么子类的虚方法表中该方法的实际入口将会被替换为指向子类实现版本的入口地址。
那么虚方法表什么时候被创建？虚方法表会在类加载的连接阶段被创建并开始初始化，类的变量初始值准备完成之后，JVM会把该类的方法表也初始化完毕。


# 方法的执行

## 解释执行

在jdk 1.0时代，Java虚拟机完全是解释执行的，随着技术的发展，现在主流的虚拟机中大都包含了即时编译器(JIT)。因此，虚拟机在执行代码过程中，到底是解释执行还是编译执行，只有它自己才能准确判断了，但是无论什么虚拟机，其原理基本符合现代经典的编译原理，如下图所示：

![image-20180815144728322](https://ws4.sinaimg.cn/large/006tNbRwly1fxekc80iwwj30d10ie3z3.jpg)

在Java中，javac编译器完成了词法分析、语法分析以及抽象语法树的过程，最终遍历语法树生成线性字节码指令流的过程，此过程发生在虚拟机外部。

## 基于栈的指令集与基于寄存器的指令集

Java编译器输入的指令流基本上是一种基于**栈**的指令集架构，指令流中的指令大部分是零地址指令，其执行过程依赖于操作栈。另外一种指令集架构则是基于**寄存器**的指令集架构，典型的应用是x86的二进制指令集，比如传统的PC以及Android的Davlik虚拟机。

两者之间最直接的区别是，基于栈的指令集架构不需要硬件的支持，而基于寄存器的指令集架构则完全依赖硬件，这意味基于寄存器的指令集架构执行效率更高，单可移植性差，而基于栈的指令集架构的移植性更高，但执行效率相对较慢，c除此之外，相同的操作，基于栈的指令集往往需要更多的指令，比如同样执行2+3这种逻辑操作，其指令分别如下：
基于栈的计算流程（以Java虚拟机为例）：

```java
iconst_2  //常量2入栈
istore_1  
iconst_3  //常量3入栈
istore_2
iload_1
iload_2
iadd      //常量2、3出栈，执行相加
istore_0  //结果5入栈
```

而基于寄存器的计算流程：

```java
mov eax,2  //将eax寄存器的值设为1
add eax,3  //使eax寄存器的值加3
```

## 基于栈的代码执行示例

下面我们用简单的案例来解释一下JVM代码执行的过程，代码实例如下：

```java
public class MainTest {
	public  static int add(){
		int result=0;
		int i=2;
		int j=3;
		int c=5;
		return result =(i+j)*c;
	}
	
	public static void main(String[] args) {
		MainTest.add();
	}
}

```

使用javap指令查看字节码：

```java
{
  public MainTest();
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 2: 0

  public static int add();
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=0     //栈深度2，局部变量4个，参数0个
         0: iconst_0  //对应result=0,0入栈
         1: istore_0  //取出栈顶元素0，将其存放在第0个局部变量solt中
         2: iconst_2  //对应i=2,2入栈
         3: istore_1  //取出栈顶元素2，将其存放在第1个局部变量solt中
         4: iconst_3  //对应 j=3，3入栈
         5: istore_2  //取出栈顶元素3，将其存放在第2个局部变量solt中
         6: iconst_5  //对应c=5，5入栈
         7: istore_3  //取出栈顶元素，将其存放在第3个局部变量solt中
         8: iload_1   //将局部变量表的第一个slot中的数值2复制到栈顶
         9: iload_2   //将局部变量表中的第二个slot中的数值3复制到栈顶
        10: iadd      //两个栈顶元素2,3出栈，执行相加，将结果5重新入栈
        11: iload_3   //将局部变量表中的第三个slot中的数字5复制到栈顶
        12: imul      //两个栈顶元素出栈5,5出栈，执行相乘，然后入栈
        13: dup       //复制栈顶元素25，并将复制值压入栈顶.
        14: istore_0  //取出栈顶元素25，将其存放在第0个局部变量solt中
        15: ireturn   //将栈顶元素25返回给它的调用者
      LineNumberTable:
        line 4: 0
        line 5: 2
        line 6: 4
        line 7: 6
        line 8: 8

  public static void main(java.lang.String[]);
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=1, args_size=1
         0: invokestatic  #2                  // Method add:()I
         3: pop
         4: return
      LineNumberTable:
        line 12: 0
        line 13: 4
}

```

执行过程中代码、操作数栈和局部变量表的变化情况如下：

![image-20180815144855571](https://ws1.sinaimg.cn/large/006tNbRwly1fxekc8y6rsj30ir0b6gm9.jpg)

![image-20180815144922377](https://ws2.sinaimg.cn/large/006tNbRwly1fxekc9z7vdj30im0a7750.jpg)

![image-20180815144947715](https://ws3.sinaimg.cn/large/006tNbRwly1fxekcavgr9j30im0a90te.jpg)

![image-20180815145016645](https://ws3.sinaimg.cn/large/006tNbRwly1fxekcbu7uoj30im0afaat.jpg)

![image-20180815145054788](https://ws3.sinaimg.cn/large/006tNbRwly1fxekccoh1lj30im09kt9e.jpg)

![image-20180815145116824](https://ws4.sinaimg.cn/large/006tNbRwly1fxekce4h9qj30im0abt9h.jpg)

![image-20180815145148167](https://ws4.sinaimg.cn/large/006tNbRwly1fxekcfgfqcj30in0a8gmb.jpg)

![image-20180815145234272](https://ws4.sinaimg.cn/large/006tNbRwly1fxekcg3vhkj30il09twf9.jpg)

![image-20180815145309935](https://ws1.sinaimg.cn/large/006tNbRwly1fxekche4wzj30im09zjs3.jpg)

![image-20180815145332856](https://ws2.sinaimg.cn/large/006tNbRwly1fxekciec41j30im09vwf9.jpg)

![image-20180815145355355](https://ws2.sinaimg.cn/large/006tNbRwly1fxekcjego7j30ij09igmd.jpg)

![image-20180815145420538](https://ws4.sinaimg.cn/large/006tNbRwly1fxekgtoz9hj30im09h0tg.jpg)

![image-20180815145546202](https://ws2.sinaimg.cn/large/006tNbRwly1fxekguoejvj30im09nq3p.jpg)

![image-20180815145610281](https://ws3.sinaimg.cn/large/006tNbRwly1fxekcn04nkj30in09w0tg.jpg)

![image-20180815145645730](https://ws3.sinaimg.cn/large/006tNbRwly1fxekcnzf9cj30im09tjs5.jpg)

![image-20180815145711517](https://ws1.sinaimg.cn/large/006tNbRwly1fxekcqpt2cj30il09ndgk.jpg)



# 附注

- Solt宗量：方法的接受者与方法的参数称为方法的宗量。举个例子：

  ```java
	public void dispatcher(){
       		 int result=this.execute(8,9);
    	}
    	
	public void execute(int pointX,pointY){
       		 //TODO
    }
  ```
 在dispatcher()方法中调用了execute(8,9)，那此时的方法接受者为当前this指向的对象，8、9为方法的参数，this对象和参数就是我们所说的宗量。  

- 容量槽，虚拟规范中并没有规定一个Slot应该占据多大的内存空间。
- 操作数栈中元素的数据类型必须与字节码指令的序列严格匹配本质就是:要求字节码操作的栈中的实际元素类型必须要字节码规定的元素类型一致。比如iadd指令规定操作两个整形数据，那么在操作栈中的实际元素的时候，栈中的两个元素也必须是整形。  

- 静态类型与动态类型:Animal dog=new Dog();其中的Animal我们称之为静态类型，而Dog称之为动态类型。两者都可以发生变化，区别在于静态类型只在使用时发生变化，变量本身的静态类型不会被改变，最终的静态类型是在编译期间可知的，而实际类型则是在运行期才可确定。

