---
title: "Tinyhttp源码分析"
date: "2021-10-01T07:13:58+08:00"
tags:
categories: ["开源项目"]
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


### Tinyhttpd源码解析

Tinyhttpd是J. David Blackstone写的一个轻量型Http Server程序，里面包含了socket编程、多线程编程等知识，值得学习一下。

[Tinyhttp源码Github下载](https://github.com/EZLippi/Tinyhttpd)

`Tinyhttpd`运行流程图如下：

![image.png](https://i.loli.net/2021/10/01/EasTYuCfd8n1ki7.png)

**整体工作过程：**

1.启动服务器，可以指定或随机选取端口进行启动，调用startu函数创建服务端socket。

2.当收到一个http的连接请求时，分配一个线程调用accept_request函数去处理该请求。

3.解析http请求消息来获取请求方法method和请求的url。如果是GET方法则额外保存url中?后面的参数信息

4.将格式化的url存入path字符串中来表示此次请求服务器的文件路径，该tinyhttpd程序的"/"目录是在htdpcs下。如果客户端请求消息是以"/"结尾或者请求的是目录，则在path后加上index.html，表示访问主页。

5.如果文件路径合法，对于无参数的GET请求则直接调用serve_file函数将文件内容写入客户端socket；其他情况（带参数 GET，POST 方式，url 为可执行文件），则调用 excute_cgi 函数执行 cgi 脚本。

6.如果是POST方法，则需要额外获取Content-Length信息

7.fork出子进程和创建两个管道cgi_input和cgi_output给父子进程交互信息

8.子进程执行cgi脚本，并将子进程的标准输入修改重定向cgi_input的读取端，标准输出重定向到cgi_output的写入端；另外对于GET方法则设置request_method的环境变量，对于POST则设置content-length环境变量。请调用execl函数执行cgi脚本程序来替换该子进程。

9.父进程关闭管道cgi_input的读取端和cgi_output的写入端。当请求方式是POST时将从客户端获取的信息通过管道cgi_input[1]向子进程写入，此时是写入到了子进程的标准输入；然后通过cgi_output[0]管道从子进程读取数据发送给客户端，此时是从子进程的标准输出读取的。最后关闭管道，等待子进程结束。



源码阅读顺序： main -> startup -> accept_request -> execute_cgi。



#### 1.Http的GET/POST请求消息格式



![image.png](https://i.loli.net/2021/10/01/Q5dCI4EicpNeuoS.png)

标准GET请求

```http
GET / HTTP/1.1
Host: 192.168.0.1:47310
Connection: keep-alive
...
```



```http
GET / HTTP/1.1\r\nHost: www.sina.com.cn\r\nConnection: close\r\n\r\n
```

标准POST请求​

```http
POST / myhttpd.cgi HTTP / 1.1
Host: 192.168.0.1 : 47310
Connection : keep - alive
Content - Length : 10
...
Form Data
```



#### 2.Main函数



以下是`main`函数代码实现

```c
int main(void)
{
    int server_sock = -1;       //创建服务端套接字
    u_short port = 4000;        //定义端口
    int client_sock = -1;
    struct sockaddr_in client_name;     //客户端地址变量
    socklen_t  client_name_len = sizeof(client_name);
    pthread_t newthread;        //线程变量
    server_sock = startup(&port);       //调用自定义函数startup，初始化服务器段套接字
    printf("httpd running on port %d\n", port);

    /* 一直循环处理新连接的到来，并创建并分配线程去处理该连接 */
    while(1)
    {
        client_sock = accept(server_sock,
                (struct sockaddr *)&client_name,
                &client_name_len);
        if (client_sock == -1) error_die("accept");
        
        /*创建线程，执行线程主函数accept_request(),并将连接套接字传入该函数*/
        if (pthread_create(&newthread , NULL, 
            (void *)accept_request, (void *)(intptr_t)client_sock) != 0)
        {
            perror("pthread_create");
        }
    }
    close(server_sock);
    return(0);
}
```

函数流程如下：

1. 调用startup函数初始化一个服务器socket
2. 进入循环，通过accep函数来受理客户端连接请求
3. 如果接收到新请求在调用pthread_create建立新线程
4. 线程中调用accept_request函数，处理请求

下文会分析startup函数，这里分析下accept函数：

```c++
accept(server_sock, (struct sockaddr *)&client_name, &client_name_len);
```

该函数接受一个套接字、ipv4地址结构变量指针以及对应长度，函数返回是新的连接套接字，CS端通过其通信。

`accept`函数有两个重要特点：

1. 如果没有 `connect` 请求，函数调用会被阻塞，直到接收到 `connect` 请求
2. 在与客户端的套接字建立连接时，`accept` 函数创建一个新的套接字，并用新的套接字与客户端连接，原始的套接字依然处于打开状态，可以用于继续监听端口。



#### 3.套接字初始化函数startup



原始代码如下：

```c
int startup(u_short *port)
{
    int httpd = 0;
    int on = 1;
    struct sockaddr_in name;

    httpd = socket(PF_INET, SOCK_STREAM, 0);
    if (httpd == -1) error_die("socket");
    memset(&name, 0, sizeof(name));
    name.sin_family = AF_INET;
    name.sin_port = htons(*port);
    name.sin_addr.s_addr = htonl(INADDR_ANY);

    /*
   
    */
    /*此处调用setsockopt方法，SO_REUSEADDR参数表示允许重用本地地址和端口*/
    if ((setsockopt(httpd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))) < 0)  
    {  
        error_die("setsockopt failed");
    }
    if (bind(httpd, (struct sockaddr *)&name, sizeof(name)) < 0) error_die("bind");
    if (*port == 0)  /* if dynamically allocating a port */
    {
        socklen_t namelen = sizeof(name);
        /*调用 getsockname()获取系统给 httpd 随机分配的端口号*/
        if (getsockname(httpd, (struct sockaddr *)&name, &namelen) == -1)
            error_die("getsockname");
        *port = ntohs(name.sin_port);
    }
    if (listen(httpd, 5) < 0) error_die("listen");
    return(httpd);
}
```

初始化流程比较简单，大多是固定的代码：

1. 调用 `socket` 函数，创建一个一套接字
2. 创建 `sockaddr_in` 结构体，并设置对应的 ip 和 端口
3. 通过 `bind` 函数，绑定套接字的 ip 和 端口
4. 调用 `listen` 函数，监听请求

其中具体分析写`sockaddr_in`结构体，主要包含了地址和端口信息。

```c
struct in_addr {                /* IPv4 4-byte address */
    in_addr_t s_addr;           /* Unsigned 32-bit integer */
};

struct sockaddr_in {            /* IPv4 socket address */
    sa_family_t    sin_family;  /* Address family (AF_INET) */
    in_port_t      sin_port;    /* Port number */
    struct in_addr sin_addr;    /* IPv4 address */
    unsigned char  __pad[X];    /* Pad to size of 'sockaddr'
};                                 structure (16 bytes) */
```

接下来startup函数调用`setsocket`函数，设置套接字的一些属性，其中SO_REUSEADDR是一个很有用的选项参数。

```c
setsockopt(httpd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on))
```

一般服务器的监听socket都应该打开它。其允许服务器bind一个地址，即使这个地址当前已经存在已建立的连接，比如：

- 服务器启动后，有客户端连接并已建立，如果服务器主动关闭，那么和客户端的连接会处于TIME_WAIT状态，此时再次启动服务器，就会bind不成功，报：Address already in use。
- 服务器父进程监听客户端，当和客户端建立链接后，fork一个子进程专门处理客户端的请求，如果父进程停止，因为子进程还和客户端有连接，所以再次启动父进程，也会报Address already in use。

在 TCP 连接中，当端口收到或者发送 FIN/ACK 请求后，端口并不会立即释放，而是处于 `TIME_WAIT` 状态（该状态一般持续 2 分钟），此时端口是无法与套接字绑定的。设置 `SO_REUSEADDR` 可以让套接字绑定处于 `TIME_WAIT` 状态的端口。具体分析可参考《TCP/IP详解》

最后调用`listen`函数，让套接字进入被动监听状态。

```c++
listen(httpd, 5)		
```

其中第二个参数5代表请求队列的长度。

> 当套接字正在处理客户端请求时，如果有新的请求进来，套接字是没法处理的，只能把它放进缓冲区，待当前请求处理完毕后，再从缓冲区中读取出来处理。如果不断有新的请求进来，它们就按照先后顺序在缓冲区中排队，直到缓冲区满。这个缓冲区，就称为请求队列（Request Queue）。



#### 4.请求消息解析函数accept_request



函数源代码如下：

```c
void accept_request(void *arg)
{
    int client = (intptr_t)arg;
    char buf[1024];         //接收客户端消息的缓冲区
    size_t numchars;
    char method[255];       //获取客户端的请求方式
    char url[255];          //获取客户端请求的URL
    char path[512];         //获取客户端请求的路径
    size_t i, j;
    struct stat st;         //用来存储文件状态的结构体
    int cgi = 0;      		//CGI程序调用标记
                      
    char *query_string = NULL;

    /*http消息中一行，numchars表示这行信息的结尾在buf中的位置*/
    numchars = get_line(client, buf, sizeof(buf));      
    i = 0; j = 0;
    while (!ISspace(buf[i]) && (i < sizeof(method) - 1))
    {
        method[i] = buf[i];         //用method来表示请求方式
        i++;
    }
    j=i;
    method[i] = '\0';

    /*此处判断方法是否支持解析或者方法是否正常，不支持则调用unimplemented函数返回错误方法的响应消息给客户端并退出解析结束线程*/
    if (strcasecmp(method, "GET") && strcasecmp(method, "POST"))        
    {
        unimplemented(client);
        return;
    }

    /*如果是post方法，则将调用cgi函数的标记置为true*/
    if (strcasecmp(method, "POST") == 0) cgi = 1;

    i = 0;
    while (ISspace(buf[j]) && (j < numchars)) j++;
    
    /*继续往后移动j来获取url信息作为字符串*/
    while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < numchars))     
    {
        url[i] = buf[j];
        i++; j++;
    }
    url[i] = '\0';

	/*比较method字符串和"GET"是否相同，相同返回0，大于返回正值，反正返回负值。此处是处理GET方法*/
    if (strcasecmp(method, "GET") == 0)
    {
        /*用query_string来获取?后面的GET请求的参数*/
        query_string = url;                     
        while ((*query_string != '?') && (*query_string != '\0')) query_string++;
        
        /*如果有参数则将cgi标记置为true，此时query_string指向了?后面的GET参数*/
        if (*query_string == '?')       
        {
            cgi = 1;
            *query_string = '\0';
            query_string++;
        }
    }

	/*将字符串"htdoc"与字符串url进行拼接，并保存在path变量中*/
    sprintf(path, "htdocs%s", url);
    
	/*如果此时path是以"/"结尾，则将字符串"index.html"追加到path后面*/
    if (path[strlen(path) - 1] == '/') strcat(path, "index.html");

    /* 调用stat函数，通过path中的文件路径获取文件或文件路径信息，并保存到struct stat st中 */
   
    if (stat(path, &st) == -1)
    {
        /*stat()函数执行失败返回-1，处理并丢弃剩下的请求头并返回错误信息*/
        while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
        not_found(client);
    }
    else
    {
        /*如果文件路径合法，对于无参数的 GET 请求，直接将文件内容写入客户端套接字。其他情况（带参数GET，POST 方式，url 为可执行文件），则调用 excute_cgi 函数执行 cgi 脚本*/
        if ((st.st_mode & S_IFMT) == S_IFDIR) strcat(path, "/index.html");
        if ((st.st_mode & S_IXUSR)||(st.st_mode & S_IXGRP)||(st.st_mode & S_IXOTH))
            cgi=1;
        
   /*简单的GET请求且？后无参数时，不执行cgi函数，而是调用serve_file函数将客户端请求的文件发送给客户端*/
        if (!cgi) serve_file(client, path);
        
   /*执行cgi函数，将连接套接字，文件路径，请求方式和?后的GET参数传入。*/
        else execute_cgi(client, path, method, query_string);
    }
    close(client);
}
```

`accept_request` 函数主要涉及的是对于 request header 的处理，整体流程如下：

1. 提取 request 的类型（GET 或 POST）
2. 提取 url 信息
3. 如果是 GET 请求，则提取 url 中的参数信息（?之后参数）
4. 如果 url 结尾是 / 或者该地址对应一个目录，默认调用该目录下的 index.html
5. 如果不是 CGI，则调用 `serve_file`，将文本内容返回给客户端
6. 如果是 CGI，调用 `execute_cgi`，执行 CGI 脚本程序
7. 断开连接

在对 request header 的处理中，大量调用了 `get_line` 函数。该函数用于读取文件的下一行信息，适用于不同的换行符（`\n` 或 `\r\n`）。



该函数使用的`stat`函数，用于获取文件或文件路径状态信息。

```c
int stat(const char *pathname, struct stat *statbuf);
```

其中`stat`结构体内容信息较多，具体源码如下：

```c
struct stat  
{   
    dev_t       st_dev;     /* ID of device containing file -文件所在设备的ID*/  
    ino_t       st_ino;     /* inode number -inode节点号*/    
    mode_t      st_mode;    /* protection -保护模式?*/    
    nlink_t     st_nlink;   /* number of hard links -链向此文件的连接数(硬连接)*/    
    uid_t       st_uid;     /* user ID of owner -user id*/    
    gid_t       st_gid;     /* group ID of owner - group id*/    
    dev_t       st_rdev;    /* device ID (if special file) -设备号，针对设备文件*/    
    off_t       st_size;    /* total size, in bytes -文件大小，字节为单位*/    
    blksize_t   st_blksize; /* blocksize for filesystem I/O -系统块的大小*/    
    blkcnt_t    st_blocks;  /* number of blocks allocated -文件所占块数*/    
    time_t      st_atime;   /* time of last access -最近存取时间*/    
    time_t      st_mtime;   /* time of last modification -最近修改时间*/    
    time_t      st_ctime;   /* time of last status change - */    
}; 
```



#### 5.CGI执行函数execute_cgi


函数源代码如下：

```c
/*参数为连接套接字，文件路径，请求方式和?后的GET参数*/
void execute_cgi(int client, const char *path,
        const char *method, const char *query_string)       
{
    char buf[1024];
    int cgi_output[2];
    int cgi_input[2];
    pid_t pid;
    int status;
    int i;
    char c;
    int numchars = 1;
    int content_length = -1;

    buf[0] = 'A'; buf[1] = '\0';
    
	/*如果是GET请求，无须处理下面的信息，所以继续执行处理并丢弃剩下的请求头信息*/
    if (strcasecmp(method, "GET") == 0)     
    {    while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
    }
    else if (strcasecmp(method, "POST") == 0) /*POST方法*/
    {
        numchars = get_line(client, buf, sizeof(buf));      //获取下一行信息
        while ((numchars > 0) && strcmp("\n", buf))
        {
            /*如果是POST请求，就需要得到 Content-Length,content-Length 字符串长度为15从 17 位开				  始是具体的长度信息*/
            buf[15] = '\0';
            if (strcasecmp(buf, "Content-Length:") == 0)
            {
                content_length = atoi(&(buf[16]));
            }
            numchars = get_line(client, buf, sizeof(buf));
        }
        if (content_length == -1)
        {
            bad_request(client);
            return;
        }
    }
    else/*HEAD or other*/
    {
    }

	/*pipe系统调用建立管道*/
    if (pipe(cgi_output) < 0) {
        cannot_execute(client);         //调用cannot_execute函数通知客户端无法执行CGI脚本
        return;
    }
    if (pipe(cgi_input) < 0) {
        cannot_execute(client);
        return;
    }

    if ( (pid = fork()) < 0 ) {         //创建子进程执行CGI脚本
        cannot_execute(client);
        return;
    }
    sprintf(buf, "HTTP/1.0 200 OK\r\n");        //发送200OK状态码给客户端
    send(client, buf, strlen(buf), 0);
    if (pid == 0)  /* child: CGI script */
    {
        char meth_env[255];
        char query_env[255];
        char length_env[255];

        /*复制cgi_output[1]文件描述符到标准输出，此时标准输出被关闭*/
        dup2(cgi_output[1], STDOUT);   
        /*复制cgi_input[0]文件描述符到标准输入，此时标准输入被关闭*/
        dup2(cgi_input[0], STDIN);       
        close(cgi_output[0]);
        close(cgi_input[1]);
        
        /*将拼接字符串，保存到meth_env中*/
        sprintf(meth_env, "REQUEST_METHOD=%s", method);
        
        /*调用putenv函数将meth_env添加到环境变量中*/
        putenv(meth_env);

        /*如果是GET请求，将？后面的参数也加入环境变量*/
        if (strcasecmp(method, "GET") == 0)
        {
            sprintf(query_env, "QUERY_STRING=%s", query_string);
            putenv(query_env);
        }
		/*如果是POST方法，将content-length加入环境变量*/
        else {   /* POST */
            sprintf(length_env, "CONTENT_LENGTH=%d", content_length);
            putenv(length_env);
        }
        
        /*exec函数族启动一个新程序，替换原有的进程。因此这个新的被exec执行的进程的PID不会改变，和调用exec函数的进程一样*/
        execl(path, NULL);                                  
        exit(0);
    }

    /*子进程的标准输入输出和cgi_input和cgi_output绑定了，父进程可以通过 cgi_input 和 cgi_output 获取子进程中cgi脚本的标准输入和标准输出*/
    else {
        close(cgi_output[1]);
        close(cgi_input[0]);
        if (strcasecmp(method, "POST") == 0)
            /*根据Content-Length读取客户端信息，并通过cgi_inputp[1]管道传入子进程的标准输入*/
            for (i = 0; i < content_length; i++) {
                recv(client, &c, 1, 0);
                write(cgi_input[1], &c, 1);
            }
        /*通过cgi_output管道获取子进程标准输出，并写入到客户端*/
        while (read(cgi_output[0], &c, 1) > 0) send(client, &c, 1, 0);
        
        close(cgi_output[0]);
        close(cgi_input[1]);
        waitpid(pid, &status, 0);
    }
}
```

函数的整体流程如下：

1. 对 POST 请求，根据 Content-Length 提取 body 中的信息
2. 创建两个管道 cgi_input 和 cgi_output 用于进程间通信
3. 调用 `fork` 建立子进程
4. 子进程调用 `dup2` 将标准输入与标准输出分别重定向到对应管道的读端和写端
5. 在子进程中设置环境变量，并调用 `execl`，执行 CGI 脚本
6. 父进程通过管道向 CGI 脚本传入参数，并获取脚本的返回结果，再将结果传给客户端
7. 父进程等待子结束

这里的重点是父子进程利用管道实现IPC。

```c
int pipe(int filedes[2]);
```

调用 `pipe` 函数，得到两个文件描述符，分别对应管道的读端 `filedes[0]` 和写端 `filedes[1]`，当程序在写端写入数据时，在读端可以读取到写入的数据。接着，通过 `fork` 函数，得到一个子进程。由于子进程与父进程拥有完全相同的变量，因此子进程也有对应管道读端和写端的两个文件描述符。之后，只需要关闭一侧的读端和另一侧的写端，就可以实现进程间的通信。

![image.png](https://i.loli.net/2021/10/01/9G2ZODXWJiEAM4F.png)

接下来调用`dup2`系统调用，对子进程的标准输入输出进行重定向。

```c
int dup2(int oldfd, int newfd);  //dup2可以用参数newfd指定新文件描述符的数值。
```

若参数newfd已经被程序使用，则系统就会将newfd所指的文件关闭，若newfd等于oldfd，则返回newfd,而不关闭newfd所指的文件。dup2所复制的文件描述符与原来的文件描述符共享各种文件状态、共享所有的锁定、读写位置和各项权限或flags等

在这里实现了以下功能：

- 子进程的标准输出将会写入到 `cgi_output` 的写端
- `cgi_input` 读端读取的数据将会作为子进程的标准输入

在子进程完成标准输入和标准输出的重定向之后，调用 `execl` 函数，执行 CGI 脚本。

```c
execl(path, NULL);
```

该函数会让进程加载新的程序，之前的程序包括缓存的数据都会被丢弃掉。此时，子进程就是 CGI 脚本的执行程序。

父进程只需要做两件事：
- 调用 `recv` 函数，从客户端中接收数据，并将数据通过 `cgi_input[1]` 写入传入 CGI 脚本
- 从 `cgi_output[0]` 中读取 CGI 脚本的返回结果，并调用 `send` 函数，将结果发送给客户端



该Tinyhttpd的源码主体已经解析完毕，当然还有获取一行信息的get_line()、从文件读入内容的cat()等函数，逻辑比较简单，读一遍大致就理解了，这里就不加以赘述。
