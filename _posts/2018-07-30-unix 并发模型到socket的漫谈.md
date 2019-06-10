---
    layout: post
    title: unix 并发模型到socket的漫谈
---

-  并发到 i/o

1.  操作系统实现并发的三种方式  
    实现一个简单的 web 服务器为例：

    - 基于进程的，fork 一个子进程

      ```c
          int main(int argc, char **argv)
          {
              int listenfd, connfd, port;
              socklen_t clientlen=sizeof(struct sockaddr_in);
              struct sockaddr_in clientaddr;
              if (argc != 2) {
              fprintf(stderr, "usage: %s <port>\n", argv[0]);
              exit(0);
              }
              port = atoi(argv[1]);
              Signal(SIGCHLD, sigchld_handler);
              listenfd = Open_listenfd(port);
              while (1) {
              connfd = Accept(listenfd, (SA *) &clientaddr, &clientlen);
              if (Fork() == 0) {
                  Close(listenfd); /* Child closes its listening socket */
                  echo(connfd);    /* Child services client */ //line:conc:echoserverp:echofun
                  Close(connfd);   /* Child closes connection with client */ //line:conc:echoserverp:childclose
                  exit(0);         /* Child exits */
              }
              Close(connfd); /* Parent closes connected socket (important!) */ //line:conc:echoserverp:parentclose
              }
          }
      ```

    * 一个进程中 i/o 多路复用 

      ```c
      int main(int argc, char **argv)
      {
          int listenfd, connfd, port;
          socklen_t clientlen = sizeof(struct sockaddr_in);
          struct sockaddr_in clientaddr;
          static pool pool;

          if (argc != 2) {
          fprintf(stderr, "usage: %s <port>\n", argv[0]);
          exit(0);
          }
          port = atoi(argv[1]);

          listenfd = Open_listenfd(port);
          init_pool(listenfd, &pool); //line:conc:echoservers:initpool
          while (1) {
          /* Wait for listening/connected descriptor(s) to become ready */
          pool.ready_set = pool.read_set;
          pool.nready = Select(pool.maxfd+1, &pool.ready_set, NULL, NULL, NULL);

          /* If listening descriptor ready, add new client to pool */
          if (FD_ISSET(listenfd, &pool.ready_set)) { //line:conc:echoservers:listenfdready
              connfd = Accept(listenfd, (SA *)&clientaddr, &clientlen); //line:conc:echoservers:accept
              add_client(connfd, &pool); //line:conc:echoservers:addclient
          }

          /* Echo a text line from each ready connected descriptor */
          check_clients(&pool); //line:conc:echoservers:checkclients
          }
      }
      ```

    * 基于线程

2.  unix i/o 模型


    - 一个输入操作通常包括两个阶段：

        * 等待数据准备好
        * 从内核向进程复制数据  

        对于一个套接字上的输入操作，第一步通常涉及等待数据从网络中到达。分组到达时，它被复制到内核中的某个缓冲区。第二步就是把数据从内核复制制到应用进程缓冲区。

    - Unix 下有五种 I/O 模型：
       * 阻塞式 I/O
       * 非阻塞式 I/O
       * I/O 复用（select 和 epoll，select（）这个内核调用是堵塞的）
       * 信号驱动式 I/O（SIGIO）
       * 异步 I/O（AIO）

    - 前四种 I/O 模型的主要区别在于第一个阶段，而第二个阶段是一样的：将数据从内核复制到应用进程过程中，应用进程会被阻塞。
    ![i/o](/images/unix_i:o.png)

    - syn and asyn：一个 i/o 操作中在请求返回前不会堵塞 就是 异步；x
    - 按这种定义，只有第五种AIO才是异步i/o

3. java中 BIO,NIO,AIO
    - BIO:面向stream的blocked i/o
    - NIO: 是上面的 i/o多路复用模型，虽然selector.select()是堵塞的，但是在eventLoop中将准备好的数据放入到了 buffer中，这样面对buffer而言就是非堵塞的
    - AIO: 不知道是不是上面的AIO,netty中还是以NIO为主

4. kafka的网络层
    - 网络层相当于一个 NIO 服务，在java NIO 上简单封装
    - a non-user-space writing of bytes use FileChannel.transferTo.  FileChannel就是建立文件数据与用户空间的内存映射，省略了数据从内核->用户空间的这一步。 sendfile(零拷贝) 的实现是通过 MessageSet 接口的 writeTo 方法完成的.这样的机制允许 file-backed 集使用更高效的 transferTo 实现,而不在使用进程内的写缓存.
    - 线程模型： 线程模型是一个单独的接受线程和 N 个处理线程,每个线程处理固定数量的连接. 就是Reactor模型

5. netty的i/o封装


6. 一些引用：
    - > [Doug Lea 的nio](../file/nio.pdf)（有Reactor 模型的介绍）