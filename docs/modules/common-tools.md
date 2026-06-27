# 模块 common-tools · 淘米各游戏共用的运维/反作弊/代理工具集

> 价值:★★  ·  游戏:基础设施(横跨摩尔庄园/赛尔号/功夫派/小花仙等全线)  ·  源码量:约 3 万行(C/C++ 计 29547 行,另含大量 PHP/Perl/Shell/conf)
> 📖 深度功能解析(每个部分实际在干什么):[deep/common-tools.md](../deep/common-tools.md) · 彩蛋见 [easter-eggs.md](../easter-eggs.md)

## 是什么 / 在整套架构里的角色

`common-tools` 不是某一款游戏的服务端,而是上海淘米运维与平台团队为**所有游戏共用**的一组独立工具/小服务的集合。它和 `c`(共享 C 库底座)、`libtaomee`(框架库)一样属于"基础设施"层,但 `c`/`libtaomee` 是被链接进游戏进程的库,而 `common-tools` 里的东西大多是**独立运行的进程或离线脚本**,部署在游戏集群旁边:

- 反作弊(anticheat):基于 UDP 上报 + 共享内存 + libevent 的"加速器/连发"检测守护进程,所有游戏(mole/seer/gf...)各自部署一份。
- 协议代理(proxy / gameproxy):跨游戏的客户端请求路由/转发层,把客户端来包按 `game+线路` 路由到对应的 dbproxy/switch。
- 广告/脏话过滤(advert_fliter):聊天文本反垃圾,贝叶斯 + mmseg 分词统计,接收聊天 server 的请求做判定。
- 明信片下发(postcard_ser / ip_postcard_ser_v3):给各游戏玩家发系统明信片/IP 归属相关卡片。
- 主登录(main_login):一份通用登录服务的源码骨架(CliProto1 注释明确写"目前摩尔、赛尔和小花仙使用")。
- 协议字典(proto/trunk):各游戏协议命令字的 **PHP 描述文件**,运维/测试脚本据此构包。
- 杂项运维(tools / shterm / win):core 文件巡检、在线人数监控、堡垒机、Windows 端网络诊断与邮件工具。

目录顶层分 `linux/`(549 文件,主体)与 `win/`(47 文件,两个 Windows 小工具)。多数子工具用 SVN `trunk/` 结构保留。它在整套架构里的角色是**旁路支撑系统**:不在游戏主链路 login→online→dbsvr→cache→switch 上,而是挂在旁边做检测、路由、过滤、运营下发。

## 关键子目录与文件(每条 = 可跳转超链接 + 一句话作用)

