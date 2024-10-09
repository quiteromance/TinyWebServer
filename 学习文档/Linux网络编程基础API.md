# Linux网络编程基础API

## 目录

- [Linux网络编程基础API](#linux网络编程基础api)
  - [目录](#目录)
  - [1 socket地址API](#1-socket地址api)
    - [1.1 通用socket地址](#11-通用socket地址)
    - [1.2 专用socket地址](#12-专用socket地址)
    - [1.3 IP地址转换函数](#13-ip地址转换函数)
  - [2 创建socket](#2-创建socket)
  - [3 命名socket](#3-命名socket)
  - [4 监听socket](#4-监听socket)
  - [5 接收连接](#5-接收连接)
  - [6 发起连接](#6-发起连接)
  - [7 关闭连接](#7-关闭连接)
  - [8 数据读写(TCP读写)](#8-数据读写tcp读写)
  - [9 地址信息函数](#9-地址信息函数)
  - [10 网络信息API](#10-网络信息api)

## 1 socket地址API

最开始的含义是一个IP地址和端口(ip,port),其唯一的表示使用TCP通信的一端。  
由于不同的机器都由自己的主机字节序(大端:高位放低地址,小端:高位放高地址),所以为了统一数据在不同字节序的主机上接收,需要将发送的数据全部转换成**大端字节序**,即**网络字节序**。

```cpp
//Linux提供了四个函数转换字节序

#include<netinet/in.h>
unsigned long int htonl(unsigned long int hostlong);
unsigned short int htons(unsigned short int hostshort);
unsigned long int ntohl(unsigned long int netlong);
unsigned short int ntohs(unsigned short int netshort);
```

`htonl`表示`host to network long`，即将长整型（32 bit）的主机字节序数据转化为网络字节序数据。

### 1.1 通用socket地址

socket网络编程接口中表示socket地址的是结构体`sockaddr`，其定义如下：

```cpp
#include<bits/socket.h>
struct sockaddr
{
    sa_family_t sa_family;
    char sa_data[14];
}
```

- sa_family成员是地址族类型（sa_family_t）的变量.地址族类型通常与协议族类型对应。常见的协议族(protocol family，也称domain)。
- sa_data成员用于存放socket地址值。但是，不同的协议族的地址值具有不同的含义和长度。
- 14字节的sa_data根本无法完全容纳多数协议族的地址值。因此，Linux定义了下面这个新的通用socket地址结构体：

```cpp
#include<bits/socket.h>
struct sockaddr_storage
{
sa_family_t sa_family;
unsigned long int__ss_align;
char__ss_padding[128-sizeof(__ss_align)];
}
```

这个结构体不仅提供了足够大的空间用于存放地址值，而且是内
存对齐的（这是__ss_align成员的作用）。

### 1.2 专用socket地址

上面两个通用的都不好用,比如设置获取IP地址和port都需要执行位操作。Linux为各个协议族提供了专门的socket地址结构体。

- UNIX本地域协议族使用如下专用socket地址结构体:

```cpp
#include＜sys/un.h＞
struct sockaddr_un
{
    sa_family_t sin_family;/*地址族：AF_UNIX*/
    char sun_path[108];/*文件路径名*/
};
```

- TCP/IP协议族有sockaddr_in和sockaddr_in6两个专用socket地址结构体，它们分别用于IPv4和IPv6：

```cpp
struct sockaddr_in
{
sa_family_t sin_family;/*地址族：AF_INET*/
u_int16_t sin_port;/*端口号，要用网络字节序表示*/
struct in_addr sin_addr;/*IPv4地址结构体，见下面*/
};

struct in_addr
{
u_int32_t s_addr;/*IPv4地址，要用网络字节序表示*/
};

struct sockaddr_in6
{
sa_family_t sin6_family;/*地址族：AF_INET6*/
u_int16_t sin6_port;/*端口号，要用网络字节序表示*/
u_int32_t sin6_flowinfo;/*流信息，应设置为0*/
struct in6_addr sin6_addr;/*IPv6地址结构体，见下面*/
u_int32_t sin6_scope_id;/*scope ID，尚处于实验阶段*/
};

struct in6_addr
{
unsigned char sa_addr[16];/*IPv6地址，要用网络字节序表示*/
};
```

- **所有专用socket地址（以及sockaddr_storage）类型的变量在实际使用时都需要转化为通用socket地址类型sockaddr（强制转换即可），因为所有socket编程接口使用的地址参数的类型都是sockaddr。**

### 1.3 IP地址转换函数

下面3个函数可用于用点分十进制字符串表示的IPv4地址和用网络字节序
整数表示的IPv4地址之间的转换：

```cpp
#include<arpa/inet.h
in_addr_t inet_addr(const char*strptr);
int inet_aton(const char*cp,struct in_addr*inp);
char*inet_ntoa(struct in_addr in);
```

- inet_addr函数将用**点分十进制字符串**表示的IPv4地址转化为用**网络字节序整数**表示的IPv4地址。它失败时返回INADDR_NONE。
- inet_aton函数完成和inet_addr同样的功能，但是将转化结果存储于参数inp指向的地址结构中。它成功时返回1，失败则返回0。
- inet_ntoa函数将用网络字节序整数表示的IPv4地址转化为用点分十进制字符串表示的IPv4地址。但需要注意的是，该函数内部用一个静态变量存储转化结果，函数的返回值指向该静态内存，因此inet_ntoa是不可重入的。

下面这对更新的函数也能完成和前面3个函数同样的功能，并且它们同时适用于IPv4地址和IPv6地址：

```cpp
#include<arpa/inet.h>
int inet_pton(int af,const char*src,void*dst);
const char*inet_ntop(int af,const void*src,char*dst,socklen_t cnt);
```

- inet_pton函数将用字符串表示的IP地址src（用点分十进制字符串表示的IPv4地址或用十六进制字符串表示的IPv6地址）转换成用网络字节序整数表示的IP地址，并把转换结果存储于dst指向的内存中。其中，af参数指定地址族，可以是AF_INET或者AF_INET6。inet_pton成功时返回1，失败则返回0并设置errno[1]。
- inet_ntop函数进行相反的转换，前三个参数的含义与inet_pton的参数相同，最后一个参数cnt指定目标存储单元的大小。

## 2 创建socket

UNIX/Linux的一个哲学是：所有东西都是文件。socket也不例外，**它就是可读、可写、可控制、可关闭的文件描述符**。下面的socket系统调用可创建一个socket：

```cpp
#include<sys/types.h>
#include<sys/socket.h>
int socket(int domain,int type,int protocol);
```

- **domain参数告诉系统使用哪个底层协议族**。对TCP/IP协议族而言，该参数应该设置为PF_INET（Protocol Family of Internet，用于IPv4）或PF_INET6（用于IPv6）；对于UNIX本地域协议族而言，该参数应该设置为PF_UNIX。
- **type参数指定服务类型**。服务类型主要有SOCK_STREAM服务（流服务）和SOCK_UGRAM（数据报）服务。对TCP/IP协议族而言，**其值取SOCK_STREAM表示传输层使用TCP协议，取SOCK_DGRAM表示传输层使用UDP协议**。
- **protocol参数是在前两个参数构成的协议集合下，再选择一个具体的协议**。不过这个值通常都是唯一的（前两个参数已经完全决定了它的值）。**几乎在所有情况下，我们都应该把它设置为0，表示使用默认协议**。
- socket系统调用成功时返回一个socket文件描述符，失败则返回-1并设置errno。

## 3 命名socket

- 创建socket时，我们给它指定了地址族，但是并未指定使用该地址族中的哪个具体socket地址。**将一个socket与socket地址绑定称为给socket命名**。
- 在服务器程序中，我们通常要命名socket，因为只有命名后客户端才能知道该如何连接它。
- 客户端则通常不需要命名socket，而是采用匿名方式，即使用操作系统自动分配的socket地址。

命名socket的系统调用是bind，其定义如下：

```cpp
#include<sys/types.h>
#include<sys/socket.h>
int bind(int sockfd,const struct sockaddr*my_addr,socklen_t addrlen);
```

- bind将my_addr所指的socket地址分配给未命名的sockfd文件描述符，addrlen参数指出该socket地址的长度。
- bind成功时返回0，失败则返回-1并设置errno。其中两种常见的errno是EACCES和EADDRINUSE，它们的含义分别是：
  
  - **EACCES，被绑定的地址是受保护的地址，仅超级用户能够访问**。比如普通用户将socket绑定到知名服务端口（端口号为0～1023）上时，bind将返回EACCES错误。
  - **EADDRINUSE，被绑定的地址正在使用中**。比如将socket绑定到一个处于TIME_WAIT状态的socket地址句号

## 4 监听socket

socket被命名之后，还不能马上接受客户连接，我们需要使用如下系统调用来创建一个监听队列以存放待处理的客户连接：

```cpp
#include<sys/socket.h>
int listen(int sockfd,int backlog);
```

- sockfd参数指定被监听的socket。
- backlog参数提示内核监听队列的最大长度(参数指定了这个队列中最多能有多少个等待处理的连接请求)。监听队列的长度如果超过backlog，服务器将不受理新的客户连接，客户端也将收到ECONNREFUSED错误信息。
- listen成功时返回0，失败则返回-1并设置errno。

```cpp
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<signal.h>
#include<unistd.h>
#include<stdlib.h>
#include<assert.h>
#include<stdio.h>
#include<string.h>

static bool stop = false;
/*SIGTERM信号的处理函数，触发时结束主程序中的循环*/
static void handle_term(int sig)
{
    stop = true;
}

int main(int argc,char*argv[])
{
    signal(SIGTERM,handle_term);
    if(argc <= 3)
    {
        printf("usage:%s ip_address port_number backlog\n",basename(argv[0]));
        return 1;
    }

    const char*ip = argv[1];
    int port = atoi(argv[2]);
    int backlog = atoi(argv[3]);
    int sock = socket(PF_INET,SOCK_STREAM,0); // TCP/IP 创建socket
    assert(sock >= 0);

    /*创建一个IPv4 socket地址*/
    struct sockaddr_in address;
    bzero(&address,sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET,ip,&address.sin_addr);
    address.sin_port = htons(port);
    int ret = bind(sock,(struct sockaddr*)&address,sizeof(address)); // 命名socket
    assert(ret!=-1);
    ret=listen(sock,backlog);
    assert(ret!=-1);
    /*循环等待连接，直到有SIGTERM信号将它中断*/
    while(!stop)
    {
        sleep(1);
    }
    /*关闭socket，见后文*/
    close(sock);
    return 0;
}
```

## 5 接收连接

下面的系统调用从listen监听队列中接受一个连接：

```cpp
#include<sys/types.h>
#include<sys/socket.h>
int accept(int sockfd,struct sockaddr*addr,socklen_t*addrlen);
```

- sockfd参数是执行过listen系统调用的监听socket[1]。
- addr参数用来获取被接受连接的远端socket地址，该socket地址的长度由addrlen参数指出。
- accept成功时返回一个新的连接socket，该socket唯一地标识了被接受的这个连接，服务器可通过读写该socket来与被接受连接对应的客户端通信。accept失败时返回-1并设置errno。

```cpp
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<errno.h>
#include<string.h>
int main(int argc,char*argv[])
{
    if(argc <= 2)
    {
        printf("usage:%s ip_address port_number\n",basename(argv[0]));
        return 1;
    }
    const char*ip=argv[1];
    int port=atoi(argv[2]);
    struct sockaddr_in address;
    bzero(&address,sizeof(address));
    address.sin_family=AF_INET;
    inet_pton(AF_INET,ip,&address.sin_addr);
    address.sin_port=htons(port);
    int sock=socket(PF_INET,SOCK_STREAM,0);
    assert(sock>=0);
    int ret=bind(sock,(struct sockaddr*)&address,sizeof(address));
    assert(ret!=-1);
    ret=listen(sock,5);
    assert(ret!=-1);
    /*暂停20秒以等待客户端连接和相关操作（掉线或者退出）完成*/
    sleep(20);
    struct sockaddr_in client;
    socklen_t client_addrlength=sizeof(client);
    int connfd=accept(sock,(struct sockaddr*)&client,&client_addrlength);
    if(connfd<0)
    {
    printf("errno is:%d\n",errno);
    }
    else
    {
    /*接受连接成功则打印出客户端的IP地址和端口号*/
    char remote[INET_ADDRSTRLEN];
    printf("connected with ip:%s and port:%d\n",inet_ntop(AF_INET,&client.sin_addr,remote,INET_ADDRSTRLEN),ntohs(client.sin_port));
    close(connfd);
    }
    close(sock);
    return 0;
}
```

- accept只是从监听队列中取出连接，而不论连接处于何种状态（如上面的ESTABLISHED状态和CLOSE_WAIT状态），更不关心任何网络状况的变化。

## 6 发起连接

如果说服务器通过listen调用来被动接受连接，那么客户端需要通过如下系统调用来主动与服务器建立连接：

```cpp
#include<sys/types.h>
#include<sys/socket.h>
int connect(int sockfd,const struct sockaddr*serv_addr,socklen_t addrlen);
```

- sockfd参数由socket系统调用返回一个socket。
- serv_addr参数是服务器监听的socket地址，addrlen参数则指定这个地址的长度。
- connect成功时返回0。一旦成功建立连接，**sockfd就唯一地标识了这个连接**，客户端就可以通过读写sockfd来与服务器通信。connect失败则返回-1并设置errno。其中两种常见的errno是ECONNREFUSED和ETIMEDOUT，它们的含义如下：

  - ❑ECONNREFUSED，目标端口不存在，连接被拒绝。
  - ❑ETIMEDOUT，连接超时。

```cpp
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

#define SERVER_IP "192.168.1.109"  // 服务器 IP 地址
#define SERVER_PORT 12345           // 服务器端口

int main() {
    int sockfd;
    struct sockaddr_in server_addr;

    // 创建 socket
    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    if (sockfd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    // 设置服务器地址结构
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    
    // 将 IP 地址从文本转换为二进制
    if (inet_pton(AF_INET, SERVER_IP, &server_addr.sin_addr) <= 0) {
        perror("inet_pton");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    // 发起连接
    if (connect(sockfd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("connect");
        close(sockfd);
        exit(EXIT_FAILURE);
    }

    printf("Connected to %s:%d\n", SERVER_IP, SERVER_PORT);

    // 可以在这里发送和接收数据
    // ...

    // 关闭 socket
    close(sockfd);
    return 0;
}
```

## 7 关闭连接

关闭一个连接实际上就是关闭该连接对应的socket，这可以通过如下关闭普通文件描述符的系统调用来完成：

```cpp
#include<unistd.h>
int close(int fd);
```

- fd参数是待关闭的socket。不过，close系统调用并非总是立即关闭一个连接，而是将fd的引用计数减1。只有当fd的引用计数为0时，才真正关闭连接。
- 多进程程序中，一次fork系统调用默认将使父进程中打开的socket的引用计数加1，因此我们必须在父进程和子进程中都对该socket执行close调用才能将连接关闭。

如果无论如何都要立即终止连接（而不是将socket的引用计数减1），可以使用如下的shutdown系统调用（相对于close来说，它是专门为网络编程设计的）：

```cpp
#include<sys/socket.h>
int shutdown(int sockfd,int howto);
```

sockfd参数是待关闭的socket。howto参数决定了shutdown的行为:

- SHUT_RD：应用程序不能对socket文件描述符执行读操作，并且socket的接收缓冲区数据全部被抛弃
- SHUT_WR：socket的发送缓冲区数据全部发送后，应用程序不能对socket文件描述符执行写操作
- SHUT_RDWR：同时关闭读写

由此可见，shutdown能够分别关闭socket上的读或写，或者都关闭。而close在关闭连接时只能将socket上的读和写同时关闭。

## 8 数据读写(TCP读写)

对文件的读写操作read和write同样适用于socket。但是socket编程接口提供了几个专门用于socket数据读写的系统调用，它们增加了对数据读写的控制。其中用于TCP流数据读写的系统调用是：

```cpp
#include<sys/types.h>
#include<sys/socket.h>
ssize_t recv(int sockfd,void*buf,size_t len,int flags);
ssize_t send(int sockfd,const void*buf,size_t len,int flags);
```

- recv读取sockfd上的数据
  
  - buf和len参数分别指定读缓冲区的位置和大小。
  - flags参数的含义见后文，通常设置为0即可。
  - recv成功时返回实际读取到的数据的长度，它可能小于我们期望的长度len。因此我们可能要多次调用recv，才能读取到完整的数据。recv可能返回0，这意味着通信对方已经关闭连接了。recv出错时返回-1并设置errno。

- send往sockfd上写入数据

  - buf和len参数分别指定写缓冲区的位置和大小。
  - send成功时返回实际写入的数据的长度，失败则返回-1并设置errno。

```cpp
// 服务器接收数据
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<stdlib.h>
#include<errno.h>
#include<string.h>
#define BUF_SIZE 1024
int main(int argc,char*argv[])
{
if(argc<=2)
{
printf("usage:%s ip_address port_number\n",basename(argv[0]));
return 1;
}
const char*ip=argv[1];
int port=atoi(argv[2]);
struct sockaddr_in address;
bzero(&address,sizeof(address));
address.sin_family=AF_INET;
inet_pton(AF_INET,ip,&address.sin_addr);
address.sin_port=htons(port);
int sock=socket(PF_INET,SOCK_STREAM,0);
assert(sock>=0);
int ret=bind(sock,(struct sockaddr*)&address,sizeof(address));
assert(ret!=-1);
ret=listen(sock,5);
assert(ret!=-1);
struct sockaddr_in client;
socklen_t client_addrlength=sizeof(client);
int connfd=accept(sock,(struct sockaddr*)&client,&client_addrlength);
if(connfd<0)
{
printf("errno is:%d\n",errno);
}
else
{
char buffer[BUF_SIZE];
memset(buffer,'\0',BUF_SIZE);
ret=recv(connfd,buffer,BUF_SIZE-1,0);
printf("got%d bytes of normal data'%s'\n",ret,buffer);
memset(buffer,'\0',BUF_SIZE);
ret=recv(connfd,buffer,BUF_SIZE-1,MSG_OOB);
printf("got%d bytes of oob data'%s'\n",ret,buffer);
memset(buffer,'\0',BUF_SIZE);
ret=recv(connfd,buffer,BUF_SIZE-1,0);
printf("got%d bytes of normal data'%s'\n",ret,buffer);
close(connfd);
}
close(sock);
return 0;
}
```

```cpp
// 客户端发送数据
#include<sys/socket.h>
#include<netinet/in.h>
#include<arpa/inet.h>
#include<assert.h>
#include<stdio.h>
#include<unistd.h>
#include<string.h>
#include<stdlib.h>
int main(int argc,char*argv[])
{
if(argc<=2)
{
printf("usage:%s ip_address port_number\n",basename(argv[0]));
return 1;
}
const char*ip=argv[1];
int port=atoi(argv[2]);
struct sockaddr_in server_address;
bzero(&server_address,sizeof(server_address));
server_address.sin_family=AF_INET;
inet_pton(AF_INET,ip,&server_address.sin_addr);
server_address.sin_port=htons(port);
int sockfd=socket(PF_INET,SOCK_STREAM,0);
assert(sockfd>=0);
if(connect(sockfd,(struct sockaddr*)&
server_address,sizeof(server_address))<0)
{
printf("connection failed\n");
}
else
{
const char*oob_data="abc";
const char*normal_data="123";
send(sockfd,normal_data,strlen(normal_data),0);
send(sockfd,oob_data,strlen(oob_data),MSG_OOB);
send(sockfd,normal_data,strlen(normal_data),0);
}
close(sockfd);
return 0;
}

```

## 9 地址信息函数

在某些情况下，我们想知道一个连接socket的本端socket地址，以及远端的socket地址。下面这两个函数正是用于解决这个问题：

```cpp
#include<sys/socket.h>
int getsockname(int sockfd,structsockaddr*address,socklen_t*address_len);
int getpeername(int sockfd,structsockaddr*address,socklen_t*address_len);
```

- getsockname获取sockfd对应的本端socket地址，并将其存储于address参数指定的内存中，该socket地址的长度则存储于address_len参数指向的变量中。如果实际socket地址的长度大于address所指内存区的大小，那么该socket地址将被截断。getsockname成功时返回0，失败返回-1并设置errno。
- getpeername获取sockfd对应的远端socket地址，其参数及返回值的含义与getsockname的参数及返回值相同。

## 10 网络信息API

