title: 从VFS谈文件读写
date: 2015-12-05 22:41:57
toc: true
tags: [Linux,VFS,内核态,系统调用,I/O缓冲区,文件读写]
categories: technology
description: 在Linux中当我们修改txt文件的一个字,这期间发什么什么?虚拟文件系统是什么?内核态与用户态又是什么?

------

# 内核态与用户态

在CPU的所有指令中,有一些指令是非常危险的,误用将导致系统崩溃.为了安全起见CPU将指令分为***特权指令和非特权指令***，对于那些危险的指令，只允许操作系统及其相关模块使用，普通的应用程序只能使用那些不会造成灾难的指令.Intel CPU提供0到3四种级别的运行模式，数字越小特权越高.在Linux机器上，CPU只会在以下两种模式下运行:

- 受信任的内核模式(0级别),对应于Linux中的内核态
- 受限制的用户模式(3级别): 对应于LInux中的用户态

处理器总处于以下状态中的一种：

1. 内核态，运行于进程上下文，内核代表进程运行于内核空间；
2. 内核态，运行于中断上下文，内核代表硬件运行于内核空间；
3. 用户态，运行于用户空间。

用户空间的应用程序,通过系统调用进入内核空间.此时用户空间的进程要传递很多变量、参数的值给内核，内核态运行的时候也要保存用户进程的一些寄存器值,变量等.而所谓的进程上下文可以被看做是用户进程传递给内核的这些参数以及内核要保存的那些变量和寄存器值和当时的环境等。

硬件通过触发中断信号,导致内核调用中断处理程序进入内核空间.这个过程中，硬件的一些变量和参数也要传递给内核，内核通过这些参数进行中断处理。所谓的“中断上下文”，其实也可以看作就是硬件传递过来的这些参数和内核需要保存的一些其他环境（主要是当前被打断执行的进程环境）

## 系统调用

除了内核本身处于内核模式以外,所有的用户进程都运行在用户模式之中.用户空间的程序无法直接执行内核代码,它们不能直接调用内核空间中的函数.运行于用户态的进程可以执行的操作和访问的资源都会受到极大的限制，而运行在内核态的进程则可以执行任何操作并且在资源的使用上没有限制。很多程序开始时运行于用户态，但在执行的过程中，一些操作需要在内核权限下才能执行.在Linux中,除异常和中断陷入外,系统调用是用户空间访问内核的唯一合法入口.

> 系统调用的本质也是中断.相对外围硬件设备产生的硬中断信号而言,系统调用产生的是软中断.异常,中断和系统调用三者最大的区别在于系统调用是进程主动请求切换的,而其他两者则是被动的.

# 虚拟文件系统

虚拟文件系统(VFS)作为Kernal的子系统,为用户空间提供了文件和文件系统相关的接口.平常我们说对文件的操作就建立在这之上.那为什么要有虚拟文件系统呢?你可以虚拟机文件系统理解为Java中接口的概念:一个操作系统可以支持不同的底层文件系统(具体Java接口的实现),如 NTFS, FAT, ext3等,为了给内核和用户进程提供统一的文件系统视图,而在内核和用户进程之间加入了抽象层,即虚拟文件系统(Java接口定义).

内核通过VFS能够方便,简单地支持各种类型的文件系统.底层文件系统通过提供VFS所期望的抽象接口和数据结构,这样内核就可以简单的和任何文件系统协作,并且这样提供给用户空间的接口,也可以和任何文件系统无缝连接在一起.也即是说所有的文件操作都通过VFS,由VFS来适配不同的底层文件系统,最终完成操作.

