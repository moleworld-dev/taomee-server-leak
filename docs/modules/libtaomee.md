# 模块 libtaomee · 淘米内部框架库(Libtaomee C 版 + Libtaomee++ C++ 版)

> 价值:★★  ·  游戏:基础设施(摩尔庄园/赛尔号/功夫派等所有页游共用)  ·  源码量:约 3.9 万行(libtaomee C 含内嵌 Lua ≈3 万行 + libtaomee++ ≈0.9 万行)

本模块不是一个解压在仓库工作树里的目录,而是仓库根的**两个发布版 tar.gz**:

- [Libtaomee-0.7.5.tar.gz](../../Libtaomee-0.7.5.tar.gz) — C 版框架库,namespace `libtaomee/`,头文件统一以 `#include <libtaomee/...>` 引用。
- [Libtaomee++-0.7.2.tar.gz](../../Libtaomee++-0.7.2.tar.gz) — C++ 版框架库,namespace `taomee::`,头文件以 `#include <libtaomee++/...>` 引用。

下文路径形如 `libtaomee/crypt/qdes.h` 指的是**对应 tar.gz 解包后**的内部路径(tar 内部文件无法直接 GitHub 链接,故用代码块/表格列出),所有结论均来自解包到临时目录后真实 Read 的文件。

## 是什么 / 在整套架构里的角色

`libtaomee` / `libtaomee++` 是上海淘米(`TaoMee, Inc. ShangHai CN`,见每个头文件版权头注)沉淀的**安装级公共框架库**——和 [c 模块](./c.md)(SVN 连版本目录泄漏的"库仓库")不同,这两个是打成 tar.gz、走 `INSTALL` 脚本装到系统(`/usr/include/libtaomee/`、链接 `-ltaomee` / `-ltaomee++`)的**正式发布件**。各游戏服务端进程把它当作系统库直接 `#include` + 链接。

从安装/使用方式即可证实其"系统库"定位(见 [Libtaomee-0.7.5.tar.gz](../../Libtaomee-0.7.5.tar.gz) 内 `libtaomee/doc/readme.txt`):

```text
1. 把代码checkout到命名为libtaomee的子目录下。
2. 进入libtaomee子目录，运行INSTALL安装脚本，需要sudo或root权限。
依赖库: 1. libglib2.0以上
使用: #include <libtaomee/crypt/qdes.h> ... gcc a.c -ltaomee
```

C++ 版同理(`libtaomee/doc` 同位置的 readme),`g++ a.cpp -ltaomee++`,额外依赖 `libxml2`、`libgd2`,且**依赖 libtaomee 本身**(C++ 版建立在 C 版之上)。

**年代与演进(来自两份 changelog,纯文本实证):**
- C 版:`2008-12-08` 起步(log.c 自动轮转)→ `2009` 陆续加 shmq/mcast/random/config → `2011` 加 tcp/inet/脏词库 tm_dirty → `2012-04-12` 收束到 **0.7.5**(TLOG KEYLOG)。
- C++ 版:`2008-12-29` 起步(byteswap/pdumanip)→ `2009` 线程/线程池/xml → `2010` 主登录框架/MemPool/A* 寻路 → `2011` EventPoll/Socket/Barrier/自旋锁 → `2012-06-15` 收束到 **0.7.2**。
- 一条关键史料:C++ changelog `2011-03-24` 写"**删除了 main_login 代码。公共库不再维护主登陆,改由游戏项目线统一维护**"——解释了为什么各游戏(如 gf)有自己的 `main_login/`。

**它和 [c 模块](./c.md) 的分工:**
- `c/`(net-io-server / serverbench / newbench / mysql-iface / mmtree)= **重型网络与进程框架**(epoll 反应堆、dispatcher+worker+shmq 多进程骨架、MySQL 封装),是"服务怎么跑起来"的骨架。
- `libtaomee` / `libtaomee++` = **轻量通用件**(加密、日志、共享内存队列、时间/定时器、协议字节序、脏词、寻路、验证码),是"业务代码每天都 `#include` 的小工具箱"。
- 二者有交集且互相借用:`c/` 的 `others/README` 注明其 `log.h/log.c` 抄自 `Libtaomee-0.6.7.tar.gz`;反过来 libtaomee++ 的 `proto_base.h` 又 `extern "C" { #include <libtaomee/log.h> }`。可理解为:**c/ 是框架,libtaomee 是被框架和业务双方共享的基础件。**

## 框架包内主要子模块(每条 = 可跳转超链接 + 一句话作用)

### Libtaomee(C 版,tar 内 `libtaomee/`)— 见 [Libtaomee-0.7.5.tar.gz](../../Libtaomee-0.7.5.tar.gz)

