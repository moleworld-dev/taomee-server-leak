# 模块 gf · 功夫派(KungFu Pai)完整游戏服务端

> 价值:★★★  ·  游戏:功夫派(KungFu Pai)  ·  源码量:约 22 万行(C/C++ 实测 220490 行,2016 文件)

> ⚠️ **游戏名修正**:baseline 曾把本模块误标为"小花仙"。经实证为**功夫派(KungFu Pai)** —— 武侠/格斗题材 MMO。证据:货币叫"**功夫豆**"([gf/trade_svr/conf/items_nor.xml:837](../../gf/trade_svr/conf/items_nor.xml)),怪物体系是"**木人/木王/铜人**"([gf/battle_svr/conf/stage_easy/monsters.xml](../../gf/battle_svr/conf/stage_easy/monsters.xml)),问答配置直接写"功夫派里可以选几种角色?"([gf/gongfu_online/conf/puzzle.xml:123](../../gf/gongfu_online/conf/puzzle.xml))。全仓未见任何"小花仙"字样。

---

## 是什么 / 在整套架构里的角色

`gf/` 是淘米"功夫派"这款网页 Flash 页游的**完整一套服务端**,不是单个进程,而是一组按职责拆分的服务,通过 `switch`(转发/路由层)+ `db`(落库)+ `cache`(用户态缓存)串成集群。和摩尔庄园同源——都建立在 `libtaomee` / `libtaomee++` 框架与 `async_serv` 异步网络库之上,业务逻辑编译成 `.so`(如 `libkf.so`)由通用网络底座动态装载(见 [gf/gongfu_online/Makefile](../../gf/gongfu_online/Makefile),`BINS = libkf.so`、`SRCS = dll_interface.cpp ...`)。

它在整套泄漏源码里是**内容最完整、最适合做"一款完整页游怎么落地"通读样本**的模块:登录、在线主逻辑、独立战斗服、交易、家园、聊天、邮件、删号、监控全有,XML 内容配置高达 522 个,是各游戏里最全的。

**进程/服务拓扑(按目录职责)**:

| 服务目录 | 角色 | 证据 |
|---|---|---|
| `gongfu_login` / `main_login` / `login` | 登录与会话校验,不支持游客登录 | [gf/gongfu_online/login.cpp:51](../../gf/gongfu_online/login.cpp) 注释 "Don't support guest login";[gf/login/login_impl.cpp](../../gf/login/login_impl.cpp) `version_str="2010-08-05"` |
| `gongfu_online` | **在线主逻辑服**(背包/任务/家园/商城/交易/灵兽/称号…),编出 `libkf.so` | [gf/gongfu_online/Makefile](../../gf/gongfu_online/Makefile) |
| `battle_svr` | **独立战斗服**:回合/格斗战斗结算 + 怪物 AI + 技能引擎 | [gf/battle_svr/battle_logic.cpp](../../gf/battle_svr/battle_logic.cpp)、[gf/battle_svr/ai/](../../gf/battle_svr/ai/) |
| `gongfu_switch` / `btl_switch` / `cache_switch` | 转发/路由层(在线、战斗、缓存各一套 switch) | [gf/gongfu_switch/online.cpp](../../gf/gongfu_switch/online.cpp) |
| `cache_svr` / `cache` | 用户信息缓存服(USERINFO_* 协议) | [gf/cache_svr/pkg/proto.h](../../gf/cache_svr/pkg/proto.h) |
| `db` | DB 服(`ser/` 二进制服务 + `sql/` 建库脚本 + `php/`),负责 PP_DB 落库 | [gf/db/ser/readme](../../gf/db/ser/readme) |
| `home_svr` / `kf_home` | 家园服(种植/施肥) | [gf/home_svr/](../../gf/home_svr/) conf 有 `plants.xml`/`ferilizer.xml` |
| `trade_svr` | 交易/市场服 | [gf/trade_svr/trade.cpp](../../gf/trade_svr/trade.cpp) |
| `chat_svr` | 聊天/分组服 | [gf/chat_svr/](../../gf/chat_svr/) |
| `mail_project` | 邮件系统(自带 mail_dbserver) | [gf/mail_project/](../../gf/mail_project/) |
| `deluser_svr` | 删号/清理服 | [gf/deluser_svr/](../../gf/deluser_svr/) |
| `kf_common` | **共享业务库**(`kf/` 命名空间):玩家属性/灵兽/战队/道具管理等,被各服 `#include <kf/...>` | [gf/kf_common/src/](../../gf/kf_common/src/) |

