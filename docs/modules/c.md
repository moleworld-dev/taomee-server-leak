# 模块 c · 淘米全公司共享的 C/C++ 服务端底座库(网络框架 + 数据结构 + 规范)

> 价值:★★  ·  游戏:基础设施(摩尔庄园/赛尔号/功夫派等所有页游共用)  ·  源码量:约 22 万行

## 是什么 / 在整套架构里的角色

`c/` 不是某一款游戏,而是上海淘米(TaoMee.Inc, ShangHai)技术部门沉淀下来的**公共 C/C++ 基础库仓库**——所有游戏服务端(mole 摩尔庄园、pea 赛尔号、monster 小怪兽、gf 功夫派……)都从这里取走同一套"轮子":网络 IO 框架、共享内存队列、MySQL 封装、INI 配置解析、红黑树容器、客户端连接库,以及一份全公司统一的《C&C++ 代码规范》和《数据库应用规范》。

从作者头注释可证实它是公司级公共件而非外购:
- MySQL 封装头注 `Company: TaoMee.Inc, ShangHai` / `@author jasonwang (王国栋)`(见 [c_mysql_iface 接口头 #L11-L14](../../c/library/mysql-iface/tags/mysql-iface-1.0.1/i_mysql_iface.h#L11)),创建于 2010-03-24。
- 红黑树 [mmtree.cpp #L13-L14](../../c/library/mmtree-1.0.0/mmtree.cpp#L13) 署名 `henry (韩林)`,2010-08-17。
- 环形队列 [i_ring_queue.h #L4-L6](../../c/library/ring-queue/trunk/i_ring_queue.h#L4) 署名 `xml` / `richard`,2010-03-22。

**在整套架构里的角色 = 最底层的"地基"。** 这里定义的接口/结构被上层各游戏服务进程(login/online/gamesvr/dbsvr/switch)直接 `#include` 或链接。它本身不是一个跑起来的服务,而是给"跑起来的服务"提供:
- **网络收发**:`net-io-server` 单线程 epoll 反应堆 + 回调式事件处理(on_new_connection / on_recv_data / on_connection_closed)。
- **进程模型**:`serverbench` / `newbench` 提供"主进程 dispatcher + 多 worker + 共享内存队列(shmq)"的经典多进程服务骨架,带 daemon 化、日志轮转、信号通知。
- **数据通道**:`mysql-iface`(同步阻塞 SQL 封装)、`ring-queue`(进程内/共享内存环形缓冲)、`mmtree`(mmap 持久化红黑树,可做内存索引)。
- **配套**:`net-client`(主动连接对端服务的客户端库,用于 svr 间互联)、`ini-file`(配置解析)。

同一组件有 `trunk` / `tags/x.y.z` / `branches/`(如 `net-io-server-2.0.0-moledoudou`、`newbench-beta/branches/4moledoudou`、`4newsfeed`)多份并存——这是直接从公司 SVN 仓库连版本目录一起泄漏出来的痕迹,`-moledoudou`(摩尔豆豆)、`4newsfeed`(动态/信息流)是为具体业务拉的分支。

## 关键子目录与文件(每条 = 可跳转超链接 + 一句话作用)

### 网络 IO 框架(核心)
- [c/library/net-io-server/](../../c/library/net-io-server) — epoll 反应堆网络服务库,有 1.1.0→2.1.0 多个 tag。
- [net-io-server-2.1.0/include/i_net_io_server.h](../../c/library/net-io-server/tags/net-io-server-2.1.0/include/i_net_io_server.h#L158) — 核心接口:`i_net_io_server`(init/send_data/broadcast/do_io/close_connection)与回调集 `i_net_io_event_handler`,理解淘米网络层从这一个文件入手。
- [net-io-server-2.1.0/src/source/c_net_io_server.cpp](../../c/library/net-io-server/tags/net-io-server-2.1.0/src/source/c_net_io_server.cpp) — 实现(全仓最大单文件 ~37KB):连接表、收发缓冲、epoll 事件循环。
- [net-io-server-2.0.0-moledoudou/](../../c/library/net-io-server/branches/net-io-server-2.0.0-moledoudou) — 为"摩尔豆豆"业务定制的分支,可对比看摩尔庄园周边产品的改动。
- [net-io-server-2.1.0/demo/simple_file_server.cpp](../../c/library/net-io-server/tags/net-io-server-2.1.0/demo/simple_file_server.cpp) — 用该框架写的最小文件服务示例,配合 PHP 客户端,最佳上手样例。

### 多进程服务骨架
- [c/library/serverbench/](../../c/library/serverbench) — "3g" 代多进程服务框架(C 语言);版本号见 [main.c #L16](../../c/library/serverbench/tags/serverbench-1.1.0/src/main.c#L16) `"3g.1.1.0"`。
- [serverbench-1.1.0/src/dispatcher.c](../../c/library/serverbench/tags/serverbench-1.1.0/src/dispatcher.c) / [worker.c](../../c/library/serverbench/tags/serverbench-1.1.0/src/worker.c) / [shmq.c](../../c/library/serverbench/tags/serverbench-1.1.0/src/shmq.c) — 主从分发 + worker + 共享内存队列三件套,经典页游服务进程模型。
- [serverbench-1.1.0/src/benchapi.h](../../c/library/serverbench/tags/serverbench-1.1.0/src/benchapi.h) — 对外 API + 一堆 gcc 内核风格宏(likely/unlikely、rdtsc、`__init` 段),日志等级 `APP_EMERG…` 也在这。
- [c/library/newbench/](../../c/library/newbench) — serverbench 的"新一代"(2.0.x),把网络层换成内嵌的 net-io-server,带 `pkg-filter`(balanced 负载/dmux 分流)。
- [c/library/newbench-beta/](../../c/library/newbench-beta) — newbench 早期 beta,含 `4moledoudou` / `4newsfeed` 业务分支。
- [async-serv-0.1/framework/](../../c/library/async-serv/tags/async-serv-0.1/framework) — 另一套全异步反应堆框架(tcp/udp io event、timer queue、connections pool),比 serverbench 更现代的尝试。

### 数据结构 / 数据访问
- [c/library/mysql-iface/](../../c/library/mysql-iface) — MySQL C API 的薄封装,提供 `select_first_row` / `execsql` 变参接口(见 [i_mysql_iface.h #L77](../../c/library/mysql-iface/tags/mysql-iface-1.0.1/i_mysql_iface.h#L77))。
- [c/library/ring-queue/](../../c/library/ring-queue) — 环形缓冲族:fixed/variable/waitable/multipopable 多种队列变体 + 统一接口 [i_ring_queue.h](../../c/library/ring-queue/trunk/i_ring_queue.h)。
- [c/library/mmtree-1.0.0/mmtree.cpp](../../c/library/mmtree-1.0.0/mmtree.cpp) — mmap 文件支撑的红黑树,可做持久化/共享内存索引。
- [c/library/net-client/](../../c/library/net-client) — 主动连接对端服务的客户端连接库(svr↔svr 互联用)。
- [c/library/ini-file/](../../c/library/ini-file) — INI 配置解析库(`i_ini_file` 接口 + `c_ini_file` 实现)。

### 文档与规范
- [c/specification/C&C++代码规范.pdf](../../c/specification/C&C++代码规范.pdf) — 全公司 C/C++ 编码规范(解释代码里 4 空格缩进、`i_`/`c_` 接口实现命名等约定的来源)。
- [c/specification/数据库应用规范.pdf](../../c/specification/数据库应用规范.pdf) — DB 设计/使用规范。
- [c/specification/templates/](../../c/specification/templates) — 协议设计/需求/系统/数据库设计文档的 .doc 模板,可对照其它模块的 proto 设计风格。
- [newbench/.../doc/平台项目详细设计文档.doc](../../c/library/newbench/trunk/doc) — 平台框架的详细/概要设计文档(.doc,需另开)。
- [c/library/doc/html/index.html](../../c/library/doc/html/index.html) — 这些库的 Doxygen 生成 API 文档(net_io_server / ring_queue / mysql_iface / net_client 全有)。

## 协议 / 数据亮点(指到 proto.h 命令字、conf XML、sql 表)

本模块是**底座,不承载游戏业务协议**,所以这里只有"框架级"的协议骨架与配置,真正的游戏命令字在各游戏模块(见 [../protocol.md](../protocol.md))。

### 框架协议骨架(可作所有淘米协议的"模板")
唯一的 proto.h 是 demo 用的,但它精确体现了淘米通用包头格式——[demo/proto.h #L17-L24](../../c/library/newbench/trunk/src/test/demo/proto.h#L17):
```c
#define PROTO_CMD_GETFILE 0x1001          // 命令字 cmd 用 16 进制
typedef struct {
    int16_t cmd;      // 命令字
    int16_t status;   // 状态/错误码
    int32_t length;   // 包总长
    int32_t user_id;  // 米米号(Taomee 账号)
}__attribute__((packed)) proto_header_t;  // 全部 packed,小端裸结构体
```
这套 `cmd/status/length/user_id` packed 包头 + 16 进制命令字 + 柔性数组 body(`char name[0]`)的写法,正是 mole/monster/gf 等游戏协议的统一范式。

### 配置文件亮点
- [serverbench bench.conf](../../c/library/serverbench/tags/serverbench-1.1.0/conf/bench.conf) — 框架主配置:`worker_num`、`shmq_length`、`pkg_timeout`、`log_level`(5=normal);其 `log_dir` 残留真实路径 `/home/lemon/vip-mgr/trunk/my-vip-mgr/logs/`,暴露当年开发机目录结构。
- [newbench-beta bench.conf](../../c/library/newbench-beta/trunk/conf/bench.conf) — INI 分节配置,出现关键业务参数 `use_barrier`(同一**米米号**并发请求过滤器)、`max_pkg_len`、`shmq_length`——证明这是面向"按账号(米米号)分发"的页游服务框架。
- [bind.conf](../../c/library/newbench-beta/trunk/conf/bind.conf) — 监听绑定表(ip/port/type/idle-timeout),如 `0.0.0.0 12212 tcp 1`。

### SQL / 表结构
本模块内**没有游戏业务 .sql**(底座库不含 schema);MySQL 只到接口封装层。游戏表结构请看各游戏模块与 [../content-data.md](../content-data.md)。

## 对我们有什么用(moleworld-server / MoleWorld-offline 能复用什么)

定位:**主要"参照",局部可"直用"。** 摩尔庄园复活若用现代语言(Rust/Go/Node)重写服务端,这里最大的价值是"读懂当年服务端长什么样",而不是把 2010 年的 C 代码搬过去编译。

- **直用(数据/契约层面)**:[demo/proto.h](../../c/library/newbench/trunk/src/test/demo/proto.h) 的包头布局可**直接照抄**成 moleworld-server 的线协议定义——只要保持 `cmd(int16)/status(int16)/length(int32)/user_id(int32)` packed 小端,就能和 Flash 客户端/各游戏模块的真实包对齐。这是离线复活解析真实抓包/复用真实客户端的硬约束。
- **参照(架构层面)**:`net-io-server` 的 epoll 反应堆 + 回调接口、`serverbench/newbench` 的"dispatcher+worker+shmq 多进程 + 按米米号分发"模型,是理解"为什么 mole 各 svr 这样拆进程、为什么用共享内存队列"的钥匙。重写时可参照其分层(网络层/分发层/业务回调层)而非照搬实现。
- **参照(配置层面)**:`bench.conf` / `bind.conf` 的字段(worker 数、包超时、日志分级、监听表)说明了一个淘米服务进程需要哪些运维旋钮,可作 moleworld-server 配置设计的清单。
- **可直接拿来当工具**:`mmtree`(mmap 红黑树)、`ring-queue` 这种纯算法/容器代码若确实要在 C/C++ 侧用,授权风险与依赖都最小,是最容易"直用"的部分。

反过来说,**业务逻辑/数值/玩法这里一点都没有**——想复活摩尔庄园的玩法,得去 [mole 模块](./mole.md) 和内容配置,本模块只回答"传输与进程怎么搭"。

## 注意 / 坑(纯二进制无源的部分、缺依赖、年代局限)

- **年代久远的工具链假设**:`benchapi.h` 里大量 `#if __GNUC__ == 2 && __GNUC_MINOR__ < 96` 兼容分支、内联 `rdtsc` 汇编、`.init.text` 段属性——面向 32 位 Linux + 老 GCC,现代 64 位环境直接编译会有坑(指针宽度、段属性、内核风格宏)。
- **borrow 来的文件不是淘米原创**:[others/README](../../c/library/newbench/tags/newbench-2.0.2/src/others/README) 明确写了 `list.h` 抄自 Linux 内核、`log.h/log.c` 来自 `Libtaomee-0.6.7.tar.gz`(对应本仓 libtaomee 模块)、`setproctitle.*` 来自 Chromium——评估授权/出处时要剥离这些第三方件。
- **同组件多版本/多分支并存**:`trunk` 与一堆 `tags/`、`branches/` 内容高度重复(net-io-server、newbench 各 6+ 份)。**22 万行里相当大比例是版本副本**,真正独立代码量远小于此;阅读时认准 trunk 或最高 tag 一份即可,不要被行数吓到。
- **设计文档是 .doc 二进制**:`平台项目详细设计文档.doc` / 规范类 `.pdf/.doc` 无法在代码视图直接读,需另用工具打开;本文结论基于头注释、接口头、conf、README 等纯文本实证,.doc 内部细节标记为(待证)。
- **mysql-iface 是同步阻塞封装**:`select_first_row/execsql` 是阻塞调用,放进单线程反应堆会卡住事件循环——当年靠"worker 多进程"规避;现代重写不应照此模式,应走异步/连接池(此为设计局限的提醒)。
- **没有 DB schema**:别指望在 `c/` 里找到任何游戏表结构,它只到 MySQL 接口层为止。

---
返回:[总入口 README](../../README.md) · 相关:[mole 摩尔庄园](./mole.md) · [protocol 协议专题](../protocol.md) · [content-data 内容数据](../content-data.md)
