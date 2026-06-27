# c 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/c.md](../modules/c.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)

本文是概览的"深水区"补充。概览回答"是什么"、"有哪些子目录";本文只回答一件事——**这些代码跑起来的时候,一个字节从网卡进来、到 worker 处理、再写回客户端,中间到底发生了什么。** 所有结论都来自实读源码,给出真实 `#L` 行号。

> 先记住一个全局事实:`c/library/` 下同一个组件有 `trunk` / `tags/x.y.z` / `branches/xxx` 多份并存(net-io-server 6+ 份、newbench 6+ 份),那是从淘米 SVN 连版本目录一起泄漏的副本,**内容高度重复**。本文一律拿"某个具体 tag 一份"举证(优先 serverbench-1.1.0 / net-io-server-2.1.0 / newbench trunk),读懂一份即读懂全部。

---

## 一图看懂:进程/组件拓扑与数据流

`c/` 提供两套并行演进的"服务骨架",外加一组可独立链接的零件库。它们的运行拓扑(以 serverbench 为准)如下:

```
                       客户端(Flash / 别的 svr)
                              │ TCP/UDP
                              ▼
        ┌──────────────────────────────────────────────┐
        │  MAIN 进程(prog:[MAIN])                       │
        │   main.c: 解析 conf → load_bind_file →         │
        │   register_plugin(.so) → pipe_create →         │
        │   shmq_create → daemon 化 → fork()             │
        └───────────────┬───────────────┬───────────────┘
            fork 子进程  │               │ fork N 份
                         ▼               ▼
        ┌──────────────────────┐   ┌──────────────────────────┐
        │ CONN 进程(单进程)    │   │ WORKER 进程 ×worker_num   │
        │ conn_loop()          │   │ worker_run()              │
        │ epoll/poll 反应堆     │   │ 死循环:                   │
        │ 收包→handle_input()   │   │  shmq_pop(recvq)          │
        │  切出整包             │   │  → dll.handle_process()   │
        │  → shmq_push(recvq) ──┼──▶│  → shmq_push(sendq) ──┐   │
        │                      │   └───────────────────────┼──┘
        │ shmq_pop(sendq)◀─────┼───────────────────────────┘
        │  → schedule_output() │      (worker 写完 sendq 后
        │  → 写回 socket        │       write_pipe() 戳醒 CONN)
        └──────────────────────┘
              recvq / sendq 都是 mmap MAP_ANON|MAP_SHARED 的共享内存环形队列
              (fork 后父子共享同一块映射 + 同一组 sem 信号量)
```

**两条主线 + 一组零件:**
- **serverbench(C,"3g" 代)**:上图这套 `MAIN+CONN+WORKER+shmq` 经典模型,网络层是它自己写的 poll/epoll(`net.c`)。
- **newbench(C++,2.0 代)**:同样的多进程骨架,但把网络层换成内嵌的 `net-io-server`(C++ epoll 反应堆类),并加了 `pkg-filter`(按米米号做并发屏障的 barrier ring queue)。
- **net-io-server(C++ 库)**:可被 newbench 内嵌、也可被任何 svr 单独链接的纯 epoll 反应堆类,提供 `on_new_connection/on_recv_data/on_connection_closed/on_wakeup` 四个回调。
- **零件库**:`mysql-iface`(同步 SQL)、`ring-queue`(环形队列家族)、`mmtree`(mmap 红黑树)、`ini-file`(配置)、`net-client`(主动连对端)。它们不构成进程,谁需要谁链接。

---

## 逐部分拆解

### 1. net-io-server —— epoll 反应堆类(理解淘米网络层的源头)

这是一个**纯被动的反应堆库**:它自己不开线程、不开进程,而是把"socket 收发 + epoll 事件分发"打包成一个对象,谁调用谁负责在自己的循环里反复 `do_io()`。