**年代**:changelog 显示框架 2009-03 起步([gf/gongfu_online/doc/changelog.txt](../../gf/gongfu_online/doc/changelog.txt)),战斗服 2010-02([gf/battle_svr/doc/changelog.txt](../../gf/battle_svr/doc/changelog.txt)),登录服源码版本串到 2010-08,内容配置(台服 `taiwan`、开心网 `kaixin` 分支)延续到 2012–2013。

---

## 关键子目录与文件(可跳转 + 一句话作用)

- [gf/gongfu_online/](../../gf/gongfu_online/) — 在线主逻辑服源(279 文件,14.8 MB),全游戏功能汇集地。
- [gf/gongfu_online/player.cpp](../../gf/gongfu_online/player.cpp) — 玩家对象与上线/属性同步主流程。
- [gf/gongfu_online/item.cpp](../../gf/gongfu_online/item.cpp) — 道具/背包系统(22 万字节,最大业务单文件)。
- [gf/gongfu_online/task.cpp](../../gf/gongfu_online/task.cpp) — 任务系统(配套 [conf/tasks.xml](../../gf/gongfu_online/conf/tasks.xml))。
- [gf/gongfu_online/trade.cpp](../../gf/gongfu_online/trade.cpp) — 在线侧交易逻辑。
- [gf/gongfu_online/summon_monster.cpp](../../gf/gongfu_online/summon_monster.cpp) — 召唤兽/灵兽(宠物)系统(配 [conf/summon_monster.xml](../../gf/gongfu_online/conf/summon_monster.xml))。
- [gf/gongfu_online/Makefile](../../gf/gongfu_online/Makefile) — 把业务编成 `libkf.so`,可见全部源文件清单与 `-DVERSION_TAIWAN/-DVERSION_KAIXIN` 分支宏。
- [gf/battle_svr/](../../gf/battle_svr/) — 独立战斗服(1172 文件),回合/格斗结算引擎。
- [gf/battle_svr/battle_logic.cpp](../../gf/battle_svr/battle_logic.cpp) — 战斗主结算逻辑。
- [gf/battle_svr/skill.cpp](../../gf/battle_svr/skill.cpp) — 技能引擎(配 [conf/skills.xml](../../gf/battle_svr/conf/skills.xml),54 万字节)。
- [gf/battle_svr/ai/BossAI.cpp](../../gf/battle_svr/ai/BossAI.cpp) — Boss AI(目录下还有 GuardAI/HighLvAI/LongRangeAtkAI/SpecialMonAI 等多套行为树)。
- [gf/battle_svr/monster.cpp](../../gf/battle_svr/monster.cpp) — 怪物(木人/木王/铜人)对象。
- [gf/db/ser/src/proto.h](../../gf/db/ser/src/proto.h) — DB 服协议头(`gf_*_cmd` 命令字集合)。
- [gf/db/ser/readme](../../gf/db/ser/readme) — DB 服运行说明 + `bench.conf` 各项含义(含 MySQL/switch/proxy 连接参数)。
- [gf/db/ser/sql/](../../gf/db/ser/sql/) — 建库脚本(`create_*.sh` + `t_gongfu.py`/`t_gf_attire.py` 等分表生成器)。
- [gf/cache_svr/pkg/proto.h](../../gf/cache_svr/pkg/proto.h) — 用户信息缓存服协议(`USERINFO_*` 命令字,10 万字节)。
- [gf/kf_common/src/](../../gf/kf_common/src/) — 共享 `kf/` 业务库:[player_attr.hpp](../../gf/kf_common/src/player_attr.hpp)、[pet_attr.cpp](../../gf/kf_common/src/pet_attr.cpp)、[fight_team.cpp](../../gf/kf_common/src/fight_team.cpp)、[item_manager.hpp](../../gf/kf_common/src/item_manager.hpp)。
- [gf/gongfu_online/conf/](../../gf/gongfu_online/conf/) — 在线服 XML 内容配置(任务/商城/合成/称号/灵兽…)。
- [gf/battle_svr/conf/](../../gf/battle_svr/conf/) — 战斗服 XML(技能/buff/光环/特效/各难度关卡 monsters.xml)。
- [gf/gongfu_online/doc/编码规范.doc](../../gf/gongfu_online/doc/编码规范.doc) — 团队内部 C++ 编码规范(20 万字节,二进制 .doc)。

