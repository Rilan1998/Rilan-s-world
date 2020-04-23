demo源码  LF线程池部分
=====================================
threadpool.h    
基于非阻塞队列实现的简易L/F线程池。
```
L/F 领导者跟随者模式 ：在LF线程池中，线程可处在3种线程状态之一： leader、follower或processor。处于leader状态的线程负责监听网络端口，当有消息到达时，该线程负责消息分离，并从处于 follower状态中的线程中按照某种机制如FIFO或基于优先级等选出一个来当新的leader，然后将自己设置为processor状态去分配和处 理该事件。处理完毕后线程将自身的状态设置为follower状态去等待重新成为leader。在整个线程池中同一时刻只有一个线程可以处于leader 状态，这保证了同一事件不会被多个线程重复处理。 
缺点：实现复杂性和缺乏灵活性； 
优点：增强了CPU高速缓存相似性，消除了动态内存分配和线程间的数据交换。 
```
该文件中实现了两个类：
1，抽象类ThreadHandle
作为线程处理的接口，为调用派生类 event（异步事件驱动框架)对象提供基类指针。
2，L/F线程池类ThreadPool
该类中主要实现了如下函数：
1，构造函数
主要是封装pthread_create，并调用process_task函数。
也因为pthread_create函数参数的需要，将process_task设置为静态函数，这样process_task在ThreadPool类未实例化时就有一个稳定的函数起始地址。
2，析构函数
不谈。
3，static void *process_task(void * arg)
一个静态的线程任务处理函数，在该函数中不停（while（1））从线程handle队列中pop句柄并通过调用Event类的任务执行函数threadhandle->threadhandle();（在event类中实现）来处理。
promote_leader和join_follwer作为process_task的工具函数，做到在调用process_task时，始终是leader线程在处理任务。代码不复杂，不再赘述。


```
#ifndef  _THREADPOOL_H_
#define _THREADPOO_H_

#include <pthread.h>
#include <vector>
#include <stdio.h>
#include "queue.h"
//线程池类
#define THREADNUM_MAX  64

namespace ThreadPool
{
class ThreadHandle
{
	friend class ThreadPool;
	public:
		virtual ~ThreadHandle(){};
		virtual void threadhandle() = 0;

};


class ThreadPool
{
	
	public:

		typedef void *(threadproc_t)(void *);
		typedef std::vector<pthread_t>  vector_tid_t;
		typedef Queue::Queue<ThreadHandle *>  queue_handle_t;
		
	private:
		size_t m_threadnum;//线程数量
		vector_tid_t m_thread;//存放线程的顺序表

		queue_handle_t m_taskqueue;//线程句柄队列
		bool m_hasleader;//是否有leader

		pthread_cond_t m_befollower_cond;//
		pthread_mutex_t m_identify_mutex;//

	public:

		ThreadPool(size_t threadnum, size_t tasknum): m_taskqueue(tasknum), m_hasleader(false)
		{
			m_threadnum = (threadnum > THREADNUM_MAX)? THREADNUM_MAX: threadnum;
			pthread_attr_t thread_attr;//线程属性
			pthread_attr_init(&thread_attr);//属性初始化

			for(size_t i = 0; i < m_threadnum; i++){
				pthread_t tid = i;//线程标志符
				pthread_create(&tid, &thread_attr, process_task, (void *)this);// (void *)this是process_task需要的参数
				m_thread.push_back(tid);
			}
			pthread_attr_destroy(&thread_attr);//属性去除初始化
		}

		~ThreadPool()
		{
			void *retval = NULL;
			vector_tid_t::iterator itor = m_thread.begin();
			for(; itor != m_thread.end(); itor++){
				pthread_cancel(*itor) < 0 || pthread_join(*itor, &retval) ;
				m_thread.clear();
			}
		}


        // 往线程池放入任务，非阻塞方式
		int pushtask(ThreadHandle *handle)
		{
			return m_taskqueue.push_nonblock(handle);
		}


	private:
		void promote_leader()
		{
	 		pthread_mutex_lock(&m_identify_mutex);

			while(m_hasleader){	// 已经有leader，阻塞
		 		pthread_cond_wait(&m_befollower_cond, &m_identify_mutex);
			}
			m_hasleader = true;
			pthread_mutex_unlock(&m_identify_mutex);
		}

		void join_follwer()
		{
			pthread_mutex_lock(&m_identify_mutex);
			m_hasleader = false;
			pthread_cond_signal(&m_befollower_cond);
			pthread_mutex_unlock(&m_identify_mutex);
		}

static void *process_task(void * arg)
{
	ThreadPool &threadpool = *(ThreadPool *)arg;

	while(true){
		threadpool.promote_leader();	

		ThreadHandle *threadhandle = NULL;

		int ret = threadpool.m_taskqueue.pop(threadhandle);

		threadpool.join_follwer();		


		if(ret == 0 && threadhandle)//返回成功并且有任务需处理
			threadhandle->threadhandle();
	}	

	pthread_exit(NULL);
}

};
}
#endif
```