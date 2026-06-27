# 模块 mole · 摩尔庄园主服务端全套(登录 / 在线逻辑 / 游戏服 / DB / 缓存 / 注册 / 小屋 / 策略文件)

> 价值:★★★  ·  游戏:摩尔庄园  ·  源码量:约 32 万行(实测 `wc -l` C/C++ 头源文件 324357 行,2429 个文件)
> 📖 深度功能解析(每个部分实际在干什么):[deep/mole.md](../deep/mole.md) · 彩蛋见 [easter-eggs.md](../easter-eggs.md)

这是整套泄漏里**最有价值、最完整**的一块——摩尔庄园(mole.61.com)的真·后端集群源码,从客户端 socket 接入(`online`)、玩法逻辑、小游戏(`gamesvr`)、持久层(`dbsvr`)、缓存(`cache_serv`)到登录/注册/小屋/Flash 策略文件,一应俱全。它就是 `moleworld-server` 复活摩尔庄园时**协议与数值的权威来源**。

---

## 是什么 / 在整套架构里的角色

摩尔庄园是 2008 年上线的 Flash 网页页游,后端是上海淘米典型的 **多进程 C/C++ + 共享内存队列(shmq) + UDP/TCP + 组播(mcast)广播** 的分布式集群。`mole/` 即这一集群的全部进程源码。各进程职责(从配置文件 `readme`/`bench.conf` 实证):

| 进程 | 角色 | 关键证据 |
|------|------|----------|
| **login** | 登录/鉴权:校验淘米账号、下发会话、引导到 online | [mole/login/myinit.cpp](../../mole/login/myinit.cpp) 读 `mole_dbproxy`/`time_ip`/`master_ip` |
| **register / new_login** | 角色注册(建摩尔)、新版登录入口 | [mole/register/](../../mole/register/) `create_role_cmd` |
| **online** | 核心:客户端长连接接入 + 全部庄园玩法逻辑(最大模块,455 文件 8.26MB) | [mole/online/readme](../../mole/online/readme) 描述 `online_ip 1863`、worker_num、组播 |
| **gamesvr** | 独立小游戏服务器(连连看、卡丁车 kart、扑泡球、滑沙、飞行器、卡牌等),so 插件化 | [mole/gamesvr/readme](../../mole/gamesvr/readme)、[mole/gamesvr/games/](../../mole/gamesvr/games/) |
| **cache_serv** | 缓存服:在线态/宠物态/家园热度等高频读缓存,定期回写 DB(`timer_step_time 1800`) | [mole/cache_serv/doc/readme.txt](../../mole/cache_serv/doc/readme.txt) |
| **dbsvr** | 持久层:`ser`(DB 进程)+`com`(proxy)+`logser`(日志库)+`payser`(支付),最大代码体(1248 文件 11.78MB) | [mole/dbsvr/readme](../../mole/dbsvr/readme) |
| **homeserv** | 小屋/家园服务(HomeServer) | [mole/homeserv/homeserv.c](../../mole/homeserv/homeserv.c) |
| **switch** | 网关/分流(SwitchServer),C++ 实现的连接调度 | [mole/switch/switch.cpp](../../mole/switch/switch.cpp) |
| **flashpolicy** | Flash 跨域策略文件服务器(843 端口),Flash 网页客户端连接前必须先取 policy | [mole/flashpolicy/service.c](../../mole/flashpolicy/service.c) |
| **picserv / httppic / pic 相关** | 玩家自定义图片(留言板/照片)存取 | [mole/picserv/](../../mole/picserv/) |
| **ip_counter / downloadstat / errreport / school_bar** | 运营辅助:IP 统计、下载统计、错误上报、校园吧 | [mole/ip_counter/](../../mole/ip_counter/) |

**数据流(典型登录-玩法链路)**:Flash 客户端 → `flashpolicy`(取策略)→ `login`(鉴权,经 `mole_dbproxy` 查淘米账号)→ `online`(进场景,所有逻辑都在这,玩家广播走 `mcast` 组播 + `communicator`)→ 高频态读写经 `cache_serv`,落地经 `dbsvr`(udp_dbserver / dbproxy)→ 进小游戏时由 `online` 转交 `gamesvr`。`central_online` 是跨 online 实例的中心服(`central_online_ip:4431`),做全局协调。

**年代实证**:`online/changelog` 最早记录 2008-11-20(感恩节摩尔餐厅、连连看、删除"摩尔变僵尸"),最新到 2010-03-03(拉姆第五阶段技能)。配置数据(`conf/*.xml` 文件名带 `TD20111223`、`af_*` 天使斗士)延伸到 2011 年内容。属于摩尔庄园的**鼎盛期源码快照**。

---

