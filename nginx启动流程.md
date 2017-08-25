# main #

----------

    ngx_preinit_modules

nginx代码在configure过程中会在objs/目录下生成ngx_modules.c中定义了`ngx_module_names`和`ngx_modules`数组。
该函数就是给所有的模块按照从先到后的顺序生成index。

----------

	ngx_init_cycle

`ngx_cycle_t *cycle`数据结构是nginx最核心的数据结构。在此解读一下`ngx_init_cycle`对`ngx_cycle_t`结构体中一些关键数据结构的操作。

- `ngx_module_t **modules` 

存储所有模块的`ngx_module_t`数据结构指针的数组。
在`ngx_cycle_modules`函数对`cycle->modules`和`cycle->modules_n`做了赋值操作。

-  `void ****conf_ctx`

保存所有`NGX_CORE_MODULE`类型模块的配置结构体指针的数组。
在`ngx_cycle_modules`调用之后，就会调用每个模块自定义的`create_conf`函数给`cycle->conf_ctx`数组中，该模块对应的数组成员赋值。

但并不是所有的`NGX_CORE_MODULE`都有定义`create_conf`函数，比如`ngx_http_module`。对于`ngx_http_module`模块`cycle->conf_ctx`中对应成员的赋值是在`ngx_http_block`模块完成的。

深究一下为什么有些`NGX_CORE_MODULE`有定义`create_conf`而有的没有。

以`ngx_core_module`为例，它定义了`create_conf`，函数功能就是分配并初始化了`ngx_core_module`用来保存配置信息的内存。因为`ngx_core_commands`中定义了很多属于`ngx_core_module`模块的配置项，因此需要通过`create_conf`来先申请好内存，后续的配置解析才能正常完成。

而`ngx_http_module`，`ngx_http_commands`只定义了一个`http`配置，其实这都不算是一个配置。nginx的模块架构是多层的，比如http模块，它本身是一个一层模块，它下面还有各种二层模块如`ngx_http_access_module`，对于`ngx_http_module`本身而言，它并没有属于自己的配置项，http相关的配置都是由其二层模块来定义的。因此，`ngx_http_module`是在http配置定义的`ngx_http_block`内完成了配置结构体的内存分配。

----------

    ngx_conf_parse

作为单独的一个流程来介绍。

----------

