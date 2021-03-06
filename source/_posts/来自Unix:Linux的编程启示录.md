title: 来自Unix/Linux的编程启示录
date: 2017/2/8 04:46:00
toc  : true
tags: [Unix,编程]
categories: technology
description: 写本文的最初灵感源于16年11月份我将工作环境切换到Mac OS上,其中一些使用"差异"让我开始对Unix/Linux中设计产生了浓厚的兴趣.虽然从13年开始使用redhat,再到后来使用的ubuntu,但却从来关注过这些,特此记录.

----------

在整个探究过程中,那些经典的著作再次让我获益匪浅:[C和指针][1],[C专家编程][2],[深入理解计算机系统(原书第3版)][3],[Linux/Unix设计思想][4],[Linux Shell脚本攻略][5].前两本书购于13年,断断续续的读了许久,这一次重读令人豁然开朗,其中许多不解之处在深入理解计算机系统一书得到答案,而后两本则是年前有幸读到.其中Linux/Unix设计思想不谈技术细节,却揭示了Linux/Unix隐含的指导思想,最后一本则是从Shell出发,以实践的角度教你如何利用Shell(Bash)在Linux/Unix平台上构建有效的解决方案,实乃入门进阶必备啊.



# 从女娲造人说起

俗说天地开辟,未有人民,女娲抟黄土作人.剧务,力不暇供,乃引绳于泥中,举以为人.故富贵者,黄土人；贫贱人,引绳人也...


## 神一样的程序员
"女娲造人"可谓人人皆知,女娲是神,而程序员是能力上最接近女娲的存在.如果说女娲创造的是真实的世界,那么程序员是创造数字的世界,我们所写下的每行代码,所构建每个程序,都在让这个数字世界变得丰富多彩.

没有哪个职业可以像程序员:赋予代码以生命,创造数字世界中的花花草草!尽管我们拥有近乎神的能力,本身却也受着NIH综合征的困扰.

## 程序员之殇
NIH综合征是什么?

> Not invented here (NIH) is a stance adopted by social, corporate, or institutional cultures that avoid using or buying already existing products, research, standards, or knowledge because of their external origins and costs.

我们经常为NIH所困:在求解解决方案时,经常自认为可以做得更好或者在看到别人的解决方案会提出各种质疑.这大都是我们缺乏对当时问题解决者所面临限制的认识所造成:可能迫于时间或者预算的限制,也可能受限于当时物理资源的限制.

## 穿着衣服的猴子

NIH综合征的特点就是人们会为了证明自己能够提供更加卓越的解决方案而放弃其他开发人员已经完成的工作.这种狂妄自大的行径说明此人并无兴趣去维护他人竭尽全力提供的最佳成果,也不想以此为基础去挑战新的高度.NIH综合征本质上是动物天性---"嘿,我更好"的体现,就像所有的公猴都会向母猴证明"我更好"一样.

放弃已有工作成果而另起炉灶的做法无形之间浪费了大量时间.有一些技术人员在入职新公司后,往往有想推倒一切重来的欲望.我曾经有位朋友连续跳槽很多家公司,每次入职之前都信心满满,不就后就黯然离开.朋友能力很不错,但每次入职后总是急于通过推倒重来来证明自己的能力.

更糟的是,新的解决方案往往只是做了一些改进,和之前并无本质区别,再加上现代的应用往往动辄十几万行,远超出每个人所能注意的极限,这也会让程序员产生疏漏,从而使得这个问题变得更糟糕.

当然,少数情况下,新的解决方案更好,这只是因为技术人员早已了解做过的工作,从而针对问题可以提出更高的解决方案.站在现在的角度看历史,还看不出对错的话,除非你对历史一无所知.