---

## 协议 / 数据亮点

**命令字(cmdid)实证** —— 各服自带 `proto.h`,采用 `0xXXXX | NEED_UPDATE` 的位掩码风格(`NEED_UPDATE=0x0100` 表示需写库,`NEED_CACHED_DEAL=0x0200`,见 [cache_svr/pkg/proto.h:29](../../gf/cache_svr/pkg/proto.h)):

- DB 服:`gf_get_user_info_cmd = 0x0601`([gf/db/ser/src/proto.h:34](../../gf/db/ser/src/proto.h)) —— 拉取玩家全量信息。
- DB 服:`gf_synchro_user_gfcoin_cmd = 0x0618 | NEED_UPDATE`([gf/db/ser/src/proto.h:67](../../gf/db/ser/src/proto.h)) —— 同步"功夫豆"(gfcoin)货币,再次坐实游戏货币。
- 缓存服:`USERINFO_LOGIN_CMD = 0x0001`([gf/cache_svr/pkg/proto.h:33](../../gf/cache_svr/pkg/proto.h)) —— 登录态查询;同文件 `USERINFO_REGISTER_CMD/ADD_FRIEND/SET_VIP_FLAG` 等是与摩尔庄园同构的通用账号协议。
- 趣味彩蛋:命令字命名带工程师情绪,如 `gf_fuck_version_check_cmd = 0x0604`、`gf_bitch_sell_items_cmd = 0x0605`([gf/db/ser/src/proto.h:38-40](../../gf/db/ser/src/proto.h))。

完整命令字体系与各服 proto.h 比对见 [../protocol.md](../protocol.md)。

**XML 内容配置(522 个,全仓最全)** —— 数值与玩法几乎全部外置到 XML,用 XML Spy 编辑(文件头可见 `edited with XML Spy ... by opec`):

- 装备/外观:[items_attire.xml](../../gf/gongfu_online/conf/items_attire.xml)(约 120 万字节,最大单表)、[items_nor.xml](../../gf/gongfu_online/conf/items_nor.xml)、[suit.xml](../../gf/gongfu_online/conf/suit.xml)(套装)。
- 战斗:[skills.xml](../../gf/battle_svr/conf/skills.xml)(技能,带 `Dir`/`Effect` 分步说明)、[effect.xml](../../gf/battle_svr/conf/effect.xml)、[buff.xml](../../gf/battle_svr/conf/buff.xml)、[aura.xml](../../gf/battle_svr/conf/aura.xml)。
- 关卡:[battle_svr/conf/stage_easy/monsters.xml](../../gf/battle_svr/conf/stage_easy/monsters.xml) 等 easy/normal/hard/hell/epic/legend/tower 多难度怪物表。
- 灵兽:[summon_monster.xml](../../gf/gongfu_online/conf/summon_monster.xml)(走兽/飞羽/上古/元素/变异/远古 6 系)、[pet_attribute.xml](../../gf/gongfu_online/conf/pet_attribute.xml)。
- 运营:[dailyActivity.xml](../../gf/gongfu_online/conf/dailyActivity.xml)、[storeSales.xml](../../gf/gongfu_online/conf/storeSales.xml)、[compose.xml](../../gf/gongfu_online/conf/compose.xml)(合成)、[tasks.xml](../../gf/gongfu_online/conf/tasks.xml)。

