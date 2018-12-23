本文介绍nginx的核心模块管理机制。主要涉及`ngx_modules[]`变量的使用。



# 模块初始化关键步骤

通过编译脚本生成`ngx_modules[]`与`ngx_module_names[]`这2个变量的具体值。

在`ngx_init_cycle()`之前调用`ngx_preinit_modules()`初始化模块的索引信息。

在`ngx_init_cycle()`中调用`ngx_cycle_modules（）`将模块信息保存在`cycle->modules[]`之中。

在`ngx_init_cycle()`的最后部分调用`ngx_init_modules()`回调各个模块定义的`init_module`方法。



# 关于ctx_index与index

`ngx_module_t.index`属性用于全部模块的索引，对应的是`ngx_modules[]`或`cycle->modules[]`的下标。

`ngx_module_t.ctx_index`属性用于不同类型模块，例如http模块、stream模块中的`cf->ctx[]`下标。

为了节省空间，并且保证reload失败时配置能正确回滚，`ctx_index`的值顺序与`index`的顺序是没有关系的。

