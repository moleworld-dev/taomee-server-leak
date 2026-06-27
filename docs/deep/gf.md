# gf 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/gf.md](../modules/gf.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)

本文是 [概览文档](../modules/gf.md) 的"深水区"补充。概览回答"是什么",本文回答"**它在运行时实际怎么跑**":进程怎么被网络底座装载、一个登录/战斗请求怎么穿过整套集群、战斗伤害与 AI 到底怎么算、分库分表的具体寻址公式。所有结论都带真实行号,可点击跳转。

游戏为**功夫派(KungFu Pai)**,武侠动作 MMO(非小花仙,见概览的实证)。

---

## 一图看懂:进程/组件拓扑与数据流

功夫派不是单进程,而是一组 **业务逻辑编成 `.so`、由通用异步网络底座(AsynServer)装载** 的进程,各进程之间用裸 TCP/UDP + 自定义二进制协议互联。底座本身不在 gf/ 里(它是 `c/library/async-serv/` + `libtaomee`/`libtaomee++`,见 [概览·注意/坑](../modules/gf.md)),gf/ 各目录只产出一个 `.so`(如 `libkf.so`)和一份 conf。

```
                      Flash 客户端(TCP, 自定义包头 cli_proto_t,大端序)
                                   │  (首包先发 <policy-file-request/> 换跨域策略)
                                   ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  gongfu_login (login/)   主登录,校验 session,下发推荐服列表    │
   └──────────────────────────────────────────────────────────────┘
                                   │ 客户端拿到服列表后直连某台 online
                                   ▼
   ┌──────────────────────────────────────────────────────────────┐
   │  gongfu_online  →  libkf.so   ★在线主逻辑服(全游戏功能汇集)    │
   │  背包/任务/家园/商城/交易/灵兽/称号/聊天转发/好友/邮件入口…       │
   └───┬───────┬────────┬─────────┬──────────┬─────────┬───────────┘
       │TCP    │UDP+TCP │TCP      │TCP       │TCP      │TCP
       ▼       ▼        ▼         ▼          ▼         ▼
   gongfu_   db/ser   battle_   trade_svr  home_svr  chat_svr
   switch   (PP_DB    svr       (Trade)    (HomeSVR) +mail_project
  (在线路由  分库分表) (libbtl.so 安全交易  种植/施肥  跨服聊天/邮件
   跨服转发) ↑↑↑↑↑↑   实时动作战斗)            ↑
       │     按 userid 寻址 GF_xx.t_*       cache_svr
       │                                   (USERINFO_* 账号缓存)
       ▼
   btl_switch / cache_switch(战斗、缓存各自的转发层)
```

**关键事实**:

