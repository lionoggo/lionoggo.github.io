title: OpenJDK系列(一):编译,调试与项目结构
date: 2018/6/20 13:20:50
toc  : true
tags: [OpenJDK,HotSpot]
categories: technology
description: 正如每个Android工程师离不开AOSP项目一样,每个从事Java领域的工程师,最后都免不了去了解下OpenJDK.

----


# OpenJDK编译
之前的基于OpenJDK8的资料由于人为因素丢失了,索性就重新来过:以OpenJDK10为例.此外,如无特殊说明,开发平台皆为MacOS.

## 源码下载

在mac平台上,可以通过HomeBrew进行OpenJDK源码的下载,以OpenJDK10为例.由于OpenJDK的源码采用mercurial进行管理,因此需要安装mercurial,另外由于编译需要,我们同时安装ccache和freetype工具:

```shell
 brew install mercurial
 brew install install ccache
 brew install freetype
```

接下来通过以下命令正式进行下载:

```shell
 hg clone http://hg.openjdk.java.net/jdk10/jdk10 OpenJDK10
 # 进入OpenJDK10目录下执行命令
 bash ./get_source.sh
```

![image-20180903153521528](https://i.imgur.com/DmElUJT.png)

现在我们已经将所有的源码下载到本地了,接下来我们可以进行编译来满足下好奇心,

## 编译

在正式编译之前需要我们先配置编译信息,该过程通过configure.sh脚本实现,其存放在OpenJDK源码的根目录.使用该脚本需要制定一些参数参数,在mac平台下,配置如下:

```shell
./configure --with-target-bits=64 --with-freetype=/usr/local/Cellar/freetype/2.9.1 --enable-ccache --with-jvm-variants=server,client --with-boot-jdk-jvmargs="-Xlint:deprecation -Xlint:unchecked" --disable-zip-debug-info --disable-warnings-as-errors --with-debug-level=slowdebug 2>&1 | tee configure_mac_x64.log
```

注意不要写错freetype的路径,可以通过 brew list freetype命令来查看当前freetype的安装路径.

configure中提供了大量的参数,通过命令`bash ./configure --help=short`,当然你也可以直接查看OpenJDK[的官方说明](http://hg.openjdk.java.net/build-infra/jdk9/raw-file/tip/README-builds.html),这里我们只是简单介绍几个:

| 参数                              | 含义                                                         |
| --------------------------------- | ------------------------------------------------------------ |
| --with-target-bits                | 设置32为/64编译                                              |
| --with-freetype                   | 指定freetype目录                                             |
| --with-jvm-variants=server,client | 设置要构建的JVM的变体,目前可以选择server,client,minimal,core,zero,zeroshark,custom |
| --with-boot-jdk-jvmargs           | 设置运行boot JDK所需要的JVM参数,例如--with-boot-jdk-jvmargs="-Xmx8G" |
| --with-debug-level                | 调试等级,目前可以指定为release,fastdebug,是slowdebug,optimized |

编译信息配置后之后,通过以下命令进行编译即可:

```shell
 export LANG=C
 make all LOG=debug 2>&1 | tee make_mac_x64.log
```

期间可能会遇到一些编译问题,[大多可以官网找到方法](https://bugs.openjdk.java.net/projects/JDK/issues/JDK-8210205?filter=allopenissues),编译的产物会被存放在build目录下:

![image-20180903161446477](https://i.imgur.com/bsLAExC.png)

执行以下命令,验证下编译结果

```java
cd OpenJDK10/build/macosx-x86_64-normal-serverANDclient-slowdebug/jdk/bin
./java -version
```

![image-20180903161734147](https://i.imgur.com/2SvqV9F.png)

# OpenJDK调试

对于OpenJDK调试而言,常用的工具有gdb和lldb,在MacOS使用更多的是lldb.直接使用lldb进行调试,相对比较原始,对于比较复杂的项目还可以借助Xcode或者CLion.

## Xcode调试OpenJDK

以Xcode 9为例.

### 工程创建

首先使用Xcode创建一个新的项目OpenSDK10,为Command Line Tool类型.

![step1](https://ws3.sinaimg.cn/large/006tNbRwly1fuwqqejp5vj31fu10i7py.jpg)

点击Next继续配置其他信息.

![step2](https://ws1.sinaimg.cn/large/006tNbRwly1fuwqrgvl3uj31bk0zewvt.jpg)

创建完成后,先删除默认生成的文件.

![step3](https://ws2.sinaimg.cn/large/006tNbRwly1fuwqt12oqcj310m0kq7h1.jpg)

### 源码导入

 右键OpenJDK10,选则Add Files To "OpenJDK10",将之前用于OpenJDK10的源码根目录下的文件都添加进来.

![step4](https://ws3.sinaimg.cn/large/006tNbRwly1fuwqxq6eqij30u60r0kb4.jpg)



![step5](https://ws1.sinaimg.cn/large/006tNbRwly1fuwqwpjaphj310a0pa4gn.jpg)

现在我们来配置下项目.选择Product -> Scheme -> Edit Scheme.在左侧的Run选项中先配置info标签页中的Executable,此处指定为之前我们已经编译好的java命令,按照我们之前的编译配置,此处路径即为:

`/OpenJDK10/build/macosx-x86_64-normal-serverANDclient-slowdebug/jdk/bin/java`

![step9](https://ws3.sinaimg.cn/large/006tNbRwly1fuwrbklw8ej316m0mm7a8.jpg)

接下来继续配置Arguments标签页中的Arguments Passed On Launch.随便写的一个Hello.java文件,如下内容:

```java
public class Hello{
    public static void main(String[] args){
          System.out.println("Hello World !");
    }
}
```

然后将其放在java命令所在的目录,即上述提到的`/OpenJDK10/build/macosx-x86_64-normal-serverANDclient-slowdebug/jdk/bin`中,在该目录下执行'./javac Hello.java'将其编译成class文件.

![step8](https://ws1.sinaimg.cn/large/006tNbRwly1fuwr5ainy5j31he0wa7id.jpg)

### 开始调试

到现在,所有的准备工作已经做好了,可以进行调试了.在main.c的`main()`打几个断点后开始调试.

![step10](https://ws3.sinaimg.cn/large/006tNbRwly1fuwrnvz4f4j31h6118kip.jpg)

到现在已经可以正常调试了,但继续调试可能会遇到signal SIGSEGV错误

![step11](https://ws1.sinaimg.cn/large/006tNbRwly1fuwrpwggqoj31ha0ysh9v.jpg)

此时只需要在lldb中设置`(lldb) process handle SIGSEGV --stop=false`,然后继续调试即可.

![step12](https://ws4.sinaimg.cn/large/006tNbRwly1fuwrwggk0bj31380w6aeh.jpg)

不出意外,最终我们会在lldb控制台看到输出结果"**Hello World !** ",如下:

![step12](https://ws1.sinaimg.cn/large/006tNbRwly1fuwrzyztmdj311y0qyae8.jpg)

至此关于Xcode下调试OpenJDK源码的流程已经完成.

# 源码目录结构

OpenJDK源码结构如下:

![image-20180903162039612](https://i.imgur.com/a7Nauul.png)

| 目录     | 说明                                    |
| -------- | --------------------------------------- |
| build    | OpenJDK编译产物                         |
| common   |                                         |
| corba    | 包含corba功能源代码                     |
| hotspot  | 包含Hotspot虚拟机实现源码               |
| jaxp     | jaxp功能的源代码,用于处理XML,提供了     |
| jaxws    | jax-ws功能的源代码,用于构建Web服务的API |
| jdk      | 包含Java工具包                          |
| langtool | 语言工具的源代码如javac,javap等         |
| nashorn  | JavaScript引擎实现                      |


## hotspot

jdk是我们平时开发主要关注的地方,而hotspot使我们了解虚拟机重要的目录,下面我们来了解hotspot目录,重点来看其src子目录:

![image-20180903163506096](https://i.imgur.com/u8uWBgm.png)

| 目录 | 说明         |
| ---- | ------------ |
| make | 编译流程控制 |
| src  | VM源码       |
| test | 单元测试代码 |

src目录下包含了有关VM实现的代码:

![image-20180903165531524](https://i.imgur.com/lfBx8hW.png)

### cpu

不同平台上CPU相关的代码,

![image-20180903164728943](https://i.imgur.com/R5rhLS1.png)

### os

不同操作系统相关代码.根据不同的系统分为不同的目录

![image-20180903164636220](https://i.imgur.com/qKECybi.png)

### os_cpu

操作系统和cpu组合的相关代码.

![image-20180903164806923](https://i.imgur.com/CGsQABh.png)

### share

与具体平台无关的代码.该目录下又分为tools和vm目录.其中tools中提供了相关工具代码.vm是虚拟机实现的核心代码.

![image-20180903165455427](https://i.imgur.com/jb7mLvj.png)

### tools

| 目录                 | 说明                                                      |
| -------------------- | --------------------------------------------------------- |
| hsdis                | 反汇编工具                                                |
| idealGraphVisualizer | 中间代码可视化工具                                        |
| LogCompilation       | 将使用-XX:LogCompiliation输出的日志格式化更容易阅读的工具 |

### vm

| 目录            | 说明                                                         |
| --------------- | ------------------------------------------------------------ |
| adlc            | 平台描述文件的编译器                                         |
| aot             | aot编译器实现,用于静态编译                                   |
| asm             | 汇编器接口                                                   |
| c1              | Client编译器,即我们常说的C1编译器                            |
| ci              | 动态编译器公共接口                                           |
| classfile       | 类文件处理                                                   |
| code            | 管理动态生成的代码                                           |
| compiler        | VM调用动态编译器的接口                                       |
| gc              | GC的实现,包含cms,g1,parallel,serial目录(对应四种GC器实现代码)以及有关GC器的公共代码实现目录shared |
| interpreter     | 解释器的实现,模板解释器和C++解释器                           |
| jvmci           | JVM编译器接口                                                |
| libadt          | 抽象数据结构                                                 |
| logging         | VM通过日志记录系统                                           |
| memory          | 内存管理相关                                                 |
| metaprogramming |                                                              |
| oops            | HotSpot中关于对象系统的实现                                  |
| opto            | Server编译器,即我们常说的C2编译器                            |
| precompiled     |                                                              |
| prims           | VM对外接口以及jvmti(用于调试)实现                            |
| runtime         | 运行时支持库,比如线程管理,锁,反射等                          |
| services        | 支持JMX之类的管理功能的接口                                  |
| shark           | 基于LLVM的JIT编译器                                          |
| trace           |                                                              |
| utilities       | 一些工具类代码                                               |

