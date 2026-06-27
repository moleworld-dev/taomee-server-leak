# 模块 mole2 · 摩尔庄园2 / 跨服(2 代)完整服务端集群

> 价值:★★★  ·  游戏:摩尔庄园(摩尔庄园 2 / 跨服 PK)  ·  源码量:约 12 万行(122,090 行 cpp/c/h)

## 是什么 / 在整套架构里的角色

`mole2/` 是上海淘米(TaoMee, Inc. ShangHai CN)摩尔庄园 **第二代(摩尔庄园 2)** 的完整后端集群源码。版权头一律为 `copyright: TaoMee, Inc. ShangHai CN. All rights reserved.`,编译环境 `gcc4.1.2 / Linux`。最老的协议头 [mole2_db/src/proto.h](../../mole2/mole2_db/src/proto.h#L9) 标注 `Created: 2007年11月01日`,作者 `xcwen (xcwenn@gmail.com)`;Switch 的 changelog 记录到 2009-04-27,内容配置 XML(任务/服饰/精灵)做到 ~2012-2013。

相比一代 [mole](../../mole),mole2 是一次**架构重写 + 玩法升级**:引入了独立的**回合制宠物对战服**(batrserv,内嵌 Lua 5.1 跑怪物 AI)、**跨服 PK 集群**(mole2_cross + mole2_switch),登录服也重写为 `mole2_login_new`。它是本仓库摩尔庄园主线里规模最大、最成体系的一套(★★★ 实至名归)。

整套是经典的淘米**分进程 + DBProxy + Switch 路由**架构,7 个进程角色:

| 子目录 | 进程 | 角色 | 关键证据 |
|--------|------|------|----------|
| `mole2_login_new` | M2Login | 登录服:账号校验/封禁/选服,组播下发服务器列表 | [readme](../../mole2/mole2_login_new/readme) `passwd_fail_time_limited / ban_time / addr_mcast_ip` |
| `mole2_switch` | Mole2Switch | 路由总机:把客户端连接分配到某台 online,维护 online_id→ip:port | [switch.cpp](../../mole2/mole2_switch/switch.cpp#L1)、[readme.txt](../../mole2/batrserv/doc/readme.txt) `server_num 500 / user_num 14950000` |
| `mole2_online` | m2_online | **游戏主逻辑服**:地图/服饰/背包/任务/SNS/宠物/战斗入口,最庞大 | [mole2_online.cpp](../../mole2/mole2_online/mole2_online.cpp#L1) |
| `mole2_home` | M2home | 家园服(庄园建造/摆设/市场/交易/PVP/GVG) | [mole2_home.cpp](../../mole2/mole2_home/mole2_home.cpp)、[proto.h](../../mole2/mole2_home/proto.h#L24) |
| `batrserv` | (Switch 进程名复用) | **回合制对战服**:发招/换宠/抓宠/药瓶,Lua 驱动怪物 AI | [bt_online.cpp](../../mole2/batrserv/src/bt_online.cpp#L1)、[proto.h](../../mole2/batrserv/src/proto.h#L18) |
| `mole2_cross` | Mole2Cross | **跨服 PK 调度**:报名/匹配/自动接受/自动战斗(job dispatcher 模式) | [cross.cpp](../../mole2/mole2_cross/cross.cpp#L1)、[proto.hpp](../../mole2/mole2_cross/proto.hpp#L38) |
| `mole2_db` | pubser | DB 服(DBProxy):所有持久化命令字的落库,MySQL 后端 | [service.cpp](../../mole2/mole2_db/src/service.cpp#L1)、[readme](../../mole2/mole2_db/readme) `DB_USER root / DB_PASSWD taomee` |

进程关系(实证自 bind.conf / readme):客户端 → **Login**(选服,组播 `addr_mcast_ip 239.0.0.x`)→ **Switch**(分配到一台 online,`online_id` 见 [online/bind.conf](../../mole2/mole2_online/bind.conf))→ **online** 处理游戏逻辑,需要持久化时走 **dbproxy → mole2_db**(`dbproxy_port 21001`);进对战时 online 把战斗委派给 **batrserv**;玩家发起跨服 PK 时由 **cross** 调度,**switch** 在多台 online 间转发战斗包。

## 关键子目录与文件(每条 = 可跳转超链接 + 一句话作用)

### mole2_online(游戏主逻辑,~4.4MB 最大)
- [mole2_online/mole2_online.cpp](../../mole2/mole2_online/mole2_online.cpp#L1) — online 主程序入口,装载各玩法模块、注册命令分发。
- [mole2_online/cli_proto.h](../../mole2/mole2_online/cli_proto.h#L41) — **客户端↔online 协议命令字枚举**(`proto_cli_login=1001` 起),整套客户端协议的钥匙。
- [mole2_online/battle.cpp](../../mole2/mole2_online/battle.cpp) — 战斗逻辑(158KB),与 batrserv 协作的 online 侧入口。
- [mole2_online/items.cpp](../../mole2/mole2_online/items.cpp) / [items.h](../../mole2/mole2_online/items.h) — 道具/背包系统(196KB items.xml 配套)。
- [mole2_online/sns.cpp](../../mole2/mole2_online/sns.cpp) — 好友/社交/关系链(151KB)。
- [mole2_online/pet.cpp](../../mole2/mole2_online/pet.cpp) — 宠物(精灵)养成/属性/技能。
- [mole2_online/task_new.cpp](../../mole2/mole2_online/task_new.cpp) — 新版任务系统(tasks_new.xml 配套)。
- [mole2_online/map.cpp](../../mole2/mole2_online/map.cpp) / [homemap.cpp](../../mole2/mole2_online/homemap.cpp) — 地图与家园地图。
- [mole2_online/conf/](../../mole2/mole2_online/conf) — **海量内容 XML**:服饰/技能价格/精灵/任务/地图/宝箱等(见下)。

### batrserv(回合制对战服)
- [batrserv/src/bt_online.cpp](../../mole2/batrserv/src/bt_online.cpp#L1) — 对战服主逻辑(208KB,全模块最大单文件)。
- [batrserv/src/proto.h](../../mole2/batrserv/src/proto.h#L18) — 对战协议命令字(`proto_bt_challenge_battle=1300` 起)+ `batrserv_proto_t` 包头。
- [batrserv/src/bt_skill.cpp](../../mole2/batrserv/src/bt_skill.cpp) — 技能结算。
- [batrserv/src/beast_ai_api.cpp](../../mole2/batrserv/src/beast_ai_api.cpp#L1) / [bt_beastai.cpp](../../mole2/batrserv/src/bt_beastai.cpp) — **怪物 AI 的 Lua 桥**:把战斗状态暴露给 Lua。
- [batrserv/lua-5.1/](../../mole2/batrserv/lua-5.1) — 内嵌完整 Lua 5.1 解释器源码,跑 AI 脚本。
- [batrserv/conf/BattleActionList.xml](../../mole2/batrserv/conf/BattleActionList.xml) — 战斗动作表(303KB)。

### mole2_db(DB 服)
- [mole2_db/src/proto.h](../../mole2/mole2_db/src/proto.h#L1) — **DB 命令字总表**(2200 行,`mole2_login_cmd=0x0801` 起,带 `NEED_UPDATE` 标志位区分读/写)。
- [mole2_db/src/service.cpp](../../mole2/mole2_db/src/service.cpp#L1) — DB 服主程序(MySQL 接口 + 路由初始化)。
- [mole2_db/src/Cuser.cpp](../../mole2/mole2_db/src/Cuser.cpp) / [Cpet.cpp](../../mole2/mole2_db/src/Cpet.cpp) — 用户、宠物的 ORM 风格落库类。
- [mole2_db/src/Croute_func.cpp](../../mole2/mole2_db/src/Croute_func.cpp) — 命令字→处理函数的路由分发(142KB)。
- [mole2_db/src/field_desc.txt](../../mole2/mole2_db/src/field_desc.txt) — 字段描述(替代 SQL 建表脚本,见"坑")。
- [mole2_db/readme](../../mole2/mole2_db/readme) — 部署说明 + MySQL 连接参数。

### cross / switch / home / login
- [mole2_cross/cross.cpp](../../mole2/mole2_cross/cross.cpp#L1) + [job_factory.cpp](../../mole2/mole2_cross/job_factory.cpp) — 跨服调度,Job 工厂/分发器模式。
- [mole2_cross/proto.hpp](../../mole2/mole2_cross/proto.hpp#L38) — 跨服内部命令字(`CROSS_COMMID_APPLY_PK=1004` 等)。
- [mole2_switch/switch.cpp](../../mole2/mole2_switch/switch.cpp#L1) + [online.cpp](../../mole2/mole2_switch/online.cpp) — 总机路由与 online 在线表维护。
- [mole2_home/mole2_home.cpp](../../mole2/mole2_home/mole2_home.cpp) + [gvg.cpp](../../mole2/mole2_home/gvg.cpp) / [pvp.cpp](../../mole2/mole2_home/pvp.cpp) — 家园 + 公会战/PVP。
- [mole2_login_new/login.cpp](../../mole2/mole2_login_new/login.cpp) + [login_impl.cpp](../../mole2/mole2_login_new/login_impl.cpp) — 登录服实现。

## 协议 / 数据亮点

### 命令字(实证带行号)
- **客户端协议**(明文易读枚举):[cli_proto.h](../../mole2/mole2_online/cli_proto.h#L41) `proto_cli_login = 1001`、[#L43](../../mole2/mole2_online/cli_proto.h#L43) `proto_cli_map_users = 1002`、[#L82](../../mole2/mole2_online/cli_proto.h#L82) `proto_cli_buy_cloth = 1101`(买衣服)。客户端包头是 `protocol_t { len, cmd, id, seq, ret, body[] }`(见 batrserv/proto.h)。
- **对战协议**:[batrserv/src/proto.h](../../mole2/batrserv/src/proto.h#L18) `proto_bt_challenge_battle = 1300`(发起挑战)、`proto_bt_battle_attack = 1308`(发招)、`proto_bt_catch_pet = 1338`(抓宠)——每条都有中文注释,极易对照。
- **DB 命令字**:[mole2_db/src/proto.h](../../mole2/mole2_db/src/proto.h#L38) `mole2_login_cmd = 0x0801`、[#L40](../../mole2/mole2_db/src/proto.h#L40) `mole2_user_user_create_role_cmd = 0x0802 | NEED_UPDATE`、[#L64](../../mole2/mole2_db/src/proto.h#L64) `mole2_user_pet_add_cmd = 0x080E`。注意 `NEED_UPDATE = 0x0100`([proto.h#L34](../../mole2/mole2_db/src/proto.h#L34) 附近)按位标记"涉及写库需保存交易报文",`rroto_route_field = 0xFE00` 用高 7 位做路由。
- **跨服协议**:[mole2_cross/proto.hpp](../../mole2/mole2_cross/proto.hpp#L38) `CROSS_COMMID_APPLY_PK=1004`、`CROSS_COMMID_AUTO_FIGHT=1007`;系统内部消息从 5500 起(`CROSS_COMMID_ONLINE_TIMEOUT=5501`)。

> 完整命令字矩阵见 [../protocol.md](../protocol.md)。各 online 包额外带 `online_id`,home/batrserv 包头还带 `homeid`/`battle_id_t challenger`,用于多服转发定位。

### 内容配置 XML(`conf/`,这是真正的"宝藏",可直接读懂玩法数值)
- [clothes.xml](../../mole2/mole2_online/conf/clothes.xml)(794KB)——全部服饰/装备,头部注释完整说明了 `EquipPart`(8 属性水晶/5 武器/6 盾/7 挂件/1 帽/2 衣/…)、`Prof`(职业:2 剑士/4 弓箭手/8 法师/16 传教士/32 忍者)、品阶颜色与加成。
- [beasts.xml](../../mole2/mole2_online/conf/beasts.xml)(145KB)——精灵/怪物图鉴:种族(0 人型/1 昆虫/2 植物/4 龙/5 野兽…)、向性(物理/魔法)、地水火风属性、体质基数等完整数值表。
- [skills_price.xml](../../mole2/mole2_online/conf/skills_price.xml)(312KB)、[items.xml](../../mole2/mole2_online/conf/items.xml)(196KB)、[tasks_new.xml](../../mole2/mole2_online/conf/tasks_new.xml)(192KB)、[rand_item.xml](../../mole2/mole2_online/conf/rand_item.xml)、[beastgrp.xml](../../mole2/mole2_online/conf/beastgrp.xml)、[maps.xml](../../mole2/mole2_online/conf/maps.xml)、[box.xml](../../mole2/mole2_online/conf/box.xml)、[handbook.xml](../../mole2/mole2_online/conf/handbook.xml) 等。
- 对战侧:[batrserv/conf/BattleActionList.xml](../../mole2/batrserv/conf/BattleActionList.xml)、[BattleMonsterTalk.xml](../../mole2/batrserv/conf/BattleMonsterTalk.xml)(怪物台词)。

### 数据表(无 SQL 脚本)
本模块**没有 .sql 建表文件**;表结构以 C++ 的 `Cxxx.cpp/h` ORM 类(Cuser/Cpet/Citem/Chome/Crank…)+ [field_desc.txt](../../mole2/mole2_db/src/field_desc.txt) 形式描述。要还原 schema 需从这些类的字段反推。

## 对我们有什么用(moleworld-server / MoleWorld-offline)

> 重要前提:摩尔世界(moleworld 项目)复活的是**一代摩尔庄园**(对应 [mole](../../mole) 模块,cocos2d/GLES1.1 客户端)。mole2 是**二代**,客户端协议(`proto_cli_*`)与一代不通用,因此整体只能**参照**而非直接对接一代客户端。

**可"直用"(数据/逻辑可整段搬):**
- `conf/*.xml` 是**纯数据**,与客户端无关。若做摩尔庄园 2 的离线服或数值研究,clothes/beasts/skills_price/tasks_new/items 等可直接加载,是现成、完整、带中文字段说明的内容库——这是本模块对内容侧最大的直用价值。
- batrserv 的 **Lua AI 脚本框架**(beast_ai_api 桥 + lua-5.1)是一套自包含、可复用的"配置驱动 AI"范式,做任何回合制怪物 AI 都能直接借鉴甚至移植。

**只能"参照"(设计参考):**
- 七进程 Login/Switch/online/db/home/batrserv/cross 的**分服架构**是淘米成熟做法,可作为 moleworld-server 设计跨服/多实例时的蓝本(尤其 cross 的 Job 调度 + switch 转发的解耦)。
- DB 命令字的 `NEED_UPDATE` / `rroto_route_field` 位标记设计(读写分流 + 路由)是干净的协议设计参考。
- 战斗/宠物/任务的**字段语义和数值口径**(职业、属性、品阶)可用于校准/对照一代的对应系统。

## 注意 / 坑

- **代际不匹配**:mole2 = 摩尔庄园 2,协议/客户端与一代([mole](../../mole))不互通。给一代离线服(moleworld-server)接协议时**不要照搬** `cli_proto.h`,要以一代的 proto 为准,mole2 仅作设计参考。
- **无 SQL 建表脚本**:数据库 schema 只能从 `mole2_db/src/Cuser.cpp` 等 ORM 类 + `field_desc.txt` 反推,落库时需自己写 DDL。
- **纯源码、无二进制/无资产**:仓库不含客户端、美术、Flash swf,也不含可执行体(`M2Login`/`Mole2Switch`/`Mole2Cross`/`M2home` 等只是目录占位/发布脚本目标,非真二进制)。
- **强依赖内部框架**:大量 `#include <libtaomee/...>`、`<libtaomee++/...>`、`<async_serv/...>`、`statistic_agent/stat_agent` 等淘米内部库(见 [libtaomee.md](./libtaomee.md)、[c.md](./c.md) 模块,两个框架 .tar.gz 在仓库根);缺这些无法编译。`batrserv` 还需要 `glib`、内嵌 Lua。
- **配置含真实内网信息**:bench.conf/bind.conf/readme 里有硬编码内网 IP(`10.1.1.x`)、组播地址、`DB_USER root / DB_PASSWD taomee` 等——历史明文凭据,仅供研究,勿当生产参考。
- **跨服 proto.hpp 注释乱码**:[mole2_cross/proto.hpp](../../mole2/mole2_cross/proto.hpp#L50) 中文注释是 GBK 被当 UTF-8 显示成乱码(如"系统内部消息id"),不影响命令字数值。
- **代码新旧混杂**:home 子目录里同名 `.c` 与 `.cpp` 并存(如 [home.c](../../mole2/mole2_home/home.c) 与 [home.cpp](../../mole2/mole2_home/home.cpp)),是 C→C++ 迁移过程残留,以 CMakeLists/.cpp 为准。

---
返回总入口:[../../README.md](../../README.md) · 相关:[mole.md](./mole.md)(一代) · [protocol.md](../protocol.md) · [content-data.md](../content-data.md) · [c.md](./c.md) · [libtaomee.md](./libtaomee.md)