| tar 内路径 | 作用 |
| --- | --- |
| `libtaomee/crypt/qdes.h` `.c` | **DES ECB 加密**(`des_encrypt`/`des_decrypt`/`des_encrypt_n`),8 字节块;mole 登录票据加密用的就是它 |
| `libtaomee/log.h` `log.c` | 9 级日志、按时间自动轮转、可选 syslog、`enable_multi_thread()` 多线程写;每条 ≤8000 字节 |
| `libtaomee/tlog/` | **新一代日志 TLOG**(0.7.4+),带 `set_trace_uid`/`set_trace_addr`/`set_rate_limit` 在线工具(`tlog/tools/`)、KEYLOG |
| `libtaomee/interprocess/shmq.h` `.c` `shmq_push.c` `shmq_pop.c` | **共享内存环形队列**(单读单写),进程间传包;mole 各 svr 用它跨进程 |
| `libtaomee/interprocess/lock.h` `filelock.h` | 进程间锁 / 文件锁 |
| `libtaomee/inet/tcp.h` `mcast.h` `inet_utils.h` `ipaddr_cat.h` | TCP/UDP 监听与连接(`create_passive_endpoint` 支持 IPv4/6)、组播 socket、`is_legal_ip/port`、按网卡取 IP |
| `libtaomee/timer.h` `.c` | 定时器(支持数组配置/重配) |
| `libtaomee/time/time.h` `.c` | 时间工具 |
| `libtaomee/atomic.h` | 原子操作(供 shmq 等用) |
| `libtaomee/random/random.h` | 随机数 |
| `libtaomee/project/constants.h` `types.h` `utilities.h` | **淘米项目公共常量/类型/工具**:如起始米米号 `base_user_id=50000`、游客号 `base_guest_id=2000000001`、登录 DES key 宏 |
| `libtaomee/project/stat_agent/msglog.h` `.c` | 统计信息打点(stat agent) |
| `libtaomee/tm_dirty/` | **脏词/敏感词过滤**:AC 多模式匹配(`acsm.c`)、UTF-8 标点处理、在线更新、检测通过后 UDP 上报广告检测;附词库 `tm_dirty.dat` |
| `libtaomee/attr_api/` | 属性上报 API(hash_table + shm) |
| `libtaomee/conf_parser/config.h` `.c` | 简单配置文件解析(支持读汉字/无换行结尾) |
| `libtaomee/dataformatter/bin_str.h` | 字符串↔数值二进制转换、`hex2str_s` |
| `libtaomee/bitmanip/bitmanip.h` | 位操作 |
| `libtaomee/tm_shm/` `debug/debug_mem.*` | 共享内存封装 / 内存调试 |
| `libtaomee/list.h` `hash_algo.h` `utils.h` | Linux 内核风格侵入式链表、哈希算法、杂项工具(`libtaomee_get_version`) |
| `libtaomee/lua/` | **内嵌 Lua 5.x 解释器全套**(~1.6 万行),0.7.0+ 用 lua 做配置识别自动化;非淘米原创,评估代码量/授权时应剥离 |

### Libtaomee++(C++ 版,tar 内 `libtaomee++/`,namespace `taomee::`)— 见 [Libtaomee++-0.7.2.tar.gz](../../Libtaomee++-0.7.2.tar.gz)

