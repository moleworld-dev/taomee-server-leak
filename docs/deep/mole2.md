# mole2 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/mole2.md](../modules/mole2.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)
>
> 本文是概览文档的「深水区」补充:不再罗列"它是什么",而是逐进程讲清**运行时它实际在干什么、消息怎么流、关键算法怎么跑**。所有 `#L` 行号均来自真实 Read/grep,GBK 注释已用 `iconv` 复核。

---

## 一图看懂:七进程拓扑与数据流

mole2 是经典淘米 **AsyncServer(多 worker fork)+ DBProxy + Switch 路由** 架构。七个进程都实现同一组 `init_service / get_pkg_len / proc_pkg_from_client / proc_pkg_from_serv / on_fd_closed` 框架回调(见 [batrserv.cpp#L50](../../mole2/batrserv/src/batrserv.cpp#L50) 注释"AsyncServer框架要求实现的接口之一")。

```
                 [Flash 客户端 (swf, AS3 socket)]
                        │  ① 先发 <policy-file-request/>,online 回 crossdomain xml
                        │  ② proto_cli_login=1001
   ┌────────────────────┼─────────────────────────────────┐
   ▼                    ▼                                   │
[M2Login]          [Mole2Switch]  ←在线表/负载/跨服聊天/PK报名─┐
登录/封禁/选服      │  ↑ online 每隔几十秒上报 user_num         │
   │ 反查 svrlist ──┘  │ 按负载分配 online_id                  │
   │                   ▼                                      │
   │            [m2_online ×N]  游戏主逻辑(地图/背包/任务/SNS/宠物)
   │              │      │      │            │
   │   waitcmd异步│      │委派战斗│           │家园
   │              ▼      ▼      │            ▼
   │          [pubser]  [batrserv ×K]      [M2home]
   │          DB服/MySQL  回合制对战服        家园/GVG/PVP
   │                      └ 内嵌 Lua 跑怪物AI
   ▼
[Mole2Cross] 跨服 PK 调度(报名/匹配/自动接受/自动战斗)
   ↑ 各 online 把跨服 PK 请求转给 cross,cross 回指令让 online 发起对战
```

**谁连谁(实证自 [online/bench.conf#L39](../../mole2/mole2_online/bench.conf#L39))**:online 是星型枢纽,主动连 `service_dbproxy / service_switch / service_batrserv / service_homeserv / service_cross` 五个服务。`proc_pkg_from_serv`([mole2_online.cpp#L322](../../mole2/mole2_online/mole2_online.cpp#L322))就是按 `fd` 区分回包来自哪个后端:`proxysvr_fd→DB`、`switch_fd→switch`、`magic_fd/spacetime_fd→验证码服`、`mall_fd→商城`、`batrserv_fd[]→对战服`、`home_serv_fd[]→家园服`。

---

## 逐部分拆解

### 1. m2_online — 游戏主逻辑枢纽(最庞大,~4.4MB)

入口 [mole2_online.cpp#L61](../../mole2/mole2_online/mole2_online.cpp#L61) `init_service`,启动时:① 判定测试环境(本机 IP 以 `10.` 开头则 `g_is_test_env=true` 并打开十六进制收发日志,[#L67](../../mole2/mole2_online/mole2_online.cpp#L67));② 注册 6 张命令表(cli/db/home/switch/code/mall);③ 一次性 `load_xmlconf` 加载约 **30 个内容 XML**(items/clothes/beasts/tasks_new/skills_price/maps…,[#L125-L163](../../mole2/mole2_online/mole2_online.cpp#L125));④ `connect_to_switch_timely` 连总机并定时心跳上报。

**客户端协议入口与异步串行化(核心运行行为)**——`dispatch` [cli_proto.cpp#L335](../../mole2/mole2_online/cli_proto.cpp#L335):
- 包头 `protocol_t {len,cmd,id,seq,ret,body}` 全部是 **网络字节序**,进来先 `ntohl/ntohs` 翻转([#L340](../../mole2/mole2_online/cli_proto.cpp#L335))。
- **登录前置校验**:除 `proto_cli_login` 外,fd 上必须已有 sprite(在线玩家对象);login 包则要求 fd 上还没 sprite([#L363](../../mole2/mole2_online/cli_proto.cpp#L363))。
- **一玩家同时只处理一条命令(waitcmd 机制)**:sprite 有个 `waitcmd` 字段。若 `p->waitcmd != 0`(上一条还没处理完,通常是在等 DB 回包),新来的命令不丢弃也不立即处理,而是 `g_slice_alloc` 拷一份压进 `p->pending_cmd` 队列(上限 50,超了回 `system_busy`),并把 sprite 挂到全局 `g_busy_sprite_list`([#L384-L404](../../mole2/mole2_online/cli_proto.cpp#L384))。
- 命中处理函数后 **立刻置 `p->waitcmd = cmd`**([#L468](../../mole2/mole2_online/cli_proto.cpp#L468) / 旧表 [#L500](../../mole2/mole2_online/cli_proto.cpp#L500)),处理函数往往要查/写库,于是异步等 DB。
- DB 回来后(见下文)清 `waitcmd`,`proc_events` 里的 `handle_cmd_busy_sprite`([cli_proto.cpp#L268](../../mole2/mole2_online/cli_proto.cpp#L268))把队列里缓存的下一条命令弹出来继续跑。
- 命令分发是 **双表并存**:新协议走 `g_cli_cmd_map`(libtaomee++ 的 `Ccmd_map`,自动反序列化 `Cmessage` 对象,[#L438](../../mole2/mole2_online/cli_proto.cpp#L438)),老协议走数组 `cli_handles[]` + 手工 `body/bodylen`([#L481](../../mole2/mole2_online/cli_proto.cpp#L481))。这是 C→C++ 协议框架迁移的活化石。

**Flash 专属细节**:`get_pkg_len`([mole2_online.cpp#L252](../../mole2/mole2_online/mole2_online.cpp#L252))会先识别 AS3 的 `<policy-file-request/>`,直接回一段 `allow-access-from domain="*"` 的跨域策略 XML 再继续——这是 Flash socket 必经的握手。

**与 DB 的对接**(svr_proto.cpp):`send_to_db` 把回包路由信息 **塞进 seq 字段**——`pkg->seq = (p->fd << 16) | p->waitcmd`([svr_proto.cpp#L52](../../mole2/mole2_online/svr_proto.cpp#L52))。DB 服原样回传该 seq,online 的 `handle_db_return`([dbproxy.cpp#L108](../../mole2/mole2_online/dbproxy.cpp#L108))再 `waitcmd = seq & 0xFFFF; connfd = seq >> 16`([#L116](../../mole2/mole2_online/dbproxy.cpp#L116))反解出"这是哪条 fd、当时等的哪条命令",据此找回 sprite 并校验 `p->waitcmd == waitcmd` 才认。**一个 uint32 seq 同时充当了请求-响应关联键 + 连接定位 + 命令校验**,无状态 DB 服得以纯转发。

### 2. pubser(mole2_db)— DB 服 / DBProxy

入口 [service.cpp](../../mole2/mole2_db/src/service.cpp#L1) `handle_init`([#L21](../../mole2/mole2_db/src/service.cpp#L21)):`PROC_WORK` 阶段 new 出 `mysql_interface`(连 `DB_IP/DB_USER=root/DB_PASSWD=taomee`,[#L43](../../mole2/mole2_db/src/service.cpp#L43))与 `Croute_func`,并建一个 `Cudp_sender` 用于 **CHANGE_LOG**(数据变更审计 UDP 推送,[#L48](../../mole2/mole2_db/src/service.cpp#L48))。

**命令路由 + 频控 + 事务**——`Croute_func::deal` [Croute_func.h#L159](../../mole2/mole2_db/src/Croute_func.h#L159):
1. 用 `cmd_id` 在 `cmdmaplist` 里查处理函数 `p_pri_stru`。
2. **每分钟频控**:`exec_cmd_limit.add_count()` 超阈值直接拒(`CMD_EXEC_MAX_PER_MINUTE_ERR`,[#L166](../../mole2/mole2_db/src/Croute_func.h#L166))——防刷库。
3. **长度校验** `check_proto_size`([#L175](../../mole2/mole2_db/src/Croute_func.h#L175))。
4. 调具体 ORM 函数,**每条命令一个 MySQL 事务**:函数返回后 `mysql_commit`([#L198](../../mole2/mole2_db/src/Croute_func.h#L198));函数返回非成功且没填 `sendbuf` 则 `STD_ROLLBACK`([#L193](../../mole2/mole2_db/src/Croute_func.h#L193))。
5. 老表查不到则降级到 `deal_new`([#L205](../../mole2/mole2_db/src/Croute_func.h#L205))走新协议框架——同样是新旧双表。

**读写分流**:命令字在 [proto.h#L28-L32](../../mole2/mole2_db/src/proto.h#L28) 用位标记设计——`rroto_route_field=0xFE00`(高 7 bit 路由)、`NEED_UPDATE=0x0100`(置位=写库命令,如 `mole2_user_user_create_role_cmd=0x0802|NEED_UPDATE` [#L40](../../mole2/mole2_db/src/proto.h#L40))。online 据此判断该命令是否会改库、是否需要落变更日志。

### 3. batrserv — 回合制对战服(内嵌 Lua 跑怪物 AI)

入口 [batrserv/src/batrserv.cpp#L50](../../mole2/batrserv/src/batrserv.cpp#L50) `init_service`:`init_battles` 建对战哈希表 → `load_lua_script` 加载 AI 脚本 → 加载 `skills_price.xml / BattleActionList.xml / BattleMonsterTalk.xml`(技能耗蓝、招式属性、怪物台词,[#L81-L83](../../mole2/batrserv/src/batrserv.cpp#L81))。

**online↔batrserv 的"信封"协议**:online **不直接发对战命令字**,而是统一用外层 cmd=`battle_info_msg=61006`([proto.h#L13](../../mole2/batrserv/src/proto.h#L13))把真实对战操作塞进 `batrserv_proto_t.opid`([proto.h#L67](../../mole2/batrserv/src/proto.h#L67))。`dispatch`([bt_dispatcher.cpp#L25](../../mole2/batrserv/src/bt_dispatcher.cpp#L25))只认 `cmd-61001` 这一格(`funcs[battle_info_msg-61001]=battle_info_op`,[bt_dispatcher.cpp#L22](../../mole2/batrserv/src/bt_dispatcher.cpp#L22)),并 **关键地把回程地址记下来**:`online[chead->online_id] = fdsess`([#L39](../../mole2/batrserv/src/bt_dispatcher.cpp#L39))——batrserv 不主动连 online,而是认下"哪个 online_id 走哪条 fd",回包时反查发回去(`send_to_online_error(..., online_id)` [bt_online.cpp#L470](../../mole2/batrserv/src/bt_online.cpp#L470))。

`battle_info_op`([bt_online.cpp#L6177](../../mole2/batrserv/src/bt_online.cpp#L6177))才是真入口:用 `challenger`(uint64 battle_id)在 `battles` 哈希表查/建 `battle_info_t`,再按 `opid` 经 `g_cmd_map` 二级分发到 `bt_battle_attack` 等([#L6227](../../mole2/batrserv/src/bt_online.cpp#L6227))。每个对战实例维护 `warrior_list / speed_list / timer_list / attack_list` 四条链表([#L6198](../../mole2/batrserv/src/bt_online.cpp#L6198))。

**多对战服分片**:online 端 `send_to_batrserv`([svr_proto.cpp#L335](../../mole2/mole2_online/svr_proto.cpp#L335))按 `fd_idx = BATTLE_USER_ID(btid) % MAX_BATRSERV_NUM` 选实例——**按战斗发起者 id 哈希到某台对战服**,同一场战斗永远落同一台,首次用到才 `connect_to_service` 懒连接。

### 4. Mole2Cross — 跨服 PK 调度(Job 工厂模式)

入口 [cross.cpp#L59](../../mole2/mole2_cross/cross.cpp#L59) `init_service`:`CreateJobDispatcher`([job_factory.cpp#L32](../../mole2/mole2_cross/job_factory.cpp#L32))造出唯一 `CJobDispatcher` 单例存到 `g_static_dispatcher`。所有逻辑都挂在这个 dispatcher 上,连超时回调、连接关闭都转成 dispatch 事件(`online_timeout_process→CROSS_COMMID_ONLINE_TIMEOUT` [#L31](../../mole2/mole2_cross/cross.cpp#L31))。

**两层解包**:外层 cmd 只允许 `proto_cross_data=65001` / `proto_test_alive=65002`([dispatcher.cpp#L19](../../mole2/mole2_cross/dispatcher.cpp#L19)),真实命令在 `mole2cross_proto_t.opid`([proto.hpp#L80](../../mole2/mole2_cross/proto.hpp#L80)),范围 `(0,5500)` 才放行([dispatcher.cpp#L34](../../mole2/mole2_cross/dispatcher.cpp#L34))。`init_proto_head` 固定把外层 cmd 写 `proto_cross_data`、把业务命令写进 opid([proto.hpp#L97](../../mole2/mole2_cross/proto.hpp#L97))——和 batrserv 同款"信封"思路。

**opid→处理函数靠宏表**(没有 switch):`BEGIN_JOB_DISPATCHER_MAP` 宏在 [cjob_dispatcher.hpp#L12](../../mole2/mole2_cross/cjob_dispatcher.hpp#L12) 展开成一个大 switch,注册了 `report_online_info / online_keepalive / challenge_battle / onli_apply_pk / onli_auto_accept / onli_auto_fight`([#L14-L20](../../mole2/mole2_cross/cjob_dispatcher.hpp#L14)),宏定义见 [ijob_dispatcher.hpp#L14](../../mole2/mole2_cross/ijob_dispatcher.hpp#L14)。

### 5. Mole2Switch — 总机:在线表 / 负载均衡 / 跨服聊天 / 本地 PK

[switch.cpp#L45](../../mole2/mole2_switch/switch.cpp#L45) `init_service` → `Online::init`。`dispatch`([dispatcher.cpp#L86](../../mole2/mole2_switch/dispatcher.cpp#L86))同样新老双表:新协议走 `g_cli_cmd_map`,老协议走 `funcs[cmd-60001]`。`init_funcs`([dispatcher.cpp#L53](../../mole2/mole2_switch/dispatcher.cpp#L53))一眼看尽 switch 的全部职责:
- **online 在线表**:`proto_online_svrinfo→report_online_info`、`proto_online_keepalive→online_keepalive`。`report_online_info`([online.cpp#L75](../../mole2/mole2_switch/online.cpp#L75))记下每台 online 的 `online_ip/port/user_num`,**user_num 就是负载**。
- **给登录服反查推荐/分段服务器列表**:`login_proto_get_recommended_svrlist_new→get_recommeded_svrlist_new`([dispatcher.cpp#L73](../../mole2/mole2_switch/dispatcher.cpp#L73)),按 `user_num` 算负载并 `mix=(user_num<<16)|online_id` 排序挑负载最低的服([online.cpp#L753](../../mole2/mole2_switch/online.cpp#L753))。
- **跨服功能**:`proto_chat_across_svr / proto_noti_across_svr`(跨服聊天/公告)、`proto_users_online_status`(查好友是否在线、在哪台)。
- **本地 PK 报名**:`proto_onli_apply_pk / onli_auto_accept / onli_auto_fight`——和 cross 几乎同款匹配算法,switch 负责 **同一 IDC 内** 的 PK 撮合,cross 负责 **跨 IDC**。
- **运营后台**:`adm_proto_kick_user_offline / broadcast_msg / create_npc`。

### 6. M2Login(mole2_login_new)— 登录 / 封禁 / 选服

[login.cpp#L28](../../mole2/mole2_login_new/login.cpp#L28) `init_service` → `mole2_login::init_service`([login_impl.cpp#L41](../../mole2/mole2_login_new/login_impl.cpp#L41))。核心 `get_recommeded_svr_list_cmd`([login_impl.cpp#L146](../../mole2/mole2_login_new/login_impl.cpp#L146)):
1. **会话校验** `verify_login_session(uid, body)`([#L153](../../mole2/mole2_login_new/login_impl.cpp#L153))——账号验证由淘米统一登录(passport)签发 session,login 服只验签不存密码。
2. **多级封禁**:回包里区分永久封 `mole2_user_be_ban`、24 小时、7 天、14 天([login_impl.cpp#L298-L310](../../mole2/mole2_login_new/login_impl.cpp#L298))。
3. **选服**:正常返回从 switch 拿到的推荐列表;DB/switch 不可用时降级用本地定时刷新的 `m_backup_svrlist`(每 60 秒 `update_backup_svrlist` [#L35](../../mole2/mole2_login_new/login_impl.cpp#L35))。服务器列表通过组播下发(概览所述 `addr_mcast_ip`)。

### 7. M2home(mole2_home)— 家园 / GVG / PVP

[mole2_home.cpp#L62](../../mole2/mole2_home/mole2_home.cpp#L62) `init_service`:读 `PVP_GAME_USER_COUNT`(每局人数,默认 4,[#L77](../../mole2/mole2_home/mole2_home.cpp#L77)),维护两套 PVP 房间 `g_pvp[0]/g_pvp[1]`,`proc_events` 里 `check_time_outs` 做房间超时清理([#L46](../../mole2/mole2_home/mole2_home.cpp#L46))。包头 `home_proto_t` 额外带 `homeid` 用于多家园服定位。online 端 `send_to_homeserv` 与 batrserv 同构(多实例 + 按 id 选 fd)。

---

## 关键机制实战

### A. 一个 uint32 seq 搞定 DB 异步关联(已细述于第 1、2 节)
要点回顾:online 端 `seq=(fd<<16)|waitcmd`([svr_proto.cpp#L52](../../mole2/mole2_online/svr_proto.cpp#L52))→ DB 服无状态原样回传 → online `handle_db_return` 反解 `connfd=seq>>16, waitcmd=seq&0xFFFF`([dbproxy.cpp#L116](../../mole2/mole2_online/dbproxy.cpp#L116))。配合 sprite 的 `waitcmd`+`pending_cmd` 队列,实现"单玩家命令严格串行、跨玩家高并发"的无锁模型。

### B. C↔Lua 怪物 AI 桥(本模块最有料)

**C 侧注册了 39 个 Lua API**——`register_beast_ai_function`([bt_beastai.cpp#L1137](../../mole2/batrserv/src/bt_beastai.cpp#L1137))用 `REGISTER_LUA_FUN` 把 C 函数暴露给脚本:既有读状态的(`lua_get_player_prop / lua_get_player_least_hp / lua_get_player_highspeed`),也有 **直接做伤害/命中/耗蓝结算** 的(`lua_calc_phy_atk_hurthp_1/2`、`lua_calc_mp_atk_hurthp_1/2`、`lua_calc_attack_hit_rate`、`lua_calc_skill_cost_mp_1/2/3`,[#L1171-L1180](../../mole2/batrserv/src/bt_beastai.cpp#L1171)),还有选目标的(`lua_get_player_by_prof_rand`、`lua_get_player_by_rand_distance`)。**数值口径在 C,策略(打谁、放什么招)在 Lua**。这些 C 函数的参数全是 `(batid 字符串, mark 阵营, pos 位置, ...)`,内部统一经 `check_lua_api_param`([beast_ai_api.cpp#L36](../../mole2/batrserv/src/beast_ai_api.cpp#L36))把 batid 还原成 `battle_info_t*` 再操作。

**轮到怪物发招时 C 调 Lua**——`get_beast_attack_skill_by_lua`([bt_beastai.cpp#L1039](../../mole2/batrserv/src/bt_beastai.cpp#L1039)):
1. 先 `lua_checkstack(g_lua_state, 7)`,栈不够就 **整脚本重载**(防长期运行栈碎片,[#L1057](../../mole2/batrserv/src/bt_beastai.cpp#L1057))。
2. `lua_getglobal("lua_beast_attack_ai")` 取入口函数([#L1066](../../mole2/batrserv/src/bt_beastai.cpp#L1066),函数名常量 [onlineapi.h#L34](../../mole2/batrserv/src/onlineapi.h#L34))。
3. 压 6 个参数:batid 字符串、阵营 mark、位置 pos、`type_id`(怪物种类)、`atk_seq`(第几次发招)、是否偷袭 touxi([#L1080-L1102](../../mole2/batrserv/src/bt_beastai.cpp#L1080))。
4. `lua_pcall(L, 6, 1, 0)` 调用([#L1111](../../mole2/batrserv/src/bt_beastai.cpp#L1111));出错(脚本异常)就回退到 C 兜底 AI `beast_intel_attack`([#L1122](../../mole2/batrserv/src/bt_beastai.cpp#L1122))。
5. 取返回值:`1`=Lua 已用 `lua_set_atk_skill` 设好招式;`0`=Lua 不管,C 给个随机技能([#L1132](../../mole2/batrserv/src/bt_beastai.cpp#L1132))。

**Lua 侧的 AI 实现**——`../conf/batrserv_ai.ai`(由 `batrserv_ai.lua` 经 `luac` 编译成字节码,注释见 [batrserv_ai.lua](../../mole2/batrserv/conf/batrserv_ai.lua))。脚本开头定义了一整套 `g_prop_*` 属性枚举(0=当前血量、6=速度、15=必杀、29=是否可抓…,与 C 侧 `lua_get_player_prop` 的 propname 严格对应)。入口分发函数 `lua_beast_attack_ai`([batrserv_ai.lua#L1932](../../mole2/batrserv/conf/batrserv_ai.lua#L1932))按 `typeid` 路由到每只 BOSS 的专属 AI:`2001=土豆泥王`、`2002=野猪王`、`2004=骷髅王`、`3001=彼兔老师`、`3002=杰生老师`、`3003=瑞玲老师`、`3004/3005=温蒂&瑞贝卡`。每个 BOSS 函数内部再分"是否被防御/血量高低/进身/嘲讽"等分支(如 `lua_beast_attack_3002_ai_hp_over30 / _hp_low30` [#L1060/#L1083](../../mole2/batrserv/conf/batrserv_ai.lua#L1060))调那 39 个 C API 拿数值、做决策、回设招式。**这是一套"数值在 C、策略在 Lua、热重载(后台每 10 分钟比对文件 mtime 自动 reload)"的配置驱动 AI 范式**,自包含、可移植。

### C. 跨服 PK 撮合算法(分段冒泡匹配)

`onli_apply_pk`([cjob_dispatcher.cpp#L355](../../mole2/mole2_cross/cjob_dispatcher.cpp#L355))收到报名 `{type, level, count}`:
- 参数校验:`level∈(0, pklv_per_phase*pklv_phase_max]`(即 1–100,5 级/段×20 段)、`count∈(0, pk_team_cnt_max=5]`([#L367](../../mole2/mole2_cross/cjob_dispatcher.cpp#L367),常量 [proto.hpp#L26](../../mole2/mole2_cross/proto.hpp#L26))。
- `reset_onlineid_by_usrid` 记住"这个队伍在哪台 online"([#L384](../../mole2/mole2_cross/cjob_dispatcher.cpp#L384))——撮合成功后才知道把开战指令发回哪台。
- `try_to_match`([#L413](../../mole2/mole2_cross/cjob_dispatcher.cpp#L413))是核心:按 `lvidx=(level-1)/5` 算等级段。**单人池 `m_single_pk_[段]`、组队池 `m_team_pk_[人数][段]`**,每段只存一个等待者。匹配优先级:① 本段有人 → 立即配对;② 否则看 **相邻上一段/下一段**,且等级差 `<5` 也配([#L424-L429](../../mole2/mole2_cross/cjob_dispatcher.cpp#L424));③ 都没有 → 把自己放进池子,挂一个 **30 秒重试定时器** `n_try_match_again`([#L438](../../mole2/mole2_cross/cjob_dispatcher.cpp#L438))。
- 30 秒到了 `try_match_again`([#L163](../../mole2/mole2_cross/cjob_dispatcher.cpp#L163))再 `match_again_single/team` 跨更多段位找对手——**等得越久放宽匹配范围**,经典分段冒泡撮合。
- 配对成功 → 给报名方发 `CROSS_COMMID_ONLI_AUTO_CHALLENGE` 让其 online 发起对战、给对手发 `CROSS_COMMID_AUTO_FIGHT`([#L394-L402](../../mole2/mole2_cross/cjob_dispatcher.cpp#L394)),双方进入 `m_start/m_accept` 表等待自动接受/自动开战。

---

## 端到端调用流举例:玩家发起一场宠物对战

追 `proto_cli_challenge=1300`([cli_proto.h#L129](../../mole2/mole2_online/cli_proto.h#L129)),从点"挑战"到对战服建局:

1. **客户端→online**:Flash 发 `protocol_t{cmd=1300}`。online `proc_pkg_from_client→dispatch`([cli_proto.cpp#L335](../../mole2/mole2_online/cli_proto.cpp#L335))翻转字节序、校验该 fd 有在线 sprite、置 `p->waitcmd=1300`,调 battle 模块挑战处理。
2. **online→batrserv**:battle 模块 `send_to_batrserv(challenger, bt_battle_info_msg, ..., challenger->id, battle_id, proto_cli_challenge)`([battle.cpp#L1389](../../mole2/mole2_online/battle.cpp#L1389))。`send_to_batrserv`([svr_proto.cpp#L335](../../mole2/mole2_online/svr_proto.cpp#L335))按 `BATTLE_USER_ID(btid)%MAX_BATRSERV_NUM` 选对战服实例,封 `batrserv_proto_t{cmd=61006, online_id=本服id, opid=1300, challenger=btid}` 发出。
3. **batrserv 建局**:`dispatch`([bt_dispatcher.cpp#L25](../../mole2/batrserv/src/bt_dispatcher.cpp#L25))记 `online[online_id]=fdsess` 后进 `battle_info_op`([bt_online.cpp#L6177](../../mole2/batrserv/src/bt_online.cpp#L6177)):opid=1300 且 battle_id 尚不存在 → `g_slice_alloc0` 新建 `battle_info_t`、初始化四条链表、插入 `battles` 哈希表。
4. **batrserv→online→客户端**:对战服回 `proto_bt_noti_battle_created=1305` 等,online `proc_pkg_from_serv`([mole2_online.cpp#L334](../../mole2/mole2_online/mole2_online.cpp#L334))识别来自 batrserv 的 fd → `handle_batrserv_pkg`([battle.cpp#L4367](../../mole2/mole2_online/battle.cpp#L4367))按 opid 进 `noti_battle_created_callback` 等回调,转成客户端协议下发。
5. **发招回合**:客户端发 `proto_cli_battle_attack=1308`,online 透传成 batrserv 的 `bt_battle_attack`([bt_online.cpp#L6289](../../mole2/batrserv/src/bt_online.cpp#L6289));若该位是怪物,batrserv 走 `get_beast_attack_skill_by_lua`([bt_beastai.cpp#L1039](../../mole2/batrserv/src/bt_beastai.cpp#L1039))让 Lua 决策。回合结算后回 `proto_bt_battle_attacks_info=1310 / battle_round_result=1311`,online 广播给场内所有玩家。
6. **战斗结束落库**:对战服回 `battle_end`,online 侧 `noti_battle_end_callback` 把经验/掉落/胜负写 DB——又走 §A 的 `seq=(fd<<16)|waitcmd` 异步落库链路,DB 服 `Croute_func::deal` 一条命令一个事务 commit。

**全链路命令字一致性**:online 内部用 `proto_cli_*`(1300 系)做语义,发往 batrserv 时塞进 `opid`,batrserv 回包也用同一套 opid,online 再翻成客户端 `proto_bt_*`(1305/1310/1311…见 [batrserv/src/proto.h#L17](../../mole2/batrserv/src/proto.h#L17))。三方共用一套编号,极易对照。

---

## 复活实操提示(对 moleworld 复活)

> 前提同概览:moleworld 复活的是**一代**摩尔庄园([mole](../../mole)),mole2 是二代,客户端协议不通用。本节聚焦"运行机制层面"能直接借鉴/移植什么。

**能直用(与客户端协议无关、纯逻辑/数据)**:
- **C↔Lua 怪物 AI 整套范式**:`register_beast_ai_function` 的 39 个 C API 清单([bt_beastai.cpp#L1137](../../mole2/batrserv/src/bt_beastai.cpp#L1137))+ `get_beast_attack_skill_by_lua` 的调用约定 + `batrserv_ai.lua` 的 `g_prop_*` 枚举与按 typeid 分发结构,是一套现成、自包含的"数值在宿主、策略在脚本、热重载"AI 底座。做任何回合制 BOSS AI 都可整段抄。
- **对战数值表**:`BattleActionList.xml`(招式属性,303KB)、`skills_price.xml`(耗蓝)、`BattleMonsterTalk.xml`(怪物台词)是纯数据,可直接喂给自研结算器。

**值得照搬的设计模式**:
- **`seq=(fd<<16)|waitcmd` 的无状态 DB 异步关联**([svr_proto.cpp#L52](../../mole2/mole2_online/svr_proto.cpp#L52) / [dbproxy.cpp#L116](../../mole2/mole2_online/dbproxy.cpp#L116))——让 DBProxy 完全无状态、纯转发,是单线程事件循环服务端的干净做法。
- **sprite.waitcmd + pending_cmd 队列**([cli_proto.cpp#L384](../../mole2/mole2_online/cli_proto.cpp#L384))——单玩家命令严格串行,杜绝同一玩家并发改库竞态,实现简单且无锁。
- **"信封"协议**(外层固定 cmd + 内层 opid 二级分发):batrserv 用 61006、cross 用 65001,后端服只需认一种外层 cmd,业务命令全在 opid——后端扩命令不动框架。
- **switch 按 user_num 负载均衡选服 + cross 分段冒泡撮合**:做多实例/跨服时的成熟蓝本。

**要自己补的**:
- **无 SQL 建表脚本**:schema 只能从 `mole2_db/src/Cuser.cpp / Cpet.cpp` 等 ORM 类 + `field_desc.txt` 反推,落库前需自写 DDL。
- **强依赖淘米内部框架**:`async_serv`(多 worker fork 框架)、`libtaomee/libtaomee++`(日志/定时器/`Ccmd_map`/`byte_array_t`/`Cmessage` 序列化)、`statistic_agent`——缺这些无法编译,需替换为自有等价物(见 [c.md](./c.md)、[libtaomee.md](./libtaomee.md),两个框架包在仓库根)。batrserv 还需 `glib`(`g_hash_table/g_slice/g_queue`)与内嵌 Lua 5.1。

---
返回总入口:[../../README.md](../../README.md) · 概览 [../modules/mole2.md](../modules/mole2.md) · 一代 [./mole.md](./mole.md) · 协议 [../protocol.md](../protocol.md) · 内容数据 [../content-data.md](../content-data.md)
