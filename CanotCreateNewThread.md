##### 应用端报错
```
Can't create a new thread(errno 11),if you are not out of available memory you can consult the manual for any possible OS dependent bug
```

##### 初步分析

无法创建新的线程，一般来说，是由于内存不够，如果不是因为这个，那么就可能是一些系统bug。MySQL历经过年，相对来说比较健壮。所以我们还是要在内存上面看这样的问题。

##### 复现场景1
1. 写一个python脚本不停的去创建链接
```
def main():
    conn_list = []
    while True:
        time.sleep(0.01)
        conn = Connect(host="127.0.0.1", port=3306, user="test_con", password="test_con")
        conn_list.append(conn)


if __name__ == "__main__":
    main()
```
2. 监控MySQLd的内存使用情况 
```
clear;while true; do s1=$(cat /proc/`pidof mysqld`/status|grep VmSize|awk '{print $2}'); sleep 1; s2=$(cat /proc/`pidof mysqld`/status|grep VmSize|awk '{print $2}'); echo -n "现在:" $((s2/1024)) "MB  增长:" ; echo -e $(($(($s2-$s1))/1024)) "MB\t" $(($s2-$s1)) "KB"; done;
```
输出如下
```
现在: 1341 MB  增长:0 MB	 0 KB
现在: 1341 MB  增长:0 MB	 0 KB
现在: 1341 MB  增长:0 MB	 0 KB
现在: 1341 MB  增长:0 MB	 0 KB
现在: 1341 MB  增长:0 MB	 0 KB
现在: 1341 MB  增长:0 MB	 0 KB
现在: 1341 MB  增长:0 MB	 0 KB
现在: 1360 MB  增长:19 MB	 19940 KB
现在: 1381 MB  增长:20 MB	 21020 KB
现在: 1399 MB  增长:18 MB	 18772 KB
现在: 1414 MB  增长:14 MB	 15344 KB
现在: 1427 MB  增长:12 MB	 13204 KB
现在: 1439 MB  增长:11 MB	 11508 KB
现在: 1449 MB  增长:9 MB	 10048 KB
现在: 1458 MB  增长:9 MB	 9512 KB
现在: 1467 MB  增长:8 MB	 8460 KB
现在: 1473 MB  增长:5 MB	 5948 KB
现在: 1480 MB  增长:7 MB	 7796 KB
现在: 1487 MB  增长:7 MB	 7268 KB
现在: 1494 MB  增长:6 MB	 6876 KB
现在: 1500 MB  增长:6 MB	 6212 KB
现在: 1506 MB  增长:5 MB	 6084 K
...
- 一种情况是没有到max connection 但是内存不够，直接挂了。
```
3. 一直增长, 直到MySQL奔溃也没抛出Can't create new thread
```
    packet = self._read_packet()
  File "/usr/local/python3/lib/python3.6/site-packages/pymysql/connections.py", line 692, in _read_packet
    packet_header = self._read_bytes(4)
  File "/usr/local/python3/lib/python3.6/site-packages/pymysql/connections.py", line 749, in _read_bytes
    CR.CR_SERVER_LOST, "Lost connection to MySQL server during query"
pymysql.err.OperationalError: (2013, 'Lost connection to MySQL server during query')
```

- 第二种情况到了 max_connection。mysql 有8000多线程
```
mysql> select count(*) from information_schema.processlist;
+----------+
| count(*) |
+----------+
|     7464 |
+----------+
1 row in set (0.18 sec)

Too many connections

```


##### 代码分析:x

```
// 基于mysql5.7.30
// conn_handler/connection_handler_per_thread.cc +43// 通过代码好像没有看到什么原因。
#ifndef DBUG_OFF
handle_error:
#endif // !DBUG_OFF

  if (error)
  {
    connection_errors_internal++;
    if (!create_thd_err_log_throttle.log())
      sql_print_error("Can't create thread to handle new connection(errno= %d)",
                      error);
    channel_info->send_error_and_close_channel(ER_CANT_CREATE_THREAD,
                                               error, true);
    Connection_handler_manager::dec_connection_count();
    DBUG_RETURN(true);
  }

  Global_THD_manager::get_instance()->inc_thread_created();
  DBUG_PRINT("info",("Thread created"));
  DBUG_RETURN(false);
```





##### 调整vm.max_map_count = 255
```
Last login: Tue May 10 21:00:13 on ttys003
/System/Library/Frameworks/Python.framework/Versions/2.7/Resources/Python.app/Contents/MacOS/Python: No module named virtualenvwrapper
virtualenvwrapper.sh: There was a problem running the initialization hooks.

If Python could not import the module virtualenvwrapper.hook_loader,
check that virtualenvwrapper has been installed for
VIRTUALENVWRAPPER_PYTHON=/usr/bin/python and that PATH is
set properly.
yixiang@yixiangdeMacBook-Pro ~ %
yixiang@yixiangdeMacBook-Pro ~ % ls
Applications			MySQL		    		envs
CProject			MyUnderstandProject.udb		go
Desktop				Pictures	    		go_path
Documents			Postman		    		go_projects
Downloads			Public		    		java_error_in_pycharm.hprof
GoProjects			PycharmProjects	    		log.0000000001
Library				PythonProjects	    		mysql_error_note
Movies				VueProject	    		tmp
Music				brew_instal	    		技术文档
yixiang@yixiangdeMacBook-Pro ~ % cd mysql_error_note
yixiang@yixiangdeMacBook-Pro mysql_error_note % ls
CanotCreateNewThread.md
yixiang@yixiangdeMacBook-Pro mysql_error_note %
yixiang@yixiangdeMacBook-Pro mysql_error_note %
#ifndef DBUG_OFF
handle_error:
#endif // !DBUG_OFF

  if (error)
  {
    connection_errors_internal++;
    if (!create_thd_err_log_throttle.log())
      sql_print_error("Can't create thread to handle new connection(errno= %d)",
                      error);
    channel_info->send_error_and_close_channel(ER_CANT_CREATE_THREAD,
                                               error, true);
    Connection_handler_manager::dec_connection_count();
    DBUG_RETURN(true);
  }

-- INSERT --
// 基于mysql5.7.30
##### 应用端报错
```
Can't create a new thread(errno 11),if you are not out of available memory you can consult the manual for any possible OS dependent bug
```

##### 初步分析

无法创建新的线程，一般来说，是由于内存不够，如果不是
因为这个，那么就可能是一些系统bug。MySQL历经过年，相
对来说比较健壮。所以我们还是要在内存上面看这样的问题
。

##### 复现场景1
1. 写一个python脚本不停的去创建链接
```
def main():
    conn_list = []
    while True:
"CanotCreateNewThread.md" 108L, 3454C
    Connection_handler_manager::dec_connection_count();
    DBUG_RETURN(true);
  }

  Global_THD_manager::get_instance()->inc_thread_created();
  DBUG_PRINT("info",("Thread created"));
  DBUG_RETURN(false);
```





##### 通过谷歌发现，调整vm.max_map_count = 255

执行创建线程的脚本。错误出现。
```
pymysql.err.OperationalError: (1135, 'create a new thread (errno 11); if you are not out of available memory, you can consult the manual for a possible OS-dependent bug'
-- INSERT (paste) --
```

实际上是超过了操作系统的vm.max_map_count的限制。但是没有发现是创建的时候哪里抛出的。倒像是创建之前检测的时候。DEBUG_EXECUTE_IF那个宏后面的go handler_error


堆栈信息.
```
Connection_acceptor<Mysqld_socket_listener>::connection_event_loop
  Connection_handler_manager::process_new_connection
    Per_thread_connection_handler::add_connection
    
```
