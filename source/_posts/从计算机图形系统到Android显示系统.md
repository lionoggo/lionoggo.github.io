title: Android图形显示系统基本原理
date: 2017/9/12 16:01:50
toc  : true
tags: [inotify,FileObserver,Android]
categories: technology
description: Android图形显示系统和传统计算机图形显示系统并无本质区别.移动端作为前端的一个分支,其界面的流畅性事关重要,如何定义界面的流畅性以及如何开发出流畅的界面需要对计算机图形领域有所了解,将从基础开始,逐步深入到Android图形显示原理.

-------

# 计算机图形显示系统

## 人眼与帧率

要理解应用流程度,我们首先引入FPS这个概念.FPS是Frames Per Second,它描述的是GPU在一秒内能够渲染出静态画面的数量(一张静态图片称之为一帧),通俗点讲就是GPU每秒钟能画出多少画面.FPS是衡量GPU性能的一个重要指标,通常来说性能越强的显卡,一秒内能够渲染出静态画面的数量越多,给我们的视觉效果越好.

解释完帧率之后,我们还要具备一点生物学的知识,即需要了解视觉暂留现象.由于人眼特殊的生理结构,如果所看的画面以每秒10~12帧的速度播放时,此时我们就认为这些图片时连贯的,这种现象就是视觉暂留.在视觉暂留的基础上,如果我把一张一张的静态图像按顺序以一定的速度出现在我们眼前,此时我们大脑就会认为画面中的物体是在运动的,这就是典型动画的原理.

## 屏幕刷新率与VSync

FPS是描述GPU的一个重要指标,但GPU画出来的图像需要借助显示器来显示出来才能被我们所感知.与帧率相对应,显示器也有一个指标用来描述一秒内从GPU取出画面的的数量,我们称之为刷新率.

屏幕刷新率(Refresh Rate)是指屏幕每秒钟更新画面的次数,对于特定设备而言它是个常量,单位Hz.为了方便,我们采用一张图来描述其关系:

![image-20181127192131031](https://i.imgur.com/QqVSdp2.png)

由于帧率和刷新率分别是用于描述不同设备的指标,而这两个指标因设备不同而有所差异,换句话说这两者并不总是能保持相同的节奏,根据情况,可以分为以下三种:

- 帧率和刷新率相等
- 帧率大于刷新率
- 帧率小于刷新率

在帧率和刷新率相等的情况,GPU每画出一帧,显示器就显示一帧,但实际情况却是由于硬件不同或需要渲染的界面复杂度问题,我们很难保证帧率和刷新率一定是相等的.实际上,更多的是遇到除此之外的情况,即后两者.但其余两种情况都会或多或少的产生问题.

### 帧率小于刷新率

在帧率小于刷新率的情况下,比如帧率是30fps,显示器刷新率是60HZ,此时我们一秒钟内看到屏幕内的画面还是更新了60次,只不过其中一些更新画面是没有变化的而已,因此该情况下由显卡输出的一张图片(一帧)实际上在显示器期内被刷新了两次而已,这种现象带给我们的就是卡顿感.

![image-20181128103849709](https://i.imgur.com/Aif4jk3.png)

### 帧率大于刷新率

那对于帧率大于刷新率的情况又是如何呢?比如帧率是75fps,刷新率是100Hz,这意味在显示器更新画面的时间里,GPU描画了1+1/3的画面.这样在画面显示的时候,那个1/3的画面就会覆盖那个完整画面上部的1/3,在下次的图像刷新的时候,GPU会描画剩下来得2/3和新的2/3的画面.因为屏幕的更新只能跟上画面更新的2/3,这样图像的上部的1/3或是下部的1/3就会和剩下的画面合不上,以屏幕快速显示数字为例,在帧率和屏幕显示率一致的情况下,其效果如下:

![image-20181128101842907](https://i.imgur.com/wuAnkzo.png)

但在帧率大于刷新率的情况下可能出现以下现象:

![image-20181128101938613](https://i.imgur.com/dsNKxfI.png)

对于像上图所示的现象,我们称之为画面撕裂,即一个画面上出现了两帧画面的内容.如何解决画面撕裂问题呢?在了解其解决方案之前,得先来认识Vsync.

## 显卡与VSync

### CRT显示器与VSync

VSync(Verti Synchronization)在计算机显示系统领域是一个非常古老的名词了,通常我们将它翻译成垂直同步.VSync是显卡的一项功能,用来限制GPU绘制的帧数,使其与显示器在一秒内刷新的次数相匹配,也就是指显卡的输出帧数和屏幕的垂直刷新率相等.其实VSync本来只是CRT显示器上的概念,CRT即我们常说的模拟显示器,其显示原理是通过电子枪扫描荧光屏来显示图像的.其扫描过程是从左到右,从上到下逐行刷新的过程,前者称之为水平刷新,后者称之为垂直刷新,其流程大概如下所示:

![image-20181127174542323](https://i.imgur.com/Vg3yPIb.png)

假设我们现在有一个`4*4`的图片要显示在`4*4`的显示器上(单位是像素):

| A1   | A2   | A3   | A4   |
| ---- | ---- | ---- | ---- |
| B1   | B2   | B3   | B4   |
| C1   | C2   | C3   | C4   |
| D1   | D2   | D3   | D4   |

当显示器要电子枪扫描到最后一个像素D4时,显卡会发出一个VSync信号,来通知显示器已经完成一帧画面的扫描,需要回到A1开始进行下一帧的扫描.总之,这个VSync信号由显卡发出,一方面用来告诉显示器需要回到A1位置,另一方面通知显卡准备输出下一帧画面.

在LCD显示器中已经没有垂直扫描这回事了,因此VSync名字本身已经没有意义了.但是LCD仍然需要VSync信号,不然显卡就无法知道在什么时候才可以输出下一帧画面,显示器也无法知道什么时候可以开始处理一帧画面.因此这个VSync这个名称就这样沿袭下来.

现在,我们只需要记住VSync就是用来保持显卡生成帧的速度和屏幕刷新的速度一致的存在,即帧率和刷新率.比如,如果屏幕的刷新率为60Hz,那么生成显卡生成一帧画面的时间就应该固定在`(1/60)s`,即60fps.

### 显卡工作流程

抛开具体实现不说,显卡的工作流程还是比较清晰的,简单点说它接受来自CPU和内存的数据,经过处理之后,生成一帧图片.循环往复,继续生成下一帧的图片,下图揭示了其工作流程:

![image-20181128152242597](https://i.imgur.com/FUZGCaI.png)

在上图中,需要重点注意帧缓冲器,其本质就是显存中划出来的一块区域,用于存储一帧图片每个像素点的数据,显卡会将该区域的数据依次输出到显示器中,当全部输出完毕后,会发出一个VSync信号到显示器内,如此往复.实际上,后面垂直同步,以及后面要提到双重缓存,三种缓存本质上描述的就是帧缓冲器数据输出规则.

通过上图不难发现该区域的数据是由显卡的光栅操作单元生成的,此外缓冲区采取的更新策略是"新数据覆盖老数据的",这意味着在实际数据处理过程中,缓冲区内的数据如果没有及时被输出到显示器,该区域内的部分数据可能会被由光栅操作单元生成新数据覆盖.举个例子,假设当前帧缓冲区内已经有一帧完整的图片A,此时显卡生成了下一帧的画面B,并准备写入帧缓冲区,在写到一半的时候,收到了VSync信号,这时候缓冲区的数据被输出到显示器.糟糕的是此时缓冲区的数据是一半是A画面一半是B画面,因此我们会在显示器上由两张图片拼接出来的画面,这就是画面撕裂现象的原因,之前数字4显示不全同理.

那该如何解决该问题呢?分析上面问题的根源在于写缓冲区的的操作和VSync信号到来时机没有同步导致,如果我们让他们同步起来会如何呢?回想下我们是如何利用`生产者-消费者`解决同步问题的.现在我们来定义这么一条规则就是:当显卡生成一帧完整的图片并写入帧缓冲区后,停下来等待VSync信号的到来,接下来在继续渲染下一帧图片并写入缓冲区.这条规则能够保证帧缓冲区内始终是一帧完整图片的数据,就不会出现画面撕裂现象了.

![11111111](https://i.imgur.com/AVCjaPM.png)

### 双缓冲机制

从性能角度出发,如果只对一块缓冲区进行读写无疑效率比较低下:一方面屏幕要从该区域去读,另一方面显卡要等待去写.因此在实际中,其实帧缓冲区实则被划分为两部分:

- 前缓冲区: 用来缓存要显示到屏幕的帧数据
- 后缓冲区: 用来缓存显卡生成的帧数据.

屏幕只能前缓冲区读取数据用于显示,显卡只能往后缓冲区写入新生成的帧数据.需要注意的是两块缓冲区并不发生实际上的数据拷贝操作,即将后缓冲区的帧数据拷贝到前缓冲区,而是在前缓冲区的帧数据已经推到屏幕上,且新的帧数据被写入到后台缓冲区后,进行指针交换操作,将原来的后缓冲区变为前缓冲区.

![image-20181128171645499](https://i.imgur.com/S4TuSRM.png)

### 三缓冲机制

三缓冲机制是在双缓冲机制基础上发展而来,其目的是在发生卡顿时能够充分利用CPU资源,同时保证尽可能快的从卡顿现象恢复成流畅状态.更具体的解释见下文.

# Android显示系统

在Android体系架构中,通常我们采用XML进行布局描述,CPU会对抽象的XML布局内容进行`Measure -> Layout -> Draw`操作,然后将其内容计算成Polygons(多边形)或Texture(纹理),GPU会对Polygons或Texture进行Rasterization(栅格化)操作,Rasterization后的数据会被写入到帧缓冲区等待显示器显示.下图描述了上述过程:

![image-20181128173844650](https://i.imgur.com/6OskezQ.png)和计算机显示

此外计算图形显示系统一样,Android显示系统同样会遇到屏幕刷新率和帧率一致的情况:

- 帧率小于屏幕刷新率: 可能会导致卡顿现象
- 帧率大于屏幕刷新率: 可能会导致画面撕裂现象

对于该问题,Android采用之前计算机系统一致的方案来解决.为了更好的了解Android是如何解决赶问题的,首先要对Android显示原理有所了解.

## Android显示原理

从开发者的角度出发,Google抽象出View概念,因此在开发过程中我们只需要重点关注View的样子,而无需去了解图形系统底层知识.对于View而言,Activity只是Google我们抽象出来的View控制器,以便我们在此可以对View进行一些控制,比如控制View被点击后的行为,那View真正的载体是什么?

在Android中,View真正的载体是Window,即窗口(其实用视窗系统来描述Android显示系统更形象),每一个Window都包含了各自想要显示内容,不同的Window之间有层级关系,即Z-Order,用来描述Window的显示次序,其本质就是在二维坐标系`x-y`添加了Z轴,其方向为垂直于屏幕表面指向屏幕外.

![image-20181128180954524](https://i.imgur.com/l1RiXc2.png)

每个Window对应于一个Surface,Surface内部含有一块可供"涂鸦"的画布Canvas.在Android系统提供了WindowManagerService服务用于管理系统中所有的Window.当一个应用需要渲染UI时,WindowManagerService服务会为其创建描述其Window信息的WindowState对象,然后通过SurfaceFinger服务将需要显示的多个Surface按照Z-Order次序混合输出到FrameBuffer(帧缓冲区),接下来就是等待VSync信号到来,再显示在屏幕上.

## Android之Project Butter

在Android 4.1之前,界面卡顿是Android中最受诟病的一点.为了解决界面卡顿问题,Google为Android引入了Project Butter计划,即常说的黄油计划.在该项目中,Google对Android显示系统进行重构,并引入了三个至关重要的改进:

- VSync增强: VSync不仅仅用于避免画面撕裂现象,现在它还会通知GPU在渲染下一帧之前要等待屏幕完成逐行绘制.
- Triple Buffer: 三缓冲机制
- Choreographer: 用于协同nimations,input和drawing一起工作

需要注意的是Android中一直存在VSync机制,只不过早期VSync只是为了避免画面撕裂(screen tearing)现象.为了更好的了解增强过的VSync和Triple Buffer,我们先来看一下早期图像显示的过程,即VSync只用来避免画面撕裂的情况:

![image-20181128230745131](https://i.imgur.com/uUscHLu.png)

在上图中,横轴表示时间,纵轴表示Buffer的使用者,对于GPU和CPU一行中的长方形其代表的是帧缓冲,其宽度可以认为是处理该帧所需要的时长,而长方形中数字代表当前帧数.此外两个VSync信号之间间隔16.6ms.

我们从左往右开始看,开始时当前屏幕显示第0帧,CPU和GPU开始准备第1帧的数据(CPU计算第1帧的纹理后交给GPU进行栅格化),并及时计算完成,并等待下一个VSync信号后屏幕显示第1帧画面在显示第1帧画面时,CPU和GPU开始准备第2帧的数据,但由于某些原因导致系统缓慢或者画面太复杂导致第2帧数据没有在第二个VSync信号到来时准备好,显示器仍然显示理第1帧的画面.这种同一帧数据被显示多次的情况称之为"Jank".

![image-20181128235257542](https://i.imgur.com/3h4kzGz.png)

不难发现早期的VSync尽管能够避免画面撕裂线程,但却无法避免Jank.

现在我们来看4.1之后,被增强过的VSync机制:GPU在渲染下一帧之前要等待屏幕完成逐行绘制,也就是每一帧处理都从接受到VSync信号开始,这样我们就可以充分利用这16.6ms.

![image-20181128235621846](https://i.imgur.com/wUlL8ek.png)

在上图中,时间从屏幕显示第0帧开始,CPU/GPU开始准备第1帧数据,一旦接受到下一个VSync信号,除了将原来准备好的第1帧数据显示出来,同时还会CPU/GPU还会开始准备第2帧数据…不难发现在这种情况下,CPU/GPU的工作速度和VSync保持一致,

除了增强的VSync机制,Google还采用三缓冲机制来帮助从卡段中快速恢复成流畅的状态.首先来看传统的双缓冲机制,理想情况下,其工作状态如下:

![image-20181129001112405](https://i.imgur.com/Ixtd3d5.png)

在上图中,以GPU一行为例,长方形A和B分别代表两块缓冲区域,分别代表前台缓冲区和后台缓冲区.开始时A作为前台缓冲区,此时GPU会想后台缓冲区B写入帧数据;当B缓冲区准备就绪后,A,B交换,B变成前台缓冲区,A变成后台缓冲区(其交换原理并不是实质的数据拷贝和转移,详见之前).

从上图看起来,一切都很流程,CPU/GPU充分利用每个16.6ms,在前一个VSync到来时开始准备数据,并在后一个VSync到来时准备好数据.但问题时,我们无法确保CPU/GPU都能在16.6ms内能够准备好数据,如下所示:

![image-20181129002236319](https://i.imgur.com/Szc0102.png)

还是从左边开始看起,此时进入第一个16.6ms,当前显示前台缓冲区A中的帧画面,与此同时CPU/GPU开始准备下一帧的数据.糟糕的是在下一个VSync信号到来之前,GPU未能及时准备好数据,也就是没有及时把帧数据写入到后台缓冲区B中,这种情况下后台缓冲区数据未就绪,因此不能交换前台缓冲区A和后台缓冲区B,这种情况下,显示器只能继续使用前台缓冲区A中的数据,即显示和之前相同的帧,即发生Jank.

更糟糕的是,改进后的VSync要求每一帧数据处理必须要从接受到VSync信号开始,在这种GPU未能及时在下一个VSync到来前及时完成工作的情况,CPU在后一个16.6ms内只能处于空闲状态,也就是上图第2个16.6ms内CPU一直处在空闲状态,而不是进行下一帧的处理操作.如果CPU/GPU大部分时间内都无法完成一帧数据的处理,那么就将导致连续的卡顿现象.

既然Jank难以完全避免,那如果是否能在Jank发生时充分利用CPU而不是使其处在空闲状态呢?如果能,那就可以在发生Jank后,快速恢复成流畅的状态?Google为了解决该问题,引入的三缓冲机制,即在原来双缓冲的机制上加入了第三块缓冲区.

![image-20181129004214237](https://i.imgur.com/ni8zA7z.png)

如上图,以CPU一行为例,共存在A,B,C三个缓冲区.在第一个VSync信号到来时,尽管A和B缓冲区都在使用中,但CPU仍然可以使用第三个缓冲区C来生成帧数据.从上图也可以看出,整个过程就在开始时Jank了一次,后续都是流畅的.这就是三缓冲区帮助Android从卡顿现象中快速恢复成流畅状态的原理.

值得注意的是,Android系统并非一直都是启用三缓冲机制,多一个缓冲区意味着消耗更多的资源.此外一个缓冲区的帧数据要想被显示到屏幕上,最终要跨越两个VSync信号,这样会让用户感觉到延迟.

![image-20181129005317770](https://i.imgur.com/y4n7Tea.png)

### VSync总结

现在关于VSync和Triple Buffer的作用已经明了,接下来总结下VSync相关的知识,首先需要知道的是在Android设备中存在两种VSync信号:

- 由硬件VSync产生: 由硬件中断产生,是一个脉冲信号,类似CPU时钟
- 由软件VSync产生: 在硬件不支持的情况下,通过软件模拟产生:SurfaceFlinger会模拟该信号,并通过Binder传给Choreographer,

简单描述下硬件是如何产生VSync信号:在Android启动过程中,init进程会启动SurfaceFinger进程,并执行main_surfaceflinger.cpp中的`main()`方法,在该方法的执行过程中会初始化HWComposer.而HWComposer就是基于硬件实现的VSync信号发生器,该信号用来通知SurfaceFlinger以便控制生成帧的速度.

### Choreographer总结

Choreographer被设计用来接收VSync信号(通过注册DisplayEventReceiver来接受VSync信号)的Java类,正如含义一样,它的主要工作是用来指挥和协调动画,输入和绘制的时序.如果将显示过程当做一台舞蹈剧的话,那它无疑相当于编舞者了.在Choreographer中存在FrameCallback接口:

```java
  public interface FrameCallback {
        public void doFrame(long frameTimeNanos);
    }
```

在新一帧画面被渲染时会被会调用该接口的`doFrame(long frameTimeNanos)`方法,其参数frameTimeNanos代表该帧开始渲染时的时间.基于该接口,我们可以借助它来实现帧率检测功能.

# 总结

本次主要谈了计算机图形显示系统中一些概念,如帧率和屏幕刷新率,并重点分析了卡顿和画面撕裂现象的由来.此外进一步延伸到Android显示系统部分原理.