| tar 内路径 | 作用 |
| --- | --- |
| `libtaomee++/thread/` | 线程全家桶:`thread`/`mutex`/`rwlock`/`spinlock`/`threadcondition`/`barrier`/`threadpool`(模板实现用 `.tcpp`) |
| `libtaomee++/inet/socket/socket.hpp` `.cpp` | 面向对象 Socket 封装(`taomee::Socket`,`SocketError` 异常) |
| `libtaomee++/inet/udp_pool/` | 基于线程池的 UDP 多线程处理池 |
| `libtaomee++/inet/pdumanip.hpp` `byteswap.hpp` | **PDU(协议数据单元)字节序/打解包**,网络收发包的 C++ 工具 |
| `libtaomee++/ev/eventpoll.hpp` `.cpp` | epoll 全操作封装 `EventPoll`(2011-01) |
| `libtaomee++/event/event_mgr.hpp` `eventable_obj.hpp` | 定时事件管理 `EventMgr` / 可事件化对象 |
| `libtaomee++/conf_parser/xmlparser.hpp` `.cpp` | **XML 解析器**(libxml2 封装),各游戏解析配置表的主力(gf 大量使用) |
| `libtaomee++/conf_parser/ini_parser.hpp` `.cpp` | INI 解析器 `IniParser`(readme 示例即用它) |
| `libtaomee++/conf_parser/item_manager.hpp` | **解析 item 表公共部分**(统一 items.xml 格式),供各游戏物品表复用 |
| `libtaomee++/proto/proto_base.h` `proto_util.h` `Ccmd_map.h` | **协议基类/字节序内联函数/命令字映射**(默认小端,`uint32_t_swap=ntohl`),C++ 侧协议骨架 |
| `libtaomee++/memory/mempool.hpp` | 内存池 `MemPool` |
| `libtaomee++/factory/factory.hpp` + `utils/obj_registry.hpp` | `ObjectRegistry` → 工厂模式 + 蝇量(Flyweight)模式 |
| `libtaomee++/pathfinder/astar/astar.hpp` | **A* 寻路算法**(2010-05),带地图格子配置(`1000101.txt` 示例)、区域可行走设置;摩尔庄园/赛尔号类场景移动相关 |
| `libtaomee++/verification_image/verification_image.hpp` `.cpp` | **图形验证码生成**(依赖 libgd,`-lgd`);配合登录验证码功能 |
| `libtaomee++/utils/md5.h` `strings.hpp` `tcpip.h` `utilities.hpp` `noncopyable.hpp` | MD5、字符串工具、tcpip 杂项、`libtaomeepp_get_version`、不可拷贝基类 |
| `libtaomee++/time/time.hpp` `timeval.hpp` `random/random.hpp` `bitmanip/bitmanip.hpp` | 时间/TimeVal/随机数/位操作的 C++ 版 |
| `libtaomee++/core/file_desc.hpp` `.cpp` | 文件描述符 RAII 封装 |
| `libtaomee++/doc/登录服务器编写指南.doc` | 登录服务器编写指南(.doc 二进制,内容待证;呼应 changelog 里被移除的 main_login) |

## 与各游戏服务端的关系(真实引用例子,带 #L 行号)

这两个库被各游戏服务端直接 `#include`,是最硬的"它确实是公共框架"证据:

