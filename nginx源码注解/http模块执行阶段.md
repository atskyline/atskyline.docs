nginx为在HTTP请求的处理划分了一些阶段，每个阶段都有不同的职责，并且支持通过模块化的方式，在每个阶段扩展不同的功能。本文简单介绍nginx为HTTP请求处理划分的阶段。



# 阶段划分

阶段定义在`ngx_http_phases`枚举中，下面是简单的介绍。

- `NGX_HTTP_POST_READ_PHASE`这个是第一个阶段，用于处理一些必须在所有业务发生之前处理的通用功能。发生的时机在解析完成请求头，并确定server之后。几个关键的顺序链如下`ngx_http_process_request_headers()->ngx_http_process_request()->ngx_http_core_run_phases()`这几个函数直接有部分是异步调用。
- `NGX_HTTP_SERVER_REWRITE_PHASE`这个阶段还在确定loc之前，当前代码用于rewrite模块的部分初始化工作。也可以用于对srv对象的改写。
- `NGX_HTTP_FIND_CONFIG_PHASE`这个是内置阶段，不允许模块扩展，用于查找确定loc。
- `NGX_HTTP_REWRITE_PHASE`这个是扩展阶段，用于实现改写URL类功能，是rewrite模块的重要处理阶段。
- `NGX_HTTP_POST_REWRITE_PHASE`是个内置阶段，用于通过改写后的URL确定loc。
- `NGX_HTTP_PREACCESS_PHASE`是访问控制前的阶段，这个阶段已经确定loc，用于设置流控类参数。
- `NGX_HTTP_ACCESS_PHASE`是访问控制阶段，用于实现访问控制相关功能。
- `NGX_HTTP_POST_ACCESS_PHASE`是个内置阶段，根据访问控制的结果，关闭请求。
- `NGX_HTTP_PRECONTENT_PHASE`这是内容生成前的一个阶段，此阶段已经通过访问控制，可以实现一些特殊的功能，例如请求镜像等。
- `NGX_HTTP_CONTENT_PHASE`是响应内容生成阶段，是最核心的一个阶段。
- `NGX_HTTP_LOG_PHASE`是访问日志记录阶段，也是请求的最后处理阶段。

# 阶段的处理逻辑

在ngxin中除了上面的处理阶段，每个处理阶段还会有多个处理函数。而且全部的处理流程并不是一次性调用完成的，而是可能多次异步调用组合完成。核心的处理流程在`ngx_http_core_run_phases()`中。每个阶段的checker函数是相同的，但会有多个不同的handler函数。常见的checker函数包括`ngx_http_core_generic_phase()\ngx_http_core_rewrite_phase()\ngx_http_core_access_phase()`等。

另外在r对象中还保存这个关键的变量`r->phase_handler`，用于同步多次异步事件调用进度。在check函数中有几个常见的操作在这类说明一下

```c
r->phase_handler++; // 进入下一处理函数
return NGX_AGAIN;
```
```c
r->phase_handler = ph->next;// 进入下一阶段，当前阶段的其他处理函数不再执行
return NGX_AGAIN;
```
```c
ngx_http_finalize_request(r, rc); // 结束请求，退出到事件轮询中
return NGX_OK;
```
```c
return NGX_OK; // 退出到事件轮询中,下次事件触发时重当前处理函数再执行
```