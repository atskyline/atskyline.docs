本文分析nginx源码中关于location的类型、存储结构、查找过程。



# location的类型

根据nginx文档http://nginx.org/en/docs/http/ngx_http_core_module.html#location说明。可以将location分为以下几类：

- 精确匹配 `=修饰符+前缀`
- 内部loc`@修饰符+前缀`

- 不执行正则的前缀匹配 `^~修饰符+前缀`
- 正则匹配 `~修饰符+正则`或`~*修饰符+正则`
- 前缀匹配`无修饰符+前缀`

优先级大致上也是由上而下。

具体的查找逻辑如下：

- 在所有前缀loc中查找到最长匹配locA
- 如果locA有`=修饰符`或`^~修饰符`或`@修饰符`则返回locA
- 逐个匹配所有的正则loc，找到第一个正则匹配的locB，则返回locB
- 如果所有正则匹配都不满足，才返回locA

# location嵌套

nginx中的location是允许嵌套的，子location是存放在`ngx_http_core_loc_conf_s.locations`中的。

嵌套location存在一些约束详见`ngx_http_core_location()`函数。

- 精确匹配的loc不允许嵌套
- 内部loc不允许嵌套

嵌套对查找过程的影响主要体现在递归上，每次查找都是在同一层loc中查找，查找获得的loc如果还有子loc才用相同方式递归查找。



# location结构优化

在`ngx_http_core_location()`函数解析获得后loc是存放在queue结构体中的，这样每个请求查找loc的效率不够高，ngx将同层级的前缀loc列表重新排列后形成静态二叉树结构提升查找效率。对于正则loc也单独使用列表维护。

在`ngx_http_block()`的尾部，将对所有srv的中的loc结构进行结构优化。其中涉及到2个核心函数`ngx_http_init_locations()`与`ngx_http_init_static_location_trees()`。

`ngx_http_init_locations()`函数将queue结构中的loc重新排序，并获得`pclcf->regex_locations`关键列表。

`ngx_http_init_static_location_trees()`函数将所有的前缀loc处理成二叉树结构，并保存在`pclcf->static_locations`中。

排序和二叉树化都是发生在同层的loc中的，所以每一层loc都有对应的递归操作。

# location查找过程

请求的loc结构的查找确认是发生在`ngx_http_core_find_config_phase()`阶段中的。其中的核心函数是`ngx_http_core_find_location()`。

关键点在`pclcf->static_locations`或`pclcf->regex_locations`中选择与请求匹配的loc结构。并处理嵌套loc的递归处理。

逻辑部分在上面已经描述过了，此处不再复述。



