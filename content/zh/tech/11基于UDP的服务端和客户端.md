+++
title= "11-基于UDP的C/S端"
date= "2021-08-20T02:20:04+08:00"
tags= ["网络编程"]
slug = "Network program11"
+++

**UDP中的客户端和服务端没有连接**

UDP服务器端和客户端不像TCP那般在连接状态下交换数据，它无需经过连接过程。也就是说不必调用TCP连接过程中调用的listen函数和accept函数。UDP中只有创建套接字过程和数据交换过程。



**基于UDP的数据I/O函数**

```c
#include <sys/socket.h>
ssize_t sendto(int sock, void * buff, size_t nbytes, int flags, struct sockaddr *to, socklen_t addrlen);
//成功返回传输的字节数，失败返回-1
/*
sock: 用于传输数据的UDP套接字文件描述符
buff: 保存待传输数据的缓冲地址值
nbytes: 待传输的数据长度，以字节为单位
flags: 可选项参数，若没有则传递0
to: 存有目标地址信息的sockaddr结构体变量的地址值
addrlen: 传递给参数的地址值结构体变量长度
*/
```

```c
#include <sys/socket.h>
ssize_t recvfrom(int sock, void *buff, size_t nbytes, int flags, struct sockaddr* from, socklen_t *addelen);
//成功返回接收字节数，失败返回0
/*
sock: 用于接收数据的UDP套接字文件描述符
buff: 保存待接收数据的缓冲地址值
nbytes: 可接收的最大字节数，故无法超过参数buff所指的缓冲大小
flags: 可选项参数，若没有则传递0
to: 存有发送端地址信息的sockaddr结构体变量的地址值
addrlen: 保存参数from的结构体变量长度的变量地址值
*/
```

**已连接(connected)UDP套接字和未连接(unconnected)UDP套接字**

TCP套接字中需要注册待传输数据的目标IP和端口号，而UDP中无需注册。因此通过sendto函数传输数据过程大致分为3阶段。

- 第一阶段：像UDP套接字注册目标IP和端口号
- 传输数据
- 删除UDP套接字中注册的目标地址信息

每次调用sendto函数都会重复这三个阶段。每次都变更目标地址，因此重复利用同一UDP套接字向不同目标传输数据，这种未注册目标地址信息的套接字叫做未连接套接字。但是如果向同一个地址的同一端口进行多次传输数据，则会消耗整体性能，所以可以使用connect函数注册目标地址信息来提升性能。此外使用connect函数后可以使用write和read函数传输数据。



**基于Windows平台的实现**

```c
#include <winsock2.h>
int sendto(SOCKET s, const char* buf, int len, int flags, const struct sockaddr* to, int tolen);
//成功返回接收的字节数，失败返回SOCKET_ERROR
```

```c
#include <winsock2.h>
int recvfrom(SOCKET s, const char* buf, int len, int flags, const struct sockaddr* from, int fromlen);
```

