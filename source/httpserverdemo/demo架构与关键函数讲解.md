demo架构与关键函数讲解
=====================================

一，系统架构
1，设计模式
demo系统是基于观察者模式实现的。观察者模式是非常经典的行为型设计模式，是事件驱动模型在设计层面的体现。
2，类的设计
从关键点上来说，整个系统的关键点有Leader/Follower模式的线程池部分，epoll异步事件驱动框架部分，以及http套接字部分。
其中最核心的部分是封装epoll的Event事件处理中心，Event既是L/F线程池部分中线程处理接口的实例化，利用Http类的基类指针调用http部分也要在Event事件处理过程中实现。

类列表如下：
queue :
把标准库中的queue封装成阻塞队列，加了三把锁，分别是互斥锁(实现原子操作)，pop阻塞锁，push阻塞锁。是实现线程池的工具类。

ThreadHandle ：
作为线程处理的接口，也是线程池和事件中心耦合的接口，为线程池调用派生类 Event对象提供基类指针。

ThreadPool :
线程池类，该类功能得以实现线程池功能的关键函数是构造函数和静态的线程任务处理函数。显现L/F部分特色的函数既是在静态的线程任务处理函数中调用，也是为线程任务处理函数服务。

EventHandle ：
作为事件处理的接口，也是http套接字和事件中心耦合的接口，为Event事件中心调用其派生类httpserver与httpstream（完整http套接字封装)对象提供基类指针。

Event(public ThreadHandle)
事件中心类，详情见epoll异步事件驱动框架部分。

ISocket ：
套接字抽象类。

Client(public ISocket) ：
tcp客户端套接字类。

Server(public ISocket) ：
tcp服务器端套接字类。

HttpRequest ：
http请求的封装。

HttpRespose ：
http回复的封装。

HttpServer (public EventHandle) ：
http服务器套接字。

HttpStream (public EventHandle) ：
封装http连接，在HttpServer中调用。

二，启动过程与响应流程

为清晰表示调用关系，将过程表示如下：
'''
启动：
1，main函数中构造一个HttpServer对象httpserver，绑定服务器的端口。
2，main函数调用httpserver构造函数，在此过程中httpserver成员变量m_server也进行构造。
3，main函数调用httpserver.start(http服务器端套接字类的start函数)。
    4，httpserver.start成员调用m_server.start函数(tcp服务器端套接字类的start函数)
        5，m_server.start中，调用了库函数convert_inaddr，setsockopt，bind，listen。总而言之，对tcpserver套接字初始化。
    6，httpserver.start调用m_server.set_nonblock将m_server设置为非阻塞型。
    7，httpserver.start调用EventHandle::register_event(m_server)。
        8，EventHandle::register_event创建一个静态Event对象event，并调用event.register_event将m_server注册到Event事件中心。
            9，event的构造函数初始化事件集合缓冲区，调用库函数epoll_create，创建线程池m_ithreadpool对象，
                10，m_ithreadpool的构造函数初始化线程池并让每个线程执行process_task函数。(note1)
            11，event的构造函数调用pthread_create创建一个新线程执行eventwait_thread函数。(note2)
        12,event.register_event封装一个epoll_event对象，并调用库函数epoll_ctl 的add动作将该fd注册到epfd中。

可以注意到在启动过程中的9和10分别标注了两个note
note1---process_task函数：
（while（true））从线程池handle队列中pop出fd并通过调用Event类的任务执行函数threadhandle->threadhandle();（在event类中实现）将fd从任务池中弹出并处理。
note2---eventwait_thread函数：
while（true））调用epoll_wait获取fd，并将其push入任务池（typedef std::map<int, EventType> EventTask_t;）与线程池。


响应过程：
1，epoll_wait获取fd，并将其push入任务池与线程池。
2，process_task函数从线程池handle队列中pop出fd，调用event.threadhandle->threadhandle()。
    3, threadhandle函数判断事件类型，如果是读事件，调用HttpServer::handle_in函数处理。
        4，handle_in用m_server.accept()初始化一个新client对象newconn，并用newconn初始化一个新的HttpStream对象httpstream。
            5，httpstream构造过程中将newconn设置为非阻塞模式并将其注册入事件中心。
        6，HttpServer::handle_in调用httpstream.handle_in。
            7，httpstream.handle_in调用httprequest.load_packet处理请求，调用handle_request(httprequest)生成http回复，调用m_client->send发送回复。
    8，处理完成后调用epoll_ctl的del动作将该fd在epfd中删除。
'''
在整个架构的梳理过程中发现了一些按之前思路coding时的问题，主要是响应过程的第3步，现在的代码好像是把3执行成了6，这是个很大的bug，有待修复。
