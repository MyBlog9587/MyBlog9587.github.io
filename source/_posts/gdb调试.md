---
title: gdb调试
date: 2023-05-23 17:22:34
categories: [工具, gdb]
tags:
- gdb
- private
---


# 开启core 
ulimit -c 查看 如果是0表示没有开启

ulimit -c unlimited 开启

- ulimit -c 永久生效
修改/etc/security/limits.conf
增加如下信息：
`* soft nofile unlimited` --软限制
`* hard nofile unlimited`--硬限制


然后重启生效。

# gcc生成core文件
编译加 `-g`

# 设置code dump 文件生成路径
- 设置core dump文件位置
`vi /etc/sysctl.conf`

修改（添加）如下两个变量
```bash
kernel.core_pattern =/var/core/core_%e_%p
kernel.core_uses_pid= 0
```
这里是改为生成目录在/var/core/，%e代表程序名称，%p是进程ID 如果想直接生成在可执行文件相同目录，前面不要加任何目录，直接 `kernel.core_pattern =core_%e_%p`

- 让修改生效
`sysctl -p/etc/sysctl.conf`


# 利用gdb 查死锁
1. 程序使用 `-g` 编译
2. 使用gdb + 可执行程序 运行
3. `r`运行 复现死锁
4. `ctrl + c`  停止
5. 查看线程栈信息，`info stack`，这个命令只能查看当前正在运行的某个线程的栈信息 (关注 **__GI___pthread_mutex_lock** )
```bash
(gdb) info stack
#0  __lll_lock_wait () at ../nptl/sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  0x00007ffff79bfd02 in _L_lock_791 () from /lib64/libpthread.so.0
#2  0x00007ffff79bfc08 in __GI___pthread_mutex_lock (mutex=0x627820 <mux_timer>) at pthread_mutex_lock.c:64
#3  0x00000000004114a8 in timer_register (equipment=0x6c4278, interval=120, argument=0x6c4270,
    timer_callback=0x41ab3b <dnsQueryInfolist_result_back>) at timer.c:200
#4  0x000000000041a5d5 in dnsQueryInfolist_stats_init (dnsQueryInfo_stats=0x0, commandId=1234567890,
    dnsId=0x629858 <switch_list+4248> "1", erverRoomId=0x62986b <switch_list+4267> "100",
    serverId=0x62987b <switch_list+4283> "200", visitTime=0x7fffffffccc0 "11-Jul-2019 21:21:46.793", timeout=120)
    at log_dnsQueryInfo_list.c:100
#5  0x00000000004059d7 in handle_string_log (keeper=0x64cb50,
    line=0x7fffffffd5b0 "11-Jul-2019 21:21:46.793 client 10.1.107.111 57682: view default: www.10086.com IN A NOERROR + NS NE NT ND NC NH 41484 58.251.121.110 NA\n", key_domain=0x62f010) at log_topn.c:1588
#6  0x0000000000409f8b in main (argc=2, argv=0x7fffffffe348) at log_topn.c:3115
```
6. info threads查看所有线程id，前面有*的，代表正在运行的线程，其他没有*的极有可能是在阻塞或者死锁的 （关注 **__lll_lock_wait** ）
```bash
(gdb) info threads
  Id   Target Id         Frame
  6    Thread 0x7ffff53bd700 (LWP 6828) "log_server" 0x00007ffff79c422d in read () at ../sysdeps/unix/syscall-template.S:81
  5    Thread 0x7ffff5bbe700 (LWP 6827) "log_server" __lll_lock_wait ()
    at ../nptl/sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
  4    Thread 0x7ffff63bf700 (LWP 6826) "log_server" 0x00007ffff747f42d in nanosleep ()
    at ../sysdeps/unix/syscall-template.S:81
  3    Thread 0x7ffff6bc0700 (LWP 6825) "log_server" 0x00007ffff747f42d in nanosleep ()
    at ../sysdeps/unix/syscall-template.S:81
  2    Thread 0x7ffff73c1700 (LWP 6824) "log_server" 0x00007ffff79c44ad in accept () at ../sysdeps/unix/syscall-template.S:81
* 1    Thread 0x7ffff7fcc740 (LWP 6820) "log_server" __lll_lock_wait ()
    at ../nptl/sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
```
7. 使用 `thread apply all bt` （`thread apply all`  命令，gdb会让所有线程都执行这个命令，比如命令为bt，查看所有线程的具体的栈信息）(关注 **__GI___pthread_mutex_lock** )
```bash
(gdb) thread apply all bt

Thread 6 (Thread 0x7ffff53bd700 (LWP 6828)):
#0  0x00007ffff79c422d in read () at ../sysdeps/unix/syscall-template.S:81
#1  0x000000000040bbd6 in socket_read (s=0x7fffe80008c0, buffer=0x7ffff53bcea0, len=4) at zip_socket.c:155
#2  0x0000000000407e93 in socket_thread (args=0x7fffe80008e0) at log_topn.c:2381
#3  0x00007ffff79bddc5 in start_thread (arg=0x7ffff53bd700) at pthread_create.c:308
#4  0x00007ffff74b821d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:113

Thread 5 (Thread 0x7ffff5bbe700 (LWP 6827)):
#0  __lll_lock_wait () at ../nptl/sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  0x00007ffff79bfd02 in _L_lock_791 () from /lib64/libpthread.so.0
#2  0x00007ffff79bfc08 in __GI___pthread_mutex_lock (mutex=0x627820 <mux_timer>) at pthread_mutex_lock.c:64
#3  0x0000000000410fa8 in check_time_refresh (equipment=0x6d4348) at timer.c:38
#4  0x0000000000418b12 in filterResultlist_result_back (arg=0x6d4340) at log_filterResult_list.c:591
#5  0x0000000000411364 in pth_timer_loop (arg=0x0) at timer.c:145
#6  0x00007ffff79bddc5 in start_thread (arg=0x7ffff5bbe700) at pthread_create.c:308
#7  0x00007ffff74b821d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:113

Thread 4 (Thread 0x7ffff63bf700 (LWP 6826)):
#0  0x00007ffff747f42d in nanosleep () at ../sysdeps/unix/syscall-template.S:81
#1  0x00007ffff747f2c4 in __sleep (seconds=0) at ../sysdeps/unix/sysv/linux/sleep.c:137
#2  0x0000000000409a42 in time_thread (args=0x62f010) at log_topn.c:2948
#3  0x00007ffff79bddc5 in start_thread (arg=0x7ffff63bf700) at pthread_create.c:308
#4  0x00007ffff74b821d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:113

Thread 3 (Thread 0x7ffff6bc0700 (LWP 6825)):
#0  0x00007ffff747f42d in nanosleep () at ../sysdeps/unix/syscall-template.S:81
#1  0x00007ffff747f2c4 in __sleep (seconds=0) at ../sysdeps/unix/sysv/linux/sleep.c:137
#2  0x0000000000407e42 in qps_thread (args=0x64cb50) at log_topn.c:2363
#3  0x00007ffff79bddc5 in start_thread (arg=0x7ffff6bc0700) at pthread_create.c:308
#4  0x00007ffff74b821d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:113

---Type <return> to continue, or q <return> to quit---
Thread 2 (Thread 0x7ffff73c1700 (LWP 6824)):
#0  0x00007ffff79c44ad in accept () at ../sysdeps/unix/syscall-template.S:81
#1  0x000000000040bb3c in socket_accept (s=0x64cb68) at zip_socket.c:133
#2  0x0000000000409ce1 in command_handler (args=0x64cb50) at log_topn.c:3028
#3  0x00007ffff79bddc5 in start_thread (arg=0x7ffff73c1700) at pthread_create.c:308
#4  0x00007ffff74b821d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:113

Thread 1 (Thread 0x7ffff7fcc740 (LWP 6820)):
#0  __lll_lock_wait () at ../nptl/sysdeps/unix/sysv/linux/x86_64/lowlevellock.S:135
#1  0x00007ffff79bfd02 in _L_lock_791 () from /lib64/libpthread.so.0
#2  0x00007ffff79bfc08 in __GI___pthread_mutex_lock (mutex=0x627820 <mux_timer>) at pthread_mutex_lock.c:64
#3  0x00000000004114a8 in timer_register (equipment=0x6c4278, interval=120, argument=0x6c4270,
    timer_callback=0x41ab3b <dnsQueryInfolist_result_back>) at timer.c:200
#4  0x000000000041a5d5 in dnsQueryInfolist_stats_init (dnsQueryInfo_stats=0x0, commandId=1234567890,
    dnsId=0x629858 <switch_list+4248> "1", erverRoomId=0x62986b <switch_list+4267> "100",
    serverId=0x62987b <switch_list+4283> "200", visitTime=0x7fffffffccc0 "11-Jul-2019 21:21:46.793", timeout=120)
    at log_dnsQueryInfo_list.c:100
#5  0x00000000004059d7 in handle_string_log (keeper=0x64cb50,
    line=0x7fffffffd5b0 "11-Jul-2019 21:21:46.793 client 10.1.107.111 57682: view default: www.10086.com IN A NOERROR + NS NE NT ND NC NH 41484 58.251.121.110 NA\n", key_domain=0x62f010) at log_topn.c:1588
#6  0x0000000000409f8b in main (argc=2, argv=0x7fffffffe348) at log_topn.c:3115
```

