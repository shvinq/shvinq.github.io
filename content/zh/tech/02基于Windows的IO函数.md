+++
title= "02-基于Windows的IO函数"
date= "2021-08-20T02:20:04+08:00"
tags= ["网络编程"]
slug = "Network program02"
+++
Linux中套接字也是文件，因而可以通过文件I/O函数read和write进行数据传输。而Windows严格区分文件I/O函数和套接字I/O函数。

```c
#include <winsock2.h>
int send(SOCKET s, const char * buf, int len, int flags); //成功返回传输字节数，失败返回SOCKET_ERROR。
/*
s: 表示数据传输对象连接的套接字句柄值
buf: 保存带传输数据的缓存地址值
len: 要传输的字节数
flags: 传输数据时用到的多种共选项信息，目前只需传递0，表示不设置任何选项
*/
```

```c
#include <winsock2.h>
int recv(SOCKET s, const char * buf, int len, int flags);  
//成功时返回接收的字节数(收到EOF时为0)，失败返回SOCKET_ERROR
/*
s: 表示数据传输对象连接的套接字句柄值
buf: 保存带传输数据的缓存地址值
len: 要传输的字节数
flags: 传输数据时用到的多种共选项信息，目前只需传递0，表示不设置任何选项
*/
```