- **摩尔庄园 mole/gamesvr** 同时用 C 版日志与 DES:[mole/gamesvr/game.c #L5-L6](../../mole/gamesvr/game.c#L5)
  ```c
  #include <libtaomee/log.h>
  #include <libtaomee/crypt/qdes.h>
  ```
  而且 game.c 里**真实硬编码了登录 DES 密钥并调用 `des_encrypt`**——[mole/gamesvr/game.c #L324-L335](../../mole/gamesvr/game.c#L324):
  ```c
  static char des_key[] = ">ta:mee<";   // L324
  static char des_key[] = ",.ta0me>";   // L326(条件分支另一把)
  ...
  des_encrypt(des_key, inbuf, outbuf);  // L335
  ```
  这把 8 字节 key 直接喂给 `libtaomee/crypt/qdes.h` 的 `des_encrypt`——离线复活解析真实登录票据时,加密算法(DES ECB)与 key 都在这里。
- mole 还广泛用 `libtaomee/list.h`(侵入式链表)、`libtaomee/timer.h`、`libtaomee/conf_parser/config.h`、`libtaomee/random/random.h`——见 [mole/gamesvr/sprite.c #L9-L10](../../mole/gamesvr/sprite.c#L9)、[mole/gamesvr/mcast.c #L11-L12](../../mole/gamesvr/mcast.c#L11)。
- **功夫派 gf** 同时用 C 版和 C++ 版:C 版 `log/qdes/timer/project` 一应俱全,C++ 版 XML 解析见 [gf/cache/dll_interface.cpp #L15](../../gf/cache/dll_interface.cpp#L15) `#include <libtaomee++/conf_parser/xmlparser.hpp>`、[gf/trade_svr/market.cpp #L2](../../gf/trade_svr/market.cpp#L2)。功夫派(原 baseline 误标为"小花仙",实为武侠战斗 MMO,货币"功夫豆")正是用 libtaomee++ 的 xmlparser/item_manager 解析其大量配置表。

> 关于"功夫派"游戏名的修正:gf 模块货币为"功夫豆"、含木人/木王/铜人/battle/skills/stage,确为**功夫派(KungFu Pai)**而非小花仙;本文据 gf 的源码引用关系一并确认。详见 [gf 模块文档](./gf.md)。

## 协议 / 数据亮点

本模块是**框架库,不承载游戏业务协议**,但提供了所有淘米协议都依赖的底层原语:

- **加密层**:`libtaomee/crypt/qdes.h` 的 DES ECB(8 字节块,`des_encrypt_n` 批量)是登录/票据加密的统一实现。readme.txt 给的示例就是 `des_encrypt_n("11122233", in, out, 3)`。
- **字节序约定**:`libtaomee++/proto/proto_base.h` 注释"目前默认系统内部是小端",并把 `uint32_t_swap` 实现为 `ntohl`、`uint64_t_swap` 手工高低 32 位拼接——这是 C++ 侧解析/构造网络包的统一基线,与 [c 模块](./c.md) demo proto.h 的 packed 小端包头一致(见 [../protocol.md](../protocol.md))。
- **公共常量**:`libtaomee/project/constants.h` 定义起始米米号 `base_user_id=50000`、起始游客号 `base_guest_id=2000000001`,以及登录 DES key 宏(`LOGIN_DES_KEY`)——这些是跨游戏的账号体系常量。
- **脏词/合规**:`libtaomee/tm_dirty/` 的 AC 多模式匹配 + 检测通过后 UDP 上报广告检测,是聊天/UGC 合规链路的入口(脏词服务器地址需在各游戏 `bench.conf` 强制配置)。
- **配置范式**:`item_manager.hpp` 规定了"item 表公共部分格式必须统一",`xmlparser.hpp`/`ini_parser.hpp` 是各游戏读 XML/INI 配置的统一入口(配置数据见 [../content-data.md](../content-data.md))。

本模块内**没有游戏业务 .sql 或玩法数值**——它只到框架原语层为止。

## 对我们有什么用(moleworld-server / MoleWorld-offline 能复用什么)

定位:**关键"直用"点 + 大量"参照"。** 比 [c 模块](./c.md) 更贴近"复活摩尔庄园必须吃透的契约细节"。

- **直用(加密契约,硬约束)**:`libtaomee/crypt/qdes.c` 的 **DES ECB 实现 + mole/gamesvr/game.c 里的真实 8 字节 key(`>ta:mee<` / `,.ta0me>`)**,是离线复活解密/构造真实登录票据的硬依赖——重写服务端(Rust/Go/Node)时必须复刻同一套 DES ECB + 同一把 key,否则与真实 Flash 客户端对不上。这是本模块对复活项目最大的价值。
- **直用(账号常量)**:`base_user_id=50000`、`base_guest_id=2000000001` 等米米号区间常量,直接照搬即可让发号逻辑与真实数据一致。
- **参照(字节序/协议)**:`proto_base.h` 的小端 + ntohl 约定、`pdumanip.hpp` 的 PDU 打解包,是 moleworld-server 线协议字节序处理的参照。
- **参照(玩法基础设施)**:`pathfinder/astar/astar.hpp`(A* 寻路 + 格子地图配置格式)与摩尔庄园场景移动直接相关,寻路算法和地图格式可参照甚至移植;`verification_image`(验证码)、`tm_dirty`(脏词)是合规功能的现成参照。
- **可直接拿来当工具**:纯算法件(`hash_algo.h`、`list.h`、`bitmanip`、A* )授权与依赖风险最小,最易直用。
- **基本无关**:内嵌的 `lua/`(标准 Lua 解释器)、`log/tlog`(运维日志)对复活客户端体验无直接帮助,属运维侧。

## 注意 / 坑

- **年代久远的工具链假设**:所有头注 `compiler gcc4.1.2 / platform Linux 2.6.x x86-32/64`,大量 `__builtin_expect`(likely/unlikely)、内核风格链表、`__attribute__((packed))`——面向老 GCC + 32/64 位 Linux,现代环境编译需适配。
- **lua 全套不是淘米原创**:`libtaomee/lua/` 是标准 Lua 5.x 解释器(含 `liblua.a` 预编译产物),约占 C 版近半行数;统计"淘米原创代码量"时必须剥离,真正淘米写的 C 代码约 1.4 万行。
- **shmq 是单读单写**:`shmq.h` 明确"目前只适用于一个进程只读,另一个进程只写"——多读/多写场景不能直接用,设计时要注意。
- **C++ 版依赖 C 版 + 外部库**:`libtaomee++` 依赖 `libtaomee`、`libxml2`、`libgd2`(`-lgd`),且 readme 提到依赖 "AsyncServer 头文件"(对应 [c 模块](./c.md) 的 async-serv);单独拿 C++ 包编不过。
- **主登录已被移除**:C++ changelog `2011-03-24` 删了 `main_login`,公共库不再含主登录框架——别在这里找登录服务端完整实现,它在各游戏项目线(如 gf 的 `main_login/`)。
- **设计文档是 .doc 二进制**:`libtaomee++/doc/登录服务器编写指南.doc`、`doc/html/`(Doxygen 产物)无法在代码视图直读,本文结论基于头注释/changelog/readme/接口头等纯文本实证,.doc 内部细节标记为(待证)。
- **未解进仓库工作树**:本文所有 tar 内路径来自临时解包审阅,仓库里仍只有两个 .tar.gz;引用游戏侧用法的 #L 行号才是仓库内真实可点击文件。

---
返回:[总入口 README](../../README.md) · 相关:[c 底座库](./c.md) · [mole 摩尔庄园](./mole.md) · [gf 功夫派](./gf.md) · [protocol 协议专题](../protocol.md) · [content-data 内容数据](../content-data.md)
