+++
title= "03-Windows平台socket"
date= "2021-08-20T02:20:04+08:00"
tags= ["网络编程"]
slug = "Network program03"
+++
### 1.Winsock的初始化

进行Winsock编程，首先调用WSAStartup函数，设置程序中用到的Winsock版本,并初始化相应的版本。

```c
#include <winsock2.h>
int WSAStartup(WORD wVersionRequested, LPWSADATA lpWSAData); //成功返回0，失败返回非0的错误代码值
/*
wVersionRequested: 程序员所需要的Winsock版本信息
lpWSAData: WSADATA结构体变量
*/
```

参数说明

​	第一个参数：Windsock中存在多个版本。应准备WORD(WORD是通过typedef声明定义的unsigned short类型)套接字版本信息，并传	递给该函数的	第一个参数wVersionRequested。若版本为1.2，则1是主版本号，2是副版本号，应传递0x0201。

​		高8位是副版本号，低8位是主版本号，以字节位单位收到构造版本信息有些麻烦，借助MAKEWORD宏函数则可快速构建WORD型版		本信息。

```c
MAKEWORD(1, 2); //1是主版本号，2是副版本号，返回0x0201
```

​	第二个参数：lpWSAData需要传入WSADATA型结构体变量地址(LPWSADATA是WSADATA的指针类型)。调用完函数后，相应参数中将	填充已初始化的库信息。虽无特殊含义，但为了调用函数，必须传递WSADATA结构体变量地址。以下代码成为Winsock编程的公式。

```c
int main()
{
    WSADATA wsaData;
    ...
    if(WSAStartup(2, 2), &wsaData != 0)
        ErrorHandling("WSAStarup() error!");
    ...
    return 0;
}
```

### 2.注销Winsock相关库

```c
#include <winsock2.h>
int WSACleanup(void);  //成功返回0，失败返回SOCKET_ERROR
```

### 3.基于Windows的套接字相关函数

...

### 4.Windows中的文件句柄和套接字句柄

​	Linux内部也将套接字当作文件，因此不管创建文件还是套接字都返回文件描述符。Windows中通过调用系统函数创建文件时，返回“句柄”(handle)，也就是相当于Linux的文件描述符。只不过Windows中要区分文件句柄和套接字句柄。虽然都是句柄，但不像Linux那样完全一致。文件句柄相关函数跟套接字相关函数是有区别的。

