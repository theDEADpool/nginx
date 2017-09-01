# 配置解析

------

先看一下nginx配置格式。

```
worker_processes 1;

events {
  worker_connection 1024;
}

http {
  keepalive_timeout 65;
  server {
    listen 80;
  }
}
```

worker_processes这种直接配置的，属于NGX_CORE_MODULE类型的模块配置。下面的events、http也是一样。

为什么events或http要写成带有{}的格式？

因为worker_processes所属的`ngx_core_module`并不包含子模块。而events或http都有包含子模块。

了解nginx的配置格式对于理解nginx配置解析是必要的。

------

配置解析入口函数

```
ngx_conf_parse
```

函数主要处理内存分配，配置文件读取等操作。

```
ngx_conf_handler
{
  /*先做一些配置项匹配，参数校验等操作*/
  
  /*如果是通过ngx_init_cycle流程，则cf->module_type == NGX_CORE_MODULE*/
  if (cf->cycle->modules[i]->type != NGX_CONF_MODULE
	&& cf->cycle->modules[i]->type != cf->module_type) {
  	continue;
  }

  /*如果是通过ngx_init_cycle流程，则cf->cmd_type == NGX_MAIN_CONF*/
  if (!(cmd->type & cf->cmd_type)) {
	continue;
  }
  
  /*
  上面两个判断可以理解为，通过nginx启动流程进行配置解析，启动流程只关注NGX_CORE_MODULE中，类型为	  	   NGX_MAIN_CONF的配置解析。
  */
  
  /*
  调用每个配置项自定义的set函数，完成配置解析并保存在conf指向的内存中
  */
  if (cmd->type & NGX_DIRECT_CONF) {
  	conf = ((void **) cf->ctx)[cf->cycle->modules[i]->index];
  } else if (cmd->type & NGX_MAIN_CONF) {
   	conf = &(((void **) cf->ctx)[cf->cycle->modules[i]->index]);
  } else if (cf->ctx) {
    confp = *(void **) ((char *) cf->ctx + cmd->conf);
    
    if (confp) {
		conf = confp[cf->cycle->modules[i]->ctx_index];
    }
  }
  
  rv = cmd->set(cf, cmd, conf);
}
```

那么`conf`指向的内存到底是哪一块内存？

就从`cf->ctx`的初始化说起，在`ngx_init_cycle`中，

```
cycle->conf_ctx = ngx_pcalloc(pool, ngx_max_module * sizeof(void *));
```

`cycle->conf_ctx`是一个指针数组，这里的`cycle->conf_ctx`就是配置解析函数中的`cf->ctx`。

```
for (i = 0; cycle->modules[i]; i++) {
	if (cycle->modules[i]->type != NGX_CORE_MODULE) {
    	continue;
    }

    module = cycle->modules[i]->ctx;

    if (module->create_conf) {
        rv = module->create_conf(cycle);
        if (rv == NULL) {
            ngx_destroy_pool(pool);
            return NULL;
        }
        cycle->conf_ctx[cycle->modules[i]->index] = rv;
    }
}
```

这一段逻辑就是根据每个CORE_MODULE模块的index，对`cycle->conf_ctx`指针数组中的对应成员进行初始化。

之前，在nginx启动流程中有说明，并不是所有的CORE_MODULE模块都定义了`create_conf`函数。也就意味着有些CORE_MODULE模块在`cycle->conf_ctx`指针数组中是没有初始化的。

不过先关注有定义`create_conf`函数的模块，比如`ngx_core_module`。

根据`ngx_core_module_create_conf`的实现，`cycle->conf_ctx[ngx_core_module.index]`就是指向类型为`ngx_core_conf_t`内存的指针。实际上这块内存就是用来保存`ngx_core_module`每个配置项的值。

再来关注没有定义`create_conf`函数的模块，比如`ngx_http_module`。

要解析http模块及其子模块的配置，首先会调用`ngx_http_block`函数。这个函数就完成了保存http模块及其子模块配置内存申请。

![ngx-配置解析-1](.\ngx-配置解析-1.bmp)

上图就展示了http模块及子模块之间的数据结构关系。需要说明的是，`**main_conf`、`**srv_conf`和`loc_conf`都是指针数组，指向的都是http子模块对应类型的配置项内存。http子模块每个配置通过`NGX_HTTP_MAIN_CONF_OFFSET`、`NGX_HTTP_SRV_CONF_OFFSET`、`NGX_HTTP_LOC_CONF_OFFSET`来决定每个配置存放在`**main_conf`、`**srv_conf`还是`loc_conf`。