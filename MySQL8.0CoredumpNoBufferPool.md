#### MySQL8.0.14开始支持coredump中排除buffer pool内存

- 作用是可以大大减少coredump的文件大小，加快排查问题速度
- 通过innodb_buffer_pool_in_core_file控制，默认为true，即buffer pool page在coredump中。
- 前置条件
    1. MySQL需要在8.0.14版本及以上。
    2. Linux内核需要在3.4版本及以上。（madvise()系统调用支持MADV_DONTDUMP宏。）

##### 实现原理
    