- [common-tools/linux/anticheat/trunk/anticheat.c](../../common-tools/linux/anticheat/trunk/anticheat.c) — 反作弊主守护:共享内存分级(SW lvl0~3)管理 uid 节点,libevent 事件循环,检测发包间隔/频率异常。
- [common-tools/linux/anticheat/trunk/anticheat.h](../../common-tools/linux/anticheat/trunk/anticheat.h) — 反作弊核心参数:keepalive 60s 判下线、tw/sw 检查阈值、百万级 uid hash 表与四级 single_watch 规格([#L48](../../common-tools/linux/anticheat/trunk/anticheat.h#L48))。
- [common-tools/linux/anticheat/trunk/interface.c](../../common-tools/linux/anticheat/trunk/interface.c) — UDP 收包接口层:`create_udp_socket`([#L79](../../common-tools/linux/anticheat/trunk/interface.c#L79))、`init_system`,游戏进程通过 UDP 把可疑发包上报到这里。
- [common-tools/linux/anticheat/trunk/db_agent.c](../../common-tools/linux/anticheat/trunk/db_agent.c) — 把检测出的违规记录落库的 agent(对应下面三份 create_db_*.sql)。
- [common-tools/linux/anticheat/trunk/deploy/](../../common-tools/linux/anticheat/trunk/deploy/) — 按游戏(gf/seer/mole)分目录的部署配置,每个游戏一套 conf + db + config。
- [common-tools/linux/anticheat/trunk/query_ill/](../../common-tools/linux/anticheat/trunk/query_ill/) — PHP + Sphinx 的违规查询后台(`query.php`/`sphinxapi.php`),给运营查作弊记录。
- [common-tools/linux/proxy/src/CProxyRoute.cpp](../../common-tools/linux/proxy/src/CProxyRoute.cpp) — 代理路由核心:按 `route.conf` 把 `game+线路` 映射到 dbproxy/switch 地址(`reload_svr_config` 声明见 [CProxyRoute.hpp#L177](../../common-tools/linux/proxy/src/CProxyRoute.hpp#L177))。
- [common-tools/linux/proxy/src/CProxyDealBase.hpp](../../common-tools/linux/proxy/src/CProxyDealBase.hpp) — 各游戏代理处理基类:定义 get_friends_list/get_user_info/send_mail/add_items 等通用虚接口([#L7](../../common-tools/linux/proxy/src/CProxyDealBase.hpp#L7))。
- [common-tools/linux/proxy/src/CProxyDealMole.cpp](../../common-tools/linux/proxy/src/CProxyDealMole.cpp) — **摩尔庄园**专用代理处理(与 Seer/Seer2/Gf/Xhx/Hero/Mdd 各一份)。
- [common-tools/linux/proxy/etc/route.conf](../../common-tools/linux/proxy/etc/route.conf) — 路由表:实证一份内网拓扑(gf/seer/seer2/mole/hero/xhx/mdd 各自的 dbproxy/switch IP:port)。
- [common-tools/linux/gameproxy/trunk/extend_proto.h](../../common-tools/linux/gameproxy/trunk/extend_proto.h) — gameproxy 的客户端包头定义(`cli_proto_t`/`new_cli_proto_t`,带防外挂 seqno)([#L29](../../common-tools/linux/gameproxy/trunk/extend_proto.h#L29))。
- [common-tools/linux/advert_fliter/trunk/src/svr_proto.h](../../common-tools/linux/advert_fliter/trunk/src/svr_proto.h) — 广告过滤的服务端协议/与 dbproxy 通信(`send_msg_to_db`)([#L20](../../common-tools/linux/advert_fliter/trunk/src/svr_proto.h#L20))。
- [common-tools/linux/advert_fliter/trunk/src/CChatBayes.cpp](../../common-tools/linux/advert_fliter/trunk/src/CChatBayes.cpp) — 聊天贝叶斯垃圾判定;配合 `mmseg_stat/` 中文分词统计。
- [common-tools/linux/advert_fliter/trunk/conf/GameSvr.xml](../../common-tools/linux/advert_fliter/trunk/conf/GameSvr.xml) — 接入过滤的游戏清单(摩尔/卜克/赛尔2,各自 ChatProtocol 与词表/统计文件)。
- [common-tools/linux/main_login/cli_proto.hpp](../../common-tools/linux/main_login/cli_proto.hpp) — 通用客户端包头 `CliProto1`(17 字节,注释:摩尔/赛尔/小花仙共用)([#L36](../../common-tools/linux/main_login/cli_proto.hpp#L36))。
- [common-tools/linux/main_login/main_login.hpp](../../common-tools/linux/main_login/main_login.hpp) — 通用登录服务实现骨架(`Login` 类、`dbproto_t`)([#L78](../../common-tools/linux/main_login/main_login.hpp#L78))。
- [common-tools/linux/postcard_ser/src/](../../common-tools/linux/postcard_ser/src/) — 明信片下发服务:`mole_sendcard`/`seer_sendcard`/`gongfu_sendcard`/`fairy_sendcard`/`hero_sendcard` 各游戏一份。
- [common-tools/linux/proto/trunk/](../../common-tools/linux/proto/trunk/) — 各游戏协议命令字的 PHP 描述(`mole_proto.php`/`gf_proto.php`/`xhx_proto.php`/`mole2_proto.php` 等),测试脚本据此构包。
- [common-tools/linux/tools/online_monitor/](../../common-tools/linux/tools/online_monitor/) — Perl 在线人数监控(`mole_monitor_online_cli.pl` 内含真实内/外网登录服 IP:port)。
- [common-tools/linux/tools/core_detect/](../../common-tools/linux/tools/core_detect/) — 各游戏 core 文件巡检脚本(`findcore_mole.pl`/`findcore_seer*.pl`)+ 短信告警 `sendsms`。
- [common-tools/win/netstat-checker/code/](../../common-tools/win/netstat-checker/code/) — Windows 客户网络状况诊断工具(SMTP 上报、MD5),配 `doc/客户网络状况测试工具_文档.doc`。

## 协议 / 数据亮点(指到 proto.h 命令字、conf XML、sql 表)

本模块没有一份大而全的游戏 `proto.h`,协议亮点分散在几处:

- **客户端包头结构**:[gameproxy/trunk/extend_proto.h#L29](../../common-tools/linux/gameproxy/trunk/extend_proto.h#L29) 的 `new_cli_proto_t` 带 `seqno`(注释明说用于检查 GameCheater),并定义 `cli_proto_cmd_max = 30001`、`cli_proto_extend_port = 999`([#L45](../../common-tools/linux/gameproxy/trunk/extend_proto.h#L45))。包头版本另见 [main_login/cli_proto.hpp#L36](../../common-tools/linux/main_login/cli_proto.hpp#L36)(17 字节 `CliProto1`)。游戏侧完整命令字仍以各游戏模块为准,见 [../protocol.md](../protocol.md)。
- **游戏编号字典**(全模块通用的 game id):[anticheat/trunk/docs/gameid](../../common-tools/linux/anticheat/trunk/docs/gameid) —— `1 摩尔 / 2 赛尔 / 5 小花仙 / 6 功夫派 / 7 摩尔勇士 / 8 摩尔庄园2 / 10 赛尔2 / 32 account,pay,service`。这是判断各模块归属哪款游戏的硬证据。
- **路由表数据**:[proxy/etc/route.conf](../../common-tools/linux/proxy/etc/route.conf) 用 `game  线路(tel/cnc)  服务(dbproxy/switch)  IP  port` 五列描述全线内网拓扑,实证 mole/seer/seer2/gf/hero/xhx/mdd 的 dbproxy/switch 地址。
- **反作弊建表**:三份按游戏拆分的 SQL —— [create_db_mole.sql](../../common-tools/linux/anticheat/trunk/deploy/mole/db_agent/db/create_db_mole.sql)、[create_db_seer.sql](../../common-tools/linux/anticheat/trunk/deploy/seer/db_agent/db/create_db_seer.sql)、[create_db_gf.sql](../../common-tools/linux/anticheat/trunk/deploy/gf/db_agent/db/create_db_gf.sql)。表 `*_main_anticheat` / `*_today_anticheat` 记录每条违规(uid/cmd/ip/port/tw 与 sw 的间隔、频率、上限),`*_tw_config` / `*_sw_config` 为阈值配置。
- **聊天过滤配置**:[advert_fliter/trunk/conf/GameSvr.xml](../../common-tools/linux/advert_fliter/trunk/conf/GameSvr.xml) 用 `ChatProtocol`(如摩尔=62015、赛尔2=63103)+ 词表/贝叶斯统计文件描述接入游戏。

> 反作弊术语:`tw` = time-window(整体发包间隔/频率)、`sw` = single-watch(对单个命令字的连发监控,分 lvl0~3 容量等级)。

## 对我们有什么用(moleworld-server / MoleWorld-offline 能复用什么)

- **直用 — 游戏编号与协议包头常量**:[gameid](../../common-tools/linux/anticheat/trunk/docs/gameid) 的 game id、[cli_proto.hpp](../../common-tools/linux/main_login/cli_proto.hpp) 的 17 字节包头、`cli_proto_cmd_max=30001` 等是摩尔庄园协议的事实常量,离线服可直接对齐。
- **直用 — 摩尔相关协议/明信片实现**:[CProxyDealMole.cpp](../../common-tools/linux/proxy/src/CProxyDealMole.cpp) 与 [postcard_ser/src/mole_sendcard.cpp](../../common-tools/linux/postcard_ser/src/mole_sendcard.cpp) 给出了摩尔好友列表/用户信息/发邮件/加道具/明信片下发等具体包结构,离线复活做对应功能时可直接照抄字段布局。
- **参照 — 整体网络拓扑**:[route.conf](../../common-tools/linux/proxy/etc/route.conf) 帮助理解 client→proxy→dbproxy/switch 的真实分层,设计 moleworld-server 的进程划分时可作参照(离线版可大幅简化,不需要多线路 tel/cnc)。
- **参照 — 反作弊设计**:anticheat 的 UDP 上报 + 共享内存 + 分级连发检测是一套成熟方案,但对单机离线/复活项目几乎用不上,仅作"官方当年怎么防加速器"的设计参考。
- **基本无关**:advert_fliter 的中文聊天反垃圾(贝叶斯/mmseg)、win 端网络诊断、core 巡检/短信告警、shterm 堡垒机,都是大规模线上运维需求,离线复活无需复用。

## 注意 / 坑(纯二进制无源的部分、缺依赖、年代局限)

- **多处只有二进制、无源码**:`tools/online_monitor/*/mole_monitor_online_svr` 等是 ELF 可执行文件(已确认为编译产物,非源码);[shterm/shterm.tar.gz](../../common-tools/linux/shterm/shterm.tar.gz) 是打包的堡垒机,未解包(也不应解进仓库)。win 端 `doc/*.doc`/`*.vsd` 为二进制文档。
- **强依赖内部框架**:几乎所有 C/C++ 都 `#include <libtaomee/...>` 与 `<async_serv/...>`、`<libtaomee++/...>`,脱离 `c` / `libtaomee` 模块无法独立编译,见 [c.md](./c.md)。
- **配置含真实内网信息**:route.conf、online_monitor 脚本里有当年真实的内网/外网 IP:port(如登录服 114.80.98.17),仅作历史参考,均已失效。
- **游戏名核对(重要修正)**:本模块多处出现 `gf` 与 `xhx` 两个不同代号。据 [gameid](../../common-tools/linux/anticheat/trunk/docs/gameid) 与各 `deploy/gf/...conf`(`svc_name = "gf"`)实证:**`gf` = 功夫派(KungFu Pai,武侠战斗 MMO,货币"功夫豆"),`xhx` = 小花仙(另一款独立游戏)**。早期 baseline 曾把 `gf` 误标为"小花仙",此处更正:`gf` 是功夫派,小花仙对应的是 `xhx`。详见 [gf.md](./gf.md)。
- **年代局限**:SVN trunk 结构、cmake/手写 Makefile 混用、MyISAM/InnoDB 混表,均为 2010–2012 年风格;PHP 协议字典(proto/trunk)与 query_ill 的 Sphinx 后台依赖当年环境,直接跑通成本高。
- 回总入口见 [../../README.md](../../README.md);跨模块协议汇总见 [../protocol.md](../protocol.md),内容数据见 [../content-data.md](../content-data.md)。