## 关键子目录与文件(可跳转 + 一句话作用)

- [mole/online/](../../mole/online/) — 核心在线逻辑服,几乎所有庄园玩法都在此(455 文件)。
- [mole/online/readme](../../mole/online/readme) — 部署说明 + `bench.conf` 全字段含义,理解架构的入口文档。
- [mole/online/protoheaders.h](../../mole/online/protoheaders.h) — 进程间通信的三个报文头:`server_proto_t`(对 DB/中心服)、`communicator_header_t`(广播)、`msglog_proto_t`。
- [mole/online/proto.h](../../mole/online/proto.h) — 在线服全量错误码枚举(ERR_*,见下节),覆盖好友/宠物/家园/拉姆/小屋/许愿池等几乎所有系统。
- [mole/online/dbproxy.c](../../mole/online/dbproxy.c) — online 到 DB 的代理读写(186KB,最繁忙文件)。
- [mole/online/pet.c](../../mole/online/pet.c) / [mole/online/mole_pasture.c](../../mole/online/mole_pasture.c) — 宠物(拉姆)与牧场系统逻辑。
- [mole/online/angel_fight.c](../../mole/online/angel_fight.c) — 天使斗士(摩尔对战)战斗逻辑;配套 `conf/af_skills.xml`/`af_prize.xml`。
- [mole/online/lahm_classroom.c](../../mole/online/lahm_classroom.c) — 拉姆课堂(答题/养成),配 `conf/class_questions.xml`。
- [mole/online/conf/items.xml](../../mole/online/conf/items.xml) — 全道具表(1.26MB,服饰/家具/食物全 ID+价格+属性),复活时**直用**的核心数据。
- [mole/online/conf/maps.xml](../../mole/online/conf/maps.xml) — 地图/场景定义(刷新点、聚会规则)。
- [mole/gamesvr/](../../mole/gamesvr/) — 小游戏服务器主体,`main.c` 版本号 1.0.0。
- [mole/gamesvr/proto.h](../../mole/gamesvr/proto.h) — 小游戏通用协议(进场/准备/分数/动作,见下节命令字)。
- [mole/gamesvr/games/](../../mole/gamesvr/games/) — 每个小游戏一目录(kart 卡丁车、pop 泡泡、link_link 连连看、flight_game 飞行器、new_card 卡牌等)。
- [mole/dbsvr/](../../mole/dbsvr/) — 持久层全套(`ser` DB 进程 / `com` proxy / `logser` / `payser`)。
- [mole/dbsvr/ser/src/Croute_func.cpp](../../mole/dbsvr/ser/src/Croute_func.cpp) — DB 请求路由分发巨型文件(1.06MB,单文件最大)。
- [mole/dbsvr/include/proto.h](../../mole/dbsvr/include/proto.h) — DB 层全量命令字(388KB,最大 proto.h,所有 `*_CMD` 宏)。
- [mole/dbsvr/ser/sql/get_grade.sql](../../mole/dbsvr/ser/sql/get_grade.sql) — 经验→等级换算 MySQL 函数(数值表,可直接抄)。
- [mole/cache_serv/pkg/proto.h](../../mole/cache_serv/pkg/proto.h) — 缓存服命令字(`ROOMINFO_*`/`USER_PET_*`),含 `NEED_UPDATE` 写标志位设计。
- [mole/login/](../../mole/login/) — 登录服(C++),`mydbproxy`/`mysvrlist`/`myprocess` 是鉴权与服务器列表逻辑。
- [mole/switch/switch.cpp](../../mole/switch/switch.cpp) — 网关分流服。
- [mole/flashpolicy/service.c](../../mole/flashpolicy/service.c) — Flash 策略文件服务(复活 Flash 客户端必备)。

---

## 协议 / 数据亮点

### 报文头(三套)
见 [mole/online/protoheaders.h](../../mole/online/protoheaders.h):内部统一头 `server_proto_t { len, seq, cmd(uint16), ret, id, body[] }`;广播头 `communicator_header_t` 多带 `mapid/opid`(场景内广播)。小游戏头见 [mole/gamesvr/proto.h](../../mole/gamesvr/proto.h):`protocol_t { len, ver, cmd(uint32), id, ret, body[] }`,加密 key `DES_KEY ">ta:mee<"`(见 [mole/online/proto.h](../../mole/online/proto.h))。

