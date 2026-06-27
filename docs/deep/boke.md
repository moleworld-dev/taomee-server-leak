# boke 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/boke.md](../modules/boke.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)

本文不重复"它是什么"(那是概览的事),而是逐进程、逐核心函数地讲清 **boke(《卜克》,游戏 ID=12,摩尔庄园2/mole2 代码 fork)在运行时实际怎么跑**:谁连谁、报文怎么流、关键算法怎么算、一次真实玩法从客户端到落库的完整链路。所有引用均带真实行号。

---

## 一图看懂:进程/组件拓扑与数据流

五个进程都是 **淘米 async_serv 框架的插件 .so**(各自实现 `init_service`/`get_pkg_len`/`proc_pkg_from_client`/`proc_pkg_from_serv` 等钩子)。框架负责 epoll/收发/分包,业务 .so 只填钩子。

```
                        Flash 客户端 (TCP, 小端 protocol_t 头)
                              │
            (843端口 policy-file-request → 返回 cross-domain-policy)
                              │
        ┌─────────────────────┼──────────────────────────┐
        ▼ login_*_cmd          ▼ 进游戏后所有 cli_*_cmd     
   ┌─────────┐            ┌──────────┐                   
   │ login   │            │  switch  │  网关:命令路由+在线表
   └────┬────┘            └────┬─────┘                   
        │ check_session         │ sw_get_recommend_svr / report_user_onoff
        │                       │ (从 online 池里挑一台 user_num≤45 的)
        ▼                       ▼
   ┌─────────────────────────────────────┐
   │            online (游戏世界主进程)     │  ← 代码主体,Csprite 状态机
   │  dispatch() → g_cli_cmd_map → 处理函数 │
   └───┬───────────────┬──────────────┬───┘
       │ TCP db_proto   │ TCP switch    │ UDP post (投稿/聊天审查)
       ▼                ▼               ▼
   ┌────────┐      (上报在线)      ┌──────────┐
   │ dbproxy│                     │  chat    │ 行为式刷屏检测+脏词库分发
   │  /db   │                     └────┬─────┘
   └───┬────┘                          │ adm_forbid_chat 回推
       ▼                               ▼ (禁言通知反向打到 online 网关)
   MySQL 分库 POP_00..POP_99
   每库 t_user_0..9 / t_task_* / t_item_* / t_find_map_*
```

