# monster 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/monster.md](../modules/monster.md) · 协议总表 [../protocol.md](../protocol.md) · 内容数据 [../content-data.md](../content-data.md) · 总入口 [../../README.md](../../README.md)
>
> 本文是概览的"深水区"补充:不再罗列"有什么文件",而是逐部分讲清**运行时它实际在干什么**、**关键函数怎么跑**、**进程之间怎么把一个请求接力到落库**。所有 `#L` 行号均真实 Read/grep 到。
>
> 游戏代号已核实为 **淘米"小怪兽"**(养成社区,非赛尔号);判据见概览的"重要修正"节。

---

## 一图看懂:进程/组件拓扑与数据流

小怪兽是一套**多进程 + 动态库逻辑**的页游服务端,与摩尔庄园 `mole` 完全同底座(libtaomee + async_serv 宿主 `AsynServ`,游戏逻辑编成 `.so`)。运行时的数据流:

```
                 Flash 客户端 (as,大端协议)
                        │  as_msg_*  (40000 段)
          ┌─────────────┴──────────────┐
          ▼                             ▼
      [login 进程]                  [online 进程 = libmonsteronline.so]
   要一个可用 online              ←─ 主游戏逻辑(注册/养成/种植/小屋/好友/精灵/益智)
          │ as_msg_acquire_online           │   │            │
          ▼ (=110)                          │   │            │
      [switch 进程] ── 选最空闲 online      │   │            │
        商店轮换 / 每日对战 / 在线统计       │   │            │
                                            │   │            │
        svr_msg_db_* (0x8000 段) ───────────┘   │            │
          ▼                                      ▼            ▼
      [db-proxy] ── route.xml 按 cmdid 分流  [ucount 进程]  [multi-server]
       0x8000→MONSTER  0x8200→MONSTER_CONF    访问/在线计数   二人益智撮合
       0xA000→account                          (mmap 文件)    (跨/多 online)
          ▼                       ┌──────────────┴────────────┐
      [db-server] ── MySQL +      ▼                            ▼
       memcached(role 缓存)    [db-cache-server]           account 平台
       分库分表 t_role_[0..99]   留言板/大众点评show           (外部,验 session)
       db_monster_[0..9]         (MySQL + mmap 排行文件)
          │
          ▼
        MySQL (10 库 × 100 表)

      [dirty_server] 脏字过滤(独立二进制 dirty_agent,经 libtaomee 的 tm_dirty_check 调用)
      [share-server] 分享个人小屋 / 最近登录用户(独立 so,自带 login/dispatch)
      [thttpd] 静态策略文件 + crossdomain.xml(Flash 跨域)
```

关键认知(与概览不同的"运行时真相"):

