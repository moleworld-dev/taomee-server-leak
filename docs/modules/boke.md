# 模块 boke · 淘米《卜克》(Buke) 社区/养成页游服务端,基于摩尔庄园2 代码克隆

> 价值:★★  ·  游戏:卜克(Buke,淘米 Game ID=12;复用摩尔庄园2 代码底座)  ·  源码量:约 3.7 万行(cpp/c/h ≈ 36519 行)
> 📖 深度功能解析(每个部分实际在干什么):[deep/boke.md](../deep/boke.md) · 彩蛋见 [easter-eggs.md](../easter-eggs.md)

## 是什么 / 在整套架构里的角色

`boke` 是淘米一款名为 **《卜克》(Buke)** 的网页 Flash 养成/社区游戏的完整服务端集群。强证据:

- 聊天网关配置 [boke/chat/conf/GameSvr.xml](../../boke/chat/conf/GameSvr.xml) 明确列出 `<Game ID="12" Name="卜克" .../>`(与之并列的是 `ID="1" Name="摩尔"`),说明"boke=卜克"是淘米内部正式游戏代号。
- 游戏内货币叫 **"卜克豆"**、有 **"卜克岛""卜克市市长"** 等物品,见 [boke/online/conf/ItemXmlData.xml](../../boke/online/conf/ItemXmlData.xml#L145)(`卜克岛金牌`)、[boke/online/gen_proto/proto/pop_online.h](../../boke/online/gen_proto/proto/pop_online.h#L194)(注释 `卜克豆`)。
- 项目名硬编码为 `boke`:[boke/online/conf/bench.conf](../../boke/online/conf/bench.conf#L50) `project_name boke`;DB 服里甚至有 `cd ~/boke/online/bin&&./startup.sh` 的重启脚本调用,见 [boke/db/src/Croute_func.cpp](../../boke/db/src/Croute_func.cpp#L316)。

**与摩尔庄园的关系(重要):** 《卜克》并非独立从零开发,而是**直接 fork 了摩尔庄园2(mole2)的服务端代码再换皮**。证据:客户端协议头文件的 include guard 写死为 `MOLE2_CLI_PROTO_H`,见 [boke/online/src/cli_proto.h](../../boke/online/src/cli_proto.h#L2);玩法机制(岛屿 island、找地图 find_map、NPC 对话、换装 clothes、小游戏、任务节点)与摩尔庄园完全同构;甚至 NPC 对话配置 [boke/online/conf/Talk_new.xml](../../boke/online/conf/Talk_new.xml) 里出现的"土豆岛/卜尼亚海"就是卜克自己的世界观换皮。因此对摩尔庄园复活而言,`boke` 是**极佳的同源参照样本**(见后)。

**进程拓扑(典型淘米五段式):**

```
Flash 客户端
   │ (验证码/登录/选服)
   ▼
[login]  ── 登录鉴权,下发 online 服列表/备份服列表
   │
   ▼
[switch] ── 接入网关/会话路由,把客户端命令字按 cmd_map 分发到 online
   │
   ▼
[online] ── 游戏世界主逻辑(地图/精灵 Csprite/任务/物品/小游戏/聊天)  ← 全模块代码主体
   │  ▲
   │  │ 脏字校验 (adm_check_msg)
   │  ▼
[chat]  ── 独立聊天审查服(分词+脏词库 uni.lib/tm_dirty.dat),供 mole 与 boke 共用
   │
   ▼
[db]   ── dbproxy / 数据落地服(MySQL 分库分表 POP_00..99),CMake 骨架
```

各服之间用统一的 `svr_proto_t` 报文头(len/seq/cmd/ret/id + body),见 [boke/chat/src/proto.hpp](../../boke/chat/src/proto.hpp#L33);命令字 cmd 约定 `> 0xEA60(60000)` 用于服务器间通信。

## 关键子目录与文件

| 路径 | 作用 |
|---|---|
| [boke/online/](../../boke/online/) | 游戏世界主进程(192 文件,2.1MB),整个模块的代码主体 |
| [boke/online/src/cli_proto.h](../../boke/online/src/cli_proto.h) | 客户端协议入口,include guard `MOLE2_CLI_PROTO_H` 暴露 mole2 血统 |
| [boke/online/src/proto/pop_online_enum.h](../../boke/online/src/proto/pop_online_enum.h) | 全部客户端命令字 cmdid 枚举(登录/任务/移动/聊天/换装/小游戏/抽奖) |
| [boke/online/src/Cgame.cpp](../../boke/online/src/Cgame.cpp) | 小游戏对战逻辑(请求/开始/操作/结算),作者 jim,2011-08 |
| [boke/online/src/Csprite.cpp](../../boke/online/src/Csprite.cpp) | "精灵/玩家化身"对象,玩家在场景中的状态机(最大单文件之一) |
| [boke/online/src/load_data.cpp](../../boke/online/src/load_data.cpp) | 启动时加载 conf 下各 XML 配置表 |
| [boke/online/conf/](../../boke/online/conf/) | 全部游戏内容配置(物品/NPC/任务/对话/地图/商店) |
| [boke/online/conf/ItemXmlData.xml](../../boke/online/conf/ItemXmlData.xml) | 物品/服装总表(173KB),含"卜克岛金牌""卜克市市长"等 |
| [boke/online/conf/Talk_new.xml](../../boke/online/conf/Talk_new.xml) | NPC 多轮对话脚本(138KB),含条件分支 condition/face |
| [boke/online/conf/TaskData.xml](../../boke/online/conf/TaskData.xml) | 任务/任务节点定义 |
| [boke/chat/](../../boke/chat/) | 独立聊天审查服(脏词/分词/刷屏检测),编译为 `libchatcheck.so` |
| [boke/chat/src/dispatcher.cpp](../../boke/chat/src/dispatcher.cpp#L269) | 聊天命令分发(`cmd - 60001` 查函数表),分词 segment() |
| [boke/chat/src/CChatCheck.cpp](../../boke/chat/src/CChatCheck.cpp) | 刷屏/重复消息计数与封禁元数据(作者 Ian,2011-10) |
| [boke/chat/data/uni.lib](../../boke/chat/data/) | mmseg 分词词库 + tm_dirty*.dat 脏词二进制库 |
| [boke/login/src/login.cpp](../../boke/login/src/login.cpp) | "通用登录框架,不许更改"(注释原话),细节在 login_impl |
| [boke/login/src/login_impl.cpp](../../boke/login/src/login_impl.cpp) | 卜克登录实现:验证码、选服、备份服列表下发 |
| [boke/switch/src/dispatcher.cpp](../../boke/switch/src/dispatcher.cpp#L61) | 网关命令路由,用 `Ccmd_map` + `pop_switch_bind_for_cli.h` 绑定 |
| [boke/switch/src/online.cpp](../../boke/switch/src/online.cpp) | 维护 online 服在线表/用户路由(domain_id/online_id) |
| [boke/db/src/db.py](../../boke/db/src/db.py) | 生成 MySQL 建表 DDL(POP_00..99 分库,t_user/t_task/t_item/t_find_map 各分 10 表) |
| [boke/db/src/Cuser.cpp](../../boke/db/src/Cuser.cpp#L26) | 用户档案读写(register_time/nick/last_islandid/last_mapid/last_x/last_y…) |
| [boke/db/src/Croute_func.cpp](../../boke/db/src/Croute_func.cpp) | DB 路由分发 + 运维后门(opt_groupid=997/998 直接 system() 重启拉表) |

## 协议 / 数据亮点

**客户端命令字(cmdid)** 集中在 [boke/online/src/proto/pop_online_enum.h](../../boke/online/src/proto/pop_online_enum.h),与摩尔庄园高度同构。真实例子:

- `cli_walk_cmd = 1014` 进入场景,见 [pop_online_enum.h#L20](../../boke/online/src/proto/pop_online_enum.h#L20)
- `cli_talk_npc_cmd = 1022` 和 NPC 聊天,见 [pop_online_enum.h#L28](../../boke/online/src/proto/pop_online_enum.h#L28)
- `cli_user_move_cmd = 1102` / `cli_noti_user_move_cmd = 1103` 多人移动与广播,见 [pop_online_enum.h#L44](../../boke/online/src/proto/pop_online_enum.h#L44)
- 小游戏一整套:`cli_game_request_cmd=1201 … cli_noti_game_end_cmd=1210`,见 [pop_online_enum.h#L54](../../boke/online/src/proto/pop_online_enum.h#L54)

`pop_online.h`/`pop_online.cpp` 是自动生成的协议 codec(同名文件在 `src/proto/` 与 `gen_proto/proto/` 各一份,还附带 php/python/xml 多语言生成产物 [boke/online/gen_proto/](../../boke/online/gen_proto/))。完整命令字总览参见 [../protocol.md](../protocol.md)。

**服务器间命令字**(chat 服):[boke/chat/src/proto.hpp#L19](../../boke/chat/src/proto.hpp#L19)
- `adm_check_msg = 0xF133` (61747) 聊天审查
- `adm_get_dirty_words = 65002` / `adm_check_msg_common = 65003`

**内容配置 XML**(挖宝重点,见 [../content-data.md](../content-data.md)):
- 物品/服装:[ItemXmlData.xml](../../boke/online/conf/ItemXmlData.xml)(`item id/name/clothtype/specialkind`)
- NPC 对话:[Talk_new.xml](../../boke/online/conf/Talk_new.xml)(多达 12 段 ask/answer + condition 分支)
- 任务:[TaskData.xml](../../boke/online/conf/TaskData.xml);NPC:[Npc_new.xml](../../boke/online/conf/Npc_new.xml);事件:[Eventdata.xml](../../boke/online/conf/Eventdata.xml);地图名:[Map_name.xml](../../boke/online/conf/Map_name.xml);商店:[shop.xml](../../boke/online/conf/shop.xml) / [jf_shop.xml](../../boke/online/conf/jf_shop.xml);小游戏积分规则:[gamesid.xml](../../boke/online/conf/gamesid.xml)。

**数据库表结构**(分库分表)在 [boke/db/src/db.py](../../boke/db/src/db.py):库 `POP_00`..`POP_99`,每库内 `t_user_0..9`、`t_task_*`、`t_item_*`、`t_find_map_*`。核心 `t_user` 字段:`userid/flag/register_time/sex/age/nick/last_islandid/last_mapid/last_x/last_y/last_login`——与摩尔庄园存档语义几乎一一对应。

## 对我们有什么用(moleworld-server / MoleWorld-offline)

**定位:高价值"参照"为主,少量"直用"。** 卜克是摩尔庄园2 的同源换皮,所以它能当摩尔庄园服务端逻辑的**第二份对照实现**——当 `mole`/`mole2` 模块某处代码缺失或晦涩时,可来 boke 交叉印证同一机制(精灵状态、岛屿/找图、任务节点、小游戏对战、登录选服流程)。

- **参照(强):** [pop_online_enum.h](../../boke/online/src/proto/pop_online_enum.h) 的命令字语义 = 摩尔庄园客户端↔服务端协议的活字典。复活摩尔庄园离线服时,可据此理解 walk/move/talk_npc/换装/小游戏 的请求-广播配对(`cli_xxx` 对 `cli_noti_xxx`)。
- **参照(强):** [db/src/db.py](../../boke/db/src/db.py) + [Cuser.cpp](../../boke/db/src/Cuser.cpp) 的存档表结构,可直接作为 MoleWorld-offline 设计本地存档(SQLite/JSON)的字段蓝本——last_islandid/last_mapid/last_x/last_y 就是"上次登出位置"。
- **参照(中):** [login_impl.cpp](../../boke/login/src/login_impl.cpp) 揭示了"验证码→登录→下发 online/备份服列表"的握手时序,对实现离线版可跳过的登录桩有帮助。
- **直用候选:** [chat/data/](../../boke/chat/data/) 的脏词库(tm_dirty*.dat)与 mmseg 词库 uni.lib 是通用资源;若离线版需要聊天过滤,词库数据可直接拿来用(注意是二进制,需配合 chat 服的加载逻辑解析)。
- **直用候选:** conf 下的 NPC 对话/任务文本本身是淘米文案产物,但它们是**卜克世界观**(土豆岛、卜尼亚海),与摩尔庄园不同,**不可直接当摩尔内容**用;只能作为"淘米同类游戏配置表的格式样本"。摩尔庄园自己的内容配置应去 `mole`/`mole2` 模块取。

## 注意 / 坑

- **不是博客/微博。** 名字"boke"易误读为"博客";实为淘米游戏《卜克》拼音,已由 [GameSvr.xml](../../boke/chat/conf/GameSvr.xml)(`Name="卜克"`)与"卜克豆/卜克岛"内容实证。
- **它是摩尔庄园2 的换皮 fork,不是原创。** 大量"摩尔"机制残留(`MOLE2_CLI_PROTO_H` 等),分析时要意识到代码是为摩尔写的、内容才是卜克的;混用内容会出戏。
- **db 模块是 CMake 半成品骨架。** 仅有 [boke/db/src/CMakeLists.txt](../../boke/db/src/CMakeLists.txt) 与 service 框架,无完整 .sql 文件(建表靠 [db.py](../../boke/db/src/db.py) 现场生成 DDL),要跑通需自行补 MySQL 实例与分库。
- **重度依赖淘米内部框架,仓库内不含。** 全模块 include 的 `libtaomee`/`libtaomee++`/`async_serv`(参见 [c.md](./c.md)、`libtaomee` 模块两个 .tar.gz)未在 boke 目录内提供,直接编译会缺头/缺库。
- **运维后门要注意。** [Croute_func.cpp](../../boke/db/src/Croute_func.cpp#L316) 里 `opt_groupid==997/998` 会用 `system()` 拼字符串执行 shell(重启 online、拉取 XML),是典型的"内网信任"年代写法(2007 起);任何复现都不要把这条路径暴露到公网。
- **年代局限。** 作者头注释显示核心成型于 2007(db proto.h,作者 xcwen)、大改于 2011(chat/online,作者 Ian/jim);gcc 4.1.2 + Linux,32/64 位假设、`__attribute__((packed))` 裸结构体直传,跨平台/现代编译器需适配。
- **proto 双份冗余。** `src/proto/` 与 `gen_proto/proto/` 下 `pop_online.*` 内容重复,改协议时注意以生成器 [gen_proto/](../../boke/online/gen_proto/) 的 xml 为单一真源。

---
返回总入口:[../../README.md](../../README.md) · 协议专题:[../protocol.md](../protocol.md) · 内容数据专题:[../content-data.md](../content-data.md)
