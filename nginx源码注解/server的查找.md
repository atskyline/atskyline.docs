本文介绍在nginx中http请求接收后，如何查找到对应的srv_conf对象。

# server_name与listen配置项

影响server选择的最主要配置项除了server_name，还有listen。有几个关键点

- 请求的IP:PORT需要与server中的listen一致。
- server_name的匹配优先级是：精确匹配>后缀匹配>前缀匹配>正则匹配

- 如果无法通过server_name无法匹配合适的server，请求将在IP:PORT对应的默认server中处理。（每一组监听IP:PORT都有一个默认server，通过default_server设置或第一个对应设置监听的server）。

# server的查找过程

- 在`ngx_http_init_connection()`中先对connect设置一个默认的server
- 对于SSL请求，在SNI的回调用会使用`ngx_http_ssl_servername()`重新设置connect中的server
- 在`ngx_http_create_request()`时request使用hc的server结果
- 在`ngx_http_process_request_line()`和`ngx_http_process_request_header()`解析获得请求行或请求头中的HOST信息后将调用`ngx_http_set_virtual_server()`查找最终使用的server对象。

# 修改server的核心代码

```c
// cscf是新查找到的server对象
r->srv_conf = cscf->ctx->srv_conf;
r->loc_conf = cscf->ctx->loc_conf;
```