**核心契约 = 4 个回调**(见 [i_net_io_server.h #L86-L153](../../c/library/net-io-server/tags/net-io-server-2.1.0/include/i_net_io_server.h#L86)):`on_new_connection` / `on_recv_data` / `on_connection_closed` / `on_wakeup`。它们的**实际触发时机**全在一个函数里——`c_net_io_server::do_io()`:

- **`do_io(timeout, cmd)`** ([c_net_io_server.cpp #L221](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L221)):每调一次 = 一轮 `epoll_wait`。`cmd` 是 `ACCEPT|READ|WRITE` 的位掩码,调用者可以"这一轮只收不发"。
  - **`on_wakeup` 每轮必触发一次**:`epoll_wait` 一返回(哪怕超时返回 0 个事件)就先调 `on_wakeup`([#L235](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L235))——这正是 newbench 用来"顺便去 sendq 取要发的数据"的钩子。
  - **`on_new_connection` 在监听 fd 可读时触发**:`data.ptr==NULL`(监听 socket)→ `for(;;) accept()` 一直收到 `EAGAIN`([#L245-L355](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L245));每 accept 一个就 `genereate_id_()` 分配连接 id、填 `connection_info_t`、回调 `on_new_connection`。**回调返回 -1 就当场 `close()` 拒掉这条连接**([#L276-L286](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L276))——这是业务层做连接准入(黑名单/限流)的唯一入口。通过后把 fd 设非阻塞、`EPOLLET|IN|OUT|RDHUP` 注册进 epoll。
  - **`on_recv_data` 在连接 fd 可读时触发**:`recv()` 进 `connection_info_t::buffer_recv`(4KB,见 [c_net_io_server.h #L18/#L46](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/header/c_net_io_server.h#L18)),累加 `buffer_recv_len` 后**把整个累积缓冲区**交给 `on_recv_data`([#L419-L427](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L419))。**回调的返回值是"我没消费完、剩这么多字节"**:框架把它写回 `buffer_recv_len`,下次再 recv 时拼在后面([#L429-L443](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L429))。返回负数 / ≥4KB → 视为非法,`close_connection`。这就是淘米所有 svr 处理"半包/粘包"的统一办法:回调里能拼出几个整包就处理几个,剩下的零头作为返回值留给下一次。
  - **`on_connection_closed` 触发点有三处**:对端 `recv` 返回 0([#L466-L470](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L466))、`EPOLLERR/HUP/RDHUP`([#L371-L377](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L371))、业务主动 `close_connection(id,true)`。回调本身在 `close_connection` 内统一发出([#L1057](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L1057))。

- **发送路径的三级缓冲**:`send_data()` → `send_data_()`([#L527](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L527))。一条连接有 `buffer_send`(4KB 内联)+ `extra_buffer`(最大 2MB 堆缓冲,见 [c_net_io_server.h #L20](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/header/c_net_io_server.h#L20))。`send()` 遇 `EAGAIN` 就把没发完的数据 `memmove` 靠齐、留到下次 `EPOLLOUT` 再续发([#L494-L519](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L494))。`send_data_atomic` 则是"装不下整包就一个字节都不发",保证包不被劈开。

- **连接表 = `std::map<int, connection_info_t>`**([c_net_io_server.h #L274](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/header/c_net_io_server.h#L274)),id 由 `genereate_id_()` 按 `id_base + n*id_step` 自增([#L337](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/header/c_net_io_server.h#L337))。`id_base/id_step` 的存在是为了**多进程各开一个 reactor 时让连接 id 不撞车**(进程 0 出奇数、进程 1 出偶数之类)。
- **`i_net_io_notifier`**([i_net_io_server.h #L31](../../c/library/net-io-server/tags/net-io-server-2.1.0/include/i_net_io_server.h#L31)):一个 `eventfd` 包成的"叫醒铃"。别的进程/线程往里写一下,epoll 就被叫醒,`do_io` 里发现 `data.ptr==(void*)-1` 就调 `notifier->got()`([#L358-L363](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp#L358))——这是"worker 写完 sendq、要唤醒 reactor 去发数据"的另一种实现(serverbench 用 pipe,net-io-server 用 eventfd notifier)。

> 最佳上手样例就是 demo:[c_net_io_event_handler.cpp](../../c/library/net-io-server/tags/net-io-server-2.1.0/demo/c_net_io_event_handler.cpp) 实现一个最小 "GET 文件" 服务——`on_recv_data` 里 [#L34-L92](../../c/library/net-io-server/tags/net-io-server-2.1.0/demo/c_net_io_event_handler.cpp#L34) 用 `string` 累积、找 `\r\n`、`get xxx` 就读文件 `send_data_atomic` 回去。把它读一遍,淘米网络层的"收→拼→回"心智模型就全有了。

### 2. serverbench —— "3g" 代多进程骨架(MAIN/CONN/WORKER + shmq)

版本字符串 `"3g.1.1.0"`([main.c #L15](../../c/library/serverbench/tags/serverbench-1.1.0/src/main.c#L15)),banner 还留着作者邮箱 `jasonwang@taomee.com`([#L31](../../c/library/serverbench/tags/serverbench-1.1.0/src/main.c#L31))。它的**业务逻辑全部以 `.so` 插件形式注入**——框架只管搬运字节,玩法在外挂的 dll 里。

**(a) 启动序列**(全在 [main.c #L52-L95](../../c/library/serverbench/tags/serverbench-1.1.0/src/main.c#L52)):
1. `config_init(argv[1])` 读 bench.conf;
2. `load_bind_file()` 读 bind.conf(监听表);
3. `register_plugin(argv[2])` —— `dlopen` 业务 `.so`,用 `dlsym` 抓 7 个钩子([dll.c register_plugin](../../c/library/serverbench/tags/serverbench-1.1.0/src/dll.c));**只有 `handle_input` 和 `handle_process` 是必需的**(找不到就启动失败),其余 `handle_init/open/close/timer/fini` 可选;
4. `pipe_create()` + `shmq_create()` 建 CONN↔WORKER 的通信设施;
5. `daemon_start()` 精灵化;
6. **`fork()` 一次**([#L79](../../c/library/serverbench/tags/serverbench-1.1.0/src/main.c#L79)):父进程走 `worker_procs_spawn()`(再 fork 出 N 个 worker)+ `children_monitor()`;子进程走 `conn_loop()`(它就是 CONN 进程)。

**(b) CONN 进程 = `conn_loop()`**([dispatcher.c #L116](../../c/library/serverbench/tags/serverbench-1.1.0/src/dispatcher.c#L116)):单进程反应堆。主循环([#L150-L158](../../c/library/serverbench/tags/serverbench-1.1.0/src/dispatcher.c#L150))每轮做两件事:
- `net_wait(1000)` 等网络事件,`net_loop()` 把"收到的整包"`shmq_push` 进 **recvq**;
- 顺手 `while(shmq_pop(&sendq,...)) schedule_output(mb)` —— 把 worker 已经处理完、压进 **sendq** 的回包写回对应 socket。

收包切包的关键在 `run_once()`([net.c #L304](../../c/library/serverbench/tags/serverbench-1.1.0/src/net.c#L304)):`do_read_conn` 读进缓冲后,**反复调 `dll.handle_input()` 问"这坨字节里第一个完整包有多长"**([#L356](../../c/library/serverbench/tags/serverbench-1.1.0/src/net.c#L356));够一个整包就组一个 `shm_block`、`shmq_push(&recvq)`,然后 `goto parse_again` 继续切下一个([#L376-L392](../../c/library/serverbench/tags/serverbench-1.1.0/src/net.c#L376))。**注意:CONN 不碰业务,它只知道"包有多长"(handle_input),不知道"包是什么意思"(那是 worker 的 handle_process)。** 这条边界是整个框架的设计核心。

**(c) WORKER 进程 = `worker_run()`**([worker.c #L23](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L23)):死循环干一件事:
```
shmq_pop(&recvq, &mb, SLEEP_MASK|LOCKED_MASK)   // 阻塞抢一个包(带信号量锁)
check_timeout(mb->skinfo.recvtm, mb_timeout)    // 包在队列里待太久(pkg_timeout)就丢弃
dll.handle_process(mb->data, rcvlen, &user_data, &sndlen, &mb->skinfo)  // 真正的业务!
```
([#L36-L43](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L36))。`handle_process` 返回值定义了三种结局([#L48-L68](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L48)):返回 `-1` → 标 `FIN_BLOCK` 让 CONN 发完即关连接;返回 `1` + 有回包 → `FIN_BLOCK`(发完关);返回 `0` + `sndlen>0` → `DAT_BLOCK`(正常回包)。无论哪种,worker 把结果 `shmq_push(&sendq)` 后调 **`write_pipe()`** 戳一下管道——这就是叫醒 CONN 去 `schedule_output` 的信号。

**(d) 进程守护 = `children_monitor()`**([worker.c #L78](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L78)):MAIN 每 10 秒 `select` 睡醒一次,`kill(wpids[i], 0)` 探活每个 worker;**发现 worker 死了(`ESRCH`)就当场 `fork` 一个新的顶上**([#L88-L105](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L88))。还顺便调 `dll.handle_timer(&sec)` 给业务一个定时回调。这就是"某个 worker 崩了不影响整体、自动拉起"的来源。

### 3. shmq —— 共享内存环形队列(CONN 与 WORKER 之间唯一的桥)

这是整个模型的命门。`shmq_create()`([shmq.c #L228](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.c#L228))在 **fork 之前** 用 `mmap(... MAP_SHARED|MAP_ANON ...)` 建两块匿名共享内存:`recvq`(CONN→WORKER)和 `sendq`(WORKER→CONN),长度 `shmq_length`(默认 64MB)。信号量 `sem_t` 也放在另一块共享映射里([#L235-L239](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.c#L235))。因为是 `fork` 后才分化,父子进程**天然共享同一块内存 + 同一组锁**,不需要 shm key 协商(也支持 SysV 信号量,见 `USE_SYSTEMV_SEM` 分支)。

队列单元 `shm_block`([shmq.h #L39-L46](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.h#L39)):`blk_id`(=连接 id,回包时用来校验连接还是不是原来那条)+ `length` + `type`(`DAT/PAD/FIN`)+ `skinfo`(socket 信息,含 fd、recvtm 收包时间)+ 柔性数组 `data[0]`。

**环形队列怎么转**:`head`/`tail` 两个偏移 + `blk_cnt` 原子计数([shmq.h #L21-L25](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.h#L21))。push 在 head 写、pop 从 tail 读。到尾部放不下整块时,用 `align_queue_head` 在末尾塞一个 **`PAD_BLOCK` 占位块**、把 head 绕回开头([shmq.c #L80-L114](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.c#L80))——这是定长环形缓冲处理"尾部碎片"的经典手法。pop/push 满了/空了就 `nanosleep(4ms)` 自旋等待(`CYCLE_WAIT_NANO_SEC`,注释明说是"修复 nanosleep 精度太高导致 CPU 压力过大",见 [shmq.h #L12-L13](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.h#L12))。

> **这里能直接看到当年的 bug 修复痕迹**:`align_queue_head` 里 [#L93-L99](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.c#L93) 有一段被注释掉的代码,注释 `//by jasonwang: 修复无端清空环形队列的bug`——王国栋当年踩过"环形队列莫名被清空"的坑,改法是**不要**在 `tail>head` 的诡异状态下强行重置 head/tail。`shmq_pop` 顶上还留着 `fixme: malloc data memory, free in another`([#L161](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.c#L161)):pop 出来的块是 `malloc` 拷贝([#L185-L189](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.c#L185)),用完在别处 `free`(worker.c 末尾 [#L70](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L70)),跨函数的内存所有权当年作者自己都觉得别扭。

### 4. newbench —— 2.0 代:同骨架 + net-io-server + 米米号屏障

newbench 是 serverbench 的 C++ 重写版,进程模型一样(MAIN fork 出 net 进程 + N 个 work 进程,见 [newbench.cpp #L313-L445](../../c/library/newbench/trunk/src/newbench.cpp#L313)),但有两个实质升级:

- **网络层换成 net-io-server**:net 进程内嵌上面那个 epoll 反应堆类(`src/net-io-server/` 直接拷进来),不再是 serverbench 手写的 poll。
- **`use_barrier` 米米号并发屏障**:这是页游业务的关键。开关在 conf([newbench-beta bench.conf `use_barrier=0/1`](../../c/library/newbench-beta/trunk/conf/bench.conf))。开启后 recvq 换成 **`i_barrier_ring_queue`**([i_barrier_ring_queue.h](../../c/library/newbench/trunk/src/i_barrier_ring_queue.h)):`push(data, length, key, flag)` 多了个 `key` 参数(就是**米米号/user_id**),`pop` 时保证**同一个 key 的多个请求不会被两个 worker 同时取走**——即"同一个玩家的请求串行处理",避免多 worker 并发改同一玩家存档导致竞态。它还提供 `clean_4pid`/`clean_4key`([#L70-L81](../../c/library/newbench/trunk/src/i_barrier_ring_queue.h#L70)):worker 崩了/超时了,由外部清掉它占着的 key,把屏障解开。`use_barrier` 需要业务 `.so` 额外提供 `handle_dispatch` 钩子(算出每个包属于哪个 key),否则降级关闭([newbench.cpp #L207-L208](../../c/library/newbench/trunk/src/newbench.cpp#L207))。

这解释了概览里那句"面向按米米号分发的页游框架"的底层机制:**分发的单位是米米号,屏障保证单玩家请求串行,这是页游服务端正确性的基石。**

### 5. 零件库(谁需要谁链接,不构成进程)

- **mysql-iface**([i_mysql_iface.h](../../c/library/mysql-iface/tags/mysql-iface-1.0.1/i_mysql_iface.h)):MySQL C API 的薄封装。`init(host,port,db,user,passwd,charset)` 连库,`select_first_row(row, "SELECT ... %d", ...)` 变参拼 SQL 取首行([#L77](../../c/library/mysql-iface/tags/mysql-iface-1.0.1/i_mysql_iface.h#L77)),`select_next_row` 取后续行,`execsql` 做 UPDATE/DELETE 返回影响行数([#L97](../../c/library/mysql-iface/tags/mysql-iface-1.0.1/i_mysql_iface.h#L97))。**全是同步阻塞**——放进单线程 reactor 会卡死事件循环,所以它只在 **worker 进程的 handle_process 里用**(worker 多进程天然把阻塞 SQL 摊开)。这正是"为什么要多 worker"的另一半答案。
- **ring-queue**([i_ring_queue.h](../../c/library/ring-queue/trunk/i_ring_queue.h)):进程内/共享内存环形缓冲家族,用装饰器模式拼功能——`create_fixed_queue_instance`(定长)/`create_variable_queue_instance`(变长),再用 `create_waitable_queue_instance`(加阻塞等待,[#L132](../../c/library/ring-queue/trunk/i_ring_queue.h#L132))/`create_multipopable_queue_instance`(加多进程并发 pop,[#L139](../../c/library/ring-queue/trunk/i_ring_queue.h#L139))套在外面。newbench 的 recvq/sendq 就是用它拼出来的。
- **mmtree**([mmtree.h](../../c/library/mmtree-1.0.0/mmtree.h)):`mmap` 文件支撑的**持久化红黑树**。节点 `mt_node_t`(key/data/left/right/parent/color 全是 int 下标,而非指针,见 [mmtree.h #L36-L47](../../c/library/mmtree-1.0.0/mmtree.h#L36))——用数组下标代替指针,所以整棵树能直接 `mmap` 到文件、进程重启/多进程共享都不失效。树头 `mt_state_t` 存 md5、root_id、空闲节点链(`qfirst/qlast` 回收链表,[#L48-L62](../../c/library/mmtree-1.0.0/mmtree.h#L48))。`data` 字段语义是"该 key 出现的次数"——典型用途是**做内存索引/计数**(如按某 key 统计在线数)。API 只有 `insert(key,data)`/`find(key)`/`traversal(cb)`([#L69-L77](../../c/library/mmtree-1.0.0/mmtree.h#L69))。
- **ini-file**([i_ini_file.h](../../c/library/ini-file/tags/ini-file-1.1.0/i_ini_file.h)):分节 INI 解析(`[RunInfo]`/`[LogInfo]`/...),newbench 用它读 bench.conf;serverbench 则用更土的 `config.c`(空格分隔 key value)。
- **net-client**:svr 主动连别的 svr 的客户端库(login 连 dbsvr、gamesvr 连 switch 之类),与 net-io-server 互补(一个被动 listen、一个主动 connect)。

---

## 关键机制实战:一个包从网卡到 worker 再回客户端(serverbench)

把上面拼起来,追一个完整往返:

1. **进**:客户端发来 N 字节 → CONN 进程 `net_loop` → `run_once()` `do_read_conn` 读进 per-fd 缓冲([net.c #L346](../../c/library/serverbench/tags/serverbench-1.1.0/src/net.c#L346))。
2. **切包**:`dll.handle_input(buf, len, sk)` 返回"第一个整包多长"([#L356](../../c/library/serverbench/tags/serverbench-1.1.0/src/net.c#L356))。返回 0=还不够一个包,先囤着;>0=切出来。
3. **入队**:整包组成 `shm_block`(带 `blk_id`=连接 id、`skinfo`),`shmq_push(&recvq, ...)`([#L383](../../c/library/serverbench/tags/serverbench-1.1.0/src/net.c#L383));缓冲里还有剩 → `goto parse_again` 一次切多个。
4. **处理**:某个空闲 WORKER `shmq_pop(&recvq)` 抢到包([worker.c #L36](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L36)),先 `check_timeout`(待太久的丢),再 `dll.handle_process(data, rcvlen, &user_data, &sndlen, sk)`([#L43](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L43))——**这里才是真正读写 MySQL、跑玩法逻辑、算出回包的地方**(业务 `.so` 里)。
5. **回队**:回包 `shmq_push(&sendq, mb, user_data, ...)` + `write_pipe()` 戳醒 CONN([#L63-L67](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c#L63))。
6. **出**:CONN 主循环 `shmq_pop(&sendq)` → `schedule_output(mb)`([dispatcher.c #L152-L153](../../c/library/serverbench/tags/serverbench-1.1.0/src/dispatcher.c#L152))。`schedule_output` 先**用 `blk_id` 校验"这条连接还是不是原来那条"**(`mb->blk_id != fdinfo[fd].id` 就丢弃,因为玩家可能已断线重连,见 [net.c #L575](../../c/library/serverbench/tags/serverbench-1.1.0/src/net.c#L575)),再 `safe_tcp_send_n` 写回 socket;若 `type==FIN_BLOCK` 则发完置 `CN_FIN_MASK` 关连接([#L580-L581](../../c/library/serverbench/tags/serverbench-1.1.0/src/net.c#L580))。

**一句话总结这套设计哲学**:CONN 进程纯异步、只搬字节、绝不阻塞;所有可能阻塞的脏活(SQL、复杂计算)全推给可崩溃可重启的多 WORKER;两者只通过共享内存环形队列 + 一根唤醒管道解耦。这就是 2010 年前后 C10K 时代页游服务端的标准答卷,mole/pea/gf 各 svr 都长这个样子。

---

## 端到端调用流举例:摩尔庄园的某个 svr 进程实际怎么用这套骨架

`c/` 是底座,本身不含 mole 业务,但 mole 各 svr 正是按上面契约写 `.so` 落地的。对照 [mole 深度档案](./mole.md) / [mole 概览](../modules/mole.md),一个典型 mole svr 进程的真实形态可推断为:

- 进程 = `serverbench/newbench 可执行 + mole 业务 .so`,启动命令形如 `bench bench.conf libmolexxx.so`;
- mole 的 `.so` 实现 `handle_input`(按 [demo/proto.h](../../c/library/newbench/trunk/src/test/demo/proto.h#L17) 那套 `cmd(int16)/status(int16)/length(int32)/user_id(int32)` packed 包头读出 `length` 字段返回整包长)、`handle_process`(按 `cmd` 命令字分发到各玩法,读写 MySQL 改存档,组回包);
- newbench 开 `use_barrier` 时,mole 的 `handle_dispatch` 返回包里的 `user_id`(米米号),保证同一玩家请求串行——这对"养成 + 存档"类玩法至关重要。

> 也就是说:**想知道"摩尔庄园某个请求落库的完整链路",网络/进程/队列这一半在本模块(c),业务/SQL/玩法那一半在 [mole 模块](./mole.md)**。本文给的就是那条链路的"传输与调度"骨架,命令字与表结构请转 [protocol 专题](../protocol.md) 与 [content-data](../content-data.md)。

---

## 复活实操提示(对 moleworld 复活)

**能直用的:**
- **线协议包头**([demo/proto.h #L19-L24](../../c/library/newbench/trunk/src/test/demo/proto.h#L19)):`cmd/status/length/user_id` 全 packed 小端 + 柔性数组 body。这是和真实 Flash 客户端/抓包对齐的**硬约束**,可直接抄成 moleworld-server 的协议定义。
- **半包/粘包模型**:不论用什么语言重写,"先读包头里的 length 字段确定整包边界、不够就囤着"这套逻辑(handle_input / on_recv_data 返回剩余字节)可原样照搬。
- `mmtree` / `ring-queue` 这种纯算法容器,若确实要在 C/C++ 侧用,授权风险与依赖最小。

**要自己补 / 不该照搬的:**
- **不要照搬"同步阻塞 SQL + 多进程"模型**:那是 2010 年没有协程/异步驱动的妥协(`mysql-iface` 全阻塞,靠 worker 多进程硬扛)。现代 Rust/Go/Node 重写应走**异步驱动 + 连接池**,用单进程多协程取代"CONN+N×WORKER+shmq"。
- **米米号串行屏障要保留语义、换实现**:`use_barrier` 解决的"同一玩家请求串行、防止并发改存档"是真需求;现代写法用 per-user actor / 按 user_id 哈希到固定 worker / 行级锁即可,不必复刻共享内存 barrier ring queue。
- **进程守护与自动拉起**(`children_monitor` 探活重 fork)在现代由 systemd / k8s / supervisor 接管,不必自己写。
- **年代坑**:`benchapi.h` 大量 `__GNUC__==2` 兼容分支、内联 `rdtsc` 汇编、`.init.text` 段属性、`mmap` 的 32 位假设——纯参照,别指望在 64 位现代工具链直接编译通过。

一句话:**这部分对复活的价值是"读懂当年 mole 服务端的传输与并发骨架,以及那套必须对齐的线协议",而非搬代码。** 玩法、数值、存档结构一概不在这里。

---
返回:[总入口 README](../../README.md) · 概览:[c 模块概览](../modules/c.md) · 相关:[mole 深度档案](./mole.md) · [protocol 协议专题](../protocol.md) · [content-data 内容数据](../content-data.md)