![image-20181020173556821](https://i.imgur.com/KVrP0Yk.png)

简单点说,VFS定义了一个通用文件系统的接口层和适配层:

- 接口层: 为用户进程提供一组操作文件/目录/其他对象的统一方法
- 适配层: 和不同的底层文件系统进行适配



## 虚拟文件系统结构

VFS整体采用面向对象的设计思路,采用不同的数据结构表示不同的结构对象.由于内核主要是由C代码编写,因此内核中关于对象的实现都是采用结构体实现,同时这些结构体中也包含有关操作这些数据结构的函数指针,当然操作函数具体的实现依赖于不同的底层文件系统.

在VFS中主要由以下四个主要的对象类型:

- 超级块对象
- 索引节点对象
- 目录项对象
- 文件对象

### 超级块对象

整个文件系统的第一块空间,代表一个具体已经安装的文件系统,用于保存一个文件系统的所有元数据，如块大小，inode/block的总量、使用量、剩余量，指向空间 inode 和数据块的指针等相关信息,可以说是文件系统的信息库.文件系统的任意元数据修改都要修改超级块,另外为了性能考虑,该超级块对象是常驻内存并被缓存的。

在内核中该对象由结构体`super_block`表示,其操作由结构体`super_operations`表示,定义在`linux/fs.h`中.

```c
struct super_block {
    struct list_head      s_list;          // 指向所有超级块的链表 
    const struct super_operations  *s_op;  // 超级块方法
    struct dentry         *s_root;         // 目录挂载点 
    struct mutex          s_lock;    	   // 超级块信号量
    int                   s_count;         // 超级块引用计数
    
	......
        
    struct list_head      s_inodes;        // inode链表
    struct mtd_info       *s_mtd;          // 存储磁盘信息
    fmode_t               s_mode;          // 安装权限
};
```



### 索引节点对象

索引节点对象包含了内核在操作文件或目录是需要的全部信息,如文件的长度、创建及修改时间、权限、所属关系等,通过命令`ls -li`可查看.一个索引节点代表文件系统中(索引节点仅当文件被访问时才在内存中创建)的一个文件,可以是设备或者管道这样的特殊文件.

在内核中该对象由结构体`inode`表示,其操作由结构体`inode_operations`表示,定义在linux/fs.h中.

```c
struct inode {
    struct hlist_node i_hash;          			// 散列表，用于快速查找inode
    struct list_head  i_list;    				// 索引节点链表 
    struct list_head  i_sb_list; 				// 超级块链表超级块 
    struct list_head  i_dentry;  				// 目录项链表 
	
    ......

    uid_t             i_uid;     				// 使用者id 
    gid_t              i_gid;     				// 使用组id 
    struct timespec   i_atime;   				// 最后访问时间 
    struct timespec   i_mtime;   				// 最后修改时间 
    struct timespec   i_ctime;    				// 最后改变时间 

    const struct inode_operations  *i_op;       // 索引节点操作函数 
    const struct file_operations   *i_fop;      // 缺省的索引节点操作 
    struct super_block            *i_sb;        // 相关的超级块 
    struct address_space          *i_mapping;   // 相关的地址映射 
    struct address_space          i_data;       // 设备地址映射 
    unsigned int                  i_flags;      // 文件系统标志 
    void                          *i_private;   // fs 私有指针 
};
```



### 目录项对象

在文件路径中,每个部分都是目录项对象.比如`/bin/cp`中的/,bin,cp都属于目录项对象.另外目录项对象不需要在磁盘中存储因此没有对应的磁盘数据结构,VFS根据字符串形式的路径名现场创建它.

在内核中,该对象由结构体dentry表示,其操作由结构体`dentry_operator`表示,定义在`linux/dcache.h`中.

```c
struct dentry {
    atomic_t      d_count;         						// 使用计数 
    unsigned int  d_flags;         						// 目录项标识 
    spinlock_t    d_lock;          						// 单目录项锁 
    struct inode  *d_inode;        						// 相关联的索引节点 
    struct hlist_node  d_hash;     						// 散列表 
    struct dentry      *d_parent;  						// 父目录的目录项对象 
    struct qstr        d_name;     						// 目录项名称 
    struct list_head   d_lru;      						// 未使用的链表

    struct list_head   d_subdirs;  						// 子目录链表 
    struct list_head   d_alias;    						// 索引节点别名链表
    unsigned long       d_time;    						// 重置时间 
    const struct dentry_operations *d_op; 				// 目录项操作相关函数 
    
    ......
};
```



### 文件对象

文件对象表示进程已经打开的文件,是已打开的文件在内存中的表示,该对象会由相应的`open()`系统调用创建,由`close()`系统调用撤销.需要注意的是,由于多个进程可以同时打开和操作同一个文件,这意味同一个文件也可能存在对个对应的文件对象,但由于是同一个文件，其索引节点对象是唯一的,这样就实现了共享同一个磁盘文件.

在内核中,该对象由`file`结构体表示,其操作由结构体`file_operator`表示,定义在`linux/fs.h`中.

```c
struct file {
    union {
        struct llist_node  fu_llist;      // 文件对象链表
        struct rcu_head    fu_rcuhead;    // 释放之后的RCU链表
    } f_u;
    struct path            f_path;        // 包含的目录项
    struct inode           *f_inode;      // 缓存值
    const struct file_operations  *f_op;  // 文件操作函数
    spinlock_t		f_lock;               // 锁

    atomic_long_t          f_count;       // 文件对象引用计数
    
    ......
        
    struct address_space   *f_mapping;
};

```

除了以上四种对象类型外,还需要之道以下对象类型的含义:

- address_space:它表示一个文件在页缓存中已经缓存了的物理页。它是页缓存和外部设备中文件系统的桥梁。如果将文件系统可以理解成数据源，那么address_space可以说关联了内存系统和文件系统.
- block:表示实际记录文件的内容，一个文件可能会占用多个 block。

## 文件打开列表

文件打开列表包含了内核中所有已经打开的文件。每个列表表项是一个文件对象file.在超级块对象结构体(super_block)中存在s_files指针(内核版本3.19之前存在)指向了“已打开文件列表模块”，该链表信息是所有进程共享的。

## 进程与虚拟文件系统

系统中每一个进程都有自己的一组打开的文件,其中`file_struct`,`fs_struct`,`这两个个结构体有效的将进程和VFS联系在一起.内核中使用结构体task_struct表示单个进程的描述符，它包含一个进程的所有信息,如进程的空间地址,挂起信号,进程状态进程号,打开的文件等信息,其定义如下:

```c
struct task_struct {
    volatile long state;        //进程状态
    
    ......
        
    struct fs_struct *fs;		// 文件系统信息	
    struct files_struct *files; // 打开文件信息
    
    ......
}
```

### 文件描述符

在Linux中，进程是通过文件描述符（file descriptors，简称fd）而不是文件名来访问文件的，文件描述符实际上是一个整数,它本质就是files_struct结构体中fd_array域数组的索引.在后续会进一步说明.

#### file_struct

每个进程用一个files_struct结构来记录其使用文件的情况,主要包含文件描述符表和打开文件对象的信息.站在进程的角度上,这个files_struct结构又被称为用户打开文件表，它是进程的私有数据。从task_struct定义可见,其中一个files的指针来指向files_struct.

files_struct结构在include/linux/fdtable.h中定义如下：

```c
struct files_struct {
  atomic_t     count;           				// 结构的使用计数,表示共享该表的进程数
  bool         resize_in_progress;
  wait_queue_head_t resize_wait;

  struct fdtable __rcu  *fdt;   				// 指向其他fd表的指针
  struct fdtable        fdtab;  				// 基fd表
  spinlock_t file_lock ____cacheline_aligned_in_smp;
  int next_fd;        							// 已分配的文件描述符加1,表示缓存下一个可用的fd
  unsigned long      close_on_exec_init[1];     // 执行exec()时关闭的文件描述符链表
  unsigned long      open_fds_init[1];          // 文件描述符的初值集合
  unsigned long      full_fds_bits_init[1];      
  struct file __rcu  *fd_array[NR_OPEN_DEFAULT];// 文件对象指针的初始化数组
};
```

fd_array数组指针指向已打开的文件对象,在64位机器上NR_OPEN_DEFAULT这个宏的值为64.如果进程打开的文件数目多于64，内核就分配一个新的、更大的文件指针数组，并且将fdt指针指向它.

对于在fd数组中有入口地址的每个文件来说，数组的索引就是**文件描述符（*file descriptor*）**。通常，数组的第一个元素（索引为0）是进程的标准输入文件，数组的第二个元素（索引为1）是进程的标准输出文件，数组的第三个元素（索引为2）是进程的标准错误文件.

### fs_struct

进程和文件系统相关的信息,由fs_struct结构体表示,定义在`linux/fs_struct.h`中.从task_struct定义可见,其中一个fs的指针来指向fs_struct.

```c
struct fs_struct {
	int users;             // 用户数目
	spinlock_t lock;       // 结构体的锁
	seqcount_t seq;		   		
	int umask;             // 当打开文件设置文件权限时所使用的位掩码
	int in_exec;           // 当前正在执行的文件
	struct path root, pwd; // 根目录路径,当前工作目录的路径
} __randomize_layout;
```

# I/O缓冲区

在I/O过程中，涉及到磁盘和内存数据之间的传输操作,由于磁盘读写速度远小于内存读写速度,因此需要将读取过的数据缓存在内存里。而用于缓存数据的内存区域就是高速缓冲区（buffer).通过缓冲区能够有效的减少磁盘I/O的操作.

> 需要注意buffer cache和cpu cache不同,前者用于磁盘和内存之间的缓冲,而后者用于CPU和内存之间的缓冲.

对于I/O缓冲区,根据具体I/O操作设备不同,主要由两种缓冲方案:buffer cache和page cache.buffer cache即块缓冲器，page cache即页缓冲器,两者最大的区别是缓存粒度的不同:buffer cache面向的是文件系统的块;而内核的内存管理组件采用了比文件系统的块更高级别的抽象-页page.

在linux不支持虚拟内存机制之前，还没有页的概念，此时缓冲区以块为单位对设备进行操作。在linux支持虚拟内存的机制后页是虚拟内存管理的最小单位，此时采用页缓冲的机制来缓冲内存。Linux2.6之后这两个缓存机制被整合在一起，页和块可以相互映射.另外页缓存面向的是虚拟内存，块缓存是面向块设备。

page cache是由内存中的物理页组成的,其内容对应磁盘上的物理块.文件IO操作实际上只和page cache交互.在内核中,使用结构体`page`表示一个内存中的物理页.

```c
struct page {
	unsigned long flags; // flags来记录该页是否是脏页，是否正在被写回等等

	union {
		struct {
			struct list_head lru;
            // 指向了地址空间address_space，表示是一个页缓存器中的页，对应于一个文件的地址空间
			struct address_space *mapping; 
			pgoff_t index;	// 记录这个页在文件中的页偏移量；
			unsigned long private;
		};
        
		......
            
      };
    
    ......

}
```

文件系统的inode实际维护了这个文件所有的块block的块号，通过对文件偏移量offset取模可以很快定位到这个偏移量所在的文件系统的块号。同样，通过对文件偏移量offset进行取模可以计算出偏移量所在的页的偏移量。

page cache使用address_space来作为文件系统和页缓存的中间桥梁,用来表示一个文件在页缓存器中已经缓存了的物理页.此外,在address_space中通过指针可以方便的获取文件inode和struct page的信息.

# 文件读写流程

## 读流程

1. 进程调用库函数向内核发起读文件请求
2. 内核检查进程的文件描述符定位到虚拟文件系统的已打开文件列表中的文件表项
3. 调用该文件可用的系统调用函数read()
4. read()函数通过文件表项链接到目录项模块，根据传入的文件路径，在目录项模块中检索，找到该文件的inode
5. 在inode中，通过文件内容偏移量计算出要读取的页
6. 通过inode找到文件对应的address_space
7. 在address_space中访问该文件的页缓存树，查找对应的页缓存结点.如果页缓存命中,则直接返回文件内容;否则会产生一个缺页中断异常,接下来会创建一个页缓存页，并通过inode找到文件该页的磁盘地址，读取相应的页填充该缓存页,之后再重新进行第6步查找页缓存
8. 文件读取完成

## 写流程

1. 进程调用函数库向内核发起写文件请求.
2. 内核检查进程的文件描述符定位到虚拟文件系统的已打开文件列表中的文件表项.
3. 调用该文件可用的系统调用函数write().
4. write()函数通过文件表项链接到目录项模块,根据传入的文件路径,在目录项中检索,找到该文件的inode
5. 在inode中,通过文件内容偏移量计算出要读取的页.
6. 通过inode找到文件对应的address_space.
7. 在address_space中访问该文件的页缓存树,查找对应的页缓存节点.如果页缓存命中,直接把文件修改内容同步到页缓存中,此时写入操作意味着已经完成了;如果页缓存页缺失,则会产生缺页中断异常,接下来会创建一个页缓存页,并通过inode节点找到文件该页的磁盘地址,然后读取相应的内容来填充该,重新在address_space中访问该文件的页缓存树,查找对应的页缓存节点,页缓存页命中,接下来就和之前一样,将修改同步到该缓存页即可.
8. 页缓冲中的页一旦被修改,就会被标记成脏页.脏页需要被同步写会磁盘中,以保证磁盘和缓存的中数据一致.一方面我们可以在进程中主动调用sync()或者fsync()调用把脏页写回,另一方面pdflush进程会定时把脏页写回到磁盘.需要注意的是脏页一旦处在写回磁盘的过程中,该页会设置写回标记,此时会被加锁,内核不能讲该页置换出内存,其他写请求将被阻塞直到写操作完成.



