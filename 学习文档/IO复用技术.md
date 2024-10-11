# I/O复用（并发）

由于程序执行的accept，读写函数等都是阻塞的，所以需要使用I/O复用技术（I/O并发）来这个问题。I/O复用使得程序能同时监听多个文件描述符（主要通过系统调用的方式），这对提高程序的性能至关重要。通常，网络程序在下列情况下需要使用I/O复用技术：

- 客户端程序要同时处理多个socket。比如非阻塞connect技术。
- 客户端程序要同时处理用户输入和网络连接。比如聊天室程序。
- TCP服务器要同时处理监听socket和连接socket。这是I/O复用使用最多的场合。
- 服务器要同时处理TCP请求和UDP请求。比如回射服务器。
- 服务器要同时监听多个端口，或者处理多种服务。比如xinetd服务器。

I/O复用虽然能同时监听多个文件描述符，但它**本身是阻塞**的。并且当多个文件描述符同时就绪时，如果不采取额外的措施，程序就只能按顺序依次处理其中的每一个文件描述符，这使得服务器程序看起来像是串行工作的。如果要实现并发，只能使用多进程或多线程等编程手段。

- Linux下实现I/O复用的系统调用主要有**select、poll和epoll**。

## 目录

- [I/O复用（并发）](#io复用并发)
  - [目录](#目录)
  - [1 select系统调用](#1-select系统调用)
    - [select并发处理](#select并发处理)
    - [高效率的多线程并发select服务器](#高效率的多线程并发select服务器)
  - [2 poll系统调用](#2-poll系统调用)

## 1 select系统调用

select系统调用的用途是：在一段指定时间内，监听用户感兴趣的文件描述符上的可读、可写和异常等事件。select系统调用的原型如下：

```cpp
#include<sys/select.h>
int select(int nfds,fd_set*readfds,fd_set*writefds,fd_set*exceptfds,struct timeval*timeout);
```

- nfds参数指定被监听的文件描述符的总数。它通常被设置为select监听的所有文件描述符中的最大值加1，因为文件描述符是从0开始计数的。select监听的就是后面的读集合缓冲区，写集合缓冲区和异常事件最大集合+1。（原因：由于select是线性表结构，指定最大总数方便知道程序什么时候结束）
- readfds、writefds和exceptfds参数分别指向可读、可写和异常等事件对应的文件描述符集合。应用程序调用select函数时，通过这3个参数传入自己感兴趣的文件描述符。select调用返回时，内核将修改它们来通知应用程序哪些文件描述符已经就绪。对于读函数，读缓冲区有数据即为就绪，对于写函数，写缓冲区可写即为就绪，异常事件有即为就绪。
- 这3个参数是fd_set结构指针类型。fd_set结构定义：

```cpp
#include<typesizes.h>
#define __FD_SETSIZE 1024
#include<sys/select.h>
#define FD_SETSIZE __FD_SETSIZE
typedef long int __fd_mask;
#undef __NFDBITS
#define __NFDBITS (8*(int)sizeof(__fd_mask))
typedef struct
{
#ifdef __USE_XOPEN
__fd_mask fds_bits[__FD_SETSIZE/__NFD BITS];
#define __FDS_BITS(set) ((set)-＞fds_bits)
#else
__fd_mask__fds_bits[__FD_SETSIZE/__NFDBITS];
#define__FDS_BITS(set)((set)-＞__fds_bits)
#endif
}fd_set;
```

由以上定义可见，fd_set结构体仅包含一个整型数组，该数组的每个元素的每一位（bit）标记一个文件描述符。fd_set能容纳的文件描述符数量由FD_SETSIZE指定，这就限制了select能同时处理的文件描述符的总量。由于位操作过于烦琐，我们应该使用下面的一系列宏来访问fd_set结构体中的位：

```cpp
#include<sys/select.h>
FD_ZERO(fd_set*fdset);/*清除fdset的所有位*/
FD_SET(int fd,fd_set*fdset);/*设置fdset的位fd*/
FD_CLR(int fd,fd_set*fdset);/*清除fdset的位fd*/
int FD_ISSET(int fd,fd_set*fdset);/*测试fdset的位fd是否被设置*/
```

- timeout参数用来设置select函数的超时时间。它是一个timeval结构类型的指针，采用指针参数是因为内核将修改它以告诉应用程序select等待了多久。不过我们不能完全信任select调用返回后的timeout值，比如调用失败时timeout值是不确定的

```cpp
struct timeval
{
long tv_sec;/*秒数*/
long tv_usec;/*微秒数*/
};
```

如果给timeout变量的tv_sec成员和tv_usec成员都传递0，则select将立即返回。如果给timeout传递NULL，则select将一直阻塞，直到某个文件描述符就绪。

- select成功时返回就绪（可读、可写和异常）文件描述符的总数。
- 如果在超时时间内没有任何文件描述符就绪，select将返回0。
- select失败时返回-1并设置errno。如果在select等待期间，程序接收到信号，则select立即返回-1，并设置errno为EINTR。

---

在网络编程中，下列socket可读：

- socket内核接收缓存区中的字节数大于或等于其低水位标记SO_RCVLOWAT。此时我们可以无阻塞地读该socket，并且读操作返回的字节数大于0。
- socket通信的对方关闭连接。此时对该socket的读操作将返回0。
- 监听socket上有新的连接请求。
- socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。

下列情况下socket可写：

- socket内核发送缓存区中的可用字节数大于或等于其低水位标记SO_SNDLOWAT。此时我们可以无阻塞地写该socket，并且写操作返回的字节数大于0。
- socket的写操作被关闭。对写操作被关闭的socket执行写操作将触发一个SIGPIPE信号。
- socket使用非阻塞connect连接成功或者失败（超时）之后。
- socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。

网络程序中，select能处理的异常情况只有一种：socket上接收到带外数据。

---

### select并发处理

- 1.创建监听套接字lfd = socket()
- 2.将监听的套接字和本地的IP和端口绑定bind()
- 3.给监听的socket设置listen()
- 4.创建文件描述符集合fd_set，用于存储需要检测读时间的所有文件描述符
  
  - 通过FD_ZERO()初始化
  - 通过FD_SET()将监听的文件描述符放入检测的读集合中

- 5.循环调用select()，周期性的检测文件描述符
- 6.select()接触阻塞返回，得到内核传出的满足条件的就绪的文件描述符集合

  - 通过FD_ISSET()判断集合中的标志位是否为1
  
    - 如果这个文件描述符是监听的文件描述符，调用accept()和客户端建立连接
  
      - 将得到的新的通信文件描述符，通过FD_SET()放入到检测集合中
    - 如果这个文件描述符是通信的文件描述符，调用通信函数和客户端通信

      - 如果服务器客户端断开连接，使用FD_CLR()将这个文件描述符从检测集合删除
      - 如果没有，正常通信即可
- 7.重复第六步

**并发服务器程序**：

```c
#include <stdio.h>
#include <ctype.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>

// server
int main(){
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sinport = htons(9999);
    serv_addr.sin_addr_s.addr = htonl(INADDR_ANY); // 本地多有的IP

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1){
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1){
        perror("listen error");
        exit(1);
    }

    fd_set redset;
    FD_ZERO(&redset);
    FD_SET(lfd, &redset);

    int maxfd = lfd;

    while(1){
        fd_set tmp = redset;
        int ret = select(maxfd+1, &tmp, NULL, NULL, NULL);
        // 判断是不是监听的fd
        if(FD_ISSET(lfd,&tmp)){
            int cfd = accept(lfd, NULL, NULL);
            FD_SET(cfd, &redset);
            maxfd = cfd > maxfd? cfd : maxfd;
        }

        // 判断是不是通信的描述符
        for(int i = 0; i <= maxfd; i++){
            if(i != lfd && FD_ISSET(i, &tmp)){
                // 接收数据
                char buf[1024];
                int len = recv(cfd, buf, sizeof(buf), 0);
                if(len == -1){
                    perror("recv error");
                    exit(1);
                }
                else if(len == 0){
                    printf("客户端已经断开连接 ... ...\n");
                    FD_CLR(i, &redset);
                    close(i);
                    break;
                }
                printf("read buf = %s\n", buf);

                // 传发给客户端
                ret = send(i, buf, strlen(buf)+1, 0);
                if(ret == -1){
                    perror("send error");
                    exit(1);
                }
            }
        }
    }
    close(lfd);
    return 0;

}
```

### 高效率的多线程并发select服务器

```c
#include <stdio.h>
#include <ctype.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/stat.h>
#include <string.h>
#include <arpa/inet.h>
#include <pthread.h> /* 添加线程头文件 */

pthread_mutex_t mutex; // 添加互斥锁

typedef struct fdinfo{
    int fd;
    int *maxfd;
    fd_set* rdset;
}FDInfo;

void* acceptConn(void *arg){
    printf("子线程ID：%ld\n", pthread_self());
    FDInfo *info = (FDInfo*)arg;
    int cfd = accept(info->lfd, NULL, NULL);

    pthread_mutex_lock(&mutex);
    FD_SET(cfd, info->rdset);
    *info->maxfd = cfd > *info->maxfd? cfd : *info->maxfd;
    pthread_mutex_unlock(&mutex);

    free(info);

    return NULL;
}

void* communication(void* arg){
    printf("子线程ID：%ld\n", pthread_self());
    char buf[1024];
    FDInfo *info = (FDInfo*)arg;

    int len = recv(info->fd, buf, sizeof(buf), 0);
    if(len == -1){
        perror("recv error");
        free(info);
        return NULL;
    }
    else if(len == 0){
        printf("客户端已经断开连接 ... ...\n");
        pthread_mutex_lock(&mutex);
        FD_CLR(info->fd, info->rdset);
        pthread_mutex_unlock(&mutex);
        close(info->fd);
        free(info);
        return NULL; // 如果断开连接，直接退出子线程
    }
    printf("read buf = %s\n", buf);

    // 传发给客户端
    ret = send(info->fd, buf, strlen(buf)+1, 0);
    if(ret == -1){
        perror("send error");
    }
    free(info);
    return NULL;
}
// server
int main(){
    pthread_mutex_init(&mutex, NULL);
    int lfd = socket(AF_INET, SOCK_STREAM, 0);
    if(lfd == -1)
    {
        perror("socket error");
        exit(1);
    }

    // 绑定
    struct sockaddr_in serv_addr;
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sinport = htons(9999);
    serv_addr.sin_addr_s.addr = htonl(INADDR_ANY); // 本地多有的IP

    // 绑定端口
    int ret = bind(lfd, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    if(ret == -1){
        perror("bind error");
        exit(1);
    }

    // 监听
    ret = listen(lfd, 64);
    if(ret == -1){
        perror("listen error");
        exit(1);
    }

    fd_set redset;
    FD_ZERO(&redset);
    FD_SET(lfd, &redset);

    int maxfd = lfd;
    while(1){
        pthread_mutex_lock(&mutex);
        fd_set tmp = redset;
        pthread_mutex_unlock(&mutex);
        int ret = select(maxfd+1, &tmp, NULL, NULL, NULL);
        // 判断是不是监听的fd
        if(FD_ISSET(lfd,&tmp)){
            // 接收客户端的连接
            /*创建子线程*/
            pthread_t tid;
            FDInfo *info = (FDInfo*)malloc(sizeof(FDInfo));
            info->fd = lfd;
            info->maxfd = &maxfd;
            inf->rdset = &redset;
            pthread_create(&tid, NULL, acceptConn, info); // 创建子线程
            pthread_detach(tid); // 分离线程
        }

        // 判断是不是通信的描述符
        for(int i = 0; i <= maxfd; i++){
            if(i != lfd && FD_ISSET(i, &tmp)){
                // 接收数据
                pthread_t tid;
                FDInfo *info = (FDInfo*)malloc(sizeof(FDInfo));
                info->fd = i;
                inf->rdset = &redset;
                pthread_create(&tid, NULL, communication, info); // 创建子线程
                pthread_detach(tid); // 分离线程
                
            }
        }
    }

    close(lfd);
    pthread_mutex_destory(&mutex);

    return 0;
}
```

## 2 poll系统调用

poll系统调用和select类似，也是在指定时间内轮询一定数量的文件描述符，以测试其中是否有就绪者。poll的原型下：

```cpp
#include<poll.h>
int poll(struct pollfd*fds,nfds_t nfds,int timeout);
```

- fds参数是一个pollfd结构类型的数组，它指定所有我们感兴趣的文件描述符上发生的可读、可写和异常等事件。pollfd结构体的定义如下：
  - 其中，fd成员指定文件描述符；
  - events成员告诉poll监听fd上的哪些事件，它是一系列事件的按位或；
  - revents成员则由内核修改，以通知应用程序fd上实际发生了哪些事件。

```cpp
struct pollfd
{
int fd;/*文件描述符*/
short events;/*注册的事件*/
short revents;/*实际发生的事件，由内核填充*/
};
```

- nfds参数指定被监听事件集合fds的大小,具体是最大有效元素下标+1。目的是为遍历的时候指定一个范围。其类型nfds_t的定义如下：
  
```cpp
typedef unsigned long int nfds_t;
```

- timeout参数指定poll的超时值，单位是毫秒。当timeout为-1时，poll调用将永远阻塞，直到某个事件发生；当timeout为0时，poll调用将立即返回。

