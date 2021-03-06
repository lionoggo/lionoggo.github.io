title: Android杂谈:从模块化到组件化
date: 2016-12-15 01:44:49
toc: true
tags: [模块化,组件化,Android]
categories: technology
description: 今天我们来聊聊组件化开发背后的哪些事.最早是在广告SDK中应用组件化,但是下述经验同样适用于普通应用开发.



------



模块化和组件化
===================

以下高能,请做好心理准备,看不懂请发邮件来交流.本文不推荐新手阅读,如果你刚接触Android开发不久,请立刻放弃阅读本文

模块化
--------

组件化不是个新概念,其在各行各业都一直备受重视.至于组件化什么时候在软件工程领域提出已经无从考究了,不过呢可以确认的是组件化最早应用于服务端开发,后来在该思想的指导下,前端开发和移动端开发也产生各自的开发方式.

在了解组件化之前,先来回顾下[模块化](https://en.wikipedia.org/wiki/Modular_programming)的定义
>Modular programming is a software design technique that emphasizes separating the functionality of a program into independent, interchangeable modules, such that each contains everything necessary to execute only one aspect of the desired functionality.


简单来说,模块化就是将一个程序按照其功能做拆分，分成相互独立的模块，以便于每个模块只包含与其功能相关的内容。模块我们相对熟悉,比如登录功能可以是一个模块,搜索功能可以是一个模块,汽车的发送机也可是一个模块.

组件化
------

现在来看看"组件化开发",这里我们看一下其[定义](https://en.wikipedia.org/wiki/Component-based_software_engineering):

> Component-based software engineering (CBSE), also known as component-based development (CBD), is a branch of software engineering that emphasizes the separation of concerns in respect of the wide-ranging functionality available throughout a given software system. It is a reuse-based approach to defining, implementing and composing loosely coupled independent components into systems. This practice aims to bring about an equally wide-ranging degree of benefits in both the short-term and the long-term for the software itself and for organizations that sponsor such software.

通俗点就是:组件化就是基于可重用的目的，将一个大的软件系统按照分离关注点的形式，拆分成多个独立的组件，已较少耦合。

咋样一看还是非常抽象,说了这么多好像还是不明白.什么是组件呢?组件可以是模块、web资源,软件包,比如汽车的发动机是一个模块,也是一个组件,再或者前端中的一个日历控件是一个模块,也一个组件.

模块化 vs 组件化
------
当你看到这的时候,想必心理一阵恶寒:模块化?组件化?到底是什么鬼?有啥区别.
有这种感觉才是对的,模块化和组件化本质思想是一样的,都是"大化小",两者的目的都是为了重用和解耦,只是叫法不一样.如果非要说区别,那么可以认为模块化粒度更小,更侧重于重用,而组件化粒度稍大于模块,更侧重于业务解耦.

组件化优缺点
--------


组件化开发的好处是显而易见:系统级的控制力度细化到组件级的控制力度,一个复杂系统的构建最后就是组件集成的结果.每个组件都有自己独立的版本,可以独立的编译,测试,打包和部署产品组件化后能够实现完整意义上的按需求进行产品配置和销售,用户可以选择使用那些组件,组件之间可以灵活的组建.

配置管理,开发,测试,打包,发布完全控制到组建层面,并带来很多好处.比如一个组件小版本进行升级,如果对外提供的接口没有发生任何变化,其他组件完全不需要再进行测试.

但是组件化的实施对开发人员和团队管理者提出了更高水平的要求.相对传统方式,在项目的管理和组织上难度加大,要求开发人员对业务有更深层次上的理解.



----------

Android项目实践
=================


为什么要在Android中实行组件化开发
----------
为什么要在Android中实行组件化开发呢,其根本原因在于业务的增长提高了项目的复杂性,为了更好的适应团队开发,提高开发效率,实行组件化乃大势所趋.为了更好的帮助大家理解上面这句话,我将从最早的Android 项目开发方式说起.


### 简单开发模型
所谓的简单开发模型是最基础的开发方式,工程中没有所谓的模块,没有所谓的规划,常见于初学者学习阶段或者是个人学习过程所写的demo,其结构大概如下:

![image-20181214110345831](https://i.imgur.com/VwrYIL4.jpg)



不难发现,往往是在一个界面中存在着大量的业务逻辑,而业务逻辑中充斥着各种各种网络请求,数据操作等行为,整个项目中没有所谓的模块的概念,项目组成的基本单位不是模块,而是方法级的.

关于这种开发模型没什么需要介绍的,我们早期都经历过,现在除了很少非常古老的项目以及初学者练手之作,已经很少见到.


### 单工程开发模型
该种开发模型已经有了明确的模块划分,并且通过逻辑上的分层呈现出较好结构,该模型最为我们所熟悉,通常用于早期产品的快速开发,团队规模较小的情况下.该种开发模型结构如下:

![image-20181214110516196](https://i.imgur.com/iKmaIEz.jpg)



随着产品的迭代,业务越来越复杂,随之带来的是项目结构复杂度的极度增加,此时我们面临着几个问题:

  1. 实际业务变化非常快,但是工程之前的业务模块耦合度太高,牵一发而动全身.
  2. 对工程所做的任何修改都必须要编译整个工程
  3. 功能测试和系统测试每次都要进行.
  4. 团队协同开发存在较多的冲突.不得不花费更多的时间去沟通和协调,并且在开发过程中,任何一位成员没办法专注于自己的功能点,影响开发效率.
  5. 不能灵活的对工程进行配置和组装.比如今天产品经理说加上这个功能,明天又说去掉,后天在加上.


在面临这些问题的前提下,我们重新来思考组件化,看看它是否能解决我们在Android 项目开发中所遇到的难题.


### 主工程多组件开发模型
借助组件化这一思想,我们在"单工程"模型的基础上,将业务层中的各业务抽取出来,封装成相应的业务组件,将基础库中各部分抽取出来,封装成基础组件,而主工程是一个可运行的app,作为各组件的入口(主工程也被称之为壳程序).这些组件或以jar的形式呈现,或以aar的形式呈现.主工程通过依赖的方式使用组件所提供的功能.

![image-20181214110610910](https://i.imgur.com/34Is4zJ.jpg)

(需要注意这是理想状态下的结构图,实际项目中,业务组件之间会产生通信,也会产生依赖,关于这一点,我们在下文会谈)

不论是jar还是aar,本质上都是Library,他们不能脱离主工程而单独的运行.当团队中成员共同参与项目的开发时,每个成员的开发设备中必须至少同时具备主工程和各自负责组件,不难看出通过对项目实行组件化,每个成员可以专注自己所负责的业务,并不影响其他业务,同时借助稳定的基础组件,可以极大减少代码缺陷,因而整个团队可以以并行开发的方式高效的推进开发进度.

不但如此,组件化可以灵活的让我们进行产品组装,要做的无非就是根据需求配置相应的组件,最后生产出我们想要的产品.这有点像玩积木,通过不同摆放,我们就能得到自己想要的形状.

对测试同学而言,能有效的减少测试的时间:原有的业务不需要再次进行功能测试,可以专注于发生变化的业务的测试,以及最终的集成测试即可.

到现在为止,我们已经有效解决了"单工程开发模型"中一些问题,对于大部分团队来说这种已经可以了,但是该模型仍然存在一些可以改进的点:每次修改依赖包,就需要重新编译生成lib或者aar.比如说小颜同学接手了一个项目有40多个组件,在最后集成所有组件的时候,小颜同学发现其中某组件存在问题,为了定位和修改该组件中的问题,小颜同学不断这调试该组件.由于在该模型下,组件不能脱离主工程,那么意味着,每次修改后,小颜同学都要在漫长的编译过程中等待.更糟糕的是,现在离上线只有5小时了,每次编译10分钟,为改这个bug,编译了20次,恩....什么也不用干了,可以提交离职报告了

如何解决这种每次修改组件都要连同主工程一起编译的问题?下面我们来看主工程多子工程开发模型是如何解决该问题的.



### 主工程多子工程开发模型
该种开发模型在"主工程多组件"开发模型的基础上做了改进,其结构图如下:

![image-20181214110637269](https://i.imgur.com/iCJiHL9.jpg)

不难发现,该种开发模型在结构上和"主工程多组件"并无不同,唯一的区别在于:所有业务组件不再是mouble而是作为一个子工程,基础组件可以使moudle,也可以是子工程,该子工程和主工程不同:Debug模式下下作为app,可以单独的开发,运行,调试;Release模式下作为Library,被主工程所依赖,向主工程提供服务.

在该种模型下,当小颜同学发现某个业务组件存在缺陷,会如何做呢?比如是基础组件2出现问题,由于在Debug模式下,基础组件2作为app可以独立运行的,因此可以很容易地对该模块进行单独修改,调试.最后修改完后只需要重新编译一次整个项目即可.

不难发现该种开发模型有效的减少了全编译的次数,减少编译耗时的同时,方便开发者进行开发调试.

对测试同学来说,功能测试可以提前,并且能够及时的参与到开发环节中,将风险降到最低.


到现在,我们在理论层次上讲明了采用组件化开发给我们带来的便利,空口无凭是没有说服力的,在下面的一小节中,我们来谈谈如何组件化在Android中的实施过程.

组件化过程中遇到的问题
-------

### 组件划分
组件化首要做的事情就是划分组件.如何划分并没有一个确切的标准,我建议早期实施组件化的时候,可以以一种"较粗"的粒度来进行,这样左右的好处在于后期随着对业务的理解进行再次细分,而不会有太大的成本.当然,我建议划分组件这一工作有团队架构人员和业务人员协商定制.



### 子工程工作方式切换
在"主工程多子工程模型"中,我们提到子工程在Debug模式下做为单独的Application运行,在Release模式下作为Library运行,如何去动态修改子工程的运行模式呢?我们都知道采用Gradle构建的工程中,用`apply plugin: 'com.android.application'`来标识该为Application,而`apply plugin: 'com.android.library'`标志位Library.因此,我们可以在编译的是同通过判断构建环境中的参数来修改子工程的工作方式,在子工程的gradle脚本头部加入以下脚本片段:
```groovy
if (isDebug.toBoolean()) {
    apply plugin: 'com.android.application'
} else {
    apply plugin: 'com.android.library'
}

```

除此之外,子工程中在不同的运行方式下,其AndroidMainifest.xml也是不相同的,需要为其分别提供自己AndroidManifest.xml文件:在子工程src目录下(其他位置创建)创建两个目录,用来存放不同的AndroidManifest.xml,比如这里我创建了debug和release目录

![这里写图片描述](https://i.imgur.com/a3Ic3Tj.jpg)
接下来同样需要在该子工程的gradle构建脚本中根据构建方式制定:

```groovy
android {
    sourceSets {
        main {
            if(isDebug.toBoolean()) {
                manifest.srcFile 'src/debug/AndroidManifest.xml'
            } else {
                manifest.srcFile 'src/release/AndroidManifest.xml'
            }
        }
    }
}

```





### 组件通信与组件依赖
在"主工程多组件"这种理想模型下业务组件是不存在相互通信和依赖的,但现实却是相反的,如下图:

![image-20181214110748460](https://i.imgur.com/gq0JE8I.jpg)


这里,业务组件1和业务组件3同时向业务组件2提供服务,即业务组件2需要同时依赖业务组件3和业务组件1.

现在我们再来看一种更糟糕的情况:

![image-20181214110817901](https://i.imgur.com/5puinLp.jpg)


由此看来,在业务复杂的情况下,组件与组件之间的相互依赖会带来两个问题:

- 重复依赖:比如可能存在业务组件3依赖业务组件1,而业务组件2又依赖业务组件3和业务组件1,此时就导致了业务组件1被重复依赖.
- 子系统通信方式不能依靠传统的显示意图.在该种模型下,使用显示意图将导致组件高度耦合.比如业务组件2依赖业务组件1,并通过显示意图的方式进行通信,一旦业务组件1不再使用,那么业务组件2中使用现实意图的地方会出现错误,这显然与我们组件化的目的背道而驰.

#### 解决组件通信
先来解决业务组件通信问题.当年看到上面那张复杂的组件通信图时,我们不难想到操作系统引入总线机制来解决设备挂载问题,同样,借用总线的概念我们在工程添加"组件总线",用于不同组件间的通信,此时结构如下:

![image-20181214110906701](https://i.imgur.com/ABv4JnM.jpg)

所有挂载到组件总线上的业务组件,都可以实现双向通信.而通信协议和HTTP通信协议类似,即基于URL的方式进行.至于实现的方式一种可以基于系统提供的隐式意图的方式,另一种则是完全自行实现组件总线.这篇文章不打算在此不做详细说明了.


#### 解决重复依赖
对于采用aar方式输出的Library而言,在构建项目时,gradle会为我们保留最新版本的aar,换言之,如果以aar的方式向主工程提供提供依赖不会存在重复依赖的问题.而如果是直接以project形式提供依赖,则在打包过程中会出现重复的代码.解决project重复依赖问题目前有两种做法:1.对于纯代码工程的库或jar包而言,只在最终项目中执行compile,其他情况采用provider方式;2.在编译时检测依赖的包,已经依赖的不再依赖


### 资源id冲突
在合并多个组件到主工程中时,可能会出现资源引用冲突,
最简单的方式是通过实现约定资源前缀名(resourcePrefix)来避免,需要在组件的gradle脚本中配置:
```groovy
andorid{
	...

	buildTypes{
		...
	}

	resourcePrefix "moudle_prefix"
	
}

```

一旦配置resourcePrefix,所有的资源必须以该前缀名开头.比如上面配置了前缀名为moudle_prefix,那么所有的资源名都要加上该前缀,如:mouble_prefix_btn_save.


### 组件上下文(Context)
最后需要注意在Debug模式下和Release模式下,所需要的Context是否是你所希望的,以避免产生强转异常.

----------


结束语
==========
最早接触组件化这个概念是在从事广告SDK工作中,最近陆续续的做了一些总结,因此有了这篇关于"组件化开发"的文章.另外,组件化开发不是银弹,并不能完全解决当前业务复杂的情况,在进行项目实施和改进之前,一定要多加考量.

敬请期待第二篇,我们将在第二篇内介绍如何对项目实施组件化.
















[1]: http://blog.csdn.net/dd864140130/article/details/53558011