1. **online 是中枢**。客户端只跟 online 维持长连接;打怪、交易、回家、聊天时,online 把请求**转发给对应专用服**,再把回包翻译成客户端协议下发。证据:[gongfu_online/dll_interface.cpp:410-441](../../gf/gongfu_online/dll_interface.cpp#L410) 的 `proc_pkg_from_serv` 按 fd 区分了 db/switch/battle/trade/home/chat/cache/vip 等十多种上游回包。

2. **每个业务进程的代码组织完全同构**:都有一个 `dll_interface.cpp` 实现 AsynServer 要求的 `extern "C"` 契约(`init_service`/`get_pkg_len`/`proc_pkg_from_client`/`proc_pkg_from_serv`/`proc_events`/`on_client_conn_closed`/`fini_service`),底座 fork 出子进程后回调这些函数。

3. **switch 不碰业务**,只维护"uid ↔ online_id ↔ fd"在线表并做跨服转发/广播(跨服聊天、世界喊话、好友上下线通知)。

---

## 逐部分拆解

### 1. `.so` 怎么被网络底座装载 —— AsynServer DLL 契约

`gongfu_online/Makefile` 把一堆 `.cpp` 编成 **`libkf.so`**(开心网分支编成 `libkfkxcommon` 链接的版本),`SOFLAGS = -shared -fPIC`,`BINS = libkf.so`,见 [Makefile:33-42](../../gf/gongfu_online/Makefile#L33)。底座启动时 `dlopen` 这个 so,按约定符号名取出回调。

`dll_interface.cpp` 就是这套契约的实现体。运行时序:

- **`init_service(isparent)`**([dll_interface.cpp:86](../../gf/gongfu_online/dll_interface.cpp#L86)):底座 fork 后,**只在子进程**(`!isparent`)里跑真正初始化——`srand`、装定时器、`config_init("./conf/common.conf")`、注册所有协议处理表(`init_cli_proto_handles` 等)、连 switch/battle_switch、开 db 的 UDP+TCP socket、然后**一口气 load 几十个 XML**(maps/tasks/skills/buff/compose/summon_monster/title/shop/pet_attribute/…,见 [L150-228](../../gf/gongfu_online/dll_interface.cpp#L150))、初始化 Lua 引擎并载入 `./lua/achieve.lua`([L252-257](../../gf/gongfu_online/dll_interface.cpp#L252))。父进程只打一行版本日志。
- **`get_pkg_len(fd, data, len, isparent)`**([L336](../../gf/gongfu_online/dll_interface.cpp#L336)):底座收到字节流后问"这一帧多长"。**父进程**处理面向客户端的连接:先判是不是 Flash 的 `<policy-file-request/>`,是就直接回跨域策略 XML([L338-357](../../gf/gongfu_online/dll_interface.cpp#L338));否则按 `cli_proto_t` 包头大端解出长度。**子进程**按 fd 区分上游(db/switch/battle)用各自的包头长度校验。
- **`proc_pkg_from_client`**([L401](../../gf/gongfu_online/dll_interface.cpp#L401))→ `dispatch()`:客户端整包到齐后进业务分发。
- **`proc_pkg_from_serv`**([L410](../../gf/gongfu_online/dll_interface.cpp#L410)):上游服回包,按 fd 路由到对应 handler。
- **`proc_events`**([L319](../../gf/gongfu_online/dll_interface.cpp#L319)):底座每轮事件循环回调一次,跑 `proc_cached_pkgs()`(下文的玩家级串行队列)、`handle_timer()`、自动排行榜、智力问答房间、活动广播。
- **`on_client_conn_closed` / `on_fd_closed`**:断线清理。前者按 fd 找到 player 并 `del_player`([L447](../../gf/gongfu_online/dll_interface.cpp#L447));后者处理上游服 crash(把对应 fd 置 -1 并触发 `do_while_battle_svr_crashed` 等容灾)。
- **`reload_global_data` / `before_reload`**([L543](../../gf/gongfu_online/dll_interface.cpp#L543)、[L564](../../gf/gongfu_online/dll_interface.cpp#L564)):热重载——重注册协议表 + 重载 Lua,**不重启进程**即可换逻辑/数值。
- **`proc_mcast_pkg`**([L510](../../gf/gongfu_online/dll_interface.cpp#L510)):监听组播,用于全集群同步——reload 配置、跨服交易广播、世界公告、限购数据同步、成就广播。`if (pkg->server_id != get_server_id())` 保证不处理自己发的包。

battle_svr 的 `dll_interface.cpp` 是同一套契约([battle_svr/dll_interface.cpp:58](../../gf/battle_svr/dll_interface.cpp#L58)),区别在 init 时 new 出 `GobalSkill`、`GfItemManager`,载入 7 套难度的 `stage_*/monsters.xml`、`effect/buff/aura/skill_summon` 等战斗专用 XML,并 `init_effect_func_table()` 建技能效果函数表。

### 2. `dispatch()` —— 客户端协议分发 + 三道闸门

[cli_proto.cpp:360](../../gf/gongfu_online/cli_proto.cpp#L360) 是所有客户端包的总入口,实际干四件事:

1. **大端解包头**(`bswap`,淘米协议走网络序),先处理探活包 `cli_proto_probe`(直接回一个日期串)。
2. **三重合法性校验**([L381-393](../../gf/gongfu_online/cli_proto.cpp#L381)):cmd 必须落在 `[cli_proto_cmd_start, cli_proto_cmd_end)` 且 handler 已注册;非 login 包必须已有 player、login 包必须**没有** player、且包头 uid 要与 player->id 一致。
3. **序列号防重放**([L399](../../gf/gongfu_online/cli_proto.cpp#L399) `is_invaild_seqno`):每个玩家维护 `real_seqno`,乱序/重放包直接丢。
4. **玩家级串行闸门 `waitcmd`**(见下节)、**禁游客闸门** `no_guest`([L457](../../gf/gongfu_online/cli_proto.cpp#L457))、**包体长度闸门**(`cmp_must_eq`/`cmp_must_ge` 对照注册表里的 len,[L464](../../gf/gongfu_online/cli_proto.cpp#L464))、**在线时长闸门** `is_time_limit`(防沉迷,[L471](../../gf/gongfu_online/cli_proto.cpp#L471))。

协议表用宏 `SET_CLI_HANDLE(op, bodylen, 比较方式, 禁游客, 限时)` 在 `init_cli_proto_handles()` 里一条条登记([L489 起](../../gf/gongfu_online/cli_proto.cpp#L489)),例如 `SET_CLI_HANDLE(login, 32, cmp_must_eq, 0, 0)`、`SET_CLI_HANDLE(walk, 12, cmp_must_eq, 0, 0)`。这是淘米全系通用范式(mole 同款)。

### 3. 玩家级串行队列 `waitcmd` —— 异步框架里怎么保证单玩家请求不乱序

这是整套架构最关键也最容易被忽略的机制。online 是**单线程异步**的,但很多请求要"先问 db/battle 服,拿到回包再继续"。为避免同一玩家在等回包期间又发新请求把状态搅乱,引入 `waitcmd`:

- 处理一个请求前,`dispatch` 把 `p->waitcmd = pkg_head.cmd`([cli_proto.cpp:477](../../gf/gongfu_online/cli_proto.cpp#L477))。
- 如果此时玩家 `waitcmd != 0`(上一个请求还没收尾),新包**不执行,塞进 `p->pkg_queue`**(每玩家上限 100,超了回 `system_busy`,[L409-418](../../gf/gongfu_online/cli_proto.cpp#L409))。
- 当上游回包到、逻辑收尾,代码会把 `p->waitcmd = 0`(如战斗结束 [player.cpp:1561](../../gf/gongfu_online/player.cpp#L1561))。
- 底座每轮调 `proc_events` → `proc_cached_pkgs()`([cli_proto.cpp:900](../../gf/gongfu_online/cli_proto.cpp#L900)):遍历 `awaiting_playerlist`,把 `waitcmd==0` 的玩家队列里缓存的包一个个重新 `dispatch(... , false)`(第三参 `first_tm=false` 跳过序列号校验)。

**效果**:对单个玩家,请求严格 FIFO 串行;对全服,各玩家互不阻塞。这就是淘米用单线程扛住大量在线的核心套路。

### 4. login —— 不支持游客,session 由 db 校验

[login.cpp:54](../../gf/gongfu_online/login.cpp#L54) `login_cmd`:

1. 解出客户端版本号,**RELEASE 版强制版本号匹配**(`online_version`),不匹配直接断([L62-68](../../gf/gongfu_online/login.cpp#L62))。
2. 区分淘米/小巴/台服/开心网客户端,串味直接拒([L81-99](../../gf/gongfu_online/login.cpp#L81)),开心网分支额外读 `is_adult`(防沉迷)。
3. `add_player(p)` 建正式玩家对象,拷贝 `login_session_t`。
4. `db_check_session(player, 6, ...)`(gameid=6 是功夫派)校验登录态;本地校验兜底 `verify_login_session`([L103-109](../../gf/gongfu_online/login.cpp#L103))。
5. 通过后 `db_get_player(player, id)` 去 db 拉全量存档,**进入 waitcmd 等待**,db 回包后才真正上线。

注释明写 "Don't support guest login"([login.cpp:51](../../gf/gongfu_online/login.cpp#L51))。多端登录会先踢旧连接再放行([cli_proto.cpp:434-442](../../gf/gongfu_online/cli_proto.cpp#L434))。

### 5. battle_svr —— 实时动作战斗服(不是回合制!)

读源码可证功夫派战斗是**实时动作 + 位置碰撞**,不是回合制:

- 一切战斗实体继承 `MovingObject`(有 `velocity`,单位 **像素/秒**,[moving_object.hpp:35](../../gf/battle_svr/moving_object.hpp#L35));`Player`(玩家/怪物/宠物共用)、`Skill`(技能投射物)、`SkillEffects`、`BufEffect` 都是 MovingObject 子类([skill.hpp:356/478](../../gf/battle_svr/skill.hpp#L356))。
- `proc_events` 每轮 `update_all_objects()` 推进所有对象位置、跑 AI、结算碰撞([battle_svr/dll_interface.cpp:201-209](../../gf/battle_svr/dll_interface.cpp#L201))。
- 技能命中靠几何判定:`SkillEffects::CanHurtPlayer` 用**扇形**(`check_sector`,半径+角度+朝向)或**碰撞体**(`collision`)判定([skill.cpp:345](../../gf/battle_svr/skill.cpp#L345)),`affect()` 遍历地图上所有怪物/打碎物,对命中目标调 `on_hit`([skill.cpp:382](../../gf/battle_svr/skill.cpp#L382))。
- online 只负责"开局/收尾",真正打斗在 battle_svr。online `start_battle` 把玩家设 `invisible` 并发离开地图([battle.cpp:1548](../../gf/gongfu_online/battle.cpp#L1548));建队/选关时 online 打包 `btl_create_btl_team` + 队伍信息 + 关卡/难度发给 battle_svr([battle.cpp:914-920](../../gf/gongfu_online/battle.cpp#L914))。

### 6. AI 体系 —— 状态机 + 工厂 + 一怪一类

AI 是 **6 态状态机 + 工厂分发** 两层:

- **状态机**:`AIInterface::UpdateAI(millisec)`([AI_interface.cpp:142](../../gf/battle_svr/ai/AI_interface.cpp#L142))按 `cur_ai_state_` 调虚函数 `wait/linger/move/evade/attack/stuck/defense`。被定身(`faint_effect_type`)强制回 WAIT。AI 有仇恨值 `hatred_`、目标 `target`、寻路 `target_path`、跑/走速率 `speed_radio_`(跑=1.6 倍,[L245](../../gf/battle_svr/ai/AI_interface.cpp#L245))。
- **工厂分发** `AIFactory::select_ai(ai_type, mon_id)`([MonsterAI.cpp:1075](../../gf/battle_svr/ai/MonsterAI.cpp#L1075)):先按大类(`LOW/MIDDLE/HIGH_LV_AI`、`LONG_RANGE_ATK_AI`、`SUMMON_MON_AI`、`GUARD_AI`、`BOSS_AI`),BOSS 再按 **怪物 ID 精确路由到专属 AI 类**——`13035→PumpkinBossAI`、`13005→ChifengAI`、`13148→IceDragon_AI`…([L1082-1271](../../gf/battle_svr/ai/MonsterAI.cpp#L1082)),足足几十个 boss 各有手写行为类,落不到任何 case 才退回通用 `BossAI`([L1272](../../gf/battle_svr/ai/MonsterAI.cpp#L1272))。召唤兽也按 `mon_id` 分 5 种性格 AI(狂暴/木/速/盾…,[L1292](../../gf/battle_svr/ai/MonsterAI.cpp#L1292))。
- 还预留了 **LuaAI**(`LuaAI.cpp`/select_lua_ai),`AI_interface.cpp` 里那段按 mon 的 lua 函数名切 lua AI 的代码被注释掉了([L178-185](../../gf/battle_svr/ai/AI_interface.cpp#L178)),说明上线版本最终走 C++ 手写 AI,Lua AI 是半成品。

### 7. db/ser —— PP_DB 落库 + 100×10 分库分表

`db/ser` 是负责 `PP_DB` 的二进制服务(普通 Makefile,非 cmake;[readme](../../gf/db/ser/readme))。每张业务表一个 `Cgf_xxx` 类(`Cgf_role`/`Cgf_item`/`Cgf_attire`/`Cgf_friend`/`Cgf_mail`/`Cgf_summon`…几十个),命令路由集中在 [Croute_func.cpp](../../gf/db/ser/src/Croute_func.cpp#L514)(`gf_get_user_info`、`gf_get_info_for_login`、`gf_wear_clothes`…一个 cmd 一个方法)。

协议命令字带语义位掩码:`NEED_UPDATE=0x0100`(要写库就要存交易报文)、`NEED_CACHED_DEAL=0x0200`([cache_svr/pkg/proto.h:29](../../gf/cache_svr/pkg/proto.h#L29))。例如 `gf_get_user_info_cmd=0x0601` 只读、`gf_synchro_user_gfcoin_cmd=0x0618|NEED_UPDATE` 写"功夫通宝"([db/ser/src/proto.h:34/67](../../gf/db/ser/src/proto.h#L34))。

### 8. switch / cache_svr / 其它专用服

- **gongfu_switch**:维护全集群在线表 `s_online_info_[]`、`s_fd_map_`、uid→online 映射([gongfu_switch/online.cpp:71-77](../../gf/gongfu_switch/online.cpp#L71)),用 `dispatcher.cpp` 的函数表(命令字 60001 起)处理跨服转发——跨服聊天、世界喊话、好友上下线、踢人、VIP/货币同步([dispatcher.cpp:23-50](../../gf/gongfu_switch/dispatcher.cpp#L23))。
- **cache_svr**:用户态账号缓存,`USERINFO_*` 命令字(登录态/注册/好友/黑名单/VIP,与 mole **同构**,[cache_svr/pkg/proto.h:33 起](../../gf/cache_svr/pkg/proto.h#L33))。
- **home_svr**:家园种植,init 时 new `GobalPlants("./conf/plants.xml")` + `FerilizerMrg("./conf/ferilizer.xml")`([home_svr/dll_interface.cpp:54](../../gf/home_svr/dll_interface.cpp#L54)),协议有 `sow_plant_seed`/`set_plant_status`/`get_plants_list`([home_svr/cli_proto.cpp:262](../../gf/home_svr/cli_proto.cpp#L262))。
- **trade_svr**:玩家间**安全交易(担保交易)**,`get_safe_trade_mgr()->init()`([trade_svr/dll_interface.cpp:66](../../gf/trade_svr/dll_interface.cpp#L66)),交易物品的删/加走 db 的 `dbproto_safe_trade_item`([trade_svr/dbproxy.cpp:113](../../gf/trade_svr/dbproxy.cpp#L113))保证一致性。
- **chat_svr** / **mail_project**:跨服聊天分组、邮件(自带 `mail_dbserver`/`mail_dbproxy`/`mail_server` 三件套)。
- **deluser_svr**:删号清理。

---

## 关键机制/算法实战

### 实战 A:伤害结算公式(完整链路)

这是最有料的一段。攻击命中后 `Player::on_hit`([player.cpp:1595](../../gf/battle_svr/player.cpp#L1595))调 `calc_damage`([player.cpp:1344](../../gf/battle_svr/player.cpp#L1344)),公式按顺序叠加:

1. **基础伤害**([L1354-1364](../../gf/battle_svr/player.cpp#L1354)):普攻 = `技能系数0 × (攻击+攻击buff)/100 + 技能系数1 × random(武器攻下限,武器攻上限)/100`;非普攻额外加技能属性伤 `attr_dmg` 和套装 `skill_atk`。
2. **元素伤** 5 系叠加 + **附加伤** + **额外伤**([L1366-1374](../../gf/battle_svr/player.cpp#L1366))。
3. **防御减伤** `calc_defthreshold_damage`([player.cpp:1303](../../gf/battle_svr/player.cpp#L1303)):怪打玩家时按"防御阈值 `def_threshold_value`"**分段衰减**——低于阈值满伤,超阈值后按 `damage*0.9 - (超出量)*damage/500`、`*0.7`、`*0.5`、`*0.35*(1-…)`、`*0.2` 五档递减,最后再乘 `def_rate`(下限 0.2)。无阈值时用平滑公式 `damage*(1 - sqrt(def/(1.2*lv))*0.09)`([L1310](../../gf/battle_svr/player.cpp#L1310))。
4. **等级压制**([L1386-1395](../../gf/battle_svr/player.cpp#L1386)):每差 1 级 ±5% 伤害,封顶 ±50%(差 ≥10 级)。
5. **暴击** `is_crit`([player.hpp:2032](../../gf/battle_svr/player.hpp#L2032)):`rand()%100 < crit_rate+暴击buff`,玩家/召唤兽**暴击率硬封顶 70%**;暴击伤害 = `dmgs × (1.20 + (暴伤buff+100)/100)`([player.cpp:1400](../../gf/battle_svr/player.cpp#L1400))。
6. **±5% 随机抖动**([L1404-1417](../../gf/battle_svr/player.cpp#L1404))、按血量百分比伤(`per_damage`)、伤害比例 `damage_ration`([L1421-1429](../../gf/battle_svr/player.cpp#L1421))。

回到 `on_hit`,伤害定下后还有一串 buff 后处理:**吸血**(按比例给攻击者回血,[L1633-1639](../../gf/battle_svr/player.cpp#L1633))、**几率追伤**(类暴击,[L1642-1649](../../gf/battle_svr/player.cpp#L1642))、**减伤抗性**([L1656-1662](../../gf/battle_svr/player.cpp#L1656)),最后 `chg_hp(-dmgs)` 扣血并 `send_to_btl` 把命中信息广播给同战斗的客户端([L1667-1676](../../gf/battle_svr/player.cpp#L1667)),还有**反弹伤害**(把伤害打回攻击者,[L1702-1720](../../gf/battle_svr/player.cpp#L1702))。注意命中本身先经 `atker->is_hit()`(命中/闪避判定),不中只发个空命中包([L1606-1616](../../gf/battle_svr/player.cpp#L1606))。

### 实战 B:100×10 分库分表寻址公式

gf 的玩家数据散在 **100 个库 × 每库 10 张表**。SQL 生成器 `t_gongfu.py` 循环 `range(100)` 建 `GF_00`..`GF_99`,每库再建 `t_gongfu_0`..`t_gongfu_9`([db/ser/sql/t_gongfu.py](../../gf/db/ser/sql/t_gongfu.py))。运行时寻址在框架基类 `CtableRoute100x10::get_table_name`([dbser/CtableRoute100x10.cpp](../../dbser/CtableRoute100x10.cpp)):

```c
sprintf(db_name,       "%s_%02d",        db_name_pre, id % 100);            // 库:  GF_(uid%100)
sprintf(db_table_name, "%s_%02d.%s_%01d", db_name_pre, id % 100,
                       table_name_pre, (id % 10000) / 1000);                // 表:  t_gongfu_((uid%10000)/1000)
```

即 **库选 `uid % 100`,表选 `(uid % 10000) / 1000`**。`Cgongfu` 构造时把前缀传进去:`CtableRoute100x10(db, "GF", "t_gongfu", "userid")`([db/ser/src/Cgongfu.cpp:9-12](../../gf/db/ser/src/Cgongfu.cpp#L9)),之后每条 SQL 都 `GEN_SQLSTR(... this->get_table_name(userid) ...)` 自动定位到正确分片(见 `get_user_info` [L40-48](../../gf/db/ser/src/Cgongfu.cpp#L40))。这套基类来自顶层 `dbser` 框架模块,mole2 也在用,是淘米全系的标准分库范式。

### 实战 C:db 读全量信息时的字段拼装与防溢出

`Croute_func::gf_get_user_info`([Croute_func.cpp:514](../../gf/db/ser/src/Croute_func.cpp#L514))演示了一条读请求的真实样子:先 `del_daily_loop` 处理跨天重置,再 `gongfu.get_user_info` 取全局信息、`gf_role.get_role_info` 取角色信息(等级/经验/血蓝/坐标/胜负/魔域积分…),逐字段拷进出包结构。值得注意有**数值兜底**:货币 `xiaomee > 10 亿`时强制夹回 10 亿并回写 db([L583-587](../../gf/db/ser/src/Croute_func.cpp#L583))——防溢出/防作弊的实战痕迹。读包还顺手把全局离线消息 + 角色离线消息拼进同一个 out buffer([L589-603](../../gf/db/ser/src/Croute_func.cpp#L589))。

---

## 端到端调用流举例:一次"打一关副本"的完整链路

追一个真实玩法,从客户端点"开打"到落库:

1. **客户端发起**:玩家在某地图发"创建战斗队/进副本"包 → online 父进程 `get_pkg_len` 校验长度 → 子进程 `proc_pkg_from_client` → `dispatch`([cli_proto.cpp:360](../../gf/gongfu_online/cli_proto.cpp#L360))过三道闸门 → 命中战斗类 handler。
2. **online 准备开局**:校验地图/限时/活动限制([battle.cpp:885-907](../../gf/gongfu_online/battle.cpp#L885)),`enter_room` 建本地 `battle_grp`,然后**打包发给 battle_svr**:`pack_btl_team_info` + `pack_initiate_battle_info`(关卡 stage、难度 difficulty、`btl_mode_pve`)+ `btl_create_btl_team` 包头 → `send_to_battle_svr`([battle.cpp:914-920](../../gf/gongfu_online/battle.cpp#L920))。同时 `start_battle` 把玩家设 invisible、发离开地图([battle.cpp:1548](../../gf/gongfu_online/battle.cpp#L1548))。
3. **battle_svr 实跑战斗**:收到 `btl_create_btl_team`,从 `stage_*/monsters.xml` 按难度刷怪、给每只怪 `AIFactory::select_ai` 配 AI;`proc_events` 每轮 `update_all_objects` 推进位置 + 跑 AI;玩家放技能 → `Skill`(MovingObject)飞行 → `SkillEffects::affect` 扇形/碰撞判定 → `on_hit` → `calc_damage`(实战 A 的公式)扣血,命中结果 `send_to_btl` 广播给同房客户端实时表现。
4. **结算回收**:怪清完/玩家死亡 → battle_svr 算经验/掉落(`treasure_box.xml`/`holidaydrops.xml`)→ 把结果回传 online → online `leave_battle`([battle.cpp:1565](../../gf/gongfu_online/battle.cpp#L1565))给客户端发结算包、把玩家放回地图、清 invisible、处理触发事件。
5. **落库**:经验/物品/货币变化由 online 经 dbproxy(TCP `proxysvr_fd` 或 UDP `dbsvr_udp_fd`,[gongfu_online/dbproxy.cpp:113/223](../../gf/gongfu_online/dbproxy.cpp#L113))发到 db/ser → `Croute_func` 按 cmd 找对应 `Cgf_*` 类 → `get_table_name(userid)` 用 `uid%100` / `(uid%10000)/1000` 定位分片 → 执行 `NEED_UPDATE` 的 SQL 写入 `GF_xx.t_*`。整条链全程靠玩家 `waitcmd` 串行,保证不乱。

---

## 复活实操提示(对 moleworld 复活)

**能直接拿来用/对照的**:

- **AsynServer DLL 契约是通用骨架**:`init_service/get_pkg_len/proc_pkg_from_client/proc_pkg_from_serv/proc_events/on_*_closed` 这套回调签名 mole 与 gf 完全一致,实现 moleworld 自有网络底座时可照此设计;[gf/gongfu_online/dll_interface.cpp](../../gf/gongfu_online/dll_interface.cpp) 是最干净的范本(比 item.cpp 那种 22 万字节巨文件好读)。
- **`waitcmd` 玩家级串行队列**([cli_proto.cpp:409/900](../../gf/gongfu_online/cli_proto.cpp#L900))是单线程异步服扛并发的核心技巧,直接可移植——任何"先问 db 再继续"的请求都该走这套闸门。
- **100×10 分库公式**([dbser/CtableRoute100x10.cpp](../../dbser/CtableRoute100x10.cpp))`uid%100` 选库、`(uid%10000)/1000` 选表,设计离线/在线服存档分片时可直接沿用;若只做单机离线服,知道这条公式才能把泄漏的真实存档正确归位。
- **`NEED_UPDATE | NEED_CACHED_DEAL` 命令字位掩码**是淘米全系协议约定,做 moleworld 协议时沿用可与原始客户端/抓包对得上。
- **跨域策略首包应答**([dll_interface.cpp:338-357](../../gf/gongfu_online/dll_interface.cpp#L338)):任何要接 Flash 客户端的复活服都必须先回这段 `cross-domain-policy`,否则 socket 连不上——这是 mole 复活同样要补的。

**要自己补的(gf 特有、对 mole 复活帮助小)**:

- **战斗服几乎不可复用**:功夫派是实时动作战斗(MovingObject/扇形碰撞/一怪一 AI 类),mole 是休闲社区,战斗模型完全不同;[calc_damage](../../gf/battle_svr/player.cpp#L1344) 的公式只能当"淘米数值设计长什么样"的参考。
- **依赖未泄漏框架**:`libtaomee`/`libtaomee++`/`async_serv`/`libkfcommon`/`liblua.a` 是编译前置(见 [Makefile LIBS](../../gf/gongfu_online/Makefile#L21)),`dbser` 框架虽在仓库里但需自行接 `CtableRoute` 基类的 mysql_interface,直接 `make` 跑不通。
- **db 层与 mole 纠缠**:`bench.conf` 仍写 `INIT_XIAOMEE`(摩尔豆)、表字段 `xiaomee`([t_gongfu.py](../../gf/db/ser/sql/t_gongfu.py)),说明 db 服从 mole 复制改名,字段语义要逐一甄别。