要点:
- **online↔db** 用 `db_proto_t`(TCP),见 [boke/online/src/svr_proto.h](../../boke/online/src/svr_proto.h);**所有进程间**统一 `svr_proto_t`(len/seq/cmd/ret/id+body),见概览。
- **online↔chat** 走 **UDP**:[boke/online/src/online.cpp#L123](../../boke/online/src/online.cpp#L123) 建 `udp_post_fd`,[boke/online/src/svr_proto.cpp#L69](../../boke/online/src/svr_proto.cpp#L69) `sendto()` 投递,fire-and-forget(投稿/聊天上报无需回执)。
- online 进程内自带 **轻量级脏词** `tm_dirty_check`(同步、即时拒绝),chat 进程跑 **重量级行为式刷屏检测**(异步、慢决策、累积封禁)——两套并存,见后。

---

## 逐部分拆解

### 1. online —— 游戏世界主进程(代码主体)

**框架接入点**:[boke/online/src/online.cpp](../../boke/online/src/online.cpp)
- `init_service()` [#L110](../../boke/online/src/online.cpp#L110):非父进程时 `srand(time*server_id)`、new 出全局 `g_sprite_map`/`g_timer_map`、调 `init_data()` 加载所有 XML、`connect_to_switch()` 上报、建 UDP post socket。
- `init_data()` [#L32](../../boke/online/src/online.cpp#L32):**靠首网卡 IP 前缀判生产/测试环境**——`strncmp(ip,"10.",3)==0` 即测试环境(开 hex 日志);否则生产。然后注册四张分发表(switch/cli/db/timer)+ 加载 TaskData/ItemXmlData/jf_shop/Map_name/npcxp/gamesid/movement/RandGroup 八个 XML。
- `get_pkg_len()` [#L165](../../boke/online/src/online.cpp#L165):**分包器**。父进程(面向客户端)先识别 Flash 的 `<policy-file-request/>` 直接回 `<cross-domain-policy>`(允许所有域所有端口),否则按 `protocol_t::len` 截包;子进程(面向服务器)按 `*(uint32_t*)` 头四字节截。
- `on_client_conn_closed()` [#L259](../../boke/online/src/online.cpp#L259):掉线即 `g_sprite_map->del_sprite_by_fd(fd)`,触发同地图广播"某人离开"。
- `reload_global_data()` [#L289](../../boke/online/src/online.cpp#L289):热重载——重新 `init_data()`,即不重启进程就能换 XML 配置。

**注册时的默认装扮池**:`init_user_reg_config_map()` [#L66](../../boke/online/src/online.cpp#L66) 硬编码了头发/嘴巴/上衣/裤子/眼睛五类 itemid 数组,新建角色时从中随机选,这就是"捏脸"的服务端真相。

**命令分发与串行化**:[boke/online/src/cli_proto.cpp](../../boke/online/src/cli_proto.cpp)
- `dispatch()` [#L57](../../boke/online/src/cli_proto.cpp#L57) 是每个客户端包的入口,实际逻辑链:
  1. `cmd==1` → 直接回版本串(256 字节)。
  2. **登录态闸门** [#L72](../../boke/online/src/cli_proto.cpp#L72):未在线只能发 `cli_login_cmd`,已在线不能再发登录;已登录但未注册(`flag & user_flag_reg==0`)只能发 `cli_reg_cmd`——服务端强约束协议时序。
  3. 登录时按写死的 `pkg->len==18+40` 创建 `Csprite` [#L83](../../boke/online/src/cli_proto.cpp#L83)。
  4. **同用户命令串行化** [#L100](../../boke/online/src/cli_proto.cpp#L100):若该玩家还有未处理完的命令(`get_waitcmd()!=0`),新包进 `cache_buffer_lst` 排队,处理完一条再 pop 下一条——保证单玩家命令严格串行,杜绝并发改存档竞态。
  5. 查 `g_cli_cmd_map`(由生成的 `pop_online_bind_for_cli_request.h` 初始化 [#L36](../../boke/online/src/cli_proto.cpp#L36))拿处理函数,`read_from_buf_ex` 反序列化 body,`set_waitcmd` 后调处理函数。
- 处理函数签名统一 `int f(Csprite* p, Cmessage* c_in)`,例:`cli_login` [#L134](../../boke/online/src/cli_proto.cpp#L134) 把 session 转成 `main_login_check_session_in` 发给 db,**gameid 写死 12**(=卜克);`cli_task_complete_node` [#L148](../../boke/online/src/cli_proto.cpp#L148) 是完整一笔任务结算(见后端到端示例)。

**玩家对象 Csprite**:[boke/online/src/Csprite.h](../../boke/online/src/Csprite.h)
- 一个在线玩家 = 一个 `Csprite`,聚合了 `item_list`(背包,[#L70](../../boke/online/src/Csprite.h#L70))、`task_list`(任务,[#L118](../../boke/online/src/Csprite.h#L118))、`spec_item_list`(每日限量物品/游戏积分)、`game_stat_map`(对战胜负统计)、`findmap_map`(已找到地图)、`cache_buffer_lst`(命令排队缓冲)。
- 关键内联工具:`get_onine_mapid()` [#L211](../../boke/online/src/Csprite.h#L211) `=(islandid<<16)+mapid`,把"岛+图"压成一个 32 位 key,作为 `Csprite_map::online_map` 的同屏分组键;`send_succ`/`send_err`/`send_*_server_noti` 一组 [#L226](../../boke/online/src/Csprite.h#L226) 区分"对请求的响应"与"主动系统通知"(后者带显式 cmdid)。
- `user_flag_*` [#L152](../../boke/online/src/Csprite.h#L152):`reg/newid/sample/guide` 四个位标志,落在 t_user.flag 里。

**同屏广播 / 进出图**:[boke/online/src/Csprite.cpp](../../boke/online/src/Csprite.cpp)
- `Csprite_map::cd_map()` [#L448](../../boke/online/src/Csprite.cpp#L448):换图(change-map)核心——从旧 `online_map[old_mapid]` 移除并广播 `cli_noti_user_left_map_cmd`,再插入新图、向同图其他人广播 `cli_noti_one_user_info_cmd`(我进来了)。
- `noti_to_map_others`/`noti_to_map_all`:遍历同 `online_mapid` 的玩家集合逐个发包,这就是"多人同屏移动/聊天/升级"的广播引擎(被 move/talk/level_up/seat 复用)。

### 2. switch —— 网关 / 在线表 / 选服

[boke/switch/src/switch.cpp](../../boke/switch/src/switch.cpp) + [boke/switch/src/online.cpp](../../boke/switch/src/online.cpp)
- 它**不是命令转发代理**,而是 **online 服注册中心 + 选服决策器**。每台 online 启动时 `connect_to_switch` 上报自己(IP/端口/在线人数),switch 存入 `s_online_info_[]` 数组([online.cpp#L102](../../boke/switch/src/online.cpp#L102) `sw_report_online_info` 填 `domain_id/online_id`)。
- `send_recommend_svr()` [#L148](../../boke/switch/src/online.cpp#L148):**负载均衡选服算法**——遍历所有 online,跳过 `user_num>45` 的满服 [#L168](../../boke/switch/src/online.cpp#L168),最多挑 10 台返回客户端;若玩家有"上次所在服"(last_online_id)优先把它放进列表首位(回到老朋友所在服)。
- `sw_get_recommend_svr_list()` [#L188](../../boke/switch/src/online.cpp#L188):**顶号逻辑**——若该 userid 已在某 online 在线,先 `sw_set_user_offline_cmd` 把旧连接踢下线("在其它地方登入了"),再下发服列表。
- `dispatch()` [boke/switch/src/dispatcher.cpp#L45](../../boke/switch/src/dispatcher.cpp#L45):用 `Ccmd_map` 按 cmd 查处理函数,`read_from_buf` 反序列化,**还校验"客户端是否多传了报文"**(`in_ba.is_end()`)防注入。

### 3. login —— 登录鉴权(通用框架 + 卜克实现)

- [boke/login/src/login.cpp](../../boke/login/src/login.cpp) 头注释自称"**通用的登录服务器框架代码,不许更改**"——它只是把 async_serv 钩子转调 `login` 对象。注意 `proc_pkg_from_client` [#L85](../../boke/login/src/login.cpp#L85) 里 `cmd==1` 的版本响应会把 `../conf/tm_dirty.dat` 的 md5 一并打进版本串(客户端据此判断要不要重新下脏词库)。
- 真正实现在 [boke/login/src/login_impl.cpp](../../boke/login/src/login_impl.cpp),继承自 libtaomee++ 的 `main_login` 框架:
  - `get_recommeded_svr_list_cmd` [#L143](../../boke/login/src/login_impl.cpp#L143):把客户端给的 hex session 转 bin,发 dbproxy 做 `check_session`;**dbproxy 不可用时降级到本地 `verify_login_session`** [#L155](../../boke/login/src/login_impl.cpp#L155)——容灾兜底。
  - `update_backup_svrlist` [#L20](../../boke/login/src/login_impl.cpp#L20):每 60 秒主动向 switch 拉一次"备份服列表",缓存起来,登录高峰时直接下发,减少实时查询。
  - 环境判定同 online:看首网卡是否 `10.` 开头 [#L52](../../boke/login/src/login_impl.cpp#L52)。

### 4. chat —— 聊天审查服(行为式反刷屏 + 脏词库分发)

这是 boke 里**算法最有料**的进程,作者 Ian,2011-10。它服务 mole(ID=1)与卜克(ID=12)两款游戏,见 [boke/chat/conf/GameSvr.xml](../../boke/chat/conf/GameSvr.xml)。

- 启动 [boke/chat/src/chatcheck.cpp#L37](../../boke/chat/src/chatcheck.cpp#L37):`init_funcs()` 建函数表 → new `Cdirty_word`(从 MySQL 拉脏词)→ 读 GameSvr.xml 为每个 gameid new 一个 `CChatCheck`(各自独立的刷屏检测器+禁言器)。
- `dispatch()` [boke/chat/src/dispatcher.cpp#L257](../../boke/chat/src/dispatcher.cpp#L257):**`cmd - 60001` 当数组下标**查 `funcs[50000]`([#L247](../../boke/chat/src/dispatcher.cpp#L247))。四个服务:
  - `adm_check_msg`(0xF133) → `check_msg` [#L82](../../boke/chat/src/dispatcher.cpp#L82):带 gameid/onlineid/mapid 的完整审查;`recvid>50000` 视为私聊**直接放行不审**(只管公屏)。
  - `adm_check_msg_common`(65003) → `check_msg_common` [#L147](../../boke/chat/src/dispatcher.cpp#L147):精简版。
  - `adm_get_dirty_md5` / `adm_get_dirty_words`:**把脏词库推给游戏服**(下文)。
- **脏词库 = MySQL 表,文件是兜底**:[boke/chat/src/dirty_word.cpp](../../boke/chat/src/dirty_word.cpp)
  - `Cdirty_word::reload()` [#L50](../../boke/chat/src/dirty_word.cpp#L50):跑 `select word, level from <db>.<table>` 把脏词拼成 `word\tlevel\n` 文本塞进 1MB `file_cache`,算 MD5 作为版本号;DB 挂了且首次启动则 `load_bak_file()` 读 [boke/chat/data/tm_dirty.dat](../../boke/chat/data/)([#L95](../../boke/chat/src/dirty_word.cpp#L95))。
  - `get_dirty_words` [boke/chat/src/dispatcher.cpp#L223](../../boke/chat/src/dispatcher.cpp#L223):游戏服来拉时,**按 4096 字节分片**连发,最后一片置 `endable=1`,带 MD5——这是一套"脏词库热更新下发协议"。游戏服先比 md5(`adm_get_dirty_md5`),变了才整库重拉。
- **mmseg 分词词库 `uni.lib`(8.6MB)**:在 `segment()` [boke/chat/src/dispatcher.cpp#L19](../../boke/chat/src/dispatcher.cpp#L19) 用——`g_seg_mgr.getSegmenter()` 把消息切词、查同义词(thesaurus),用于把变体广告归一。注意 `check_msg_common` 里 `segment()` 调用被注释掉了 [#L175](../../boke/chat/src/dispatcher.cpp#L175),说明线上主要靠下面的"行为式"判定,分词是实验/辅助路径。
- **白名单 `valid_words`**:[boke/chat/data/valid_words](../../boke/chat/data/valid_words) 是一批游戏内官方台词(拉姆学院/时空感应等),`check_msg` 命中即放行 [boke/chat/src/CChatCheck.cpp#L329](../../boke/chat/src/CChatCheck.cpp#L329),防误杀高频官方语。

### 5. db —— 落库服(分库分表 + 运维后门)

- [boke/db/src/service.cpp](../../boke/db/src/service.cpp):多进程模型(PROC_MAIN/WORK/CONN),每个 WORK 进程持一条 MySQL 连接 + 一个 `Croute_func`([#L33](../../boke/db/src/service.cpp#L33))。注意启动 banner 还写着 `"DuDu cur VERSION"`([#L29](../../boke/db/src/service.cpp#L29))——又一处 fork 残留(DuDu/嘟嘟另一淘米产品)。
- **建表是脚本现生成,不是 .sql 文件**:[boke/db/src/db.py](../../boke/db/src/db.py) `for i in range(100)` 打出 `POP_00..99` 100 个库,每库 10 张分表 `t_user_0..9` 等。t_user 字段即玩家存档全集(userid/flag/register_time/sex/age/nick/last_islandid/last_mapid/last_x/last_y/last_login,[#L12](../../boke/db/src/db.py#L12))。
- **分片路由**:`Cuser` 继承 `CtableRoute100x10(db,"POP","t_user","userid")`([boke/db/src/Cuser.cpp#L19](../../boke/db/src/Cuser.cpp#L19))——库名/表名由 libtaomee++ 的 `CtableRoute100x10` 按 userid 算(100 库 × 10 表 = 1000 分片),业务代码只调 `get_table_name(userid)` 拼 SQL([#L26](../../boke/db/src/Cuser.cpp#L26))。
- **路由分发 + 后门**:[boke/db/src/Croute_func.cpp](../../boke/db/src/Croute_func.cpp)
  - `pop_login` [#L54](../../boke/db/src/Croute_func.cpp#L54):一次登录把玩家的 user/task/item/day/find_map/user_log/game 七张表**一次性全读出来**打包回 online——这就是进游戏时"加载存档"的真相;`USER_ID_NOFIND_ERR` 即自动 insert 新号。
  - `pop_opt` [#L279](../../boke/db/src/Croute_func.cpp#L279):按 `opt_groupid` 分支(1物品/2任务/3地图/4每日/5…),**997/998 是运维后门**——直接 `system("cd ~/boke/online/bin&&./startup.sh")` 重启 online、拉 XML([#L314](../../boke/db/src/Croute_func.cpp#L314)),999 一键初始化。内网信任年代写法,严禁暴露公网。

### 6. gen_proto —— XML→C++/PHP/Python 多语言协议生成器

这是整个 boke(及所有淘米游戏)的**协议单一真源**机制,在 [boke/online/gen_proto/](../../boke/online/gen_proto/)。

- **真源是 XML**:[boke/online/gen_proto/xml/pop_online.xml](../../boke/online/gen_proto/xml/pop_online.xml) 用 `<struct>/<field mode=single|array|varlist|... type=uint32|char|item_t...>` 声明每个协议结构体([#L4](../../boke/online/gen_proto/xml/pop_online.xml#L4) 起)。`struct_from="pub"` 的是跨协议公共结构,`"self"` 是本协议私有。
- **怎么用**(实际运维流程):[boke/online/gen_proto/getxml.sh](../../boke/online/gen_proto/getxml.sh)
  1. 先把旧 xml 备份到 `bak/`;
  2. **`wget http://10.1.1.5/su/proto_test/download.php?project=pop_online`** 从内网协议管理网站下最新 xml([#L25](../../boke/online/gen_proto/getxml.sh#L25))——协议改动在网页工具上做,代码库只是消费方;
  3. 对每个 project 跑三个编译器:`cpp_proto`(出 .h/.cpp/enum/bind/func_def)、`php_proto`(出 php/)、`py_proto`(出 python/),并可挂插件(`gen_online_cli.py`/`gen_online_db.py` 生成 cli 请求绑定与 db 同步代码);
  4. 把生成的一长串 `cplist` 文件 `cp` 进 `../src/proto/`——这就是为什么 `src/proto/` 与 `gen_proto/proto/` 内容**逐字节重复**(改协议只能改 xml 再重生成,不能手改 .cpp)。
- **fork 血缘的铁证**:getxml.sh 里大量被注释的 `get_deal_xml seer2_btl/seer2_db/seer2_online`([#L41](../../boke/online/gen_proto/getxml.sh#L41) 附近)与拷往 `~/seer2_db`、`~/seer2_switch`、`~/seer2_btl` 的死代码块——同一套生成器、同一台开发机,曾用于赛尔号2(seer2),卜克直接沿用。
- 生成产物的语言用途:**C++** 给五个服务端进程编解码;**PHP** [boke/online/gen_proto/php/](../../boke/online/gen_proto/php/) 给网站/运营后台读同一协议;**Python** [boke/online/gen_proto/python/](../../boke/online/gen_proto/python/) 给 `tools/cli`、`tools/onlinei` 等压测/调试客户端。一份 XML,四端一致,杜绝协议漂移。

---

## 关键机制/算法实战

### A. chat 的行为式反刷屏(不是关键词黑名单!)

精髓在 [boke/chat/src/CChatCheck.cpp](../../boke/chat/src/CChatCheck.cpp):它判定"一句话是不是广告/刷屏",**靠的是这句话被多少不同人重复发、正负反馈比例**,而非匹配脏词。

`check_msg()` [#L306](../../boke/chat/src/CChatCheck.cpp#L306) 实际流程:
1. **太短放行**:`strlen<=6` 直接 true [#L315](../../boke/chat/src/CChatCheck.cpp#L315),避免"哦/嗯/好的"被误判为广告。
2. `trim_abnormal_utf8` 清异常字节 → `BKDRHash` 算 64 位 msgid [#L323](../../boke/chat/src/CChatCheck.cpp#L323);命中 `valid_words` 白名单放行。
3. 把消息挂进三级数据结构:`CMsgMetaMap`(按 msgid 聚合,记 `all_count/positive_cnt/negative_cnt/msglv`)、`CMsgUserMap`(按 userid 聚合 `userlv`)、主时间队列。
4. **动态升降级**:`CMsgMetaMap::update()` [#L92](../../boke/chat/src/CChatCheck.cpp#L92) 用公式 `ratio = negative/positive - 2 + log(all_count/4)` 决定这条消息的"可疑等级" msglv(0~3);一句话被越多人复读、负反馈越多,等级越高。
5. **判定**:首条消息记正分放行;老用户调 `check_user_msg` 给出 change_level,据此 `try_change_level`;**曾被禁言又说可疑话** → 直接 false [#L364](../../boke/chat/src/CChatCheck.cpp#L364)。
6. 返回 false 时,`dispatcher` 端调 `chat_forbid.add_forbid_user` [boke/chat/src/dispatcher.cpp#L130](../../boke/chat/src/dispatcher.cpp#L130),禁言 **300 秒**([boke/chat/src/CChatForbid.cpp#L28](../../boke/chat/src/CChatForbid.cpp#L28)),并 `send_pkg_forbid_user` 用 `adm_forbid_chat` 把禁言反推给游戏服网关,挂定时器到期自动解禁。

消息缓存有上限(`max_msg_count`),满了走 LRU 淘汰最老消息 [#L116](../../boke/chat/src/CChatCheck.cpp#L116);`max_msglv=4` 是四级优先队列。这是一套**自学习、零配置**的广告识别器——同一句新广告刚出现时放行,被越来越多小号复读后自动升级拦截。

### B. 两层脏词:在线即时 vs 离线累积

- **第一层(online 进程内,同步即时)**:`cli_talk`/`cli_set_nick` 调 `tm_dirty_check(7, msg)`([boke/online/src/cli_proto.cpp#L842](../../boke/online/src/cli_proto.cpp#L842)),命中即回 `cli_dirty_word_err` 当场拒发。词库就是 chat 服推下来的 tm_dirty.dat。
- **第二层(chat 进程,异步累积)**:消息正常发出去后,online 通过 UDP 把聊天上报给 chat 服做行为式分析(A),慢半拍但能抓"分开看每句都干净、合起来是刷屏"的广告。
- 两层职责清晰:即时层挡显式脏字,累积层挡变体广告与刷屏。

### C. 100×10 分库分表路由

userid 经 `CtableRoute100x10` 映射到 `POP_<userid 决定的库>` 的 `t_user_<表>`。业务侧零感知——`Cuser::get_all_info` 只写 `get_table_name(userid)`([boke/db/src/Cuser.cpp#L26](../../boke/db/src/Cuser.cpp#L26)),所有 7 张业务表(user/task/item/find_map/day/user_log/game)同 userid 同库同表序号,保证单玩家所有数据落在同一物理分片,登录时一次性 `pop_login` 批量读出。

---

## 端到端调用流举例:完成一个任务节点

追 **"玩家在某岛交掉一张任务卡"** 这一真实玩法,从客户端到落库:

1. **客户端** 发 `cli_task_complete_node_cmd`(=1011,[boke/online/src/proto/pop_online_enum.h#L17](../../boke/online/src/proto/pop_online_enum.h#L17)),body 含 `islandid + task_nodeid`。
2. **online::dispatch** [boke/online/src/cli_proto.cpp#L57](../../boke/online/src/cli_proto.cpp#L57):过登录态闸门 → 若该玩家有命令在处理则入 `cache_buffer_lst` 排队 → 否则查表调 `cli_task_complete_node` [#L148](../../boke/online/src/cli_proto.cpp#L148)。
3. **服务端校验链**(全在内存里做,失败即 `send_err` 返回):
   - 任务节点存在? `g_task_map.get_task_node` [#L156](../../boke/online/src/cli_proto.cpp#L156)
   - 前置任务都完成? 遍历 `req_taskid_list` 查 `task_list.check_existed_node` [#L165](../../boke/online/src/cli_proto.cpp#L165)
   - 所需物品够? 遍历 `req_item_list` 查 `item_list.check_enough` [#L173](../../boke/online/src/cli_proto.cpp#L173)
   - 末节点要加钱? `check_change_xiaomee(TASK_XIAOMEE_VALUE)` 防超上限 [#L184](../../boke/online/src/cli_proto.cpp#L184)
4. **内存先行 + 攒一个 db 事务包**:`task_list.add` 进内存 [#L192](../../boke/online/src/cli_proto.cpp#L192),回收物品 `item_list.del`、奖励物品 `item_list.add`、末节点加卜克豆 `change_xiaomee`,全部累积进一个 `pop_exchange_in db_in`(del_item_list/add_item_list),每张新卡 +10 经验 [#L229](../../boke/online/src/cli_proto.cpp#L229)。
5. **一次性发 db**:`send_msg_to_db(p, pop_exchange_cmd, &db_in)`(db_proto_t / TCP),seq 里编进 `(fd<<16)|waitcmd` 以便回包路由。
6. **db::Croute_func::pop_exchange** [boke/db/src/Croute_func.cpp#L118](../../boke/db/src/Croute_func.cpp#L118):按 userid 路由到 `POP_xx.t_item_y`,逐条 update/insert/delete,落库;特殊 itemid==2 走 `change_xiaomee`(卜克豆是用户表字段而非物品)。
7. **回包**:db 返回 → online `handle_db_return` → `p->send_succ()` 给客户端;同图广播视情况推 `cli_noti_get_item_list_cmd` 等。

**核心设计**:内存为权威态、即时响应玩家;DB 写是"事后批量同步"(攒成一个 exchange 包),崩溃风险换吞吐——这是淘米所有页游存档的统一范式。

---

## 复活实操提示(对 moleworld 复活)

boke 是 **mole2 的同源换皮**,对复活摩尔庄园是"高价值参照、少量直用":

**能直接借鉴/复用的:**
- **协议字典**:[pop_online_enum.h](../../boke/online/src/proto/pop_online_enum.h) 把每个 cmdid 的中文语义+请求/广播配对(`cli_xxx` 对 `cli_noti_xxx`)写得很全,是理解摩尔客户端↔服务端交互的活字典。
- **存档表结构**:[db/src/db.py](../../boke/db/src/db.py) 的 t_user/t_task/t_item/t_find_map 七表字段可直接当 MoleWorld-offline 本地存档(SQLite/JSON)的字段蓝本;`last_islandid/last_mapid/last_x/last_y` 就是"登出位置"。
- **同屏广播模型**:`online_map[(islandid<<16)+mapid]` 分组 + `noti_to_map_others` 遍历发包([Csprite.cpp#L448](../../boke/online/src/Csprite.cpp#L448)),离线/局域网版可照搬这套"同图玩家集合"做法。
- **反刷屏算法**:CChatCheck 的 BKDRHash+正负反馈升降级是一套自洽、可移植的反广告引擎,若做联机版聊天可直接抄。
- **脏词/分词数据**:[chat/data/](../../boke/chat/data/) 的 tm_dirty*.dat、uni.lib 是通用资源(非卜克专有),可直接用于聊天过滤。

**要自己补的:**
- **缺淘米内部框架**:async_serv、libtaomee/libtaomee++(仓库根有 `Libtaomee-0.7.5.tar.gz`/`Libtaomee++-0.7.2.tar.gz`,但 `CtableRoute100x10`、`main_login`、mysql_iface、mmseg 等需从中找齐)、`dbser`——直接编译必缺头缺库。
- **协议生成器闭环断了**:getxml.sh 依赖内网 `10.1.1.5` 的协议网站,现已不可达;要改协议只能直接改已生成的 .h/.cpp(失去单一真源保障)。
- **内容是卜克世界观**:conf 下 NPC 对话/任务文本是"土豆岛/卜尼亚海",**不可当摩尔内容**,只能作格式样本;摩尔自己的内容去 mole/mole2 模块取。
- **后门别带上线**:[Croute_func.cpp#L314](../../boke/db/src/Croute_func.cpp#L314) 的 `system()` 重启路径(opt_groupid 997/998)严禁暴露公网。

---

返回总入口:[../../README.md](../../README.md) · 概览:[../modules/boke.md](../modules/boke.md) · 协议专题:[../protocol.md](../protocol.md) · 内容数据专题:[../content-data.md](../content-data.md)
