



int main(void)
{

    int server_sock = -1;
    u_short port = 0;
    int client_sock = -1;
    struct sockaddr_in client_name;

    //这边要为socklen_t类型
    socklen_t client_name_len = sizeof(client_name);
    pthread_t newthread;

    server_sock = startup(&port);
    printf("httpd running on port %d\n", port);

    while (1)
    {
        //接受请求，函数原型
        //#include <sys/types.h>
        //#include <sys/socket.h>  
        //int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
        client_sock = accept(server_sock,
            (struct sockaddr*) & client_name,
            &client_name_len);
        if (client_sock == -1)
            error_die("accept");
        /* accept_request(client_sock); */

         //每次收到请求，创建一个线程来处理接受到的请求
         //把client_sock转成地址作为参数传入pthread_create
        /*第一个参数为指向线程标识符的指针。
        第二个参数用来设置线程属性。
        第三个参数是线程运行函数的起始地址。
        最后一个参数是运行函数的参数。*/
        if (pthread_create(&newthread, NULL, (void*)accept_request, (void*)(intptr_t)client_sock) != 0)
            perror("pthread_create");
    }

    close(server_sock);

    return(0);
}