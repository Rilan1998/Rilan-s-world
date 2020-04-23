http服务器的编写 1 底层socket
=============================

近期读了一些c++写http服务器的代码，准备自己亲手实现一个。
项目初级预期，期待post方法传来一个数字，返回该数字的阶乘。
对get请求返回一个hello语句。

一，底层实现。

1，创建套接字

#include <sys/socket.h>
//服务端套接字
int server_sockfd；
server_sockfd = socket(AF_INET, SOCK_STREAM, 0); //通信域--IPv4,通信类型--面向连接可靠传输,协议--0（根据前两个参数使用默认的协议）

//客户端套接字
client_sockfd = accept(server_sockfd,(struct sockaddr *)&client_address, &client_len);//会新创建一个套接字传给client_sockfd


2，对服务器套接字地址信息的初始化

struct sockaddr_in server_address；
memset(&server_address, 0, sizeof(server_address));//对server_address初始化
server_address.sin_family = AF_INET;//IPV4
server_address.sin_port = htons(*port);//主机字节顺序转换为网络字节顺序
server_address.sin_addr.s_addr = htonl(INADDR_ANY);//监听本机所有IP


3，绑定与监听
int server_len；
server_len = sizeof(server_address);
bind(server_sockfd, (struct sockaddr *)&server_address, server_len);
listen(server_sockfd, 5);//设置同时监听最大个数，其实就是队列长度


4，客户端部分

//客户端套接字
int client_sockfd；

//客户端套接字初始化
int client_len；
struct sockaddr_in client_address；
client_len = sizeof(client_address);

//accept
client_sockfd = accept(server_sockfd,(struct sockaddr *)&client_address, &client_len);//会新创建一个套接字传给client_sockfd

5，接收与发送字符

char buffer[5000];
char send_str[] = "hello\n";  // 准备给连接过来的客户端发送的字符串
read(client_sockfd, &buffer, 5000);    // 接收客户端传来的字符，存到buffer中
printf("%s", buffer);     //  打印我们接收到的字符
write(client_sockfd, &send_str, sizeof(send_str)/sizeof(send_str[0]));   // 向客户端发送数据，这里的 read write 和 和文件读写时没什么区别


6.关闭套接字

//客户端套接字应在处理请求的线程中关闭
close(client_sockfd);

//服务端套接字在main函数最后关闭
close(server_sockfd);




二，函数设计

1，main函数
通过startup实现网络初始化。
循环accept侦听客户端连接。
创建新线程处理客户端连接。
关闭网络，用close函数实现。