### 命令字真实样例
- 小游戏进场/分数(client↔gamesvr):`proto_player_enter = 30001`、`proto_game_score = 30006`、卡牌 `proto_new_card_get_card_info = 30019` —— [mole/gamesvr/proto.h#L37](../../mole/gamesvr/proto.h#L37)、[#L42](../../mole/gamesvr/proto.h#L42)、[#L56](../../mole/gamesvr/proto.h#L56)。
- 缓存服命令(online↔cache_serv):`ROOMINFO_CLIENT_DATA_GET_CMD = 0xB469`、`USER_PET_SKILL_SET_CMD = (0x10EA | NEED_UPDATE)` —— [mole/cache_serv/pkg/proto.h#L32](../../mole/cache_serv/pkg/proto.h#L32)、[#L53](../../mole/cache_serv/pkg/proto.h#L53)。命令字高位 `0x0100`(NEED_UPDATE)标识写操作、`0xFC00`(RROTO_ROUTE_FIELD)做路由,是淘米一贯的命令字编码约定。
- 错误码:`ERR_client_not_proc = 10000`、`ERR_multi_login = 10010`、`ERR_invalid_session = 10001` —— [mole/online/proto.h#L23](../../mole/online/proto.h#L23) 起的大枚举,几乎是摩尔庄园所有业务错误的字典。

> 因 DB/cache 的 proto.h 极大(388KB / 198KB),命令字全表与跨模块对照请见专题 [../protocol.md](../protocol.md)。

### 配置/SQL 数据亮点
- [mole/online/conf/items.xml](../../mole/online/conf/items.xml):全道具(`Kind`/`Item`,带 ID 段、价格 `Price`、米币 `Mibi`、可交易性 `Tradability`、图层 `Layer`),是复活商城/换装的**金矿**。
- [mole/online/conf/maps.xml](../../mole/online/conf/maps.xml)、`swap.xml`(交换)、`exchange.xml`、`DigTreasureElement.xml`(挖宝)、`af_skills.xml`(天使斗士技能)、`class_questions.xml`(拉姆课堂题库)。
- SQL:`get_grade.sql` 经验→等级表、`appeal_ex.sql`、各 `table.sql` 建表骨架(详见 [../content-data.md](../content-data.md))。

---

## 对我们有什么用(moleworld-server / MoleWorld-offline)

**直用(代码/数据可直接拿来)**:
- **协议字典与报文格式**:`online/protoheaders.h`、`gamesvr/proto.h`、各 `proto.h` 命令字 + 错误码,是复活服务端**必须 1:1 复刻**的契约——客户端(Ruffle/touchHLE 跑的真 Flash 包)写死了这些 cmd/字段布局,服务端不照抄就连不上。
- **数值与内容数据**:`conf/*.xml`(道具/地图/技能/题库)、`get_grade.sql` 等级表,可直接导入离线服,省去逆向客户端猜数值。
- **业务规则**:每日上限、好友/黑名单、宠物技能学习、许愿池等规则,在 `proto.h` 错误码 + 对应 `*.c` 里有完整定义,离线复刻可照搬判定逻辑。

**参照(设计参考,不直接编译)**:
- **进程拆分与数据流**:login/online/gamesvr/cache/dbsvr 的分工是为千万级并发设计的;`moleworld-server` 离线/小规模复活不必照搬这套多进程+组播+共享内存,可合并为单进程/少进程,但**端到端协议序列**(取 policy→登录→进场→玩法)要参照实现。
- **cache_serv 的"NEED_UPDATE 命令字 + 定时回写"模式**:离线服可简化为内存态 + 退出时落盘。

---

## 注意 / 坑

- **依赖内部框架库**:全模块依赖 `libtaomee`(`<libtaomee/conf_parser/config.h>`、`<libtaomee/log.h>`、`tm_dirty` 脏字过滤等),源码在仓库另一处的两个 `.tar.gz`,**未解包不能直接编译**;见 [libtaomee 模块](./libtaomee.md)。底层网络/共享内存还依赖 `c` 共享库,见 [c.md](./c.md)。
- **无客户端、无美术**:仓库只含服务端;Flash 客户端 swf 与素材需另行获取(我们用 Ruffle/touchHLE 跑真包)。`conf/*.xml` 里的道具只有 ID/名字/数值,**没有对应贴图**。
- **年代局限**:GCC 旧版(`__attribute__((packed))`、`body[]`/`body[0]` 柔性数组混用)、依赖 32/64 位与字节序(大量 `htonl`),现代工具链需适配;`TW_VER` 等条件编译针对台服。
- **dbsvr 的 `bin/` 目录可能含编译产物/无源二进制**,以源码(`ser/src`、`com`)为准。
- **加密弱**:`DES_KEY ">ta:mee<"` 等硬编码 DES 密钥就在 `proto.h` 里,仅作历史参考,复活时按需保留兼容性,勿当安全机制。
- 部分 `conf` 文件含 BOM/GBK 混编(如 `maps.xml` 带 BOM、`proto.h` 中文注释为 GBK 乱码),解析时注意编码。

---

回总入口:[../../README.md](../../README.md)
