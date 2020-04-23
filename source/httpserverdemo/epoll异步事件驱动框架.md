demo源码  epoll异步事件驱动框架部分
=====================================

在该部分中，主要实现了两个类：
1抽象类EventHandle
作为event类中部分函数调用派生类 httpserver与httpstream（完整http套接字封装)对象提供基类指针。

2，event类
该类中主要实现了如下关键函数：
1，构造函数
主要是封装epoll_create，，对类中ThreadPool对象初始化，并调用eventwait_thread函数。
也因为pthread_create函数参数的需要，将process_task设置为静态函数，这样process_task在ThreadPool类未实例化时就有一个稳定的函数起始地址。
跟L/F线程池类ThreadPool的构造函数描述差不太多。

2，static void *eventwait_thread(void *arg);
一个静态的等待处理线程任务函数，在该函数中封装epoll_wait不停（while（1））获取套接字描述符fd并将其push入任务池（typedef std::map<int, EventType> EventTask_t;）与线程池。（此处note：fd从线程池中的弹出是在线程池类的process_task函数中实现的fd从任务池中弹出是process_task调用的event类的threadhandle()函数中实现的）

3，threadhandle()函数
从任务池弹出fd，调用http的handle_xxx函数对fd进行处理，然后调用epoll_ctl函数del掉事件。

4，register_event函数
封装epoll_ctl在epoll框架中add fd事件。
这个函数时在httpserver的构造函数里实现的。



event.h
```
#ifndef _EVENT_
#define _EVENT_
#include <sys/epoll.h>
#include <map>
#include "queue.h"
#include "threadpool.h"
#include "http.h"

namespace Event
{
enum EventType
{
	EIN = EPOLLIN,		  // 读事件
	EOUT = EPOLLOUT,	  // 写事件
	ECLOSE = EPOLLRDHUP,  // 对端关闭连接或者写半部
	EPRI = EPOLLPRI,	  // 紧急数据到达
	EERR = EPOLLERR,	  // 错误事件
	EET = EPOLLET, 		  // 边缘触发
	EDEFULT = EIN | ECLOSE | EERR | EET
};

class EventHandle
{
	public:
		int register_event(int fd, EventType type = EDEFULT);
		int register_event(Socket::ISocket &socket, EventType type = EDEFULT);
		int shutdown_event(int fd);
		int shutdown_event(Socket::ISocket &);
	protected:
		virtual void handle_in(int) = 0;
};

class Event: public ThreadPool::ThreadHandle
{
    private:

		enum Limit{
			EventBuffLen = 1024, CommitAgainNum = 2,
		};
		int m_epollfd;//句柄
		struct epoll_event m_eventbuff[EventBuffLen];//事件集合缓冲区

		pthread_t m_detectionthread;

		pthread_mutex_t m_events_mutex;

		ThreadPool::ThreadPool *m_ithreadpool;        	

		typedef std::map<int, ThreadHandle *> EventMap_t;

		typedef std::map<int, EventType> EventTask_t;

		EventTask_t m_events;


	public:
		Event(size_t eventmax);
		~Event();
		int register_event(int, EventHandle *, EventType);
		int shutdown_event(int);
		void threadhandle();
	private:
		static void *eventwait_thread(void *arg);	
		int pushtask(int fd, EventType event);
		int poptask(int &fd, EventType &event);
		int Event::cleartask(int fd);
		ThreadHandle *get_observer(int fd);

};
}
#endif
```


