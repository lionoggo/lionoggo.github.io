title: 深入分析AIL语言及init.rc
date: 2016/07/21 01:22:02
toc  : true
tags: [Android,AIL,init.rc]
categories: technology
description: init.rc文件由系统第一个启动的init程序进行解析.它由"Android Init Language"语言编写而成.init.rc文件可以在你android设备根目录下找到.还记得我们上次编译的Android源码么?如果你已经编译过源码了,那么可以在out/target/generic/root/目录下找到该文件.要想读懂init.rc文件,首先要掌握Android Init Language语言,即AIL.在/system/core/init/下有一份readme.txt文件,为我们详细介绍了有关AIL的知识.我们下面的学习同样是借助了该文档来的.



---

# AIL语法
AIL语言非常简单,主要包括两部分:结构语法及注释语法.下面我们就这两点进行说明

## 结构语法
AIL语言包含主要包含五种结构语法:

 1. Actions
 2. Services
 3. Options
 4. Commands
 5. Imports

需要注意,AIL采用是面向行的代码风格,即用换行符作为一条语句的分隔符,也就是在init.rc中以一条语句通常占据一行.如果一行写不下,可以在行尾添加反斜杠来链接到下一行,换言之,通过行尾添加反斜杠符可以将多行代码链接为一行代码.

init.rc有许多Service和Action组成.那么什么是Service和Action呢?
Action和Service显式声明了一个语句块,而Commands和Options则分别用来定义Actions和Service(你可以理解为这是Action或者Service的属性).

另外,我们声明的Commands和Options属于最近声明的语句块,即就近原则.需要注意,在第一个语句块之前的commands和options会被忽略.

每个Actions或者Services应该有唯一的名字.对于名字重复的情况,Action和Service有自己不同的处理方式:
>如果第二个定义的Action的名字和之前存在Action的名字相同,第二个Action中定义的Commands将会被添加到已经存在的同名Action中.如果第二个定义的Service的名字和之前存在的Service的名字相同,第二个Service会被忽略并输出错误信息.


## 注释语法
AIL中的注释语法和Shell脚本一致,以#开头即可




# 结构语法详解
## Actions
Actions代表一些Action.Action代表一组命令,它包含一个触发器,该触发器决定了何时执行这个Action,即在什么情况下才能执行该Action中的定义命令.当一些条件满足触发器的条件时,该Action中定义的命令会被添加到要执行命令队列的尾部(如果这组命令已经在队列中,则不会再次添加).

当一个Action从队列移除时,该Action定义的命令会依次被执行.

Action的格式如下:

```
on <trgger> [&& <trigger>]*
   <command>
   <command>
   <command>
   ...
```
不难发现Action都是以on开始,随后会定义触发器(trigger),接着便是为其定义命令(Commmand).在开始讲解Trigger和Command之前,我们先来看一段Action的示例代码:

```
on boot
    # 初始化网络
    ifup lo
    hostname localhost
    domainname localdomain

```

## trigger
trigger即我们上面所说的触发器,本质上是一个字符串,能够匹配某种包含该字符串的事件.
trigger又被细分为事件触发器(event trigger)和属性触发器(property trigger).

事件触发器可由"trigger"命令或初始化过程中通过QueueEventTrigger()触发,通常是一些事先定义的简单字符串,例如:`boot`,`late-init`
属性触发器是当指定属性的变量值变成指定值时触发,其格式为`property:<name>=*` 

一个Action可以有多个属性触发器,但是最多有一个事件触发器.下面我们看两个例子:

```
on boot && property:a=b

```
该Action只有在boot事件发生时,并且属性a和b相等的情况下才会被触发.

```
on property:a=b && property:c=d
```
该Action会在以下三种情况被触发:

 - 在启动时,如果属性a的值等于b并且属性c的值等于d
 - 在属性c的值已经是d的情况下,属性a的值被更新为b
 - 在属性a的值已经是b的情况下,属性c的值被更新为d

当前AIL中常用的有以下几种事件触发器:

