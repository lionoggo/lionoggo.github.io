title: Unix Domain Socket
date: 2016-10-1 16:30:19
toc: true
tags: [Unix,Socket,UDS]
categories: technology
description: 和网络Socket不同,Unix Domain Socket适用于同一主机内进程通信的机制.

----

# Unix Domain Socket简介

Unix Domain Socket是在网络Socket架构上发展而来用来实现同一台主机内进程通信的机制,简称UDS.和网络Socket相比,使用UDS传输数据不需要经过网络协议栈,不存在数据封包和拆包等过程,只涉及数据拷贝的过程.UDS使用系统文件地址来作为通信地址,无须像网络Socket那样必须指定可用的IP和端口号,使用更加简单高效.

此外UDS支持SOCK_DGRAM和SOCK_DGRAM两种工作方式,即数据包套接字和流套接字,这两者类似UDP/TCP,但由于UDS在本机内是借助内核通信,因此不会出现丢包及发送包和接收包次序不一致的问题,换言之就是两种工作模式下,UDS都是可靠的通信机制.

尽管是POSIX标准中的一个组件,但在LInux中同样是受支持的,此外UDS是全双工的,能够用于两个没有亲缘关系的进程之间通信.

# UDS函数

和使用网络Socket类似,UDS服务端使用流程为:

```shell
socket -> bind -> listen -> accept -> recv/send -> close
```

UDS客户端使用流程如下:

```shell
socket -> connect -> recv/send -> close
```

## Socket创建

通过函数`socket()`来创建Socket,其函数原型为:

```c
int socket (int domain, int type, int protocol);
```

在使用UDS时,domain必须被设置成AF_UNIX或AF_LOCAL,而不能网络SOCKET中的AF_INET.使用AF_UNIX参数将会在系统上创建一个socket文件,不同进程通过读写这个文件来实现通信.

第二个参数type表示套接字的类型:SOCK_STREAM或SOCK_DGRAM.正如之前所说SOCK_STREAM和SOCK_DGRAM都是可靠的,不存在丢包以及发送包的次序和接收包的次序不一致的问题,两者之间唯一的区别在于SOCK_STREAM无论发送多大的数据都不会被截断,而对于SOCK_DGRAM而言,如果发送的数据超过了一个报文的最大长度,那么数据会被截断.

第三个参数protocol表示协议,对UDS而言,其值需要被设置成0.

## Socket绑定

如果服务端Socket采用SOCK_STREAM工作模式,那么需要为该Socket对应的文件描述地址绑定地址,该过程通过`bind()`函数实现,其原型为:

```c
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

sockfd即通过`socket()`返回的文件描述符,`*addr`用于描述套接字的地址,采用sockaddr_un结构体表示,addrlen表示结构体的长度.

```c
struct sockaddr_un {
    sa_family_t sun_family;         /* AF_UNIX */
    char sun_path[UNIX_PATH_MAX];   /* 路径名 */
};
```

在sockaddr_un中,字段sun_family必须要设置成AF_UNIX.而sun_path则表示路径名.在使用`bind()`时,需要创建并初始化sockaddr_un结构体,并将该结构体的指定传入作为`bind()`参数,同时将addrlen设置成该结构体的实际大小.

## 监听客户端连接

对于服务端而言,服务端Socket需要通过`listen()`监听来自客户端的连接,其使用过程和网络Socket保持一致,该函数原型为:

```c
int listen(int sockfd, int backlog);
```

参数sockfd是Socket创建后返回的文件描述符;而backlog用于设置请求排队的最大长度,也就是当有多个客户端程序和服务端相连时,该参数用来表示客户端排队的长度.

## 接受客户端连接

对于服务端而言,当接受到来自客户端请求时,需要`accept()`来处理客户端连接,其函数原型为:

```c
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

和网络Socket不同,UDS不存在客户端地址的问题,所以参数addr和addrlen参数需要被设置成NULL.在`accept()`调用后,服务器端会阻塞直到客户端发起连接. 

## 连接服务端

对于客户端Socket而言,在使用`socket()`创建套接字描述符之后,就可以通过`connect()`来连接服务端了,其函数原型为:

```c
int connect(int sockfd, struct sockaddr *addr,int addrlen);
```

和`bind()`函数类似,在该函数中同样需要制定*addr,用于表示需要连接到的服务端套接字地址.

## 收发数据

当客户端和服务端建立连接后,就可以进行数据收发了,对于工作在SOCK_STREAM模式下的服务端而言,通过`write()`和`read()`实现数据写入和读取操作,其使用方式和操作网络Socket一致:

```c
ssize_t read(int sockfd, void *buf, size_t length); 
ssize_t write(int sockfd, const void *buf, size_t length);
```

对于工作在SOCK_DGRAM模式下的服务端而言,接受数据和发送数据分别通过函数`recvfor()`和`sendto()`来完成.