**SQL/表结构** —— `db/ser/sql/` 下是 shell+python 生成脚本而非裸 .sql,直接 `mysqladmin CREATE` + 内嵌 `CREATE TABLE`。例:[create_task_day.sh](../../gf/db/ser/sql/create_task_day.sh) 建 `TASK_DAY_DB.t_task_day`(`type/userid/count`,ENGINE=memory)。注意 `t_gongfu.py`/`t_gf_attire.py`/`insert_attire.py` 等按 userid 分表生成。`bench.conf` 里默认 MySQL 账号(`root`/`ta0mee`、`jim`)与 `INIT_XIAOMEE 2000 #摩尔豆`(沿用了摩尔豆字段名,印证 db 层是从 mole 复用而来,见 [gf/db/ser/readme](../../gf/db/ser/readme))。

---

## 对我们有什么用(moleworld-server / MoleWorld-offline)

**定位:以"参照"为主,少量"直用"。** 功夫派是武侠战斗 MMO,玩法核心(回合/格斗战斗 + 灵兽)与摩尔庄园(休闲社区养成)不同,业务逻辑不能照搬;但它和 mole 是**同一套底座、同一套账号/缓存/DB 协议骨架**,因此基础设施层高度可借鉴。

**直用(可直接拿来对照实现)**:
- 账号/会话/好友/VIP 通用协议:`cache_svr/pkg/proto.h` 的 `USERINFO_*` 命令字与 mole 同构,做 moleworld-server 的登录/账号子系统时可直接拿来核对字段与命令字编号。
- `db/ser/sql/` 的建库脚本与分表思路(按 userid hash 分表、memory 引擎缓存表)可直接复用到离线服的存档落库设计。
- `0xXXXX | NEED_UPDATE | NEED_CACHED_DEAL` 的"命令字自带落库/缓存语义"位掩码约定,是淘米全系通用范式,做离线 server 时可沿用同一套约定保持协议一致性。

**参照(只作设计参考)**:
- 服务拆分拓扑(login → online(libkf.so)→ switch → db/cache,战斗独立成服)是淘米标准集群形态,对理解 mole 同款架构、设计 moleworld-server 的进程划分极有价值——尤其"业务编成 .so 由网络底座装载"这一模式。
- `kf_common` 的"共享业务库 + 各服 include"组织方式,可参照设计 moleworld 的公共逻辑层。
- XML 数值外置 + reload 机制(见 [gf/gongfu_online/tools/reload/](../../gf/gongfu_online/tools/reload/))是策划热更思路的好范例。
- 与 [mole.md](./mole.md) 对读可快速看出"哪些是通用底座、哪些是各游戏特化",对复活 mole 帮助最大。

---

## 注意 / 坑

- **不含客户端/美术**:整套泄漏只有服务端,Flash 客户端、SWF、贴图全无;功夫派玩法(技能动画、格斗手感)无法仅凭服务端还原。
- **重依赖未泄漏的框架库**:`libtaomee`/`libtaomee++`/`async_serv`/`libkfcommon`/`liblua.a`/`glib-2.0`/`libxml2` 是编译前置(见 [Makefile](../../gf/gongfu_online/Makefile) `LIBS`);其中 `libtaomee` 仅以 .tar.gz 形态在 `libtaomee` 顶层模块出现,`async_serv`/`libkfcommon` 需自行从 `c`/`common-tools` 等共享底座拼凑,直接 `make` 跑不通。
- **DB 层与 mole 纠缠**:`bench.conf` 仍写 `INIT_XIAOMEE`(摩尔豆),说明 db 服是从摩尔庄园复制改名,字段语义需逐一甄别,不能假设全部是功夫派语义。
- **多版本分支混杂**:源码内含 `taiwan`(台服)、`kaixin`(开心网)条件编译与并行 conf 目录,XML 内容不是单一线上版本,引用数值前需确认分支。
- **`db/ser/src/` 是 Makefile 而非 CMake**:与 baseline 背景里"dbser=cmake 骨架"指的是另一个顶层 `dbser` 模块,本模块 `gf/db` 用普通 Makefile,二者勿混。
- **文档稀薄**:除 20 万字节的二进制 `编码规范.doc` 和两行 changelog 外几乎无设计文档,逻辑需直接读 .cpp 反推。
- **明文口令**:`readme`/建库脚本里有明文 MySQL 口令(`ta0mee` 等),仅作历史考据,切勿当真实凭据使用。