|类型|说明|
|---|---|
|`boot`|init.rc被装载后触发|
|`device-added-<path>`|指定设备被添加时触发|
|`device-removed-<path>`|指定设备被移除时触发|
|`service-exited-<name>`|在特定服务(service)退出时触发|
|`early-init`|初始化之前触发|
|`late-init`|初始化之后触发|
|`init`|初始化时触发|


## Commands
Commands代表一组命令,在为Action设置了触发器后,就需要为其定义一组命令(command)了.AIL中内置了众多的命令,下面我们做个简单的说明:

|命令|解释|
|---|---|
|`bootchart_init`|如果配置了bootcharing,则启动.包含在默认的init.rc中|
|`chmod`|更改文件权限|
|`chown <owner> <group> <path>`|更改文件的所有者和组|
|`calss_start <serviceclass>`|启动指定类别服务下的所有未启动的服务|
|`class_stop <serviceclass>`|停止指定类别服务类下的所有已运行的服务|
|`class_reset <serviceclass>`|停止指定类别的所有服务(服务还在运行),但不会禁用这些服务.后面可以通过class_start重启这些服务|
|`copy <src> <dst>`|复制文件,对二进制/大文件非常有用|
|`domainname <name>`|设置域名称|
|`enable <servicename>`|启用已经禁用的服务|
|`exec [ <seclabel> [ <user> [ <group> ]* ]] `<br>`--<command> [ <argument> ]* `|fork一个进程执行指定命令,如果有参数,则带参数执行|
|`export <name>`|在全局环境中,将`<name>`变量的值设置为`<value>`,即以键值对的方式设置全局环境变量.这些变量对之后的任何进程都有效|
|`hostname`|设置主机名|
|`ifup <interface>`|启动某个网络接口|
|`insmod [-f] <path> [<options>]`|加载指定路径下的驱动模块。-f强制加载，即不管当前模块是否和linux kernel匹配|
|`load_all_props`|从/system，/vendor加载属性。默认包含在init.rc|
|`load_persist_props`|当/data被加密时，加载固定属性|
|`loglevel <level>`|设置kernel日志等级|
|`mkdir <path> [mode] [owner] [group]`|在制定路径下创建目录|
|`mount_all <fstab> [ <path> ]*`|在给定的fs_mgr-format上调用fs_mgr_mount和引入rc文件|
|`mount <type> <device> <dir>[ <flag> ]* [<options>]`|挂载指定设备到指定目录下.|
|`powerct`|用来应对sys.powerctl中系统属性的变化,用于系统重启|
|`restart <service>`|重启制定服务，但不会禁用该服务|
|`restorecon <path> [ <path> ]*`|恢复指定文件到file_contexts配置中指定的安全上线文环境|
|`restorecon_recursive <path> [ <path> ]*`|以递归的方式恢复指定目录到file_contexts配置中指定的安全上下文中|
|`rm <path>`|删除指定路径下的文件|
|`rmdir <path>`|删除制定路径下的目录|
|`setprop <name> <value>`|将系统属性`<name>`的值设置为`<value>`,即以键值对的方式设置系统属性|
|`setrlimit <resource> <cur> <max>`|设置资源限制|
|`start <service>`|启动服务(如果该服务还未启动)|
|`stop <service>`|关闭服务(如果该服务还未停止)|
|`swapon_all <fstab>`||
|`symlink <target> <path>`|创建一个指向`<path>`的符合链接`<target>`|
|`sysclktz <mins_west_of_gmt>`|设置系统时钟的基准,比如0代表GMT,即以格林尼治时间为准|
|`trigger <event>`|触发一个事件,将该action排在某个action之后(用于Action排队)|
|`verity_load_state`||
|`verity_update_state <mount_point>`||
|`wait <path> [ <timeout> ]`|等待一个文件是否存在,存在时立刻返回或者超时后返回.默认超时事件是5s|
|`write <path> <content>`|写内容到指定文件中|


## Services

Services代表一些Service.Service是一些在系统初始化时就启动或者退出时需要重启的程序.其格式如下:

```
service <name> <pathname> [ <argument> ]*
        <option>
        <option>
        ...
```
不难发现,首先需要为服务定义名字,并指定程序路径,然后便是通过option来修饰服务.同样先来看一下示例:

```
service ueventd /sbin/ueventd
    class core
    critical
    seclabel u:r:ueventd:s0
```


## Options

Options代表一些option.option用来修饰服务,决定了服务在什么时候运行以及怎样运行.AIL中提供了非常多的option,下面我们做个简单说明:

|选项|解释|
|----|----|
|`console`|服务需要一个控制台.|
|`critical`|表示这是一个关键设备服务.如果4分钟内此服务退出4次以上,那么这个设备将重启进入recovery模式|
|`disabled`|服务不会自动启动,必须通过服务名显式启动|
|`setenv <name> <value>`|在进程启动过程中,将环境变量`<name>`的值设置为`<value>`,即以键值对的方式设置环境变量|
|`socket <name> <type> <perm> [ <user> [ <group> [seclabel]]]`|创建一个unix域下的socket,其被命名`/dev/socket/<name>`. 并将其文件描述符fd返回给服务进程.其中,type必须为dgram,stream或者seqpacke,user和group默认是0.seclabel是该socket的SELLinux的安全上下文环境,默认是当前service的上下文环境,通过seclabel指定.|
|`user <username>`|在执行此服务之前切换用户名,当前默认的是root.自Android M开始,即使它要求linux capabilities,也应该使用该选项.很明显,为了获得该功能,进程需要以root用户运行|
|`group <groupname>`|在执行此服务之前切换组名,除了第一个必须的组名外,附加的组名用于设置进程的补充组(借助setgroup()函数),当前默认的是root|
|`seclabel <seclabel>`|在执行该服务之前修改其安全上下文,默认是init程序的上下文|
|`oneshot`|当服务退出时,不重启该服务|
|`class <name>`|为当前service设定一个类别.相同类别的服务将会同时启动或者停止,默认类名是default.|
|`onrestart`|当服务重启时执行该命令|
|`priority <priority>`|设置服务进程的优先级.优先级取值范围为-20~19,默认是0.可以通过setpriority()设置|



## Imports

用来引入一个要解析的其他配置文件,通常用于当前配置文件的扩展.
其格式如下:
```
import <path>
```
如果path是个一个目录,则该目录下的每个.rc文件都被引入.

在初始化过程中,共有两次使用import来引入.rc文件:

 1. 在初始化引导期间,引入/init.rc文件
 2. 在执行mount_all命令时,引入/{system,vendor,odm}/etc/init/或者指定路径下的.rc文件

我们来看看init.rc文件引入的.rc文件:

```
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /init.${ro.zygote}.rc
```


## Properties

Properties代表Init进程运行中的一些属性信息.在Init运行中,通过以下属性能够获取当前程序内部信息:

|类型|说明|
|----|----|
|`init.svc.<name>`|指定名称服务的状态,有stopped,stopping,runing,restarting这种四种状态|
|`init.action`|获取当前正在执行的action|
|`init.command`|获取当前正在执行的command|




# 文件示例

到现在为止,有关AIL相关的知识基本介绍完毕,下面截取init.rc文件中的一段来做个简单的说明:
```
//引入其他要解析的rc文件
import /init.environ.rc
import /init.usb.rc
import /init.${ro.hardware}.rc
import /init.usb.configfs.rc
import /init.${ro.zygote}.rc

#定义了一个action,在init初始化之前触发
on early-init
    # Set init and its forked children's oom_adj.
    write /proc/1/oom_score_adj -1000

    # Disable sysrq from keyboard
    write /proc/sys/kernel/sysrq 0

    # Set the security context of /adb_keys if present.
    restorecon /adb_keys

    # Shouldn't be necessary, but sdcard won't start without it. http://b/22568628.
    mkdir /mnt 0775 root system

    # Set the security context of /postinstall if present.
    restorecon /postinstall
    
    #启动ueventd服务
    start ueventd
    
    #...省略多行...
    
#定义ueventd服务,设置服务为/sbin/ueventd
service ueventd /sbin/ueventd
    class core#为其设置类名为core
    critical#表明这是一个关键服务
    seclabel u:r:ueventd:s0 #设置其安全上下文

```

# 总结
AIL是一种非常简单的语言,主要用于定义启动流程中需要做的事情.
