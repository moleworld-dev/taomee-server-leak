# 模块 monster · 淘米"小怪兽"养成社区网页游戏服务端

> 价值:★★  ·  游戏:小怪兽(养成社区,**非赛尔号**,见下文修正)  ·  源码量:约 5 万行(51,682 行 cpp/c/h)

## 是什么 / 在整套架构里的角色

`monster/` 是上海淘米(TaoMee)一款**怪兽养成 + 种植园 + 精灵收集 + 个人小屋 + 益智小游戏**的网页 Flash 页游服务端,2011 年立项(作者头注释 `Created 07/2011`,作者多为 luis 程龙 / henry 韩林)。它和摩尔庄园 `mole` 同属一套技术底座:基于 `libtaomee` 框架 + `async_serv`(AsynServ)宿主,游戏逻辑编译成 `.so`(`online` → `../bin/libmonsteronline.so`),由通用异步服务器进程加载运行。

### ★ 重要修正:不是"赛尔号"

baseline 把本模块标为"赛尔号(Seer)",**实证不成立**,应更正为内部代号"**小怪兽**":

- [monster/doc/db/db.org](../../monster/doc/db/db.org) 文件首行标题即「**小怪兽项目数据库设计**」,作者 `monster@taomee`。
- 部署文档 [monster/doc/小怪兽机器部署图.doc](../../monster/doc/小怪兽机器部署图.doc)、统计表 [monster/doc/msg/小怪兽stat.xls](../../monster/doc/msg/小怪兽stat.xls)。
- 数据库名 `db_monster_[0..9]`、MySQL 账号 `monster/monster@24`(见 [create_role.sh](../../monster/doc/db/create_role.sh))。
- 玩法是**养成 + 社区**(怪兽健康/愉悦/经验、种植园、精灵园、小屋、连连看)而非赛尔号式的**精灵对战**;`monster/` 下无任何 seer/赛尔号字样(仅 `test/xml/items.xml` 出现疑似共享测试数据)。
- 机器人配置 [robot.xml](../../monster/conf/robot.xml) 里的吉祥物名 "麦咭""克拉博""斗哥",指向金鹰卡通系 IP,而非赛尔号。

> 注:真正的赛尔号对战服请看兄弟模块 [pea.md](./pea.md)(pet.xml=精灵)。本目录的 `pet` 指的是养成游戏里可收集的"精灵宠物"(喵小黑、海特先生…),非战斗精灵。

### 进程拓扑(从源码与配置实证)

