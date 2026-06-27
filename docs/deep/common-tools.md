# common-tools 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/common-tools.md](../modules/common-tools.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)
>
> 概览讲"是什么";本篇讲"开起来之后,每个进程在内存里、在网络上、在数据库里**实际发生了什么**",并把跨进程数据流追到落库为止。所有行号均真实 grep/Read 到。

---

## 一图看懂:进程/组件拓扑与数据流

common-tools 不是一个进程,而是一堆**旁路守护进程 + 离线脚本 + 库代码骨架**。最值得讲的是 anticheat 反作弊,它是一条由 5 个独立进程 + 4 块共享内存 + 2 段 UDP 串起来的流水线:

```
游戏 online/逻辑进程
   │  (玩家每发一个命令包, 复制一份"米米号+cmd+ip+收包时刻"上报)
   │  UDP -> if_addr (eth1:8889)
   ▼
[ interface ]  interface.c  ── 纯收包, 不做判定
   │  写入 shm-channel (key = SHM_KEY_ANTICHEAT_CHANNEL + gameid*1000)
   ▼  (共享内存环形队列, 零拷贝交接)
[ anticheat ]  anticheat.c  ── 核心检测器, 百万 uid hash 表 + 四级 single_watch
   │  判定违规 -> 组 ill_report 包, cmd=1001
   │  UDP -> db_agent_ip:db_agent_port (192.168.4.79:11201)
   ▼
[ db_interface ] db_interface.c ── 又一个纯收包进程, bind 11201
   │  写入 shm-channel (key = SHM_KEY_DB_INTERFACE_CHANNEL, 不分业务)
   ▼
[ db_agent ]  db_agent.c   ── 每 100µs 轮询 channel, 拼 INSERT
   │  MySQL  (库 anticheat, 表 <svc>_today_anticheat / <svc>_main_anticheat)
   ▼
[ query_ill ] PHP + Sphinx ── 运营后台查"谁开了加速器"
```

旁边还挂着两组独立工具(与 anticheat 无关):

```
聊天 server ──TCP(cmd 60001/60002)──> [ advert_fliter ] ──> 禁言判定 ──> dbproxy
客户端 ──TCP──> [ proxy / gameproxy ] ──按 game+线路 路由──> dbproxy / switch
运营 ──手工触发──> [ postcard_ser ] ──发系统明信片──> dbproxy
摩尔/赛尔/小花仙客户端 ──登录包──> [ main_login(库骨架) ] ──> dbproxy 校验
```

**关键设计动机**:为什么要 interface / anticheat / db_interface / db_agent 拆成 4 个进程而不是 1 个?因为**收包**(高频、不能阻塞)、**判定**(吃 CPU + 大块共享内存)、**落库**(会被 MySQL 阻塞)三件事对延迟容忍度完全不同。用共享内存环形队列把它们解耦,任何一段卡住都不会丢上游的包——这是当年扛"全游戏在线玩家每包都上报"这种流量的工程取舍。

---

## 逐部分拆解

### 1. anticheat/interface.c —— 上报收包口(只搬运,不思考)

