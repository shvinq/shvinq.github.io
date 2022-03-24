---
title: "17epoll实现IO复用"
date: "2021-08-20T02:20:04+08:00"
tags:
categories: ["网络编程"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
disableHLJS: true
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: false
ShowBreadCrumbs: true
ShowPostNavLinks: true
---
> 实现IO复用的传统方法有select函数和epoll函数。epoll函数优于select函数。其他系统如BSD的kqueue、Solaris的/dev/poll和Windows的IOCP复用技术。

### 基于select的IO复用技术速度慢的原因

- 调用select函数后常见的针对所有文件描述符的循环语句。
- 每次调用select函数时都需要向该函数传递监视对象信息

调用select函数后，并不是把发生变化的文件描述符单独集中在一起，而是通过观察作为监视对象的fd_set变量的变化，找出发生变化的文件描述符。因此需要有针对所有监视对象的循环语句。此外作为监视对象的fd_set变量会发生变化，调用select函数前应复制保存元u你要信息，并在每次调用select函数时传递新的监视对象信息。

每次调用select函数时向操作系统传递监视对象信息造成的负担比循环语句的更大，且无法从代码上优化。所以需要通过“仅向操作系统传递一次监视对象，监视范围或内容发送变化时只通知发送变化的事项”这种方式解决。不同操作系统的支持方式不同，Linux的方式时epoll、Windows的是IOCP。

**select函数的优点**：

- 服务器端接入者少
- 程序应具有兼容性

### 实现epoll时必要的函数和结构体

epoll函数的优点

-  无需编写以监视状态变化为目的的针对所有文件描述符的循环语句
-  调用对应于select函数的epoll_wait函数时无需每次传递监视对象信息

epoll服务器端实现需要的三个函数

1. epoll_creat：创建保存epoll文件描述符的空间
2. epoll_ctl：向空间注册并注销文件描述符
3. epoll_wait：与select函数类似，等待文件描述符发送变化

epoll通过结构体epoll_event将发送变化的文件描述符单独集中在一起。声明足够大的epoll_event结构体数组后，传递给epoll_wait函数，发生变化的文件描述符信息将被填入该数组。所以无需像select函数那样对文件描述符进行循环。

```c
typedef union epoll_data
{
    void * ptr;
    int fd;
    __unit32_t u32;
    __unit64_t u64;
} epoll_data_t;

struct epoll_event
{
    __unit32_t events;
    epoll_data_t data;
}

```

**epoll_create函数**。调用epoll_create函数时创建的文件描述符保存空间叫做“epoll例程”。参数size建议epoll例程的大小，但该值只是向OS提建议供OS参考。

```c
#include <sys/epoll.h>
int epoll_create(int size);  //成功返回epoll文件描述符，失败返回-1；size表示epoll实例大小
```

**epoll_ctl函数**。生成epoll例程后，调用该函数在其内部注册监视对象文件描述符

```c
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event * event); //成功返回0，失败返回-1
/*
epfd: 用于注册监视对象的epoll例程的文件描述符
op: 用于指定监视对象的添加、删除或更改等操作
fd: 需要注册的监视对象文件描述符
event: 监视对象的事件类型
*/
epoll_ctl(A, EPOLL_CTL_ADD, B, C); epoll例程A中注册文件描述符B，主要目的是监视参数C中的事件
epoll_ctl(A, EPOLL_CTL_ADD, B, NULL); 从epoll例程A中删除文件描述符B
```

第二个参数op的选项和含义：

- EPOLL_CTL_ADD：将文件描述符注册到epoll
- EPOLL_CTL_DEL：从epoll例程中删除文件描述符
- EPOLL_CTL_MOD：更改注册的文件描述符的关注事件发生情况

第四个参数event：

```c
struct epoll_event event;
...
event.events = EPOLLIN;	//发生需要读取数据的情况(事件)时
event.data.fd = sockfd;
epoll_ctl(epfd, EPOLL_CTL_ADD, sockdf, &event);
...
```

上述代码是将sockfd注册到epoll例程epfd中，并在需要读取数据的情况下产生对应事件。

- EPOLLIN：需要读取数据的情况
- EPOLLOUT：输出缓冲为空，可以立即发送数据的情况
- EPOLLPRI：收到OOB数据的情况
- EPOLLRDHUP：断开连接或半连接的情况，这在边缘触发方式下非常有用
- EPOLLERR：发生错误的情况
- EPOLLET：以边缘触发的方式得到事件通知
- EPOLLONESHOT：发生一次事件后，相应文件描述符不在收到事件通知。因此需要向epoll_ctl函数第二个参数传递EPOLL_CTL_MOD再次设置事件。

**epoll_wait函数**

```c
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout); //成功返回发生事件的文件描述符，失败返回-1
/*
epfd: 表示事件发生监视范围的epoll例程的文件描述符
events: 保存发生事件的文件描述符集合的结构体指针
maxevents: 第二个参数中可以保存最大的事件数
timeput: 以1/1000秒为单位的等待时间，传递-1时，一直等待直到事件发生。
*/
```

需要注意第二个参数所知缓冲需要动态分配。

```c
int event_cnt;
struct epoll_event * ep_events;
...
ep_events = malloc(sizeof(struct epoll_event)*EPOLL_SIZE);  //EPOLL_SIZE是宏常量
event_cnt = epoll_wait(epfd, ep_events, EPOLL_EIZE, -1);
...
```

### 条件触发(Level Trigger)和边缘触发(Edge Trigger)

**条件触发特性**：“条件触发方式中，只要输入缓冲有数据就会一致通知该事件”

**边缘触发特性**：“边缘触发中输入缓冲收到数据时仅注册1次该事件，即使输入缓冲中还留有数据也不会再进行注册”

epoll默认以条件触发方式工作，select模型也是以条件触发方式工作。

修改成边缘触发需要将EPOLLIN---> EPOLLIN|EPOLLET。

**边缘触发服务器端中的内容**

以下两点：

- 通过errno变量验证错误原因
- 为了完成非阻塞IO，更改套接字属性

Linux套接字相关函数一般通过返回-1通知发生错误，另外Linux声明了全局变量 int errno; 访问改变量，需要引入error.h头文件，其中有errno变量的extern声明。每种函数出错时，保存到errno变量中的值不同。

将套接字改成非阻塞方式的方法，Linux提高更改或读取文件属性的方法：

```c
#include <fcntl.h>
int fcntl(int filedes, int cmd, ...);  //成功返回cmd参数相关值，失败返回-1
/*
filedes: 属性更改目标的文件描述符
cmd: 表示函数调用的目的
*/
```

fcntl函数具有可变参数的形式。如果第二个参数传递F_GETFL，可以获得第一个参数所指的文件描述符属性(int型)；如果传递F_SETFL，可以更改文件描述符属性。需要将文件(套接字)改成非阻塞模式，加入下面的代码：

```c
int flag = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flag|O_NONBLOCK);
```

**实现边缘触发的回声服务器端**

读取错误原因的方法和非阻塞模式套接字创建方法给边缘触发服务器提高了基础。

> 边缘触发中，接收数据时仅注册一次该事件，所以一旦发生输入相关事件就应该读取输入缓冲中的全部数据，因此需要验证输入缓冲是否为空——“如果read函数返回-1，变量errno中的值时EAGAIN时，说明没有数据可读取”。同时需要非阻塞模式套接字，不然read&write函数可能引起服务器端长时间停顿。

**边缘触发方式的优点：“可以分离接收数据和处理数据的时间点！”**

