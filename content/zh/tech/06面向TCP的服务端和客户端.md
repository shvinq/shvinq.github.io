+++
title= "06-基于TCP的C/S端"
date= "2021-08-20T02:20:04+08:00"
tags= ["网络编程"]
slug = "Network program06"
+++


### TCP服务端的默认函数调用顺序：

![image-20210809192110319.png](https://i.loli.net/2021/10/13/FEmBg52I6VU7anS.png)

**进入等待连接请求状态**

调用bind函数给套接字分配了地址，接下来调用listen函数进入等待连接请求状态，只有调用了listen函数后客户端才能进入可发出连接请求的状态，也就是这时客户端才能调用connect函数(若提前调用会发送错误)。

```c
#include <sys/socket.h>
int listen(int sock, int backlog); //成功返回0，失败返回-1
/*
sock: 希望进入等待连接请求状态的套接字文件描述符，传递的描述符套接字参数成为服务器端的套接字(监听套接字)
backlog: 连接请求等待队列(Queue)的长度，若为5，表示最多使5个连接请求进入队列
*/
```

**受理客户端连接请求**

```c
#include <sys/socket.h>
int accept(int sock, struct sockaddr * addr, socklen_t * addrlen);  //成功返回创建的套接字文件描述符，失败返回-1
/*
sock: 服务器套接字的文件描述符
addr: 保存发起连接请求的客户端地址信息的变量地址值，调用函数后向传递来的地址变量参数填充客户端地址信息
addrlen: 第二个参数addr结构体的长度，但是存有长度的变量地址，函数调用完成后，该变量即被填充入客户端地址长度
*/
```

accept函数受理连接请求等待队列中待处理的客户端请求连接。函数调用成功时，accept内部将产生用于数据I/O的套接字，并返回其文件描述符。套接字是自动创建的，并自动与发起连接请求的客户端建立连接。



### TCP客户端的默认函数调用顺序

![image-20210810223613521.png](https://i.loli.net/2021/10/13/8jpgYfCBRDqJTAX.png)

```c
#include <sys/socket.h>
int connect(int sock, struct sockaddr * servaddr, socklen_t addrlen);  //成功返回0，失败返回-1
/*
sock: 客户端套接字文件描述符
servaddr: 保存目标服务器端地址信息的变量指针
addelen: 以字节为单位传递以传递给第二个结构体参数servaddr的地址变量长度
*/
```

客户端调用connect函数后，发送以下情况之一才会返回(完成函数调用)：

- 服务器端接受连接请求
- 发送断网等异常情况而中断连接请求
