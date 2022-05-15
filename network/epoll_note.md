# epoll原理及简单用法
---

### epoll原理极简解释
epoll是Linux系统下一个高效的多路复用IO机制。
Linux下，一切皆文件。epoll可对大部分IO操作，以下以socket为例，对epoll机制进行简单解释。
用户进程所有的IO都由操作系统统一管理，所有的IO对象（socket，通道，文件等）在内核都对应一个文件描述符，用户只能操作文件描述符，由内核去
和具体的IO设备打交道。
linux有三个系统调用来操作epoll
~~~
int epoll_create(int size);
~~~
这个函数创建一个epoll文件，返回对应的文件描述符，入参没什么用，为了
向后兼容，大于0就好。
通过这个函数，会在内核中开辟有段高速缓存空间，保存需要监听的socket，
以及一个就绪list。
~~~
typedef union epoll_data {
	void *ptr;
	int fd;
	__uint32_t u32;
	__uint64_t u64;
} epoll_data_t;

struct epoll_event {
	__uint32_t events;
	epoll_data_t data;
};
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
~~~
这个函数修改fd在epoll中的状态，将这个fd插入/删除/修改到红黑树中，并向内核中断处理程序注册一个回调，当这个fd有事件发生，就向list中插入这个fd。epfd为之前创建的epoll文件描述符，op为注册的操作，fd为需要监听的文件描述符，event为需要监听的事件。
op有三个选择：
- EPOLL_CTL_ADD 添加fd为监听的文件
- EPOLL_CTL_MOD 修改fd的监听事件
- EPOLL_CTL_DEL 不在监听fd

event->events有7中选择，可通过位或选择多个选项：
- EPOLLIN fd可读
- EPOLLOUT fd可写
- EPOLLPRI 有紧急数据可读
- EPOLLERR fd发生错误
- EPOLLHUP fd被挂断
- EPOLLET 将EPOLL 设为ET模式
- EPOLLONESHOT 这个fd只监听一次，一次之后将它从监听中删除

event->data 联合体。设置fd为要监听的对象或设置ptr为要写的数据，其他用法不清楚。

~~~
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
~~~
这个函数会启动epoll监听，线程会在这阻塞，直到有被监听的事件发生，函数会检查list中是否有元素，如有，就将所有元素copy到events,并清空list。epfd为epoll的fd, events为为存储返回的所有发生的事件，maxevents为events的大小，timeout为超时限制。返回值为返回的event的个数。
timeout为-1表示永久等待，0表示立即返回。

### 一个服务端监听socket的例子
~~~
for( ; ; ){
    nfds = epoll_wait(epfd,events,20,500);
    for(i=0;i<nfds;++i){
        if(events[i].data.fd==listenfd){ //如果是主socket的事件，则表示有新的连接
			connfd = accept(listenfd,(sockaddr *)&clientaddr, &clilen); //accept这个连接
			ev.data.fd=connfd;
			ev.events=EPOLLIN|EPOLLET;
			epoll_ctl(epfd,EPOLL_CTL_ADD,connfd,&ev); //将新的fd添加到epoll的监听队列中
        }
        else if( events[i].events&EPOLLIN ){ //接收到数据，读socket
            if ( (sockfd = events[i].data.fd) < 0) continue;
            n = read(sockfd, line, MAXLINE)) < 0    //读
            ev.data.ptr = md;     //md为自定义类型，添加数据
            ev.events=EPOLLOUT|EPOLLET;
            epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev);//修改标识符，等待下一个循环时发送数据，异步处理的精髓
        }
        else if(events[i].events&EPOLLOUT){ //有数据待发送，写socket
            struct myepoll_data* md = (myepoll_data*)events[i].data.ptr;    //取数据
            sockfd = md->fd;
            send( sockfd, md->ptr, strlen((char*)md->ptr), 0 );        //发送数据
            ev.data.fd=sockfd;
            ev.events=EPOLLIN|EPOLLET;
            epoll_ctl(epfd,EPOLL_CTL_MOD,sockfd,&ev); //修改标识符，等待下一个循环时接收数据
        }
        else{
            //其他情况的处理
        }
    }
}
~~~


### 一个监听管道的例子
读管道端：
~~~
#include <stdlib.h>
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <fcntl.h>


int main(void)
{
    int fifo_fd1, fifo_fd2;
    int ep_fd;
    struct epoll_event event;
    struct epoll_event *ret_events = NULL;
    int cnt;
    int i;
    char c;

    mkfifo("./fifo1", 0666);
    mkfifo("./fifo2", 0666);

    fifo_fd1 = open("./fifo1", O_RDONLY);
    fifo_fd2 = open("./fifo2", O_RDONLY);

    /* 需要有进程以只写的方式打开fifo1、fifo2后才能执行于此  */
    printf("监测fifo1和fifo2中...\n");

    /* 创建epoll池 */
    ep_fd = epoll_create1(0);

    /* 将检测事件加入epoll池中 */
    event.events = EPOLLIN | EPOLLET;   /* 监测fifo1可读，且以边沿方式触发 */
    event.data.fd = fifo_fd1;
    epoll_ctl(ep_fd, EPOLL_CTL_ADD, fifo_fd1, &event);

    event.events = EPOLLIN | EPOLLET;   /* 监测fifo2可读，且以边沿方式触发 */
    event.data.fd = fifo_fd2;
    epoll_ctl(ep_fd, EPOLL_CTL_ADD, fifo_fd2, &event);

    /* ret_events用于存放被触发的事件 */
    ret_events = malloc(sizeof(struct epoll_event) * 100);

    /* 阻塞等待监测事件触发 */
    cnt = epoll_wait(ep_fd, ret_events, 100, -1);
    printf("cnt = %d\n", cnt);

    /* 判断监测事件 */
    for (i = 0; i < cnt; i++)
    {
        if (ret_events[i].events & EPOLLIN)
        {
            read(ret_events[i].data.fd, &c, 1);
            printf("fd = %d, recv data = %c\n", ret_events[i].data.fd, c);
        }
    }

    free(ret_events);
    close(ep_fd);       /* 注意关闭epoll池的描述符 */
    close(fifo_fd2);
    close(fifo_fd1);

    return 0;
}
~~~
写管道1：
~~~
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

int main(void)
{
    int fd;
    char c;

    fd = open("./fifo1", O_WRONLY);

    printf("w1: pls input char: \n");
    scanf("%c", &c);

    write(fd, &c, 1);

    close(fd);

    return 0;
}
~~~
写管道2：
~~~
int main(void)
{
    int fd;
    char c;

    fd = open("./fifo2", O_WRONLY);
    printf("w2: pls input char: \n");
    scanf("%c", &c);

    write(fd, &c, 1);

    close(fd);
    return 0;
}
~~~