入口 [interface.c#L432 `main`](../../common-tools/linux/anticheat/trunk/interface.c#L432):建 UDP socket([create_udp_socket #L79](../../common-tools/linux/anticheat/trunk/interface.c#L79),非阻塞 + reuseaddr + 放大收发缓冲),挂 libevent 事件循环。

实际干的事只有一件:[recv_req_from_client #L214](../../common-tools/linux/anticheat/trunk/interface.c#L214) 在一个 `while(1)` 里把 UDP 包一口气收干,每个包做两道**长度自检**(收到字节数 ≥ `svr_pkg_t` 头、且 == 包头里写的 `len`,[#L238](../../common-tools/linux/anticheat/trunk/interface.c#L238)、[#L248](../../common-tools/linux/anticheat/trunk/interface.c#L248)),然后把"收包来源地址 `recv_from`"贴在包前面([#L257-258](../../common-tools/linux/anticheat/trunk/interface.c#L257)),整包塞进共享内存通道 [add_to_channel_shm_mgr #L264](../../common-tools/linux/anticheat/trunk/interface.c#L264) 就完事。

通道 key 的算法很巧:[init_interface_channels #L117](../../common-tools/linux/anticheat/trunk/interface.c#L117) 用 `SHM_KEY_ANTICHEAT_CHANNEL + svc_gameid * 1000`——这样**摩尔(gameid=1)、赛尔(2)、功夫派(6)各自一条通道、互不串台**,同一台机器能并排跑多个游戏的反作弊而不撞 shmkey。配置由 [init_if_config #L128](../../common-tools/linux/anticheat/trunk/interface.c#L128) 用 libconfuse 读 `*.if.conf`(svc_name / svc_gameid / bind_ethx / bind_port)。

> 它**通信对象**:上游=游戏逻辑进程(UDP 收);下游=anticheat 进程(共享内存)。注意 `send_resp_to_client` 整段被 `#if 0` 掉([#L272](../../common-tools/linux/anticheat/trunk/interface.c#L272))——反作弊是**单向上报**,从不回包给游戏。

### 2. anticheat/anticheat.c —— 检测器主脑(本模块技术含金量最高的一块)

入口在 [anticheat.c](../../common-tools/linux/anticheat/trunk/anticheat.c#L1)。它启动时 attach 6 块共享内存:业务通道、百万 uid hash 表(`uid_node_hash_table`)、配置(`ac_config`)、以及**四级 single_watch 缓冲**([#L43-46](../../common-tools/linux/anticheat/trunk/anticheat.c#L43))。

每个在线玩家对应一个 [`uid_node_t`](../../common-tools/linux/anticheat/trunk/anticheat.h#L179),被精心填充到**正好 1KB**(末尾有 954 字节 `reserved` + `COMPILE_ASSERT` 强制断言 [#L199](../../common-tools/linux/anticheat/trunk/anticheat.h#L199))。注释直说:["1KB 一个 node,意味着 2GB 内存最多承担 200w 个在线结构"](../../common-tools/linux/anticheat/trunk/anticheat.h#L176)——这是按当年全游戏峰值在线量反推的内存预算。hash 表上限 `MAX_UID_NODE_NUM = 1000000`([anticheat.h#L51](../../common-tools/linux/anticheat/trunk/anticheat.h#L51))。

核心处理函数 [proc_one_client_request #L591](../../common-tools/linux/anticheat/trunk/anticheat.c#L591),逐包做:
1. 解出 `cmd/uid`(网络序转回),uid==0 直接丢([#L600](../../common-tools/linux/anticheat/trunk/anticheat.c#L600))。
2. 在 hash 表里找/建 uid_node([#L603](../../common-tools/linux/anticheat/trunk/anticheat.c#L603));新 uid 走"初始化路径"只记一笔不判定([#L617-625](../../common-tools/linux/anticheat/trunk/anticheat.c#L617))。
3. 对老 uid 跑**两套检测**:tw(total-watch,不分命令号的总体节奏)和 sw(single-watch,针对单个被监控命令号的连发)。

**它跟谁通信**:上游=interface(共享内存读);下游=db_interface(UDP 写,见下文 ill_report)。

### 3. anticheat/db_interface.c + db_agent.c —— 落库流水线

[db_interface.c#L432 `main`](../../common-tools/linux/anticheat/trunk/db_interface.c#L432) 又是一个 UDP 收包进程(和 interface.c 几乎同构),bind 在 `db_agent_port`(实证 mole 配为 **11201**),把 anticheat 发来的 ill_report 写进 [SHM_KEY_DB_INTERFACE_CHANNEL #L115](../../common-tools/linux/anticheat/trunk/db_interface.c#L115)。注意这条通道**不分业务**——所有游戏的违规都汇到一处再统一落库。

[db_agent.c](../../common-tools/linux/anticheat/trunk/db_agent.c#L1) 的 worker 定时器间隔被设成 [100µs #L349](../../common-tools/linux/anticheat/trunk/db_agent.c#L349)("使得每次主循环都能执行它"),每轮最多取 100 条([#L293](../../common-tools/linux/anticheat/trunk/db_agent.c#L293))。每条记录交给 [store_ill_record_to_db #L114](../../common-tools/linux/anticheat/trunk/db_agent.c#L114):**表名是动态拼出来的** `<my_dbname>.<svc_name>_<my_tab_basename>`([#L120](../../common-tools/linux/anticheat/trunk/db_agent.c#L120))——`svc_name` 由违规记录自带([struct.h#L83](../../common-tools/linux/anticheat/trunk/comm/struct.h#L83)),所以同一个 db_agent 能把摩尔的记录写进 `mole_today_anticheat`、赛尔的写进 `seer_today_anticheat`,一份代码服务全游戏。

> **安全坑(真实凭据泄漏)**:db_agent 默认 MySQL 口令在源码里写死为 `ta0mee`([db_agent.c#L408](../../common-tools/linux/anticheat/trunk/db_agent.c#L408));而 mole 的实际部署 conf 里更是明文留着真库口令 `my_user="db" / my_passwd="fsd23rg8@"`(`deploy/mole/db_agent/conf/mole.dbagt.conf`)。均为历史泄漏,已失效。

### 4. proxy / gameproxy —— 两套不同的"代理"

这两个名字像,其实分工不同:

- **proxy**([CProxyRoute.cpp](../../common-tools/linux/proxy/src/CProxyRoute.cpp#L1)):是**跨游戏的路由层**。[reload_svr_config #L128](../../common-tools/linux/proxy/src/CProxyRoute.cpp#L128) 逐行 `sscanf` 读 [route.conf](../../common-tools/linux/proxy/etc/route.conf) 的五列(`game 线路 服务 ip port`),把 `(game_flag, zone, svr_type)` 三维下标映射到一个 `m_svr[game][zone][svr]` 连接表([init_svr_info #L169](../../common-tools/linux/proxy/src/CProxyRoute.cpp#L169))。`zone` 只认 `tel`/`cnc`([#L202](../../common-tools/linux/proxy/src/CProxyRoute.cpp#L202))=电信/网通双线;`svr_type` 只认 `dbproxy`/`switch`([#L214](../../common-tools/linux/proxy/src/CProxyRoute.cpp#L214))。各游戏的具体业务包结构由 `CProxyDealMole/Seer/Gf/Xhx/Hero...` 一族派生类处理(见下文端到端流)。
- **gameproxy**([dll_interface.cpp](../../common-tools/linux/gameproxy/trunk/dll_interface.cpp#L1)):是 async_serv 框架的一个**可热插 .so 业务模块**,不是独立 main。[init_service #L25](../../common-tools/linux/gameproxy/trunk/dll_interface.cpp#L25) 从 `cliproto.conf` 读 `cliprotoheadsize` / `netbytesorder`,管理客户端连接(`ClientsMgr`)。它的存在意义在 [extend_proto.h](../../common-tools/linux/gameproxy/trunk/extend_proto.h#L29):`new_cli_proto_t` 把旧 `cli_proto_t` 的 `ret` 字段换成了 `seqno`,注释明写["used as an auto-increment sequence number for checking GameCheater"](../../common-tools/linux/gameproxy/trunk/extend_proto.h#L36)——客户端每包自增序号,服务端据此识别重放/连发外挂。常量 `cli_proto_cmd_max = 30001`、`cli_proto_extend_port = 999`([#L45](../../common-tools/linux/gameproxy/trunk/extend_proto.h#L45))。

### 5. advert_fliter —— 聊天反垃圾(贝叶斯 + mmseg 分词 + 升降级禁言)

入口路由表 [dispatcher.cpp#L138](../../common-tools/linux/advert_fliter/trunk/src/dispatcher.cpp#L138):`cmd 60001 -> check_msg`、`60002 -> check_msg_common`。聊天 server 把每条公聊转发过来;私聊(`recvid > 50000`)直接放行不查([#L39](../../common-tools/linux/advert_fliter/trunk/src/dispatcher.cpp#L39))。按 `gameid` 在 `game_chat_check` map 里取对应游戏的 [CChatCheck](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L1) 实例([#L44](../../common-tools/linux/advert_fliter/trunk/src/dispatcher.cpp#L44))——多游戏共用一个进程、各带各的词表/统计。

判定主逻辑 [CChatCheck::check_msg #L300](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L300),实际跑这些规则(顺序很有讲究):
1. **太短放行**:`strlen<=6` 直接 true,注释说怕把"哦、嗯"误判成广告([#L309](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L309))。
2. **清洗**:`trim_abnormal_utf8` 去掉夹杂的乱码字符([#L314](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L314))——对抗"广∮告"这类插字符绕过。
3. **整句哈希**:`BKDRHash` 算 msgid,纯 ASCII(哈希为 0)放行([#L317](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L317))。
4. **白名单**:msgid 命中 `valid_words` 立即放行([#L323](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L323))。
5. **贝叶斯分类**:`bayes->classify` 返回类别([#L341](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L341)),`class==0`(疑似广告)给当前玩家的"嫌疑等级"加分。Cbayes 启动时 `init_segment("../data/")` 加载 mmseg 中文分词词典、再 `train(stat_file)` 训练([#L293-296](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L293))。
6. **群体 + 个人双维升降级**:对同一条 msgid 维护 `meta_msg`(全服多少人说过这句、正/负样本计数),对玩家维护 `user_msg`(个人嫌疑等级)。综合 `check_user_msg` 的群体信号([#L357](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L357))+ 贝叶斯,调 `try_change_level` 升降级([#L367](../../common-tools/linux/advert_fliter/trunk/src/CChatCheck.cpp#L367))。

命中后,dispatcher 调 [chat_forbid.add_forbid_user #L61](../../common-tools/linux/advert_fliter/trunk/src/dispatcher.cpp#L61) 把玩家拉进禁言名单,并挂一个定时器到期自动解禁(`n_chat_forbid_pop`,[#L63](../../common-tools/linux/advert_fliter/trunk/src/dispatcher.cpp#L63))。接入游戏由 [GameSvr.xml](../../common-tools/linux/advert_fliter/trunk/conf/GameSvr.xml) 描述:摩尔(`ChatProtocol=62015`)、卜克(12)、赛尔2(`63103`),各带 `WordFile`(白名单)与 `StatFile`(贝叶斯统计)。

> 与 dbproxy 通信见 [svr_proto.h `send_msg_to_db` #L20](../../common-tools/linux/advert_fliter/trunk/src/svr_proto.h#L20):懒连接 dbproxy,封 `db_proto_t` 头(`seq = (fd<<16)|waitcmd`)再发——这套 seq 编码方式在淘米全线服务端通用。

### 6. main_login —— 摩尔/赛尔/小花仙共用的通用登录(库骨架)

它不是独立进程,而是 async_serv 的业务 .so 骨架:[login.cpp](../../common-tools/linux/main_login/login.cpp#L1) 是"框架代码,不许更改",真正的游戏差异写在 `login_impl.cpp`。框架把 async_serv 的回调 `proc_pkg_from_client` 转给 [Login::dispatch](../../common-tools/linux/main_login/login.cpp#L80)。

**最有复活价值的一处**:[cli_proto.hpp](../../common-tools/linux/main_login/cli_proto.hpp#L1) 把淘米三种客户端包头**一次性列全**了——这是对齐协议的硬证据:
- `CliProto1`(17 字节,`len/ver/cmd/uid/ret`,大端):["目前摩尔、赛尔和小花仙使用"](../../common-tools/linux/main_login/cli_proto.hpp#L36);
- `CliProto2`(18 字节,把 ver 去掉、加 `seqno` 防外挂):["摩尔勇士和功夫派使用"](../../common-tools/linux/main_login/cli_proto.hpp#L95);
- `CliProto3`(18 字节,**小端**解包):["seer2 在使用"](../../common-tools/linux/main_login/cli_proto.hpp#L154)。

登录命令分发在 [main_login.hpp 的 dispatch switch #L395](../../common-tools/linux/main_login/main_login.hpp#L395):`renew_verifimg_cmd`(刷新验证码图)/`login_cmd`/`check_active_cmd`/`set_active_code_cmd`。其中 [login_cmd #L680](../../common-tools/linux/main_login/main_login.hpp#L680) 的实际行为:
- uid==0 时按**邮箱登录**(包体多带 `email_max_len`),否则按**米米号登录**([#L686](../../common-tools/linux/main_login/main_login.hpp#L686));
- 解出 `channel / which_game / passwd(转 hex)/ 验证码`([#L705-713](../../common-tools/linux/main_login/main_login.hpp#L705));
- 校验 `which_game` 合法范围([#L717](../../common-tools/linux/main_login/main_login.hpp#L717))——印证"一份登录服务多游戏复用,靠 which_game 区分";
- 最终走 [db_login_with_verif_img #L740](../../common-tools/linux/main_login/main_login.hpp#L740),把校验请求发给 dbproxy(`dbproto_t` 头,[#L57](../../common-tools/linux/main_login/main_login.hpp#L57)),登录密码用 `req_verify_md5` 做 MD5 校验([#L512](../../common-tools/linux/main_login/main_login.hpp#L512))。

### 7. postcard_ser —— 系统明信片下发

每游戏一份(`mole_/seer_/gongfu_/fairy_/hero_sendcard`)。以 [mole_sendcard.cpp](../../common-tools/linux/postcard_ser/src/mole_sendcard.cpp#L1) 为例:[onProcPkgCli #L33](../../common-tools/linux/postcard_ser/src/mole_sendcard.cpp#L33) 只认一个命令 `0xC080`([#L46](../../common-tools/linux/postcard_ser/src/mole_sendcard.cpp#L46)),触发 [sendCard #L76](../../common-tools/linux/postcard_ser/src/mole_sendcard.cpp#L76)——带去重(超时/已发判定)后把明信片包通过 `m_proxy_fd` 发给 dbproxy 落到玩家信箱。卡片模板由 `load_cache_xml`([#L169](../../common-tools/linux/postcard_ser/src/mole_sendcard.cpp#L169))从 XML 加载。

### 8. proto/trunk —— PHP 协议字典(运维/测试构包用)

[mole_proto.php](../../common-tools/linux/proto/trunk/mole_proto.php#L1) 等是给运维脚本用的"客户端模拟器"基类:每个方法对应一条命令,用 PHP `pack()` 按字段格式拼包再 `sendmsg`。例如 [msgboard_add_qa #L16](../../common-tools/linux/proto/trunk/mole_proto.php#L16) 用 `pack("A12A16a213a361", ...)` 构造留言板加问答的请求(cmd `7120`),[msgboard_get_qa_list_ex #L23](../../common-tools/linux/proto/trunk/mole_proto.php#L23) 用 cmd `7023` 拉问答列表。**这是摩尔命令号 + 字段宽度的活字典**,对照协议时极有用。同目录还有 `gf_proto.php / xhx_proto.php / mole2_proto.php / pp_proto.php`。

### 9. gameid 字典(全模块通用的游戏编号硬证据)

[docs/gameid](../../common-tools/linux/anticheat/trunk/docs/gameid)(UTF-8,行号即 id):

| id | 游戏 | id | 游戏 |
|----|------|----|------|
| 1 | 摩尔(摩尔庄园) | 8 | 摩尔庄园2(摩尔大冒险) |
| 2 | 赛尔(赛尔号) | 9 | 校巴 |
| 3 | 2125 大玩国(已停用) | 10 | 赛尔2 |
| 4 | DUDU | 12 | POP(解密类,即卜克) |
| 5 | 小花仙 | 13 | 淘米小怪兽 |
| 6 | 功夫派 | 21 | 哈奇 |
| 7 | 摩尔勇士 | 32 | account, pay, service(基础设施) |

这张表是判断各模块归属哪款游戏、以及 anticheat `svc_gameid` / GameSvr.xml `ID` 取值的权威来源。

---

## 关键机制实战

### 机制 A:反作弊 tw/sw 双维判定算法(它到底怎么抓加速器)

anticheat 对**每个命令包**同时跑两条独立判定线,任一违规就上报:

**tw(total-watch,总体节奏)** —— 不分命令号,看玩家整体发包是否过快:
- *间隔检查* [total_cmd_check_interval #L507](../../common-tools/linux/anticheat/trunk/anticheat.c#L507):本包距上一包 `diff < min_tw_interval` → 置 `ILL_CODE_TW_INT`;
- *频率检查* [total_cmd_check_frequence #L519](../../common-tools/linux/anticheat/trunk/anticheat.c#L519):`freq = count / (now - start_tv) * 1e6`,超 `max_tw_freq` → 置 `ILL_CODE_TW_FREQ`。注释特意提醒[浮点数判断绝不能用 `>=`/`==` #L534](../../common-tools/linux/anticheat/trunk/anticheat.c#L534)。频率检查要攒够 `CHECK_TW_FREQ_MIN_COUNT=10` 包才启动([anticheat.h#L31](../../common-tools/linux/anticheat/trunk/anticheat.h#L31)),避免冷启动误判。

**sw(single-watch,单命令连发)** —— 只盯**配置里列入监控**的命令号([find_sw_config #L95](../../common-tools/linux/anticheat/trunk/anticheat.c#L95) 用**二分查找**,要求 sw_config 按 cmd 升序),逻辑同 tw 但阈值各命令独立([single_cmd_check_interval #L549](../../common-tools/linux/anticheat/trunk/anticheat.c#L549)、[single_cmd_check_frequence #L562](../../common-tools/linux/anticheat/trunk/anticheat.c#L562)),攒够 `CHECK_SW_FREQ_MIN_COUNT=3` 包启动。这是用来抓"无限连点某个采集/PK 按钮"类外挂的。

判定后([#L719-771](../../common-tools/linux/anticheat/trunk/anticheat.c#L719))把所有现场数据(uid、cmd、玩家 ip/port、上报机 ip/port、各间隔/频率/阈值/计数)塞进 `ill_report_rcd_t`([struct.h#L81](../../common-tools/linux/anticheat/trunk/comm/struct.h#L81)),`svr_pkg->cmd=1001`,UDP 发往 db_interface。**违规后会重置该维 start_tv 和 count**([#L773-783](../../common-tools/linux/anticheat/trunk/anticheat.c#L773)),让下一窗口重新计——所以一次加速会被持续记录而非只记一条。

### 机制 B:四级 single_watch 的内存分级(省内存的巧设计)

一个玩家可能只触发 1 个被监控命令,也可能触发几百个。若给每人都按最大 512 命令预留,内存爆炸。anticheat 的做法是**按需升级缓冲等级**:

- 四级容量 SPEC = 16 / 32 / 128 / 512([anticheat.h#L54](../../common-tools/linux/anticheat/trunk/anticheat.h#L54)),节点数预算 lvl0 留 50w、lvl1 留 10w、lvl2 留 2w、lvl3 留 1w([#L67-70](../../common-tools/linux/anticheat/trunk/anticheat.h#L67),注释直接算了每级总字节数);
- 玩家首次触发监控命令时拿一个 lvl0(16 槽)的 [single_watch_lvl0_t](../../common-tools/linux/anticheat/trunk/anticheat.h#L147),命令种类超 16 就 [upgrade_sw_cmd_list_lvl #L375](../../common-tools/linux/anticheat/trunk/anticheat.c#L375) 搬到 lvl1,以此类推;
- 空槽用 [get_sw_empty_node #L335](../../common-tools/linux/anticheat/trunk/anticheat.c#L335) 的 first-zero 算法分配。

这套"绝大多数人 16 槽就够、极少数升级"的分布,把百万在线的 sw 内存压到可控。anticheat.h 里还有一连串 `#error` 编译期断言([#L60](../../common-tools/linux/anticheat/trunk/anticheat.h#L60)、[#L78](../../common-tools/linux/anticheat/trunk/anticheat.h#L78))确保各级容量与上限不矛盾——非常严谨的系统级 C 代码。

---

## 端到端调用流举例(摩尔庄园:加好友请求一直追到落库)

以 proxy 处理摩尔"拉好友列表"为例,看 client→proxy→dbproxy→callback→client 全链路:

1. 客户端发摩尔某 cmd 到 proxy。proxy 按 `route.conf` 查到 `mole tel dbproxy 10.1.1.23:21001`([route.conf](../../common-tools/linux/proxy/etc/route.conf)),交给摩尔的处理器。
2. [CProxyDealMole::get_friends_list_cmd #L12](../../common-tools/linux/proxy/src/CProxyDealMole.cpp#L12):`init_db_proto_head(..., mole_dbproto_get_friends_list, ...)` 重新打一个 dbproxy 包头,转发给 dbproxy。
3. dbproxy 落库查询后回包。proxy 的 [handle_db_return #L99](../../common-tools/linux/proxy/src/CProxyDealMole.cpp#L99) 按 `pkg->cmd` 分派:[case mole_dbproto_get_friends_list #L121](../../common-tools/linux/proxy/src/CProxyDealMole.cpp#L121) → `dbproto_get_friends_list_callback` 把结果整理成客户端格式发回。
4. **同一玩家的每个包还会被游戏 online 进程复制一份上报给 anticheat 流水线**(机制 A),若这次拉好友是外挂高频刷的,就会在 `mole_today_anticheat` 表里留下一条违规记录,运营用 query_ill 后台能查到。

> 同一个 `CProxyDealMole` 还实现了 [get_user_info](../../common-tools/linux/proxy/src/CProxyDealMole.cpp#L30)、[send_mail](../../common-tools/linux/proxy/src/CProxyDealMole.cpp#L48)、[add_items](../../common-tools/linux/proxy/src/CProxyDealMole.cpp#L73) 等——这些方法的字段布局(如 `send_mail_t`、`add_items_t`+`add_item_info_t` 变长数组)就是摩尔对应玩法的真实协议结构,复活时可直接照抄。

---

## 复活实操提示(对 moleworld 复活)

**能直接用的:**
- **包头常量与字节布局**:[cli_proto.hpp](../../common-tools/linux/main_login/cli_proto.hpp#L36) 的 `CliProto1`=17 字节大端(摩尔正用)是事实标准,离线服的客户端编解码可直接对齐;`clipkg_max_size=8192`、`cli_proto_cmd_max=30001` 也是硬约束。
- **gameid=1**:摩尔的业务 id,凡是配置/分库/路由里要填游戏号的地方都用它。
- **摩尔玩法协议字段**:[CProxyDealMole.cpp](../../common-tools/linux/proxy/src/CProxyDealMole.cpp#L1)(好友/用户信息/邮件/加道具)+ [mole_sendcard.cpp](../../common-tools/linux/postcard_ser/src/mole_sendcard.cpp#L1)(明信片 cmd `0xC080`)+ [mole_proto.php](../../common-tools/linux/proto/trunk/mole_proto.php#L1)(留言板等 cmd 号 + `pack` 格式)三处互相印证,是离线实现这些功能时最省事的字段对照表。
- **登录流程参考**:[login_cmd #L680](../../common-tools/linux/main_login/main_login.hpp#L680) 给出"米米号/邮箱双模式、which_game 区分、密码 MD5、验证码"的完整登录交互骨架,离线服可裁剪掉验证码/多游戏分支后照搬主干。

**要自己补的:**
- 离线/单机复活**几乎用不到 anticheat 整条流水线**(interface/anticheat/db_interface/db_agent/query_ill)——它是为大规模线上防加速器设计的,单人服没有意义。仅作"官方当年怎么抓外挂"的设计参考。
- proxy 的 `tel/cnc` 双线路、`route.conf` 多游戏拓扑离线版可彻底砍掉,直连即可。
- advert_fliter 依赖外部 `data/` 词典 + 贝叶斯统计文件 + mmseg,且是多游戏运营级需求,离线复活无需复用。
- 所有 C/C++ 都强依赖 `libtaomee` / `libtaomee++` / `async_serv`(见 [c.md](../modules/c.md) 概览),脱离这套框架无法独立编译;真正可"拿来即用"的是**协议字段定义与常量**,而非可执行体。

> 注意:本模块多处 conf/脚本含当年真实内网 IP 与 MySQL 明文口令(如 db_agent 的 `ta0mee`、mole.dbagt.conf 的 `fsd23rg8@`、route.conf 的 10.1.1.* 段),均为历史泄漏已失效,仅作考据,切勿当作有效凭据。
>
> 回总入口 [../../README.md](../../README.md) · 协议汇总 [../protocol.md](../protocol.md) · 内容数据 [../content-data.md](../content-data.md)
