# C++ Epoll

```cpp
 int epoll_create(int size);  
 int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
 int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```

## Server side:

```cpp
int listenfd = socket(AF_INET, SOCK_STREAM, 0);  //create socket file descriptor
SetSocket(listenfd);                             //设置 socket 为非阻塞

struct epoll_event epev;  //记录套接字相关信息
epev.events = EPOLLIN;    //监视有数据可读事件
epev.data.fd = listenfd;  //文件描述符数据，其实这里可以放任何数据。

int epollfd = epoll_create(1024);
//创建 epoll 文件描述符，出错返回-1

epoll_ctl(epollfd, EPOLL_CTL_ADD, listenfd, &epev);
//加入监听列表，当 listenfd 上有对应事件产生时， epoll_wait 会将 epoll_event 填充到 events_in 数组里
//出错返回 -1

struct epoll_event events_in[16];
while(true)
{
    int event_count = epoll_wait(epollfd, events_in, 16, -1);
    //等待事件， epoll_wait 会将事件填充至 events_in 内
    //返回 获得的事件数量，若超时且没有任何事件返回 0 ，出错返回 -1 。 timeout设置为-1表示无限等待。

    for (int i = 0; i < event_count; i++)      //遍历所有事件
    {
        if (events_in[i].data.fd == listenfd)  //新连接请求
        {
            int new_fd = accept(listenfd, NULL, NULL);
            epev.events = EPOLLIN;  //参见 man 7 epoll 如果要使用 Edge Trigger 还需将 new_fd 设为非阻塞
            epev.data.fd = new_fd;
            epoll_ctl(epollfd, EPOLL_CTL_ADD, new_fd, &epev);  //将新连接加入监视列表
        }
        else if (events_in[i].events & EPOLLIN)  //有数据可读
        {
            int new_fd = events_in[i].data.fd;
            // ... handle(new_fd);
            epoll_ctl(epollfd, EPOLL_CTL_DEL, new_fd, NULL); //不再监听fd，最后一个参数被忽略
            close(new_fd);
        }
    }
}
```

## epoll_create():

```cpp
int epoll_create(int size);
/* 生成 epoll 文件描述符，用来存放 socket fd 上是否发生以及发生了什么事件。
size: 返回的 epoll fd 上能关注的最大 socket fd 数。
*/

```

## epoll_ctl():
```cpp
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
/* 控制 epoll 文件描述符上的事件：注册、修改、删除。
epfd: epoll_create() 创建的 epoll 文件描述符。
op: epoll 监听事件，
  EPOLL_CTL_ADD 增加一个新的 epoll 事件。
  EPOLL_CTL_DEL 减少一个 epoll 事件，
  EPOLL_CTL_MOD 改变一个事件的监听方式。
fd: socket() 返回值
event: struct epoll_event;
*/

```

## struct epoll_event:

```cpp
// 结构体 epoll_event 被用于注册事件和回传所发生待处理的事件，定义如下：
typedef union epoll_data
{
    void* ptr;
    int fd;
    __uint32_t u32;
    __uint64_t u64;
}
epoll_data_t;  /* 保存触发事件的某个文件描述符相关的数据 */

struct epoll_event
{
    __uint32_t events;  /* epoll event */
    epoll_data_t data;  /* User data variable */
};
/* epoll_event.events:
  EPOLLIN  表示对应的文件描述符可以读
  EPOLLOUT 表示对应的文件描述符可以写
  EPOLLPRI 表示对应的文件描述符有紧急的数据可读
  EPOLLERR 表示对应的文件描述符发生错误
  EPOLLHUP 表示对应的文件描述符被挂断
  EPOLLET  表示对应的文件描述符有事件发生
*/

```

##epoll_wait():

```cpp
int epoll_wait(int epfd, struct epoll_event* events,int maxevents, int timeout);
/* 等待I/O事件的发生
epfd: epoll_create() 生成的 epoll 文件描述符；
epoll_event: 用于回传代处理事件的数组；
maxevents: 每次能处理的事件数；
timeout: 等待I/O事件发生的超时值；返回发生事件数。

```
