---
title: "01Linux文件操作"
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
### 1.打开文件：

```c++
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *path, int flag);  //成功返回文件描述符，失败返回-1。
									   //path：文件名的字符串地址；flag：文件打开模式信息
```

| 打开模式 | 含义                       |
| -------- | -------------------------- |
| O_CREAT  | 必要时创建文件             |
| O_TRUNC  | 删除全部现有数据           |
| O_APPEND | 维持现有数据，保存到其后面 |
| O_RDONLY | 只读打开                   |
| O_WRONLY | 只写打开                   |
| O_RDWR   | 读写打开                   |

### 2.关闭文件

```c
#include <unistd.h>
int close(int fd);		//成功返回0，失败返回-1。fd：需要关闭的文件或套接字的文件描述符
```

### 3.将数据写入文件

```c
#include <unistd.h>
ssize_t write(int fd, const void * buf, size_t nbytes); //成功返回写入的字节数，失败返回-1
/*
fd: 显示保存数据传输对象的文件描述符。
buf: 保存要传输数据的缓冲地址值。
nbytes: 要传输的字节数。
该函数定义中，size_t通过typedef声明的unsigned int类型。对于ssize_t,size_t前面多加的s代表signed，即ssize_t时通过typedef声明的signed int类型。
*/
```

### 4.读取文件中的数据

```c++
#include <unistd.h>
ssize_t read(int fd, void * buf, size_t nbytes); //成功返回接受的字节数(但是遇到文件结尾则会返回0)，失败返回-1。
/*
fd: 显示数据接收对象的文件描述符fd
buf: 要保存接收数据的缓存地址值
nbytes: 要接收数据的最大字节数
*/
```