| 进程 | 产物 | 角色 |
|------|------|------|
| **login** | [login.cpp](../../monster/login/src/login.cpp) | 登录接入,向 switch 要一个可用 online(`as_msg_acquire_online=110`,见 [login_impl.cpp:54](../../monster/login/src/login_impl.cpp#L54)) |
| **switch** | [switch.cpp](../../monster/switch/src/switch.cpp) | 调度/分流 + 商店/每日对战逻辑([shop.cpp](../../monster/switch/src/shop.cpp)、[everyday_fight.cpp](../../monster/switch/src/everyday_fight.cpp)) |
| **online** | `libmonsteronline.so` | 主游戏逻辑(注册/登录/背包/房间/种植园/好友/精灵/益智),最大一坨 |
| **multi-server** | [work_proc.cpp](../../monster/multi-server/src/work_proc.cpp) | 二人益智游戏(跨/多服对战撮合) |
| **db-server** | [db-server/src](../../monster/db-server/src/) | 直连 MySQL 的持久化服务(按业务拆 role/plant/friend/...) |
| **db-cache-server** | [work_proc.cpp](../../monster/db-cache-server/src/work-proc/work_proc.cpp) | DB 前置缓存层 |
| **db-proxy** | [db-proxy](../../monster/db-proxy/) | 按 route.xml 把请求路由到分库 |
| **share-server** | [share-server/src](../../monster/share-server/src/) | 分享个人房间/最近登录用户([cli_proto.h](../../monster/share-server/src/cli_proto.h)) |
| **ucount-server** | [work_proc.cpp](../../monster/ucount-server/src/work-proc/work_proc.cpp) | 在线人数/计数统计 |
| **dirty_server** | [dirty_server](../../monster/dirty_server/) | 脏字过滤(含二进制 `dirty_agent`) |
| **thttpd** | [thttpd](../../monster/thttpd/) | 静态/策略文件 HTTP(含 crossdomain.xml) |

调用链大致为:Flash 客户端(as)→ login(要 online 地址)→ online(逻辑,`.so`)→ db-cache → db-proxy → db-server → MySQL,switch 负责 online 节点选择与全局商店。

## 关键子目录与文件

- [monster/online/src/cli_proto.cpp](../../monster/online/src/cli_proto.cpp) — 客户端协议总分发(`init_cli_handle_funs` 注册 cmd→handler,210KB,主逻辑入口)。
- [monster/online/src/cli_proto.h](../../monster/online/src/cli_proto.h) — online 客户端协议头(luis,2011-07-30)。
- [monster/online/src/function.cpp](../../monster/online/src/function.cpp) — 核心业务函数库(怪兽吃食、profile、宠物增删等)。
- [monster/online/src/plantation.cpp](../../monster/online/src/plantation.cpp) — 种植园:种植/维护/收获/阳光奖励逻辑。
- [monster/online/src/db_proxy.cpp](../../monster/online/src/db_proxy.cpp) — online 侧 DB 代理封装(各 svr_msg_db_* 的打包/回调)。
- [monster/online/src/c_user_cache.cpp](../../monster/online/src/c_user_cache.cpp) — 在线用户对象缓存(role/bag/room 内存态)。
- [monster/online/src/xml_parse.cpp](../../monster/online/src/xml_parse.cpp) — 加载 conf/*.xml(pet/game/system/imitate…)到内存表。
- [monster/online/src/multi_svr.cpp](../../monster/online/src/multi_svr.cpp) — online 对接 multi-server(二人益智)。
- [monster/common/message.h](../../monster/common/message.h) — **全局协议字典**:客户端 `as_msg_*` 与服务端 `svr_msg_db_*` 命令字定义。
- [monster/common/data_structure.h](../../monster/common/data_structure.h) — 跨进程共享数据结构(role/pet/plant/room…)。
- [monster/switch/src/shop.cpp](../../monster/switch/src/shop.cpp) — 全局商店物品轮换。
- [monster/multi-server/src/work_proc.cpp](../../monster/multi-server/src/work_proc.cpp) — 二人益智撮合/对局(henry,2011-12)。
- [monster/db-server/src/role-info/role_info.cpp](../../monster/db-server/src/role-info/role_info.cpp) — 角色表读写(分库分表落地)。
- [monster/db-server/src/plantation/plantation.cpp](../../monster/db-server/src/plantation/plantation.cpp) — 种植园持久化。
- [monster/conf/](../../monster/conf/) — 全部内容配置(见下节)。
- [monster/doc/db/](../../monster/doc/db/) — 全套建表脚本 `create_*.sh`(role/pet/plant/room/friend/badge/task…)+ `db.org` 设计文档。
- [monster/doc/msg/message_svr.docx](../../monster/doc/msg/message_svr.docx) — 服务端协议/消息设计文档。
- [monster/bin/](../../monster/bin/) — 宿主二进制 `AsynServ-1.6.x-64bit`、`dirty_agent`、`flashpolicyd.tar.gz`(纯二进制,无源)。

## 协议 / 数据亮点

### 命令字(均在 [monster/common/message.h](../../monster/common/message.h),真实行号)

客户端→服务端 `as_msg_*`(十进制段 40000+):

- [`as_msg_acquire_userinfo = 40101`](../../monster/common/message.h#L33) — as 向 online 发注册请求(对应 `process_register`,[cli_proto.cpp:427](../../monster/online/src/cli_proto.cpp#L427))。
- [`as_msg_login_online = 40102`](../../monster/common/message.h#L34) — 登录 online。
- [`as_msg_enter_two_puzzle = 40134`](../../monster/common/message.h#L65) — 进入二人益智游戏(multi-server)。

服务端内部 DB 协议 `svr_msg_db_*`(0x8000 段):

- [`svr_msg_db_add_role = 32768 (0x8000)`](../../monster/common/message.h#L185) — 新增 role。
- [`svr_msg_db_add_role_value = 32780 (0x800c)`](../../monster/common/message.h#L197) — 给 role 字段累加值(金币/经验等)。

账号系统对接 `account_msg_*`(0xA000 段),如 [`account_msg_login_verify = 0xA026`](../../monster/common/message.h#L134)。

### conf 内容配置([monster/conf/](../../monster/conf/))

- [pet.xml](../../monster/conf/pet.xml) — 可收集精灵表(id/name/稀有度 type 1常见~4濒危/三档颜色),如"喵小黑""呼呼侠"。
- [game.xml](../../monster/conf/game.xml) — 益智小游戏"怪卡连连看"关卡/星级分数/掉落物。
- [system.xml](../../monster/conf/system.xml) — 全局数值:怪兽健康/愉悦衰减时间、种植园成长/维护参数、每日金币/经验上限。
- [imitate.xml](../../monster/conf/imitate.xml)(158KB)、[name_set.xml](../../monster/conf/name_set.xml)(142KB)— 模仿题库 / 随机昵称词库。
- [robot.xml](../../monster/conf/robot.xml) — 街道机器人(吉祥物 NPC)。

### SQL 表(建表脚本在 [monster/doc/db/](../../monster/doc/db/))

分库分表规则(见 db.org):`db_suffix = uid % 10`,`table_suffix = (uid/100) % 100`,即 10 库 × 100 表。

- `t_role_[0..99]` — 角色主表,含 `coins/mon_health/mon_happy/mon_exp/mon_level` 等养成核心字段([create_role.sh](../../monster/doc/db/create_role.sh))。
- `t_pet_[0..99]` — 精灵收集(pet_id/total_num/follow_num,[create_pet.sh](../../monster/doc/db/create_pet.sh))。
- `t_plant_[0..99]` — 种植园(hole_id/plant_id/growth/maintain,[create_plant.sh](../../monster/doc/db/create_plant.sh))。

> 完整协议矩阵汇总见 [../protocol.md](../protocol.md);内容配置专题见 [../content-data.md](../content-data.md)。

## 对我们有什么用(摩尔庄园复活)

定位:**参照为主,部分直用**。对 [moleworld-server / MoleWorld-offline] 而言,monster 是"摩尔庄园同底座的姊妹项目",架构高度同构(libtaomee + async_serv + .so 逻辑 + 分库分表 + login/switch/online/db 链),是极佳的**架构对照样本**。

- **直用(可直接拿来)**:无——它服务的是另一款游戏的客户端,协议号/数据结构与摩尔庄园不互通。
- **参照(设计参考,价值高)**:
  - **分库分表落地范式**:`uid%10` + `(uid/100)%100` 的 10×100 拆表、`db-proxy` 路由,可直接照搬到摩尔离线存档分片思路。
  - **进程分层模板**:login→switch→online→db-cache→db-proxy→db-server 的清晰职责切分,比单看 mole 更易读(命名更规整)。
  - **养成数值模型**:[system.xml](../../monster/conf/system.xml) 的健康/愉悦衰减、种植园成长公式,与摩尔庄园拉姆/种植玩法同源,可作数值复刻参考。
  - **协议字典写法**:[message.h](../../monster/common/message.h) 把 as/svr/account 三段命令字集中带中文注释,是给摩尔补协议文档的好模板。

## 注意 / 坑

- **纯二进制无源**:[monster/bin/](../../monster/bin/) 下的 `AsynServ-1.6.x-64bit`、`dirty_agent`、`flashpolicyd.tar.gz` 均为编译产物,宿主框架源在 `libtaomee` 模块(.tar.gz)里,本目录不可独立编译运行。
- **依赖外部框架**:Makefile 需 `-ltaomee -ltaomee++ -lxml2 -lssl -lglib-2.0` 及 `<async_serv/dll.h>`,缺 libtaomee 头/库则无法构建([online Makefile](../../monster/online/src/Makefile))。
- **建表脚本含硬编码生产信息**:`create_*.sh` 里写死了内网 host(`10.1.1.24`)与库口令 `monster@24`,仅供参考,勿当真实凭据。
- **db-server 模块齐全但 online 是核心**:share-server / ucount-server / multi-server 体量小;真正业务厚度在 `online/src`(`cli_proto.cpp` + `db_proxy.cpp` + `function.cpp` 三件套占近半源码)。
- **年代局限**:32/64 位混编(thttpd 有 32bit/64bit 两版),依赖 glib-2.0 老接口,直接在现代发行版上编需要补 toolchain。
- **赛尔号标注已修正**:见上文"重要修正",勿沿用 baseline 的"赛尔号"结论。

---
↩ 返回 [总入口 README](../../README.md)