event.cpp
```

#include <unistd.h>
#include <sys/socket.h>
#include <string.h>
#include "threadpool.h"
#include "event.h"
#include "socket.h"
#include "http.h"

#define INVALID_FD(fd) (fd < 0)
#define INVALID_POINTER(p) (p == NULL)

namespace Event
{

Event::Event(size_t eventmax):   m_detectionthread(0)
{
	bzero((void *)&m_eventbuff, sizeof(m_eventbuff));//初始化事件集合缓冲区
	
	m_epollfd = epoll_create(eventmax);

	m_ithreadpool = new ThreadPool::ThreadPool(4, 1024);
	
	pthread_t tid = 0;
	if(pthread_create(&tid, NULL, eventwait_thread, (void *)this) == 0)//创建一条线程eventwait_thread，在这里调用epoll_wait等待事件
		m_detectionthread = tid;
}

Event::~Event()
{
	if(m_detectionthread != 0 && pthread_cancel(m_detectionthread) == 0){	//cancel thread
		pthread_join(m_detectionthread, (void **)NULL);//pthread_cancel后，部分资源仍未回收，调用pthread_join等待线程结束，到线程的退出代码后回收其资源
	}
	
	if(!INVALID_FD(m_epollfd))
		close(m_epollfd);
}

int Event::register_event(int fd, EventHandle *handle, EventType type)
{
	struct epoll_event newevent;
	newevent.data.fd = fd;
	newevent.events = type;
	epoll_ctl(m_epollfd, EPOLL_CTL_ADD, fd, &newevent) ;
	return 0;
}


void *Event::eventwait_thread(void *arg)
{
	Event &cevent = *(Event *)(arg);
	for(;;){
		int eventnum = epoll_wait(cevent.m_epollfd, &cevent.m_eventbuff[0], EventBuffLen, -1);//参数cevent.m_eventbuff[0]用来从内核得到事件的集合

		for(int i = 0; i < eventnum; i++){//事件发生后将事件放入队列并向线程池投入任务。

			int fd = cevent.m_eventbuff[i].data.fd;
			EventType events = static_cast<EventType>(cevent.m_eventbuff[i].events);
			
			if(cevent.pushtask(fd, events) == 0){
				cevent.m_ithreadpool->pushtask(&cevent);
			}
		}
	}
	pthread_exit(NULL);
}

int Event::pushtask(int fd, EventType event)
{
	pthread_mutex_lock(&m_events_mutex);
	EventTask_t::iterator itor = m_events.find(fd);
	if(itor == m_events.end()){
		m_events[fd] = event;
		pthread_mutex_unlock(&m_events_mutex);
		return 0;
	}
	// exist, update it
	itor->second = (EventType)(itor->second | event);	
	pthread_mutex_unlock(&m_events_mutex);
	return 0;
}


void Event::threadhandle()
{
	int fd = 0x00;
	EventType events;
	poptask(fd, events) ;


	Socket::Client  ob(fd);
	Socket::Client  *obs=&ob;
	Http::HttpStream  obse(obs);
	Http::HttpStream *observer=&obse;

	if(observer == NULL)
		return;

	if(events & ECLOSE){
		cleartask(fd);		
	
	}else{	
		if(events & EIN){
			observer->handle_in(fd);
		}

	}	
	
	epoll_ctl(m_epollfd, EPOLL_CTL_DEL, fd, NULL);
	observer->handle_close(fd);	
	
}



int Event::poptask(int &fd, EventType &event)
{
	pthread_mutex_lock(&m_events_mutex);
	EventTask_t::iterator itor = m_events.begin();
	if(itor == m_events.end()){
		pthread_mutex_unlock(&m_events_mutex);
		return -1;
	}
		

	fd = itor->first;
	event = itor->second;

	m_events.erase(itor);
	pthread_mutex_unlock(&m_events_mutex);
	return 0;
}

int Event::cleartask(int fd)
{
	pthread_mutex_lock(&m_events_mutex);
	if(fd == -1){	// clear all
		m_events.clear();
		pthread_mutex_unlock(&m_events_mutex);
		return 0;
		
	}else if(fd >= 0){
		EventTask_t::iterator itor = m_events.find(fd);
		if(itor == m_events.end()){
			pthread_mutex_unlock(&m_events_mutex);
			return -1;
		}
			

		m_events.erase(itor);
		pthread_mutex_unlock(&m_events_mutex);
		return 0;	
	}

	return -1;
}

}
```