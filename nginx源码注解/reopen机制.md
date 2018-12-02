本文主要分析nginx中的reopen机制是如何辅助实现日志回滚、信号的基本处理流程、master与worker之间的信号交互。

# nginx中的日志回滚

仔细观察nginx的配置项中发现并没有日志回滚的相关配置。其实nginx设计中有考虑日志回滚的需求。实现方式大致如下：

1. 重命名日志文件
2. 向master进程发生USR1信号
3. 归档重命名后的日志文件

```bash
mv access.log access.log.0
kill -USR1 `cat master.nginx.pid`
sleep 1
gzip access.log.0    # do something with access.log.0
```

官网链接：https://www.nginx.com/resources/wiki/start/topics/examples/logrotation/

原理大致为外部脚本重命名日志文件后，由于nginx内部维护的文件句柄没有更新，日志写入依然能记录到重命名后的文件中。当nginx收到USR1信号（nginx内部也称为reopen信号）后，重新打开日志文件，更新文件句柄，后续内容就可以重写写入旧文件中。

这个要求nginx在全局维护所有打开的文件句柄，并且能在master与worker进程都处理reopen信号。

# reopen信号处理流程

信号处理函数`ngx_signal_handler()`，接受到reopen信号后将全局变量`ngx_reopen`置1。

master进程主循环`ngx_master_process_cycle()`，检测到`ngx_reopen`置1后执行`ngx_reopen_files()`，并将信号传递到worker进程中。

worker进程住循环`ngx_worker_process_cycle`，检测到`ngx_reopen`置1后执行`ngx_reopen_files()`。

`ngx_reopen_files()`函数中遍历所有已打开的文件，将触发缓存flush进文件、重新打开文件获得新句柄、关闭原句柄。

# 文件管理

所有文件打开的信息保存在全局`cycle->open_files`中。

文件打开时使用`ngx_conf_open_file()`函数进行open。

在`ngx_init_cycle()`中的最后`/* open the new files */`会真正打开文件获取句柄。