![mage-20180708123207](https://i.imgur.com/wKCDiEp.png)

## 为什么Linux会成功

Linux为什么会成功有很多的的因素,除却当时环境因素外,其成功之处在于开源的世界让每个人都能得到Linux源码,从而使得经验可以被借鉴.事实上,在原有软件基础上进行扩展是Unix的核心概念之一.当然,在Linux出现之前,借鉴他人编写的软件已成为相当普遍的做法,这也是GUN宣言中的GPL下所阐释的思想.

----------


# 来自Unix/Linux的编程启示录

现在我们来看看Unix/Linux中那些隐藏的编程哲学,需要说明的是Unix/Linux中的编程哲学非常通俗易懂,他们大多是你已知的,更多的时候是我们不曾注意到.

无论你现在负责的系统多么复杂,这总有可以帮助你的地方.(要说复杂,还有什么比Unix/Linux更为复杂的应用?)



## 小既是美

我猜当你看到"小既是美"一词时会首先想到一句谚语"麻雀虽小,五脏俱全".那就用麻雀做引子来说说为什么小即是美.

大凡生物,无论形体如何,愈小愈稳定.麻雀相对人而言,出现脑残鸟的几率要小的多,而单细胞生物相对多细胞生物产生显著变异体的概率也要小的多...在软件工程中,同样如此.Unix中的程序大凡遵循这条准则,小带来的显著优点如下:


 >  2. 易于编写
 3. 易于理解
 4. 易于维护
 5. 消耗更少的系统资源
 6. 容易与其他工具组合

尽管我知道你已经对其有过了解,我还是例行的啰嗦一下.

每个程序员心中都有编写一个万能软件的想法,渴望像上帝一样创世,能够名垂青史,但这实在是太难了.越是复杂的问题越令人畏缩不前,想想在上学期间,你是更乐于立刻开始做一道题还是一套题呢?不出意外,我们大都喜欢从一道题开始.编写小程序同样如此:我们能够尽快的动手去做,并且针对某个点找出更合理的解决方案.另外,由于大脑的结构,我们无法同时关注超过四个以上的点,而复杂的程序需要被关注的点却远大于四个,这就意味着我们可能会犯更多的错误.

小程序带来的另一个好处就是容易理解,一个100行的程序比一个1000行的程序要容易理解的多(复杂度相同的情况下).

当程序足够小的时候它就是函数,还有什么比函数更灵活地自由组合呢?当然大多数小程序不能够小到只有一个函数,但尽可能的减小程序大小仍然很重要,unix/linux中的命令便是最佳的示例:cp用来拷贝,rm用来删除,组合起来就可以实现移动,简直棒呆了.

我想,没人会认为麻雀每天摄入的能量会比人多吧?换到程序当中就是越小的程序消耗的系统资源会更少,大多数情况下如此.

## 让每个程序只做好一件事情

程序员也许只想编写一个简单的应用程序,但是随着他的创新精神占据上风,促使他在会在这里添加一个功能,在那里添加一个选项,很快你就发现本来你只想要一个文本编辑器,但是他却给你一个IDE,最终他给你的往往不是惊喜而是大杂烩.这个问题不但在程序员身上体现出来,同样也会出现在产品经理上:想要一个计算器对么,我这里有个超级计算机可以给你用?现在你只能面对这这么一个庞然大物了
![mage-20180708123241](https://i.imgur.com/0YDZFRi.png)

该思想和"小即是美"相辅相成,"专注一件事"的应用最终会产生一个较小的程序,而小程序往往只有单一的功能,单一功能的程序往往也会很小.另外,每个程序只做一件事情也意味着我们可以集中精力去解决当前任务,全新全意做好本职工作.

## 建立原型

原型的建立是学习的过程:在该过程中可以知道哪些想法可行,哪些不可行.每一个正确设计的背后都有数百个错误的设计方案,通过建立原型,我们可以在早期剔除掉不良的设计方案,以此来降低风险.换句话说,越早的建立原型,离你的软件发布的时间越近.


### 你的软件发布了吗?
从开始接触软件工程开始起,我们的前辈就告诉我们:"软件永远不会有开发完的时候,记住,是永远!".

"怎么会完不成呢,我只要不做了不就行了么?",现在想想,这比让一个沉迷在购物中的女人停下脚步更难(如果钱是无尽的话).编写软件亦如此,你永远都会想要加入的更多的功能.想加入的更多功能的可能不是你,或者是你的产品经理,当然你的boss也会掺一脚.

我一直在向周围的朋友传递这么一种观点"**世界上唯有不变的事物那就是变**",对于软件工程同样如此.既然变化无可避免,那软件是怎么完成的?尽管我们做事都希望看到最后的结局,但是对于软件而言并非如此.可以说,没有做完的软件,只有发布的软件.

我曾犯过类似的错误.16年我在负责一款移动端产品的开发,希望在一个月内上线第一个版本.由于个人疏忽,当我着手原型工作时离发布时间只有20天的时间了,在这期间发现原有的一些方案无法奏效,我不得不临时更改一些既有的设计.最终导致第一个版本正式发布的时间超出15天.我也曾因此被认为是个糟糕的工程师.在团队没有任何移动应用产品开发的经验下,及早的开始建立原型或许不会导致如此后果.在离开这家公司之后,我仍倍感惭愧.

## 舍弃高效率而取可移植性

偏向高效率往往会导致代码不可移植,而选择可移植性往往会让软件的的性能不那么尽如人意.每当有为了追求高效率而放弃移植性的想法的时候,请在心中默念以下两句话:

 1. 速度欠佳的缺点会被明天的机器克服
 2. 好程序不会消失,而被移植到新平台.

### 关于优化的两点建议
在unix环境中,可移植性的含义通常意味着人们要转而采用shell脚本来编写软件.抛开平台不谈,关于优化,我有以下两点建议:

 - 不要立刻优化:如无必要,无须优化.换句话说,不要想当然的优化,尤其是在用户无感觉的情况下.
 - 关注微优化:影响性能的往往只有几处,在需要优化的时候只要解决这几处就好,切莫以优化的名义重构全部.


## 使用纯文本来存储数据

文本不一定是性能最好的格式,但却是最通用的.另外,文本文件易于阅读和编辑:在任何平台上我们可以轻松的阅读文本数据,或使用标准文本编辑器对其进行编辑.

你可能会考虑到使用纯文本文件拖慢了系统的处理速度,并会找出一系列的证据来证明处理字符串的性能更低.我们承认文本文件确实拖累了系统的性能,但是记住明天系统的机器的性能必将有大幅度的提高.

## 利用软件的杠杆效应

给你一个杠杆,在不考虑杠杆质量的前提下,你真的可以撬动地球.一个人精力只有这么多,如果想要取得非凡的成就,你就必须放大自己对这个世界的影响力,这就需要你找到其中的"杠杆点",也就是关键点.

![mage-20180708123321](https://i.imgur.com/ec39f5c.png)

### 君子性非异也,善假于物也
软件中的杠杆效应就是要善于利用他人已有成果,具体点就是学会利用他人写的代码.在早期,我希望每一行代码都是从自己手中出来,并以为为傲,并认为这就是所谓的优秀的程序员.多年之后,我才明白成为一个优秀程序员的关键却和手敲每行代码并无必要联系.

善于利用他人的代码会给程序员自身增加砝码.这可能和很多人的认知相违背.查看别人工作并夸口说自己做的更好,不代表你能力更强;推导现有的方案在重来,那只是模仿不是创造.想要解决掉你手中的任务,编写更多软件的最好方法就是善于借用别人的成果.和大多程序员一样,我曾认为亲自编写代码能带来"就业保障",并将这种做法视为核心竞争力.但实际工作中却是相反,在实现功能的前提下,企业往往认为花费时间更少的程序员更具有高效的生产了,你可能对此感到委屈和不解.这里,我想要说的是:认识企业对你的认识和你对自己认识之间的差异是非常重要的.

如果有可能,请尽量让你的工作自动化.通过加强自动化工作来利用软件的杠杆效应能产生巨大的生产力,并且能够更充分的利用好时间.如果你是个linux开发者,shell脚本便是你在寻找的杠杆点.

### 化作春泥更护花
从另外一个角度来说,允许他人使用你的代码来发挥杠杆作用.大部分工程师喜欢私藏自己的源代码,并视为独一无二,举世珍宝,仿佛自己的"核武器".他们认为公开自己的源代码,自己就失去地位,会被公司所抛弃.

哪些你以为是"核武器"的代码真的就有如此威力么?要记住:任何一个有着合理思维的正常人都可以编写出像样的代码.更何况,一个有耐心的程序员完全可以将程序反汇编出来,通过一步一的分析,最终发现其中的蛛丝马迹.


Linux的开发人员认为执意保留源码的控制权并无太大必要.现在你发现,尽管你拥有全部的源码,但却仍然不能创造出比Linux更成功的操作系统.我想这足打消你心中哪点忧虑了.


### 重新认识自豪感
现在重新认识所谓的自豪感.很多程序员将写出优秀的代码,使用先进的技术视为自豪感,但我们最终的产出是用来服务用户的,如果用户不能满意,那么我们所谓的自豪感不过是"孤芳自赏".

## 使用shell脚本来提高杠杆效应和可移植性

谈起脚本语言,很多工程师觉得脚本太弱了,只能用来编写简单的应用无法工程化,并且不少程序员更乐于哪些看起来宏大的java或者net应用,甚至有些程序员认为脚本语言算不上一门语言.现实给了我们一巴掌:我们从来没有想象过javascript会有如此发展,以node为代表的新型开发平台,让整个JavaScript语言变得非常有活力.

那么我们是否也小觑了shell这种简单的脚本语言呢?有很长一段时间我不能正确的认识shell是什么,甚至觉得"嗨,这东西好像对我没什么用,它能用来干什么".让我改变想法的恰好是15年的工作经历.

和其他脚本语言(如:python,JavaScript)一样,shell脚本同样由一个或者多个语句组成,通过调用本地程序,解释程序和其他脚本来执行任务.shell脚本将每条指令加载到内存执行.大部分情况下,这些指令是由无数的unix/linux开发者事先编写完成,我们要做的就是用shell来间接的利用这些高效,安全的代码.比如在shell中使用wget命令来下载文件时,我们实际书写的不过几行语句,背后支撑我们的确有数万行的代码,如果让你来用java写一个下载器会需要多少工程量呢,又需要耗费多少时间?现在有些程序员会反驳我:我用python写起来也很快,代码也不多?那问题来了,python的运行环境可不是先天就存在unix/linux中的,但是大部分unix/linux却都是支持shell的.如果编写的脚本程序要运行在不同的系统中或者运行在许多设备中,shell无疑具有更好的移植性,不是么?


### 如你所想,及时运行
脚本语言有一个先天的优势就是他们是解释型的语言,运行前不需要事先编译.先来看看以C语言为代表的编译型语言,要构建一个程序的基本流程如下:
`思考->编辑代码->编译->测试`
而在shell当中,整个过程更短:
`思考->编辑->测试`

相比C语言,shell不存在编译过程,这带来的优势就是:无论shell的应用规模多大多复杂,只要你想你就可以运行它观察它,而无需像C语言一样等待编译过程结束后再运行.

### shell真的慢吗?
一些程序员对shell执行效率过于担忧,认为其效率太低,这就导致了他们为了更高的效率企图采用C语言来重写脚本.shell脚本的执行效率相对较慢,一旦C程序被装载到内存中运行,纯C程序与那些从脚本中调用C程序相比,并没有太大的性能优势.另外,由于Unix/Linux中大部分的命令做过相关的优化,反而会比由你编写的纯C程序更快.这也引出我想说的另外一点:为了追求更高的性能而更换语言这一做法有待商榷.如果真是为了追求性能,那么改变通过改良实现方法更有效,比如有序数列中查找指定数字采用二分查找比传统的遍历查找的时间复杂度更低,而不是将原来实现遍历查找的java代码改成C代码实现.

## 避免强制性的用户界面

我最早接触计算机系统是Windows 2000,应该是在上小学期间,尽管我不知道这个大个"计算器"能够做什么,但是却可以在没人教的情况下琢磨去做,并学会玩编辑文字,利用文本编辑器和共享来相互交流,当然,少不了的纸牌和扫雷.

### window和unix/linux理念差异


用户界面能够让任何一个小白在不需要专业知识的前提迈出操作的第一步.window系列产品和unix/linux系列产品的不同之一在于对于他们对用户的看法上:window下的设计者认为用户是畏惧和计算机打交道的,因而提供足够的用户界面来消除用户的恐惧心理,而unix的设计理念则不同,它认为一个接触unix/linux的用户已经具备了基本的计算机素养,并力求更全面的掌握它,从这个角度而言,unix/linux是选择用户的,而window是服务用户的.

这也是早期window比unix/linux更为大众所接受的关键因素之一.但随着unix/linux的逐渐发展以及大众计算机素养的提升,越来越多的人开始接受unix/linux产品,并热衷与此.


### 像搭积木一样编程
对于程序员而言,我们希望编程应该像搭积木一样,通过不同的组合来实现多样的产品,未来的产品也应该是如此:一些程序员负责开发基础的小模块,另外一些程序员则负责将这些小模块对接成客户所需要的产品.对Unix/Linux开发者而言的确如此,我们可以利用系统中提供的众多非界面性小程序来组合成我们所需要的.但是对于提供用户界面的小程序而言我们却很难做到这点:用户界面渴望与用户沟通而非另一个程序,而模拟人在用户界面上的沟通并非易事,你需要花费更多的精力放在非核心需求上.

即使我们能够模拟,从操作效率来看也不容乐观.毕竟用户界面相对命令程序而言本身意味着更多的资源消耗,我想没人愿意在服务器上装一个KDE吧?这就是window下同样拥有众多的小程序但却很难将其组合成新产品的原因之一,换言之我们很难利用前人已产生的成果来发挥杠杆作用.

### 嗨,我们需要再加个按钮
程序员和产品之间总有一个矛盾点,来看看下面这个对话.
>PM:"这里加个按钮"
RD:"又没有人用,不加不行么?"
PM:"听我的,加上吧!加了按钮之后就会有人点击..."
RD:"好吧..."
PM:"你看这个地方空白好大,能再加一个按钮么?"
RD:"我...."


好吧,这是个笑话,我并非想挑起程序员和产品之间的战争.如果你在设计/开发一个界面化的产品,你经常会不由自主的为他加上更多的按钮,企图让他提供更丰富的功能:你总觉得有人会需要它.最终导致项目更加复杂,但用户满意度却未见提升.

有这么一件事到现在对我影响甚大.我想把Kindle中的笔记导出到印象笔记,这件事情本来很简单,只需要导出指定的文本文件,解析后导入到印象笔记就可以.来看看当时我想了什么:我应该做一个界面,可以方便的我操作,别人也可能需要一个保存笔记成html格式的功能,所以我要添加个转换按钮,当然,保存,导出按钮也不能忘了,等等...要不要编辑功能?...

这非常好笑,不是么?由于我第一时间想到了图形界面,接下来不由自主地想在界面上添加更多的功能,更重要的是我明明是自己使用的,却站在别人的角度上去给自己提了需求?

如果最终呈现出的是图形化的产品,会发生很有意思的事情.来想一下,我希望它能在每天十二点自动完成导出任务?难道我要再写一个程序来模拟点击操作么?既然这样,为什么不能让两个程序有能够直接交流的能力么?(感谢unix/linux中的管道机制)

现在来看到底发生了什么?当我们想提供界面的时候,就假设是人在操作.这个隐形的假设最终让这个程序失去了自由,好比送给孩子一个用胶水粘合积木而成的城堡而非一盒积木.

## 让每个程序都成为过滤器

程序是什么?早期认为`程序=算法+数据`.但从功能的角度来说,程序即过滤器.

先不要着急骂我或者反驳我.你不觉得在你编写的每个程序中,无论是复杂还是简单,都是以某种形式接受数据,并产生一些数据作为输出么?这些要输出的数据不就是取决于程序中的算法么?有一些人为过滤器就该是过滤掉不符合期望的东西,而非转换,比如存在一个程序用于过滤掉集合中元素值为0的元素,这对于大部分人而言是最直观的有关过滤的理解.但要记住过滤器同样也包含转换的概念.

对于计算器而言,我们输入错误会产生错误提示,而决定产生什么样错误信息的算法就是错误状态输入的过滤器.可以说算法是人根据需求定义的规则,即算法由人定义.那数据又是怎么来的?一个没有数据输入/输出的程序就像一个不会进食和排泄的人,这种程序没有存在的价值,现实世界也不存在这种人.





## 并行思考

计算机界有个经典的笑话:如果一个女人要花是十个月生下一个婴儿,是否意味着能让十个女人在一个月内生出一个婴儿呢?笑话中透出的含义非常明显:自然特性导致某些任务必须被串行,任何试图让其并行运行的举动都无法加快它的进程.

在数字计算机的整个历史中,有两个需求不断的驱动我们:一是我们想计算机做的更多,令一个是我们想要计算机运行的更快.而并发则是实现这两个需求有效途径.对程序设计而言,并行思考意味着我们应该尽可能利用CPU运算性能,这也要求我们要对任务进行细致有效的分解,寻找其串行路径和并行路径.

现代操作系统通过提供进程,线程及I/O多路复用作为并行思考的实现手段,这里我们对这三者做个简单的说明:

 >- 进程:每个逻辑控制流都是一个进程,由内核来调度和维护.因为进程有独立的虚拟地址空间,想要和其他控制流通信必须依靠显示的进程间通信,即我们所说的IPC机制
> - I/O多路复用:应用程序在一个进程的上下文中显式地调度他们自己的逻辑流.逻辑流被模型化为状态机,数据到达文件描述符之后,主程序显式地从一个状态转换为另一个状态.因为程序都是以一个单独的进程,所以所有的流都共享同一个地址空间.基本的思路就是使用select函数要求内核挂起进程,只有一个或多个I/O事件发生后,才将控制权返回给应用程序
> - 线程:线程应该是我们最为熟知的.它本质是运行在一个单一进程上下文中的逻辑流,由内核进行调度.

现代程序设计语言大都提供对这三种开发方式的支持,比如C,Node,Python中常用多进程/线程技术,java中的多线程及NIO技术等等.除此之外,协程也是一种不错的技术方案,比如Lua,Python中也提供了对协程的支持.关于这几种实现并发编程的技术方案,我将在随后的文章中去解析.

最后补充一点,无论机器速度快慢,我们都可以将多个机器串联在一起,从而得到一个运行速度更快的集群,这就是现代服务器技术核心之一:大凡高性能的服务器应用背后都有集群的身影,而这些集群大都依托于高性能,廉价的Unix/Linux平台.

## 各部分之和大于整体

水泥、沙子、石子和水个各自的硬度不高,但是混合起来去能得到高硬度的混凝土.这个现实展示了一个很普通的道理就是对不同特性的物质进行组合往往能产出令人意外的结果,这种不遵循数学上"1+1=2"的规律却和化合反应非常类似.

放在软件工程中同样如此:通过组合小程序来构建集成性的应用程序.这就和我在上文中提到像搭积木一样编程.深究下来,无论是搭积木还是编程,其中共性就是**组合的思想**.从原子到分子,从树木到高楼大厦,从函数到程序,看似不相关的东西经组合产生出意想不到的结果,在某种意义上,组合的思想是我们人类大脑最重要的能力之一.

## 寻找90%的解决方案

"金无足赤,人无完人"这朗朗上口的话放在软件领域却代表一个不争的事实:没有一个软件能百分百的满足用户.说的更直白点,你的软件在用户看来总缺少点什么,无论你做出什么努力,这也从另一个角度说明"只有已发布的软件而没有完成的软件".

### 中国邮政 vs 顺丰
既然没有完美的软件,那对于软件开发者而言,有一个非常艰难的决定:"我们该向用户提供什么".这是个非常有意思的问题,其答案取决你所开发软件的性质以及面向的用户群体.

说到这,就不得不谈起中国邮政和顺丰速递.在中国的每个地方,你都可以用中国邮政来收发快递,而顺丰却只见于经济发展不错的地区.很多人像我一样曾抱怨过邮政的效率太低,接下来感叹一句要是有顺丰该多好.现在我们客观的来谈谈同为快递这两者本质差异:邮政作为一家国企,在政府的要求下,邮政必须服务所有的人,必须提供100%的解决方案,而顺丰作为一家独立的商业公司,却可以选择性服务,通过专注于高利润且容易做到的90%.最终的结果就是我们普遍觉得顺丰具有更高的效率.

这其中的关键在于我们需要故意忽略哪些代价昂贵,费时费力或难以解决的问题.从商业的角度来说,这是一种"低投入,高回报"的解决方案.现在我们可以来谈谈软件开发了.无论你是个人开发者还是商业软件公司负责人,都应该考虑这个问题"向用户提供什么才最有价值",然后去解决这其中的90%.这可能令很多人不适,但现在互联公司最不应该做的就是企图提供向邮政一样的100%的服务.提供90%的方案除了能获得更高的效益之外,还可以帮助聚集你的用户群体,这也是15年创业期间最大的感触之一.



## 层次化思考

层次化思想是什么?说起来很抽象,不如直接的问自己一个问题"金字塔是如何建成的?"如果你的回答是一层一层建成的,那么你可以直接略过本章节了.

![mage-20180708123408](https://i.imgur.com/hwkVSzg.png)

层次化思考并并非新名词,对大部分程序员而言,"分层"是所有解决方案的指导思想,这里的分层便是层次化思考的结果.与层次化思考相关的一种分析方法是AHP(Analytic hierarchy process),即层次分析法.(关于AHP更多的信息就不多讲了),它是思考和构造复杂系统的一个基本方法,按照该方法设计出的系统具有明显的层次体系.

现实世界充斥着层次化结构,小到细胞结构大到宇宙,从小团队到跨国公司,其都呈现出层次化结构,对计算机软件而言亦如此.在层次体系中,下层构件为上层构件提供服务支撑,上层和下层之间的关系就像是方法的"调用-返回".采用层次化设计的应用就像金字塔,先有底层构件,然后在之上构建高层构件,相对低层构件而言,高层构件更容易理解.

可以说,一个系统无论最初采用什么样的组织方式,最终都会走向分层体系.
我经常说的一句话是:**无分层不架构**,通俗点就是没有分层的应用是没有架构可言的.分层结构在计算机世界中无处不在,来看几个最典型的例子:
### ISO/OSI七层网络模型
![mage-20180708123451](https://i.imgur.com/5brqCvK.png)


### WEB应用分层结构
![mage-20180708123512](https://i.imgur.com/ZAy1xwo.png)

### Unix/Linux系统分层结构

![mage-20180708123557](https://i.imgur.com/UoTUfJd.png)

### Android 系统分层结构

![mage-20180708123632](https://i.imgur.com/wFFdoJt.png)


### MVC & MVP
除了上述几个典型示例外,两种典型的开发方式MVC 和MVP也都是典型的分层结构:
![mage-20180708123700](https://i.imgur.com/ahtYCc6.png)

无论是MVC还是MVP,亦或是MVVM,还是将来出现的什么M*P,只要记住所有的开发方式本质都是层次思考的结果,最终呈现分层体系,这就是所谓的万变不离其中.如果将分层体系和小程序结合起来,我们可以得到一个更为通用的程序结构设计图:
![mage-20180708123723](https://i.imgur.com/zIFORFn.png)

----------

# 总结

也许你会像我当初一样纳闷这么优秀的操作系统就靠这些看起来无比简单理念引导?事实确是如此,这有点像我们所说的"大道至简",**复杂的东西简单做**,也是我们解决问题的最重要的一步.


关于Unix/Linux中的编程哲学,我想以上十三条已经足够.其中最有启发的应属"小既是美",这也是我决定以后不写长文的原因.

[1]: https://book.douban.com/subject/3012360/
[2]: https://book.douban.com/subject/2377310/
[3]: https://book.douban.com/subject/26912767/
[4]: https://book.douban.com/subject/7564417/
[5]: https://book.douban.com/subject/6889456/