# 监听端口

------

​	Nginx在启动流程`ngx_init_cycle`中会对`cycle->listening`进行初始化。这个数据结构是用来保存Nginx各个监听端口信息的数组。监听端口信息有两个来源一是从旧的进程中继承下来，二是来自于配置解析。本次主要关注配置解析的流程。

​	完成上述的初始化过程中之后就是要对数组进行赋值。赋值的过程是由各个模块的配置解析函数完成的。以http模块为例。http模块及子模块的配置解析入口函数为`ngx_http_block` ，该函数会调用`ngx_http_core_listen`对`listen`配置项进行解析，再调用`ngx_http_add_listen`保存解析到的监听端口信息。完成所有配置项解析之后，`ngx_http_block`会调用`ngx_http_optimize_servers`->`ngx_http_init_listening`->`ngx_http_add_listening`将解析到的监听端口加入到`cycle->listening`的数组中，且设置监听端口回调函数为`ngx_http_init_connection`。

​	之后`ngx_init_cycle`调用`ngx_open_listening_sockets`根据`cycle->listening`中保存的监听端口信息创建socket。

​	接下来会做的就是将连接和监听端口进行关联，每个监听端口对应一个连接。以父子进程模式工作的Nginx为例，`ngx_worker_process_init`函数会调用`ngx_event_process_init`，创建`cycle->connections`，这个数据结构是一个数组，也是一个单向链表。每个成员的`data`字段都指向下一个成员。而`cycle->free_connections`字段则是指向了`cycle->connections`的第一个空闲连接。

​	然后会将`cycle->listening.elts`和`cycle->connections`逐一关联。关键是设置`rev->handler`为`ngx_event_accept`。同时对`rev`增加epoll读事件。完成这些操作，监听端口就正式开始监听数据。待这些socket收到数据，触发epoll读事件回调`ngx_event_accept`。

​	`ngx_event_accept`做的就是调用`accept`函数，完成TCP建连。重新设置连接的读回调`c->recv`和写回调`c->send`。并调用`ngx_epoll_add_connection`重新设置epoll事件。至此监听端口就可以开始接受和发送数据。