> 需要注意的是：如果系统运行着很多线程的时候，不可能使用thread  id（这个id比如上面的1 ,2 ,3, ,4, 5, 6），因此最好还是直接使用thread apply all bt

8. 看到的lock_wait就是被死锁的线程


# 用gdb 运行调试 带参数的程序 过程

- 正常启动：
`./log_server /usr/local/zddi/dns/log/query.log.`

- 前提：
    - 运行程序 编译时 需要加 -g
    - 程序如果带参数，可以在gdb 中使用set args 指定
`(gdb) set args /usr/local/zddi/dns/log/query.log. `

- 带参数 运行：
 1. 如果运行gdb 时指定了可执行文件，那么需要在进入以后设置参数`set args`后，再运行`run`
 2. 如果运行gdb 时未指定可执行文件，那么 需要在进入以后通过`file` 或 `exec`
 3. 可以在运行时 指定可执行程序 以及 指定参数

```bash
# gdb log_server
(gdb) set args /usr/local/zddi/dns/log/query.log.
(gdb) r
```

```bash
# gdb
(gdb) file log_server
Reading symbols from /root/zddi/build/repo/log_server/log_server...done.
(gdb) set args /usr/local/zddi/dns/log/query.log.
(gdb) r
```

```bash
# gdb --args log_server /usr/local/zddi/dns/log/query.log.
(gdb) r
```

