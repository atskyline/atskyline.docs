近日看《许式伟的架构课》中一段关于系统分解的描述所有思考，略记于此。



原文片段如下：

> 系统设计，简单来说就是 “对系统进行分解” 的能力。这个阶段核心要干的事情，就是明确子系统的职责边界和接口协议，把整个系统的大框架搭起来。
>
> 那么怎么分解系统？
>
> 首先我们需要明确的是分解系统优劣的评判标准。也就是说，我们需要知道什么样的系统分解方式是好的，什么样的分解方式是糟糕的。
>
> 最朴素的评判依据，是这样两个核心的点：
>
> - 功能的使用界面（或者叫接口），应尽可能符合业务需求对它的自然预期；
> - 功能的实现要高内聚，功能与功能之间的耦合尽可能低。
>
> 在软件系统中有多个层次的组织单元：子系统、模块、类、方法 / 函数。子系统如何分解模块？模块如何分解到更具体的类或函数？每一层的分解方式，都遵循相同的套路。也就是分解系统的方法论。
>
> ......
>
> 一个程序员的系统分解能力强不强，其实一眼就可以看出来。你都不需要看实现细节，只需要看他定义的模块、类和函数的使用接口。如果存在大量说不清业务意图的函数，或者存在大量职责不清的模块和类，就知道他基本上还处在搬砖阶段。
>
> 无论是子系统、模块、类还是函数，都有自己的业务边界。它的职责是否足够单一足够清晰，使用接口是否足够简单明了，是否自然体现业务需求（甚至无需配备额外的说明文档），这些都体现了架构功力。

完整原文 https://time.geekbang.org/column/article/117783

佩服作者对系统设计这样大问题的能简化到2个朴素的判断依据，而不是学院式的抽象含糊的说明。

近期由于工作内容需要，也常思考业务需求对接口的预期是什么？

对于服务端软件而言，接口在形式上比有UI交互的前端软件更加稳定且标准；然而对于业务需求的预期有时会缺少思考，设计上更倾向于技术的可行便利。

当然这两者并不矛盾，如果业务在接口标准上达成共识，就能取得很好的效果。

对于服务端软件，软件配置的组织与设计是一个重要而容易被忽视的接口。

以nginx软件为例，个人认为nginx配置方式的设计对业务需求的表达有时就不够自然清晰。甚至在某些场景下功能配置的耦合情况不少。

**示例1**

```nginx
# 示例例来源 http://nginx.org/en/docs/http/websocket.html
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        ...

        location /chat/ {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
        }
    }
}
```

上面这段示例的目的是实现WebSocket的代理，但从配置上看不出WebSocket的任何信息，而是依赖于配置人员对WebSocket升级流程与Nginx代理流程十分了解的基础上才能判断出这段配置能实现WebSocket代理。

**示例2**

```nginx
# 示例例来源 http://nginx.org/en/docs/http/converting_rewrite_rules.html
location / {
    root       /var/www/myapp.com/current/public;

    try_files  /system/maintenance.html
               $uri  $uri/index.html $uri.html
               @mongrel;
}

location @mongrel {
    proxy_pass  http://mongrel;
}
```

上面这段示例目的是优先尝试本地文件，不存在再像后端获取。其中的`location @mongrel {}`块部分是涉及到nginx开发中subrequest的概念。

subrequest的设计在代码内部很好的处理复杂请求的业务流程，是nginx开发中模块解耦，降低开发复杂度，并保持高性能的重要利器。

而这样一个重要特性却可能会造成配置片段的隔离性，当这个特性被大量使用时，配置文件看上去就显得特别复杂。与公司运维童鞋交流过程中也收到过反馈，说ngxin配置的格式，看着特别像代码而不是配置项。

关于subrequest更多信息可以参考http://nginx.org/en/docs/dev/development_guide.html#http_subrequests

---

然而这个视角的分析太片面。从另一个视角看，nginx的配置是提供了足够的灵活性和可扩展性，为实现各类不同的业务提供了可能性。nginx的配置不仅是业务需求的展示，而且还能承载业务逻辑，实现灵活扩展的功能。openresty的ngx_http_lua_module就是一个很好的实例。

``` nginx
# 示例来自 https://github.com/openresty/lua-nginx-module/
location = /request_body {
    client_max_body_size 50k;
    client_body_buffer_size 50k;

    content_by_lua_block {
        ngx.req.read_body()  -- explicitly read the req body
            local data = ngx.req.get_body_data()
            if data then
            ngx.say("body data:")
            ngx.print(data)
            return
            end

            -- body may get buffered in a temp file:
            local file = ngx.req.get_body_file()
            if file then
            ngx.say("body is in file ", file)
            else
            ngx.say("no body found")
            end
    }
}
```

在配置文件上承载业务逻辑，对于配置的管理与维护就提出了新的挑战。

孰是孰非，我并没有明确的倾向性，在实际项目中还要考虑组织结构、协作流程、历史数据、现有架构等等因素……才能设计出一个恰如其分的方案。

再看开头提到徐大牛提出的2个判断依据。接口满足业务预期；功能高内聚低耦合。用于判断方案的合理性确实是一个很不错的标准。