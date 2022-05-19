#### MySQL8.0.14开始支持coredump中排除buffer pool内存

- 优点
    1. 可以大大减少coredump的文件大小，减少磁盘空间占用。
    2. 可以加快问题分析速度。
    3. 因为buffer pool里面是数据，也可以减少数据泄漏风险（有时候需要提供coredump给专业人士分析。

- 如何控制

    1. 通过innodb_buffer_pool_in_core_file控制，默认为true，即buffer pool page在coredump中。

- 前置条件
    1. MySQL需要在8.0.14版本及以上。
    2. Linux内核需要在3.4版本及以上。（madvise()系统调用支持MADV_DONTDUMP宏。）

##### 实现原理(MySQL8.0.28)
    
1. 通过innodb选项innodb_buffer_pool_in_core_file控制
``` 
/* Boolean @@innodb_buffer_pool_in_core_file. */
bool srv_buffer_pool_in_core_file = TRUE;
```

2. 通过bool innobase_should_madvise_buf_pool();获取到用户设置的选项。
3. 通过madvise()系统调用设置，madvise()系统调用的功能是给使用内存的建议，前两个参数分别为地址，以及长度，第三个参数为建议参数。宏MADV_DONTDUMP建议在核心转储中排除由 addr 和 length 指定的范围内的页面。 这在具有大量内存且已知在核心转储中无用的应用程序中很有用。 MADV_DONTDUMP 的效果优先于通过 /proc/PID/coredump_filter 文件设置的位掩码
```
 for (ulint i = 0; i < srv_buf_pool_instances; i++) {
      buf_pool_t *buf_pool = buf_pool_from_array(i);
      bool success = buf_pool_should_madvise ? buf_pool->madvise_dont_dump()
                                             : buf_pool->madvise_dump();
      if (!success) {
        innobase_disable_core_dump();
        break;
      }
  }
```

##### 参考

1. https://man7.org/linux/man-pages/man2/madvise.2.html
2. https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool-in-core-file.html
3. MySQL8.0.28 source code