- 查看设置的参数：
`(gdb) show args`

> 如果存在Makefile 可以在gdb中直接执行make 命令进行编译，并运行r 重新运行（前提是要gdb要运行在编译环境下，并且可执行文件在当前路径下）


# gdb调试cpu占用率高
1. 启动进程，top查看进程pid
2. top -H -p 进程pid,观察占用cpu较高的pid
3. gdb icdn 进程pid
4. 如果是多线程thread 线程标号
5. bt 找到在多少行那个函数


# attach调试：(可以调试正在运行的程序)
- gdb（进入gdb）
- gdb pid（通过ps aux查看的程序id）
- bt（查看当前状态）
![C0EFA4EC-0725-4344-A5AD-FF09BA246DA5](https://user-images.githubusercontent.com/32731294/156599014-a6da0e76-e1a3-4b9c-a854-40d946b80bef.png)
- 使用 f  线程id（可以查看某个线程 ）
- 查看具体变量的内容（利用地址）
- 追加 zymap内容（目标）
```bash
(gdb) p app
$1 = (struct tsdb_app *) 0x19882f0
(gdb) p app->settings
$2 = {next = 0x1988938, prev = 0x199aaf8}
```

```bash
* (gdb) p ((struct tsdb_setting *)((char *)(0x199aaf8)-(unsigned long)(&((struct tsdb_setting *)0)->app_next)))
            $3 = (struct tsdb_setting *) 0x199aa50

* (gdb) p *((struct tsdb_setting *)((char *)(0x199aaf8)-(unsigned long)(&((struct tsdb_setting *)0)->app_next)))
            $4 = {init = 0, crc = 0, name = 0x1988540 "secops", listener_name = "ztms", '\000' <repeats 59 times>, dir = 0x199ab20 "/var/taw/statistics/secops", field_count = 11,
field_size = 44, statics_field_count = 1, statics_field_size = 8, intervals = 1, fields = 0x199ab50, trunk_time = 600, trunk_size = 20971520, statistics = 0x199db00,writer = 0x0, listener_next = {next = 0x199aae8, prev = 0x199aae8}, app_next = {next = 0x1988368, prev = 0x199dd68}, pattern = 0x0}

* (gdb) p *((struct tsdb_setting *)((char *)(0x1988938)-(unsigned long)(&((struct tsdb_setting *)0)->app_next)))
            $5 = {init = 0, crc = 0, name = 0x1988cb0 "ads", listener_name = "flexer", '\000' <repeats 57 times>, dir = 0x1988960 "/var/taw/statistics/ads", field_count = 9,field_size = 36, statics_field_count = 19, statics_field_size = 152, intervals = 1, fields = 0x1996000, trunk_time = 600, trunk_size = 20971520,statistics = 0x198dee0, writer = 0x0, listener_next = {next = 0x1988928, prev = 0x1988928}, app_next = {next = 0x19967d8, prev = 0x1988368}, pattern = 0x0}  
```

```bash
(gdb) p app->settings->next                                                                                   
        $6 = (struct ts_list_head *) 0x1988938
(gdb) p app->settings->next->next
        $7 = (struct ts_list_head *) 0x19967d8
```

```bash
(gdb) p *((struct tsdb_setting *)((char *)(app->settings->next)-(unsigned long)(&((struct tsdb_setting *)0)->app_next)))
        $10 = {init = 0, crc = 0, name = 0x1988cb0 "ads", listener_name = "flexer", '\000' <repeats 57 times>, dir = 0x1988960 "/var/taw/statistics/ads", field_count = 9,field_size = 36, statics_field_count = 19, statics_field_size = 152, intervals = 1, fields = 0x1996000, trunk_time = 600, trunk_size = 20971520,statistics = 0x198dee0, writer = 0x0, listener_next = {next = 0x1988928, prev = 0x1988928}, app_next = {next = 0x19967d8, prev = 0x1988368}, pattern = 0x0}

(gdb) p *((struct tsdb_setting *)((char *)(app->settings->next->next)-(unsigned long)(&((struct tsdb_setting *)0)->app_next)))
        $11 = {init = 0, crc = 6648852076915354675, name = 0x1988500 "antivirus", listener_name = "antivirus", '\000' <repeats 54 times>,dir = 0x1996800 "/var/taw/statistics/antivirus", field_count = 15, field_size = 60, statics_field_count = 3, statics_field_size = 24, intervals = 1,fields = 0x1996a70, trunk_time = 600, trunk_size = 20971520, statistics = 0x199b890, writer = 0x0, listener_next = {next = 0x19883c8, prev = 0x19883c8}, app_next = {next = 0x19968d8, prev = 0x1988938}, pattern = 0x0}
```

```bash
(gdb) p ((struct tsdb_setting *)((char *)(app->settings->next->next)-(unsigned long)(&((struct tsdb_setting *)0)->app_next)))       
        $13 = (struct tsdb_setting *) 0x1996730
```

```bash
(gdb) p ((struct tsdb_setting *) 0x1996730)->statistics
        $14 = (struct tsdb_stats *) 0x199b890
```

```bash
(gdb) p ((struct tsdb_setting *) 0x1996730)->statistics->simap.str_map
        $16 = (ZYMap *) 0x7fdb7e49a380
```

```bash
(gdb) p *((struct tsdb_setting *) 0x1996730)->statistics->simap.str_map     
        $18 = {val_list = {next = 0x7fdb7af16380, prev = 0x7fdb7af66480}, len = {cnt = 34}, ref = {cnt = 1}, rwlock = {cnt = 0}, root = {0x7fdb7af16580, 0x7fdb7af16780, 0x7fdb7af16480, 0x7fdb7af16880, 0x7fdb7af1ef80, 0x7fdb7af16680, 0x7fdb7af16380, 0x0}, max_size = {{cnt = 22}, {cnt = 4}, {cnt = 0}, {cnt = 2}, {cnt = 0}, {cnt = 3}, {cnt = 0}, {cnt = 0}}, tree_len = {{cnt = 22}, {cnt = 4}, {cnt = 1}, {cnt = 2}, {cnt = 1}, {cnt = 3}, {cnt = 1}, {cnt = 0}}, flags = 1}
(gdb) p zyprintf("${1@MP}",((struct tsdb_setting *) 0x1996730)->statistics->simap.str_map)
        $19 = 903
    打印出结果: "testaawafdsaf (3": 29.000000 
```

# debugme调试:



# gdb 调试 clickhouse core dump

## 查看启用版本以及运行参数：
```bash
# ps aux|grep clickhouse
root     128471  0.0  0.0 112812   976 pts/0    S+   10:02   0:00 grep --color=auto clickhouse
clickho+ 173192  326 27.4 528776420 36144064 ?  Ssl   2022 1841141:13 clickhouse-server --daemon --pid-file=/var/run/clickhouse-server/clickhouse-server.pid --config-file=/etc/clickhouse-server/config.xml
```


## 查看启动位置：
```bash
# whereis clickhouse-server
clickhouse-server: /usr/bin/clickhouse-server /etc/clickhouse-server

# ll /usr/bin/clickhouse-server
lrwxrwxrwx 1 root root 10 Jun 24  2020 /usr/bin/clickhouse-server -> clickhouse
# whereis clickhouse
clickhouse: /usr/bin/clickhouse /usr/share/clickhouse
# ll /usr/bin/clickhouse
-rwxr-xr-x 1 root root 280453752 Jun 11  2020 /usr/bin/clickhouse
```

## 使用gdb启动clickhouse服务：
```bash
$ 指定clickhouse用户启动
# sudo -u clickhouse gdb

$ 指定启动文件
(gdb) file /usr/bin/clickhouse 
$ 指定启动参数
(gdb) set args server --config-file=/etc/clickhouse-server/config.xml
$ 运行
(gdb) r 
(gdb)
......
2023.03.30 10:15:29.499293 [ 87833 ] {} <Debug> DiskLocal: Reserving 1.00 MiB on disk `disk5`, having unreserved 5.14 TiB.
2023.03.30 10:15:29.499680 [ 87882 ] {} <Error> void DB::StorageKafka::threadFunc(): Code: 36, e.displayText() = DB::Exception: external dictionary 'ipip4' not found, Stack trace (when copying this message, always include the lines below):
......
Program received signal SIGUSR1, User defined signal 1.
[Switching to Thread 0x7ffa52775700 (LWP 88494)]
pthread_cond_wait@@GLIBC_2.3.2 () at ../nptl/sysdeps/unix/sysv/linux/x86_64/pthread_cond_wait.S:185
185     62:     movl    (%rsp), %edi
(gdb)

$ run以前设置程序收到SIGUSR1信号时，不会退出就可以了
(gdb) handle SIGUSR1  nostop
Signal        Stop      Print   Pass to program Description
SIGUSR1       No        Yes     Yes             User defined signal 1
(gdb)
(gdb) r 
(gdb)
......
(gdb) 
---Type <return> to continue, or q <return> to quit---
$ 阻塞住等待用户操作，解决办法
(gdb) set pagination off
(gdb) 
.......
Program received signal SIGSEGV, Segmentation fault.
[Switching to Thread 0x7fffd6bf5700 (LWP 103631)]
0x000000000ba97830 in memcpy ()
(gdb) bt
#0  0x000000000ba97830 in memcpy ()
#1  0x0000000008a07548 in DB::DataTypeString::serializeBinaryBulk(DB::IColumn const&, DB::WriteBuffer&, unsigned long, unsigned long) const ()
#2  0x0000000008f63c95 in DB::MergeTreeDataPartWriterWide::writeColumn(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > const&, DB::IDataType const&, DB::IColumn const&, std::__1::set<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> >, std::__1::less<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > >, std::__1::allocator<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char> > > >&) ()
#3  0x0000000008f646eb in DB::MergeTreeDataPartWriterWide::write(DB::Block const&, DB::PODArray<unsigned long, 4096ul, Allocator<false, false>, 15ul, 16ul> const*, DB::Block const&, DB::Block const&) ()
#4  0x0000000008fdee72 in DB::MergedBlockOutputStream::writeImpl(DB::Block const&, DB::PODArray<unsigned long, 4096ul, Allocator<false, false>, 15ul, 16ul> const*) ()
#5  0x0000000008f2fd45 in DB::MergeTreeDataMergerMutator::mergePartsToTemporaryPart(DB::FutureMergedMutatedPart const&, DB::MergeListEntry&, DB::TableStructureReadLockHolder&, long, std::__1::unique_ptr<DB::IReservation, std::__1::default_delete<DB::IReservation> > const&,bool, bool) ()
#6  0x0000000008e24b53 in DB::StorageReplicatedMergeTree::tryExecuteMerge(DB::ReplicatedMergeTreeLogEntry const&) ()
#7  0x0000000008e51b5b in DB::StorageReplicatedMergeTree::executeLogEntry(DB::ReplicatedMergeTreeLogEntry&) ()
#8  0x0000000008e51f9c in DB::StorageReplicatedMergeTree::queueTask()::{lambda(std::__1::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&)#2}::operator()(std::__1::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&) const [clone .isra.0] ()
#9  0x000000000901a1de in DB::ReplicatedMergeTreeQueue::processEntry(std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()>, std::__1::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&, std::__1::function<bool (std::__1::shared_ptr<DB::ReplicatedMergeTreeLogEntry>&)>)
    ()
#10 0x0000000008e08e37 in DB::StorageReplicatedMergeTree::queueTask() ()
#11 0x0000000008ec1481 in DB::BackgroundProcessingPool::threadFunction() ()
#12 0x0000000008ec1e2d in _ZZN20ThreadFromGlobalPoolC4IZN2DB24BackgroundProcessingPoolC4EiRKNS2_12PoolSettingsEPKcS7_EUlvE_JEEEOT_DpOT0_ENKUlvE_clEv ()
#13 0x0000000005013ead in ThreadPoolImpl<std::__1::thread>::worker(std::__1::__list_iterator<std::__1::thread, void*>) ()
#14 0x000000000501251f in void* std::__1::__thread_proxy<std::__1::tuple<std::__1::unique_ptr<std::__1::__thread_struct, std::__1::default_delete<std::__1::__thread_struct> >, void ThreadPoolImpl<std::__1::thread>::scheduleImpl<void>(std::__1::function<void ()>, int, std::__1::optional<unsigned long>)::{lambda()#3}> >(std::__1::tuple<std::__1::unique_ptr<std::__1::__thread_struct, std::__1::default_delete<std::__1::__thread_struct> >, void ThreadPoolImpl<std::__1::thread>::scheduleImpl<void>(std::__1::function<void ()>, int, std::__1::optional<unsigned long>)::{lambda()#3}>) ()
#15 0x00007ffff72eeea5 in start_thread (arg=0x7fffd6bf5700) at pthread_create.c:307
#16 0x00007ffff7b0b8dd in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:111

(gdb) f 1
#1  0x0000000008a07548 in DB::DataTypeString::serializeBinaryBulk(DB::IColumn const&, DB::WriteBuffer&, unsigned long, unsigned long) const ()

(gdb) info frame
Stack level 1, frame at 0x7fffd6bee8c0:
rip = 0x8a07548 in DB::DataTypeString::serializeBinaryBulk(DB::IColumn const&, DB::WriteBuffer&, unsigned long, unsigned long) const; saved rip 0x8f63c95
called by frame at 0x7fffd6beea70, caller of frame at 0x7fffd6bee840
Arglist at 0x7fffd6bee838, args:
Locals at 0x7fffd6bee838, Previous frame's sp is 0x7fffd6bee8c0
Saved registers:
  rbx at 0x7fffd6bee888, rbp at 0x7fffd6bee890, r12 at 0x7fffd6bee898, r13 at 0x7fffd6bee8a0, r14 at 0x7fffd6bee8a8, r15 at 0x7fffd6bee8b0, rip at 0x7fffd6bee8b8
```

### handle
在GDB中定义一个信号处理。信号可以以SIG开头或不以 SIG开头，可以用定义一个要处理信号的范围（如：SIGIO-SIGKILL，表示处理从SIGIO信号到SIGKILL的信号，其中包括SIGIO， SIGIOT，SIGKILL三个信号），也可以使用关键字all来标明要处理所有的信号。一旦被调试的程序接收到信号，运行程序马上会被GDB停住，以 供调试。其可以是以下几种关键字的一个或多个。

* nostop：当被调试的程序收到信号时，GDB不会停住程序的运行，但会打出消息告诉你收到这种信号。
* stop：当被调试的程序收到信号时，GDB会停住你的程序。
* print：当被调试的程序收到信号时，GDB会显示出一条信息。
* noprint：当被调试的程序收到信号时，GDB不会告诉你收到信号的信息。
* pass
* noignore：当被调试的程序收到信号时，GDB不处理信号。这表示，GDB会把这个信号交给被调试程序会处理。
* nopass
* ignore：当被调试的程序收到信号时，GDB不会让被调试程序来处理这个信号。

info signals
info handle
查看有哪些信号在被GDB检测中


