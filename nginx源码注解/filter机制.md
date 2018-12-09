本文介绍nginx中过滤器的设计，以及功能模块如何使用过滤器。

# nginx中的过滤器

nginx中一共设计了3组过滤器，分别用于处理请求体、响应头、响应体。对应的代码声明再`ngx_http.c`中。

```c
ngx_http_output_header_filter_pt  ngx_http_top_header_filter;
ngx_http_output_body_filter_pt    ngx_http_top_body_filter;
ngx_http_request_body_filter_pt   ngx_http_top_request_body_filter;
```

对于过滤器设计一般要特别关注最后一个filter，一般会关心到核心流程。中间的filter一般用于模块之间扩展，模块扩展时要注意顺序问题。

# 请求filter

请求体的过滤器用处比较少，比较少有模块对其扩展。一个重要的filter是ngx_http_request_body_save_filter（）`，它默认是最后一个filter。行为是将请求数据缓存到临时文件中。

# 响应filter

响应数据分为heder与body两部份分别处理，处理方式类似。

2个过滤器链分别在`ngx_http_send_header()`与`ngx_http_output_filter()`中调用，content阶段的模块根据流程需要选择发送的数据。

模块扩展的方式类似，一般在postconfiguration阶段，将需要新增的filter链接到当前已有链的最前端。实例如下

```C
ngx_http_next_header_filter = ngx_http_top_header_filter;
ngx_http_top_header_filter = ngx_http_addition_header_filter;
ngx_http_next_body_filter = ngx_http_top_body_filter;
ngx_http_top_body_filter = ngx_http_addition_body_filter;
```

然后在过滤器的处理函数中，通过调用`ngx_http_next_xxxx_filter()`实现后续过滤器的调用。示例如下

```C
static ngx_int_t
ngx_http_addition_body_filter(ngx_http_request_t *r, ngx_chain_t *in) {
...
return ngx_http_next_body_filter(r, in);
...
}
```

对于响应类的过滤器，最后一级的过滤器是` ngx_http_header_filter()`与`ngx_http_write_filter()`，都是讲数据通过send()调用发送到客户端。

# 过滤器顺序

由于是每个模块单独注册，所以过滤器的顺序与模块的初始化顺序相关。而nginx的设计中静态模块的初始化顺序其实是取决于编译顺序的。所以要注意编译选型的顺序，或者在模块的config文件中显示设置模块顺序。

# 注意事项

根据实践经验在过滤器的使用上有几点要注意的

- 一个请求body_filter要会被多次调用。由于body可能比较大，根据事件的情况，并不会按固定的大小，或数据全部缓存后才调用body_filter，所以编写模块的body_filter时要特别注意。每次body_filter调用的数据buf是不完整的长度不可控的，要注意需要处理的数据被分割到多次调用中。
- body_filter要注意数据是否经过压缩，如果必须对响应的数据进行解析，要特别注意数据被压缩的场景。如果解压和数据处理使用不同模块处理，还要关注模块的调用顺序。