- **online 不直连 MySQL**。它只跟 `switch`、`db-proxy`、`ucount`、`multi-server`、`db-cache-server` 这几个内部服务建长连接(见 [dll_interface.cpp:44-130](../../monster/online/src/dll_interface.cpp#L44)),所有持久化都打包成 `svr_msg_db_*` 走 db-proxy。
- **db-proxy 只是"按命令号路由的转发器"**,不碰业务、不连库;真正连 MySQL 的是 **db-server** 和 **db-cache-server** 两个不同的后端。
- **"db-cache-server" 名字有误导**:它不是 db-server 的通用缓存前置,而是**专职处理留言板(pinboard)和"大众点评"show 排行**的 DB 服务,自己连 MySQL 还 mmap 排行榜文件(详见下文)。真正的"缓存"是 **db-server 内嵌的 memcached**(缓存 role 行)。

---

## 逐部分拆解:每个进程实际职责 + 关键函数 + 跟谁通信

### 1. online —— 主游戏逻辑(`libmonsteronline.so`,占全模块近半源码)

这是一坨被宿主 `AsynServ` 加载的动态库,作者 henry/luis,2011-07。它的运行骨架是三张"命令字→处理函数"映射表:

- **客户端请求表** `g_cli_cmd_map`,在 [init_cli_handle_funs() cli_proto.cpp:56](../../monster/online/src/cli_proto.cpp#L56) 一次性注册约 90 个 `as_msg_* → process_*`。例如 `as_msg_eat_food=40128 → process_eat_food`、`as_msg_grow_plant=40123 → process_grow_plant`、`as_msg_attract_pet=40125 → process_attract_pet`。
- **DB 回包表** `g_db_cmd_map`,在 [init_db_handle_funs() db_proxy.cpp:47](../../monster/online/src/db_proxy.cpp#L47) 注册约 90 个 `svr_msg_db_* → process_db_*_return`(例 `svr_msg_db_eat_food=0x801e → process_db_eat_food_return`)。
- **cache-server 回包**走单独的 switch-case,见 [handle_cache_svr_return() db_cache.cpp:27](../../monster/online/src/db_cache.cpp#L27)(只处理 pinboard / show)。

**入口分发** [dispatch() cli_proto.cpp:155](../../monster/online/src/cli_proto.cpp#L155):每个客户端包先做大端转换,然后一连串严格校验(见下文"包校验"),再 `cmd_iter->second(...)` 调到对应 `process_*`。

它跟谁通信:

| 连接 | 用途 | 建立处 |
|------|------|--------|
| `g_p_switch_svr` | 向 switch 注册自己、上报在线数、取商店 | [dll_interface.cpp:46-67](../../monster/online/src/dll_interface.cpp#L46) |
| `g_p_db_proxy_svr` | 所有持久化(role/bag/plant/pet/friend…)、向 account 验 session | [dll_interface.cpp:74-95](../../monster/online/src/dll_interface.cpp#L74) |
| `g_p_ucount_svr` | 上报访问/在线计数 | [dll_interface.cpp:97-119](../../monster/online/src/dll_interface.cpp#L97) |
| multi-server / db-cache-server | 二人益智 / 留言板 show | multi_svr.cpp / db_cache.cpp |

核心业务函数库 [function.cpp](../../monster/online/src/function.cpp):怪兽**养成数值的纯函数**全在这里 —— 愉悦衰减 [get_new_happy() function.cpp:188](../../monster/online/src/function.cpp#L188)、健康衰减 [get_now_health() function.cpp:234](../../monster/online/src/function.cpp#L234)、等级换算 [get_level_from_exp() function.cpp:338](../../monster/online/src/function.cpp#L338,二分查 `g_level[]` 经验表)。

在线用户内存态由 [c_user_cache.cpp](../../monster/online/src/c_user_cache.cpp) 维护(role/bag/room/plant/day_restrict 全缓存在用户对象里,登录时一次性拉齐,之后增量改、随手回写 DB)。

### 2. login —— 接入与"要一个 online"

login 自己几乎不做业务,基于 libtaomee 的 c_xxx_login 骨架(见 [login_impl.cpp:49 proc_other_cmds](../../monster/login/src/login_impl.cpp#L49)):客户端发 `as_msg_acquire_online` 时,它调 [get_svr_list_cmd() login_impl.cpp:167](../../monster/login/src/login_impl.cpp#L167) 向 switch 问一个 IP:port,把结果回给客户端,客户端再直连那台 online。回包在 [handle_switch_return() login_impl.cpp:87](../../monster/login/src/login_impl.cpp#L87)。

### 3. switch —— 调度大脑 + 全局玩法

switch 持有所有 online 的在线人数表,做三件事:

- **负载均衡选 online**:[online.cpp:332-380](../../monster/switch/src/online.cpp#L332) 遍历 `m_online_info_map` 找 `conn_num` 最小的那台返回;若最小都 `>= MAX_NUM_PER_ONLINE` 就拒绝(满载),`> 3/4` 触发告警。online 上线/下线时 `conn_num` 增减并同步全局在线数([online.cpp:268-285](../../monster/switch/src/online.cpp#L268))。
- **全局商店轮换**:[shop.cpp](../../monster/switch/src/shop.cpp) 按 `shop_update_time=900s`([system.xml](../../monster/conf/system.xml))定时刷新在售物品,所有 online 共享同一份。
- **每日对战 / 在线统计**:[everyday_fight.cpp](../../monster/switch/src/everyday_fight.cpp)、在线低于 `m_warning_line` 时告警([online.cpp:846](../../monster/switch/src/online.cpp#L846))。

### 4. db-proxy —— 纯路由转发器(无业务)

最小的一个进程(源码 0.00 MB 级)。靠 [route.xml](../../monster/db-proxy/conf/route.xml) 把 `svr_msg_*` 命令号段映射到后端 IP:port:

- `0x8000` / `0x8400`(MONSTER 业务)→ `db_type=10`,按 `db_startid=0 ~ db_endid=9` 路由到 10 个分库后端;
- `0x8200`(MONSTER_CONF,全局配置)→ `db_type=1` 单实例;
- `0xA000`(account)→ 外部账号平台 `10.1.1.47:11014`。

注释写明可用命令号范围 `0x8000-0x9FFFF`,每区段间隔 512(0x200)。online 永远不知道库在哪,只认命令号 —— db-proxy 替它把请求送到正确的分库 db-server。

### 5. db-server —— 真正连 MySQL 的持久化层(分库分表 + memcached)

按业务拆成几十个子目录([role-info](../../monster/db-server/src/role-info/role_info.cpp) / [plantation](../../monster/db-server/src/plantation/plantation.cpp) / [friend](../../monster/db-server/src/friend/friend.cpp) / [pet](../../monster/db-server/src/pet/) / [badge](../../monster/db-server/src/badge/) / [task](../../monster/db-server/src/task/) …),每个收对应 `svr_msg_db_*` 拼 SQL 落库。

- **分库分表**:库 = `uid % 10`,表 = `(uid/100) % 100`,宏在 [util.h:26-27](../../monster/db-server/src/util.h#L26)。落地形如 `db_monster_<DB_ID>.t_role_<TABLE_ID>`(见 INSERT [role_info.cpp:91](../../monster/db-server/src/role-info/role_info.cpp#L91)、UPDATE [role_info.cpp:982](../../monster/db-server/src/role-info/role_info.cpp#L982))。
- **memcached 写穿透**:role 行同时进 memcached。读 role 先查缓存 [role_info.cpp:262 get_role](../../monster/db-server/src/role-info/role_info.cpp#L262),写后回填 [role_info.cpp:621 set_role](../../monster/db-server/src/role-info/role_info.cpp#L621);累加金币/经验等也走"读缓存→改→回 set"([role_info.cpp:979/1220](../../monster/db-server/src/role-info/role_info.cpp#L979))。这才是本模块真正的"DB 缓存"。

### 6. db-cache-server —— 留言板 + "大众点评"show 专属 DB(名字误导)

它**也直连 MySQL**([work_proc.cpp:119-128](../../monster/db-cache-server/src/work-proc/work_proc.cpp#L119) `create_mysql_iface_instance` + `init`),但只服务一小撮命令([work_proc.cpp:232-249](../../monster/db-cache-server/src/work-proc/work_proc.cpp#L232)):`svr_msg_db_get_pinboard_info`(留言板)、`svr_msg_db_add_message`、以及 `enter_show / join_show / vote_show / history_show`(社区"大众点评"投票)。show 的实时排行用 **mmap 文件**做双缓冲(cur/last,[work_proc.cpp:303/354](../../monster/db-cache-server/src/work-proc/work_proc.cpp#L303))。online 这侧的回包专门走 [handle_cache_svr_return() db_cache.cpp:27](../../monster/online/src/db_cache.cpp#L27)。

### 7. multi-server —— 二人益智实时撮合(henry,2011-12)

跨多台 online 的"怪卡连连看/答题"双人对战服。撮合模型很朴素:**全局只有一个等待位** `m_waiting_match_user`:

- 第一个进入的人占住等待位、记时间戳,回 `is_found=0`([work_proc.cpp:408-424](../../monster/multi-server/src/work_proc.cpp#L408));
- 第二个人进来立刻配对:生成题库 `set_question()`、把对手信息和题目一起回给双方、把两人放进 `m_matched_gamer_map`([work_proc.cpp:430-470](../../monster/multi-server/src/work_proc.cpp#L430));
- 超时(`match_opp_time=10s`)还没等到真人,定时器 `check_waiting_user` 把等待者**配给机器人**对战([work_proc.cpp:185](../../monster/multi-server/src/work_proc.cpp#L185));配对成功后 `click_start_time=30s` 没点开始也由定时器清理([work_proc.cpp:188](../../monster/multi-server/src/work_proc.cpp#L188))。

状态机用一组 map 互斥:等待 / 已匹配 / 游戏中 / 配机器人中 / 与机器人游戏中,enter 时逐个查重防重复进入([work_proc.cpp:344-391](../../monster/multi-server/src/work_proc.cpp#L344))。每日咕咚果上限 `two_puzzle_day_restrict=50`([work_proc.cpp:173](../../monster/multi-server/src/work_proc.cpp#L173))。

### 8. ucount-server —— 访问量/在线计数(纯文件)

不连库,把 uid 计数写进自管理的文件 [ucount.cpp:54 init](../../monster/ucount-server/src/ucount/ucount.cpp#L54)(`O_RDWR|O_CREAT`,`ucount_prefix`/`visit_prefix` 两种前缀文件)。online 通过 `g_p_ucount_svr` 上报"谁访问了谁"(`ucount_msg_add_visit`)。

### 9. share-server —— 分享小屋 / 最近登录(独立 so)

结构是 online 的"迷你版":自带 [init_cli_handle_funs / dispatch cli_proto.h](../../monster/share-server/src/cli_proto.h)、自己的 db_proxy 封装。职责是对外暴露"看别人的小屋"与"最近登录用户列表"。注意它复刻了一份养成数值函数,如 [get_other_monster_health() function.cpp:167](../../monster/share-server/src/function.cpp#L167) —— 展示别人怪兽时也要按离线时长把健康值衰减算到"现在"。

### 10. dirty_server —— 脏字过滤(外部共享设施)

本目录只有编译好的 `dirty_agent` 二进制 + 配置。真正的调用入口是 libtaomee 提供的 `tm_dirty_check()`,online 在注册改名等处直接调:注册时校验角色名和怪兽名 [cli_proto.cpp:440/456](../../monster/online/src/cli_proto.cpp#L440)、专门的 `process_check_dirty_word`/`process_modify_monster_name` [cli_proto.cpp:3645/3678](../../monster/online/src/cli_proto.cpp#L3645)。脏词库由远端脏词服务下发并定时(600s)更新([dirty_agent.conf](../../monster/dirty_server/dirty_agent.conf),`tm_dirty_server_addr=192.168.4.68:28000;...`,可见是跨游戏共享的基础设施)。

---

## 关键机制实战(本模块最有料的三处)

### 机制 A:每用户"单飞行请求"串行化 + 命令缓存队列

这是整个 online 异步模型的核心,理解它才能看懂为什么所有 `process_*` 都"发完 DB 包就 return"。

每个在线用户对象有个 `waitcmd` 字段:**同一时刻只允许一条命令在途**。

1. 客户端包进 [dispatch()](../../monster/online/src/cli_proto.cpp#L155),若该用户 `waitcmd != 0`(上一条还没等到 DB 回包),新包不丢弃也不并发,而是 [cache_a_pkg() cli_proto.cpp:317](../../monster/online/src/cli_proto.cpp#L317) 塞进该用户的 `cached_pkg_queue`(上限 `MAX_REQ_PER_USER`,满了回 `ERR_TOO_MANY_PKG`),并把用户挂到 `g_waiting_user_list`。
2. 可以处理时设 `p_user->waitcmd = msg_type`,调 `process_*`([cli_proto.cpp:288-290](../../monster/online/src/cli_proto.cpp#L288))。`process_*` 通常**只把请求打包发给 db-proxy 就返回**(如 eat_food [cli_proto.cpp:1708](../../monster/online/src/cli_proto.cpp#L1708)),并把"待用的请求参数"暂存进 `p_user->buffer`([cli_proto.cpp:1714](../../monster/online/src/cli_proto.cpp#L1714))。
3. DB 回包到 [handle_db_return() db_proxy.cpp:131](../../monster/online/src/db_proxy.cpp#L131),按命令号找到 `process_db_*_return`,在那里**真正改内存态并回客户端**,最后清 `waitcmd=0`。
4. `waitcmd` 一清零,主循环调 [proc_cached_pkgs() cli_proto.cpp:343](../../monster/online/src/cli_proto.cpp#L343) 把该用户队列里缓存的下一条 `dispatch(..., first_tm=false)` 取出来跑(注意 `first_tm=false` 会跳过序列号/包体校验,因为这是服务端自己缓存的可信包)。

**为什么这么设计**:单 online 进程内对同一个用户的状态改动天然串行,避免"还没落库就被下一条改了"的竞态,也避免一个用户的慢 DB 拖垮整个 reactor。

### 机制 B:DB 请求/回包的 seq_id 关联(conn_fd<<16 | counter)

online 是单进程异步,发出去的 DB 请求和回来的回包必须对上号、还要防"连接已断/超时回包打到新用户"。做法是把 `seq_id` 编码成 **高 16 位 = 连接 fd,低 16 位 = 该用户的递增 counter**:

回包时 [db_proxy.cpp:143-148](../../monster/online/src/db_proxy.cpp#L143) 解出 `conn_fd = seq_id>>16`、`counter = seq_id & 0xFFFF`,用 fd 找回用户,再校验 `counter == p_user->counter`。对不上(用户已断开/换人/超时)就直接丢弃,绝不误改。cache-server 回包用同一套编码([db_cache.cpp:38](../../monster/online/src/db_cache.cpp#L38))。

### 机制 C:养成数值的"惰性结算"(读时按时间差现算)

健康/愉悦不存"实时值",而是存"上次值 + 上次时间",**读到才按经过时间倒推现在值** —— 经典页游省 CPU 写法。

- 愉悦:每 `happy_desc_time=30` 分钟掉 1 点,[get_new_happy() function.cpp:188](../../monster/online/src/function.cpp#L188)。
- 健康:**分段衰减**——高段(600-999)每 2 分钟掉 1,中段(300-600)每 28 分钟,低段每 1000 分钟,离线再按 43 分钟/点单算([system.xml](../../monster/conf/system.xml) + [get_now_health() function.cpp:234](../../monster/online/src/function.cpp#L234) + [get_monster_health_on_login() function.cpp:207](../../monster/online/src/function.cpp#L207))。

喂食时先把当前值结算到 `now`,再算"还能加多少"(封顶 MAX),把净增量发给 DB([process_eat_food cli_proto.cpp:1694-1706](../../monster/online/src/cli_proto.cpp#L1694))。

---

## 端到端调用流举例:追一次"喂食"从客户端到落库

以 `as_msg_eat_food=40128`(给怪兽喂一份食物)为例,跑完整条链:

1. **客户端 → online**:Flash 发包 `[as_msg_header | item_id]`(大端)。进 [dispatch() cli_proto.cpp:155](../../monster/online/src/cli_proto.cpp#L155):大端转换 → 校验(用户已登录、uid 匹配、序列号 `get_new_seq` 与包体校验和 `check_body_sum` 都对,[cli_proto.cpp:223-238](../../monster/online/src/cli_proto.cpp#L223))。
2. **设 waitcmd + 调处理函数**:`waitcmd=40128`,调 [process_eat_food() cli_proto.cpp:1640](../../monster/online/src/cli_proto.cpp#L1640)。
3. **本地预校验(零 DB)**:查 `g_item_map` 确认是食物且 `category==ITEM_FOOD`;查内存背包 `get_stuff_num` 确认有货(没货直接回 `ERR_NOT_ENOUGH_FOOD`,不发 DB,[cli_proto.cpp:1668](../../monster/online/src/cli_proto.cpp#L1668))。
4. **算净增量**:用机制 C 把当前健康/愉悦结算到 now,算出 `reward_health/reward_happy`(封顶到 MAX,[cli_proto.cpp:1694-1706](../../monster/online/src/cli_proto.cpp#L1694))。
5. **online → db-proxy**:打包 `svr_msg_db_eat_food=0x801e` 发 `g_p_db_proxy_svr`([cli_proto.cpp:1708](../../monster/online/src/cli_proto.cpp#L1708)),把这份 `db_msg_eat_food_req_t` 暂存进 `p_user->buffer`([cli_proto.cpp:1714](../../monster/online/src/cli_proto.cpp#L1714)),函数 return(此时 waitcmd 仍=40128)。
6. **db-proxy → db-server**:route.xml 按 `0x8000` 段把包路由到 `uid%10` 对应的 db-server 实例。
7. **db-server 落库**:扣道具、累加 role 的 health/happy,UPDATE `db_monster_<uid%10>.t_role_<(uid/100)%100>` 并回填 memcached,把结果回 db-proxy。
8. **回包 → online**:db-proxy 原路回 online,进 [handle_db_return() db_proxy.cpp:131](../../monster/online/src/db_proxy.cpp#L131),seq_id 校验通过 → 调 [process_db_eat_food_return() db_proxy.cpp:2387](../../monster/online/src/db_proxy.cpp#L2387)。
9. **改内存态 + 回客户端**:从 `p_user->buffer` 取回奖励值,扣背包 `desc_stuff`、把 health/happy 加到内存 role([db_proxy.cpp:2406-2420](../../monster/online/src/db_proxy.cpp#L2406)),`notify_as_current_user_info` 把新状态推回客户端;若愉悦变化还连带触发种植园成长重算 `update_plant_info_after_eat_food`(怪兽越开心植物长越快,[system.xml](../../monster/conf/system.xml) `happy_growth_time` 对应)。
10. **清 waitcmd + 放下一条**:回包处理结束 waitcmd 归零,`proc_cached_pkgs` 把该用户排队的下一条命令取出来跑(机制 A)。

一次喂食 = 1 个客户端往返 + 1 个 online↔db 往返,中间还顺手触发了种植园联动。这条链就是小怪兽所有玩法的通用模板(种植、买物、答题…都同构)。

---

## 复活实操提示(对 moleworld 复活的价值)

定位:**架构参照为主,代码不可直用**(协议号/数据结构是另一款游戏的)。但它是比 mole 更规整的同底座样本,几处可直接抄设计:

- **可直接照搬的范式**:
  - 分库分表落地 `uid%10` + `(uid/100)%100`([util.h:26](../../monster/db-server/src/util.h#L26))+ db-proxy 按命令号路由([route.xml](../../monster/db-proxy/conf/route.xml)),给摩尔离线存档分片是现成模板。
  - **每用户单飞行请求 + 缓存队列 + seq_id(fd<<16|counter)关联**(机制 A/B)是这套异步框架的"心法",复活任何 online 逻辑都要复刻它,否则会撞内存竞态。
  - **惰性数值结算**(机制 C):摩尔的拉姆/种植/喂食与此同源,数值公式可直接参照 [system.xml](../../monster/conf/system.xml) + [function.cpp](../../monster/online/src/function.cpp) 复刻。
- **要自己补的**:
  - 宿主 `AsynServ` 与 libtaomee/libtaomee++ 头库不在本目录(`bin/` 只有二进制),需从仓库根的 `Libtaomee-*.tar.gz` 取头文件才能理解 `send_data/timeout_cb/fdsession_t` 等接口。
  - dirty_server / account 平台是**外部共享设施**(脏词服 `192.168.4.68`、account `10.1.1.47`),复活时要么自建 stub,要么把 `tm_dirty_use_dirty_logical` 关掉绕过([dirty_agent.conf](../../monster/dirty_server/dirty_agent.conf))。
  - db-cache-server 的 mmap 排行文件、ucount 的计数文件都是有状态本地文件,迁移环境要一起带或重建。

---
↩ 返回 [总入口 README](../../README.md) · 概览 [../modules/monster.md](../modules/monster.md)