```c
int recvfrom(int sockfd, void *buf, int length, unsigned int flags, struct sockaddr *addr, int *addrlen);

int sendto (int sockfd, const void *buf, int length, unsigned int flags, const struct sockaddr *addr, int addrlen);
```

# UDS实例

## 服务端代码

```c
#include <stdlib.h>  
#include <stdio.h>  
#include <stddef.h>  
#include <sys/socket.h>  
#include <sys/un.h>  
#include <errno.h>  
#include <string.h>  
#include <unistd.h>  
#include <ctype.h>   
 
#define MAXLINE 80  
 
char *socket_path = "server.socket";  
 
int main(void)  
{  
    struct sockaddr_un server_addr, client_addr;  
    socklen_t client_addr_len;  
    int serverfd; 
    int clientfd;
    int size;  
    char buf[MAXLINE];  
    int i, n;  
    
    // 1. 创建Socket
    if ((serverfd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0) {  
        perror("Socket创建错误");  
        exit(1);  
    }  
    printf("Socket创建完成\n");
    memset(&server_addr, 0, sizeof(server_addr));  
    server_addr.sun_family = AF_UNIX;  
    strcpy(server_addr.sun_path, socket_path);  
    size = offsetof(struct sockaddr_un, sun_path) + strlen(server_addr.sun_path);  
    unlink(socket_path);  
    
    // 2. 绑定Socket
    if (bind(serverfd, (struct sockaddr *)&server_addr, size) < 0) {  
        perror("Socket绑定错误");  
        exit(1);  
    }  
    printf("Socket绑定成功\n");  
    
    // 3. 监听Socket  
    if (listen(serverfd, 10) < 0) {  
        perror("Socket监听错误");  
        exit(1);          
    }  
    printf("Socket开始监听\n");  
 
    while(1) {  
        client_addr_len = sizeof(client_addr);         
        if ((clientfd = accept(serverfd, (struct sockaddr *)&client_addr, &client_addr_len)) < 0){  
            perror("连接建立失败");  
            continue;  
        }  
          
        while(1) {  
            n = read(clientfd, buf, sizeof(buf));  
            if (n < 0) {  
                perror("读取客户端数据错误");  
                break;  
            } else if(n == 0) {  
                printf("连接断开\n");  
                break;  
            }  
              
            printf("接受客户端数据: %s", buf);  
 
            for(i = 0; i < n; i++) {  
                buf[i] = toupper(buf[i]);  
            } 
             
            write(clientfd, buf, n);  
            printf("向客户端发送数据: %s", buf); 
        }  
        close(clientfd);  
    }  
    close(serverfd);  
    return 0;  
}

```

## 客户端代码

```c
#include <stdlib.h>  
#include <stdio.h>  
#include <stddef.h>  
#include <sys/socket.h>  
#include <sys/un.h>  
#include <errno.h>  
#include <string.h>  
#include <unistd.h>  
 
#define MAXLINE 80  
 
char *client_path = "client.socket";  
char *server_path = "server.socket";  
 
int main() {  
    struct  sockaddr_un client_addr, server_addr;  
    int len;  
    char buf[100];  
    int sockfd, n;  
 
    // 1. Socket创建
    if ((sockfd = socket(AF_UNIX, SOCK_STREAM, 0)) < 0){  
        perror("Socket创建失败");  
        exit(1);  
    }  
      
    memset(&client_addr, 0, sizeof(client_addr));  
    client_addr.sun_family = AF_UNIX;  
    strcpy(client_addr.sun_path, client_path);  
    len = offsetof(struct sockaddr_un, sun_path) + strlen(client_addr.sun_path);  
    unlink(client_addr.sun_path);  
    
    // 2. Socket绑定
    if (bind(sockfd, (struct sockaddr *)&client_addr, len) < 0) {  
        perror("Socket绑定错误");  
        exit(1);  
    }  
 
    memset(&server_addr, 0, sizeof(server_addr));  
    server_addr.sun_family = AF_UNIX;  
    strcpy(server_addr.sun_path, server_path);  
    len = offsetof(struct sockaddr_un, sun_path) + strlen(server_addr.sun_path);  
    
    // 3. 连接服务端
    if (connect(sockfd, (struct sockaddr *)&server_addr, len) < 0){  
        perror("连接服务端失败");  
        exit(1);  
    }  
 
    while(fgets(buf, MAXLINE, stdin) != NULL) {    
         write(sockfd, buf, strlen(buf));    
         n = read(sockfd, buf, MAXLINE);    
         if ( n < 0 ) {    
            printf("服务端关闭\n");    
         }else {    
            write(STDOUT_FILENO, buf, n);    
         }    
    }   
    close(sockfd);  
    return 0;  
}

```

## 编译并执行

使用gcc命令对其进行编译后,并运行

```shell
# 编译
gcc uds_server.c -o server
gcc uds_client.c -o client

# 运行
./server
./client
```

