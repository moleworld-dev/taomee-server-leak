# dbser 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/dbser.md](../modules/dbser.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)

本文不复述"dbser 是什么"(概览已讲),而是把它**当成一段在 dbsvr 进程里真实运行的代码**逐层拆开:一个报文从网卡进来,经过 cmd_id 查表 → 频率/长度校验 → 调用处理函数 → 拼 SQL → 选库选表 → 落 MySQL → (可选)写缓存/同步老库 → 组包回去,每一步分别由哪个文件、哪个函数完成。

---

## 一图看懂:进程/组件拓扑与数据流

dbser 自己**不是进程**,它是被各游戏 dbsvr 静态链接的库。把它放进一台真实运行的 dbsvr 里,数据流如下:

```
                    ┌──────────────────────────────────────────────────────────┐
   gamesvr ──TCP──▶ │  dbsvr 进程(链接 libdbser + libtaomee 的网络框架)       │
  (摩尔/功夫/…)     │                                                            │
                    │  1. 收到 recvbuf,头是 PROTO_HEADER(proto_header.h#L32)   │
                    │           │ cmd_id, id(米米号)                            │
                    │           ▼                                                │
                    │  2. Croute_func::deal()  ← Cfunc_route_base.h#L51          │
                    │       ├ cmdmaplist.getitem(cmd_id)  O(1) 查表(65536 vec)  │
                    │       ├ exec_cmd_limit.add_count()  每分钟频率闸          │
                    │       ├ check_proto_size()          报文定长/变长校验      │
                    │       ├ do_sync_data()              在线分裂(扩容期)      │
                    │       ├ (this->*p_deal_fun)(...)    业务处理函数(各游戏写)│
                    │       └ mysql_commit()              每条命令一个事务        │
                    │           │                                                │
                    │           ▼ 业务函数内部                                   │
                    │  3. CtableRoute::get_table_name(id)  ← CtableRoute.cpp#L46  │
                    │       选库 id%100、选表 (id/100)%100 → "USER_07.t_pet_03"   │
                    │           │                                                │
                    │           ▼                                                │
                    │  4. mysql_interface::exec_query/update_sql ← mysql_iface.cpp│
                    │           │                              ↘ 可走 Cbig_cache  │
                    └───────────┼──────────────────────────────┼────────────────┘
                                ▼                              ▼
                         MySQL 集群(分 100 库×100 表)   big_cache 缓存进程(TCP)
```

关键事实:
- **每个 cmd_id 既是"调用哪个函数"的钥匙,也是"落哪个业务库"的钥匙**(见下文 GET_ROUTE)。
- **一条命令 = 一个 MySQL 事务**:`deal()` 跑完业务函数立刻 `mysql_commit`([Cfunc_route_base.h#L84](../../dbser/Cfunc_route_base.h#L84)),连接建立时关掉了 autocommit([mysql_iface.cpp#L76](../../dbser/mysql_iface.cpp#L76))。

---

## 逐部分拆解

### 1) 报文头 `PROTO_HEADER` —— 所有数据流的入口结构

[proto_header.h#L32-L42](../../dbser/proto_header.h#L32) 定义了全平台通用的 20 字节定长头(packed):`proto_length / proto_id(序列号原样回) / cmd_id / result / id(米米号)`。功夫派编译时打开 `GF_PROTO_HEADER` 宏会多出一个 `role_tm` 字段([#L38](../../dbser/proto_header.h#L38)),所以**功夫派的头比其它游戏长 4 字节** —— 这是抓包时区分 gf 协议的硬特征。

这个头同时是一组"组包/拆包"内联工具的载体:
- `set_std_return()`([#L94](../../dbser/proto_header.h#L94))—— 业务函数返回时 `malloc` 出 `头+私有数据` 的回包,把请求头原样 `memcpy` 回去再改 `proto_length/result`。下游所有 `STD_RETURN_*` 宏都靠它。
- 一组 `RECVBUF_*` / `SNDBUF_PRIVATE_POS` 宏([#L55-L67](../../dbser/proto_header.h#L55))—— 让业务代码用 `RECVBUF_USERID`、`PRI_IN_POS` 这种名字直接取头字段和私有数据区,不用手算偏移。
- `#define NEED_UPDATE 0x0100`([#L50](../../dbser/proto_header.h#L50))—— cmd_id 里的一个 bit,标记"这条命令会改库,要存交易报文"(给重放/审计用)。

**通信对象**:gamesvr 发来的就是这个头,dbsvr 回去的也是这个头。要解析任何淘米服间协议,这里是第 0 步。

### 2) `Ccmdmaplist` —— 65536 槽的 O(1) 命令表

[Ccmdmaplist.h#L86-L90](../../dbser/Ccmdmaplist.h#L86):构造时 `v_cmdmap.reserve(65536)` 再 `memset` 清零,直接拿 **cmd_id 当 vector 下标**。`getitem(cmd_id)`([#L92](../../dbser/Ccmdmaplist.h#L92))判断 `cmd_id!=0` 就返回槽里的私有结构指针。因为 cmd_id 是 16 位,整个命令空间被一次性铺平成数组,查表是纯下标访问,没有哈希、没有树。代价是每个 dbsvr 实例固定吃掉 `65536 × sizeof(槽)` 内存。

注意 `set()` 用 `memcpy(this,&amap,sizeof(...))`([#L42](../../dbser/Ccmdmaplist.h#L42))整块拷贝——典型 2007 风格 POD 操作。

### 3) `pri_stru_check_len` + `stru_exec_cmd_limit` —— 每条命令的"门禁规则"

[Ccmdmap_private_checklen.h#L54-L77](../../dbser/Ccmdmap_private_checklen.h#L54)是塞进上面每个槽的私有结构,装着四样东西:
- `check_type`(`MUST_EQ`/`MUST_GE`,[#L33](../../dbser/Ccmdmap_private_checklen.h#L33))+ `predefine_len` —— 报文长度规则。定长命令必须严格等于,变长命令(带可变列表)用 `>=`。`check_proto_size()`([#L65](../../dbser/Ccmdmap_private_checklen.h#L65))就是这套判断。
- `p_deal_fun` —— 指向业务处理函数的成员函数指针。
- `exec_cmd_limit`([#L34-L52](../../dbser/Ccmdmap_private_checklen.h#L34))—— 每分钟频率限制。`add_count()` 是个**滑动整分钟计数器**:`time(NULL) > start_time+60` 就重置归零,否则累加,超过 `limit_max_count_per_min` 就拒绝。`limit==0` 表示不限。这是淘米对单条命令(如"领奖""购买")做防刷的第一道闸,默认关,需调 `set_cmd_max_limit_per_minute()`([Cfunc_route_cmd.h#L181](../../dbser/Cfunc_route_cmd.h#L181))显式打开。

### 4) `Cfunc_route_cmd` / `Cfunc_route_base` —— 命令分发引擎(被各 gamesvr 继承的核心)

这是 dbser 被复用的主入口。各游戏写一个 `Croute_func` 类继承它,用 `FILL_CMD_WITH_PRI_IN(xxx)` 这组宏([Cfunc_route_cmd.h#L36-L50](../../dbser/Cfunc_route_cmd.h#L36))声明一张 `{cmd_id, {长度规则, 函数指针, 频率}}` 静态表,再 `initlist()`([#L171](../../dbser/Cfunc_route_cmd.h#L171))把它灌进 cmdmaplist。`initlist` 里有 `assert(槽.cmd_id==0)`,**同一个 cmd_id 注册两次会直接 assert 崩**——这是它强约束"一个命令号只能有一个处理函数"的方式。

运行时核心是 `deal()`。两个版本:
- [Cfunc_route_cmd.h#L192](../../dbser/Cfunc_route_cmd.h#L192) 基础版:查表 → 频率闸 → 长度闸 → `(this->*p_deal_fun)(recvbuf,sendbuf,sndlen)` 调业务函数 → 返回。
- [Cfunc_route_base.h#L51](../../dbser/Cfunc_route_base.h#L51) 派生版:在基础版之上**多两步**——若开了 `SYNC_USER_DATA_FLAG` 配置先 `do_sync_data()`([#L76](../../dbser/Cfunc_route_base.h#L76))(在线扩容搬数据),业务函数跑完后**强制 `mysql_commit`**([#L84](../../dbser/Cfunc_route_base.h#L84))。所以"一命令一事务"语义在这层落地。

业务函数签名统一是 `int(char *recvbuf, char **sendbuf, int *sndlen)`([#L28/L31](../../dbser/Cfunc_route_cmd.h#L28)),返回值就是写进回包 `result` 的错误码。函数体几乎都是"取私有数据 → 调 CtableRoute → 用 STD_RETURN 宏组包"三段式。

`Cfunc_route_cmd_v2.h` / `Cfunc_route_db_v2.h` 是后期演进版本(同一套思路的迭代),老模块仍用 v1。

**通信对象**:上接 gamesvr(收 recvbuf),下接 CtableRoute / mysql_interface。

### 5) `mysql_interface` —— 真正碰 MySQL C API 的唯一一层

[mysql_iface.cpp](../../dbser/mysql_iface.cpp) 把 libmysqlclient 包成一个对象。值得注意的运行行为:
- 连接用 `CLIENT_FOUND_ROWS`([#L70](../../dbser/mysql_iface.cpp#L70)),让 UPDATE 的 affected_rows 返回**匹配行数而非实际改动行数**——上层 `exec_update_sql` 靠 `acount==1` 判断"这条用户记录存在"([Ctable.cpp#L75](../../dbser/Ctable.cpp#L75)),没这个 flag 逻辑会错。
- **断线自动重连**:`execsql()` 检测到 `CR_SERVER_GONE_ERROR` 会 `connect_server()` 后重发一次([mysql_iface.cpp#L130-L137](../../dbser/mysql_iface.cpp#L130))。
- **测试环境护栏**:`is_test_env` 下 `select_db()` 会校验"你只能选某个固定库"且"两次 select 的库名必须一致"([#L151-L166](../../dbser/mysql_iface.cpp#L151)),防止测试代码误打生产分库。`is_only_exec_select_sql` 可把实例设成只读([#L101](../../dbser/mysql_iface.cpp#L101))。
- SQL 错误统一进 `show_error_log`([#L46](../../dbser/mysql_iface.cpp#L46))打 errno+msg+原 SQL。

### 6) `Ctable` —— 单库单表的 CRUD 原语

[Ctable.cpp](../../dbser/Ctable.cpp) 是不分库时的基类。把 mysql_interface 的裸调用收敛成带语义错误码的方法:`exec_insert_sql` 把 `ER_DUP_ENTRY` 翻成 `existed_err`([#L104](../../dbser/Ctable.cpp#L104))、`exec_update_sql` 把"影响 0 行"翻成 `nofind_err`([#L77](../../dbser/Ctable.cpp#L77))、`exec_insert_sql_get_auto_increment_id` 取 `mysql_insert_id`([#L53](../../dbser/Ctable.cpp#L53))。这套"DB 原生错误 → 业务错误码"的翻译,是上层能用 `db_error_base.h` 统一码的基础。

### 7) `CtableRoute` 家族 —— 分库分表的落地

`CtableRoute` 继承 `Ctable`,核心就一个 `get_table_name(id)`([CtableRoute.cpp#L46-L56](../../dbser/CtableRoute.cpp#L46)):
```c
sprintf(db_name, "%s_%02d", db_name_pre, id%100);              // 选库
db->select_db(db_name);                                         // 切库
sprintf(db_table_name, "%s_%02d.%s_%02d",
        db_name_pre, id%100, table_name_pre, (id/100)%100);    // 选表
```
**同一米米号永远落在 `PREFIX_<id%100>.t_xxx_<(id/100)%100>`**——这是淘米全部用户数据物理分布的钥匙。注意它在拼表名时**顺手 `select_db` 切了当前连接的默认库**,所以业务函数里"先 get_table_name 再执行 SQL"的顺序不能乱。

在此之上提供了一批"对某用户某字段"的通用读写,业务代码几乎不用手写 SQL:
- `get/set_int_value`([#L81/L90](../../dbser/CtableRoute.cpp#L81))、带二级 key 版([#L193/L202](../../dbser/CtableRoute.cpp#L193))。
- `set_int_value_bit`([#L97](../../dbser/CtableRoute.cpp#L97))—— 按位 `|`/`&` 设/清某个 bit(flag 类字段,如"是否领过奖")。
- `change_int_value`([#L115](../../dbser/CtableRoute.cpp#L115))—— **带上下限的增减**:先读后算,越界时按 `is_change_to_max_min` 要么报 `VALUE_NOENOUGH_E`/`VALUE_MAX_E`,要么夹到 0/max 并回传"实际变化量"。游戏里加减摩尔豆/经验/体力都走这个。
- `get_insert_sql_by_userid`([#L229](../../dbser/CtableRoute.cpp#L229))—— 把某用户某表整行 dump 成 `delete+insert` SQL 文本(可换 userid),是搬数据/换号的底层工具。

`CtableRoute10/100/10x10/100x10/100x1` 是不同分片粒度的变体(10 库、100 库、库×表二维等),`CtableWithKey`/`CtableString` 加二级 key 和字符串字段,`CtableDate/CtableDate_100/CtableMonth` 按日/按月分表(时间序列日志用)。

### 8) `db_macro.h` —— 把 MySQL 结果集变成 C 结构体的宏工厂

[db_macro.h](../../dbser/db_macro.h) 是业务函数"不写裸 MySQL API"的秘密:
- `STD_QUERY_ONE_BEGIN/END`([#L204](../../dbser/db_macro.h#L204))—— 期望恰好一行,行数 ≠1 直接返回 `no_finderr`;行内用 `INT_CPY_NEXT_FIELD`/`DOUBLE_CPY_NEXT_FIELD`([#L81/L84](../../dbser/db_macro.h#L81))按列顺序 `row[++_fi]` 取值灌进结构体。
- `STD_QUERY_WHILE_BEGIN/END`([#L150/L124/L92](../../dbser/db_macro.h#L150))—— 取列表,有三种编译开关:`malloc` 数组 / 复用 `dbser_return_buf` 不 malloc(`DB_GET_LIST_NO_MALLOC`,带 `PROTO_MAX_SIZE` 溢出保护)/ `push_back` 进 STL 容器(`DB_GET_LIST_NEW`)。
- `set_mysql_string`([#L26](../../dbser/db_macro.h#L26))包 `mysql_real_escape_string`,`atoi_safe/atof_safe` 处理 NULL 列。
- 时间工具 `get_date`(返回 `YYYYMMDD` 整数,[#L247](../../dbser/db_macro.h#L247))、`get_year_month`、`hash_str`(经典 `h=31*h+c`,[#L262](../../dbser/db_macro.h#L262)),按日/月分表和散列就靠它。

### 9) `Cbig_cache` —— 可选的外置 SQL 缓存客户端

[Cbig_cache.h#L47](../../dbser/Cbig_cache.h#L47) 继承 `Ctcp`,通过 TCP 连一个独立的 `big_cache` 进程,自带二进制协议 `proto_head_t`(`len/ver/cmd/key/err/row`,[#L39](../../dbser/Cbig_cache.h#L39))。运行逻辑:
- 构造时 host_port 为空就 `use_cache=false`([#L86](../../dbser/Cbig_cache.h#L86)),即缓存可热关。
- 发请求时看全局 `g_db_cache_state.transaction_open_flag`:**在事务中把 cmd 设成 2,等于命令缓存"丢掉这条 key 的数据"**([#L58-L62](../../dbser/Cbig_cache.h#L58));不在事务中 cmd=1 正常走。这是"写穿透时让缓存失效"的实现。
- `exec_insert_sql_get_auto_increment_id`([#L129](../../dbser/Cbig_cache.h#L129))按返回包长度(头 / 头+4)判断是否带回自增 id;`get_db_err`([#L29](../../dbser/Cbig_cache.h#L29))把缓存侧 1~99 的错误码统一翻成 `DB_ERR`。

### 10) `Citem_change_log` —— 道具变动流水(按用户+按天分表)

[Citem_change_log.cpp](../../dbser/Citem_change_log.cpp) 是 `CtableRoute` 的一个具体子类,记录每一次道具增减,是经济系统审计/回溯的数据来源。
- `opt_type` 只有三种([Citem_change_log.h#L24](../../dbser/Citem_change_log.h#L24)):0=消耗、1=获得、2=米币购买获得。
- `add(...,bool is_vip_opt, ..., int count)`([Citem_change_log.cpp#L88](../../dbser/Citem_change_log.cpp#L88))把"是否米币渠道 + count 正负"翻译成上面三种枚举,负 count 自动转正存储。
- **表名按天滚动**:`get_table_name(userid,logtime)` 拼出 `<db>.t_item_change_log_<YYYYMMDD>`([#L67](../../dbser/Citem_change_log.cpp#L67))。
- **自建表兜底**:`add()` 先盲插,插失败(表当天还没建)就 `create_table` 现建 innodb 表再重插([#L51-L54](../../dbser/Citem_change_log.cpp#L51)、建表 SQL [#L77](../../dbser/Citem_change_log.cpp#L77))。这种"insert 失败→建表→重试"是 dbser 处理按日分表的标准套路,无需预建表脚本。

### 11) `db_error_base.h` —— 全平台统一错误码

[db_error_base.h](../../dbser/db_error_base.h) 是写进回包 `result` 的码表:`SUCC=0`、`DB_ERR=1002`、`PROTO_LEN_ERR=1005`、`CMD_EXEC_MAX_PER_MINUTE_ERR=1018`(频率超限)、`CHECK_PASSWD_ERR=1103`(登录密码错)、`USER_ID_EXISTED_ERR=1104`/`USER_ID_NOFIND_ERR=1105`(米米号已存在/不存在)、`LIST_ID_*`(好友/黑名单列表)。gamesvr 拿到回包就靠这张表判错,所以它是协议契约的一部分。

---

## 关键机制实战:把"路由信息"编码进 cmd_id 本身

这是 dbser 最精彩、也最值得逐 enum 解读的设计。一个 16 位 cmd_id 同时承载"调哪个函数"和"落哪个业务库"两重含义。

### 解码公式

[proxy_route.h#L25](../../dbser/proxy_route.h#L25)(`db_macro.h#L24` 有同一份):
```c
#define GET_ROUTE(cmdid) ((cmdid&0x8000)?(70+((cmdid&0x7E00)>>6)+((cmdid&0x00E0)>>5)):(cmdid>>9))
```
按注释([#L22-L24](../../dbser/proxy_route.h#L22)),它是两套并存的编码:
- **最高 bit=0**(`cmdid<0x8000`):`cmdid>>9` → 取高 7 位当库号(可表示 512 个库,每库低 9 位 = 512 个命令号空间)。
- **最高 bit=1**(`cmdid>=0x8000`):换一套密集编码 `70 + ((cmdid&0x7E00)>>6) + ((cmdid&0x00E0)>>5)`,在 0x8000 之后再细分(64 个库 × 每库 32 命令号)。这就是为什么 0x8000 以下的 ROUTE 间隔是 0x0200,而 0xB400 之后出现 0xB420/0xB440 这种 0x20 间隔。

`get_db_type(cmdid)`([#L26](../../dbser/proxy_route.h#L26))把结果转成 `enum_route_type`。dbsvr 前置的 proxy/router 就是用它把每个报文**按 cmd_id 路由到对应业务库的 dbsvr 实例**,无需任何配置表。

### 逐 enum 解读这张"分库地图"

`enum enum_route_type`([proxy_route.h#L29-L108](../../dbser/proxy_route.h#L29))把命令空间切成一张全平台分库地图。按游戏归类(注释为源码原文,中文经 GBK 复核):

**基础设施 / 跨游戏(0x0000–0x0E00 段)**
- `ROUTE_USERINFO=GET_ROUTE(0x0000)`([#L31](../../dbser/proxy_route.h#L31))用户基础信息库
- `ROUTE_SESSION=GET_ROUTE(0x0A00)`([#L36](../../dbser/proxy_route.h#L36))base.session server(会话)
- `ROUTE_REG_USERID=GET_ROUTE(0x0C00)`([#L37](../../dbser/proxy_route.h#L37))base.注册米米号(全平台账号注册)
- `ROUTE_EMAIL=GET_ROUTE(0x4000)`([#L58](../../dbser/proxy_route.h#L58))email→米米号映射
- `ROUTE_USER_LOGIN=GET_ROUTE(0x3C00)`([#L56](../../dbser/proxy_route.h#L56))用户登入信息
- `ROUTE_VIP=GET_ROUTE(0x2000)`([#L48](../../dbser/proxy_route.h#L48))会员系统
- `ROUTE_PRODUCE=GET_ROUTE(0x2400)`([#L50](../../dbser/proxy_route.h#L50))产品管理
- `ROUTE_ADMIN=GET_ROUTE(0x5400)`([#L63](../../dbser/proxy_route.h#L63))整体权限管理

**小花仙(注意:与功夫派是两回事)**
- `ROUTE_XHX=GET_ROUTE(0x0200)`([#L32](../../dbser/proxy_route.h#L32))小花仙
- `ROUTE_XHXOTHER=GET_ROUTE(0x0400)`([#L33](../../dbser/proxy_route.h#L33))小花仙.other

**功夫派(gf,注释写"功夫")**
- `ROUTE_GF=GET_ROUTE(0x0600)`([#L34](../../dbser/proxy_route.h#L34))功夫
- `ROUTE_GF_OTHER=GET_ROUTE(0x0E00)`([#L38](../../dbser/proxy_route.h#L38))功夫_OTHER

**摩尔庄园 2 / 派派 / DUDU**
- `ROUTE_MOLE2=GET_ROUTE(0x0800)`([#L35](../../dbser/proxy_route.h#L35))MOLE2
- `ROUTE_USER_2=GET_ROUTE(0x3000)`([#L53](../../dbser/proxy_route.h#L53))mole.用户数据-摩尔庄园2
- `ROUTE_MOLE2_RAND_ITEM=GET_ROUTE(0xD420)`([#L97](../../dbser/proxy_route.h#L97))
- `ROUTE_PAIPAI=GET_ROUTE(0x1800)` / `ROUTE_PAIPAI_2=GET_ROUTE(0x1200)`([#L43/L41](../../dbser/proxy_route.h#L43))派派
- `ROUTE_PP_EMAIL=GET_ROUTE(0x1A00)`([#L44](../../dbser/proxy_route.h#L44))派派明信片、`ROUTE_PPTASKDAY=GET_ROUTE(0x3800)`([#L55](../../dbser/proxy_route.h#L55))派派每日每周任务、`ROUTE_PPOTHER=GET_ROUTE(0x4800)`([#L60](../../dbser/proxy_route.h#L60))、`ROUTE_PP_PK_TEAM=GET_ROUTE(0x5800)`([#L64](../../dbser/proxy_route.h#L64))派派战队
- `ROUTE_DD=GET_ROUTE(0x1C00)`([#L45](../../dbser/proxy_route.h#L45))DUDU

**摩尔庄园(mole)—— 占了整张表的大半,正是本仓库主菜**
- `ROUTE_USER=GET_ROUTE(0x1000)`([#L40](../../dbser/proxy_route.h#L40))mole.用户数据-摩尔庄园(主用户库)
- `ROUTE_HOPE=GET_ROUTE(0x1400)`([#L42](../../dbser/proxy_route.h#L42))许愿池
- `ROUTE_MOLE_DINING=GET_ROUTE(0x2200)`([#L49](../../dbser/proxy_route.h#L49))餐厅
- `ROUTE_BAOBAO=GET_ROUTE(0x2800)`([#L51](../../dbser/proxy_route.h#L51))mole 宝宝
- `ROUTE_GROUPMAIN=GET_ROUTE(0x3400)`([#L54](../../dbser/proxy_route.h#L54))产生群号、`ROUTE_GROUP=GET_ROUTE(0xD400)`([#L96](../../dbser/proxy_route.h#L96))群组
- `ROUTE_MMS=GET_ROUTE(0x4400)`([#L59](../../dbser/proxy_route.h#L59))毛毛树/金蘑菇向导
- `ROUTE_GAME_SCORE=GET_ROUTE(0x5000)`([#L62](../../dbser/proxy_route.h#L62))游戏排名、`ROUTE_GAME=GET_ROUTE(0x8000)`([#L73](../../dbser/proxy_route.h#L73))用户游戏积分
- `ROUTE_SERIAL=GET_ROUTE(0x6000)`([#L66](../../dbser/proxy_route.h#L66))神奇密码(序列号兑换)
- `ROUTE_PARTY=GET_ROUTE(0x6400)`([#L67](../../dbser/proxy_route.h#L67))PARTY
- `ROUTE_MSGBOARD=GET_ROUTE(0x7000)`([#L69](../../dbser/proxy_route.h#L69))记者投稿、`ROUTE_USERMSG=GET_ROUTE(0x9000)`([#L74](../../dbser/proxy_route.h#L74))用户投稿
- `ROUTE_MOLE_NOAH=GET_ROUTE(0x7200)`([#L70](../../dbser/proxy_route.h#L70))诺亚舟、`ROUTE_PICTURE=GET_ROUTE(0x7400)`([#L71](../../dbser/proxy_route.h#L71))图片
- `ROUTE_SU=GET_ROUTE(0xA000)`([#L76](../../dbser/proxy_route.h#L76))客服系统/定时信息
- `ROUTE_ROOM/_2/_4/_6/_8`(0xB400 起,0x20 步进,[#L78-L82](../../dbser/proxy_route.h#L78))小屋信息——**这里开始进入"最高 bit=1"的密集编码区**,同一逻辑库被切成多个物理分片
- `ROUTE_SYSARG/_2/_4/_6`(0xC000 起,[#L84-L87](../../dbser/proxy_route.h#L84))系统参数
- `ROUTE_ROOMMSG/_2`(0xC400,[#L89](../../dbser/proxy_route.h#L89))小屋留言
- `ROUTE_TEMP/_2/_4`(0xD000,[#L92-L94](../../dbser/proxy_route.h#L92))**活动临时库,活动结束即删**
- `ROUTE_EMAIL_SYS=GET_ROUTE(0xE000)`([#L99](../../dbser/proxy_route.h#L99))邮件系统

**IM / 即时通讯**
- `ROUTE_IM_USER_CACHE/IM_ONLINE_INFO/IM_USER/IM_USER_2`(0xF020 起,[#L103-L106](../../dbser/proxy_route.h#L103))IM 用户缓存/在线信息/用户库

**特殊**
- `ROUTE_NULL=-1`([#L30](../../dbser/proxy_route.h#L30))、`ROUTE_NOROUTE=GET_ROUTE(0xF000)`([#L101](../../dbser/proxy_route.h#L101))不走路由

> 这张表的价值:它就是淘米 2007–2013 全部网页游戏的"数据库目录"。复活摩尔庄园时,所有 `ROUTE_*//mole.*` 的条目都对应一个你需要重建的逻辑库,cmd_id 区段就是该库命令号的取值范围。

---

## CMake 骨架实际编出什么

[CMakeLists.txt](../../dbser/CMakeLists.txt) 行为很直白:
- `find_package(glib/mysql/taomee/taomee++)`([#L7-L10](../../dbser/CMakeLists.txt#L7))—— 四个依赖,后两个是仓库根的 `Libtaomee-0.7.5.tar.gz` / `Libtaomee++-0.7.2.tar.gz` 提供的私有框架。
- `ADD_LIBRARY(dbser ${SRC_LIST})`([#L61](../../dbser/CMakeLists.txt#L61))—— 编出**一个静态库** `libdbser`,只编 15 个 `.cpp`(`SRC_LIST` [#L11-L27](../../dbser/CMakeLists.txt#L11))。注意分发框架(`Cfunc_route_*`、`Ccmdmap*`、`Cbig_cache`、宏头)全是**纯头文件、不在 SRC_LIST**——它们靠模板/内联,在各 gamesvr **编译期**展开,这就是 dbser 能被多游戏复用的根本:库只装"碰 MySQL 的实现",分发/路由逻辑随头文件注入到每个 gamesvr 自己的二进制里。
- `ADD_DEFINITIONS(-g -DDEBUG -Wall -funroll-loops -fPIC ...)`([#L62](../../dbser/CMakeLists.txt#L62))—— 带 `-DDEBUG`,所以那些 `DEBUG_LOG` 在编出来的库里默认是开的。
- `INSTALL(TARGETS dbser DESTINATION lib)` + `INSTALL(FILES ${HEADER_LIST} DESTINATION include/dbser/)`([#L63-L64](../../dbser/CMakeLists.txt#L63))—— 装到 `/usr/lib/libdbser.a` 和 `/usr/include/dbser/*.h`。各 gamesvr 的构建脚本就是 `find_package` 这个 + `#include <dbser/...>`。

**它怎么被各 gamesvr 复用**(总结):gamesvr → 链 `libdbser.a` → `#include <dbser/Cfunc_route_base.h>` 等头 → 写自己的 `Croute_func` 子类 + cmd 表 → 运行时 `deal()` 驱动。所有游戏共享同一套头/事务/路由语义,只各自填业务函数和表名前缀。

---

## 端到端调用流举例(mole):一次"喂宠物加饱食度"的落库

假设摩尔庄园客户端给宠物喂食,触发一条改"饱食度"字段的命令(cmd_id 落在 `ROUTE_USER` 摩尔用户库区段 0x1000~):

1. **客户端→gamesvr→dbsvr**:封成 `PROTO_HEADER{cmd_id, id=米米号, ...}` + 私有数据(itemid 等),TCP 发到 mole 用户库 dbsvr。proxy 用 `GET_ROUTE(cmd_id)` 得 `ROUTE_USER`,投递到对应库实例。
2. **deal() 入口**([Cfunc_route_base.h#L51](../../dbser/Cfunc_route_base.h#L51)):`cmdmaplist.getitem(cmd_id)` O(1) 取到该命令的规则;`add_count()` 过频率闸;`check_proto_size()` 过长度闸(喂食是定长 `MUST_EQ`)。
3. **业务函数**(在 mole 模块的 `Croute_func` 子类里,非本模块):取私有数据里的目标字段,调 `CtableRoute::change_int_value(userid, "fullness", +delta, MAX, ...)`([CtableRoute.cpp#L115](../../dbser/CtableRoute.cpp#L115))。
4. **选库选表**:`change_int_value` 内部先 `get_int_value` → `get_table_name(userid)`([CtableRoute.cpp#L46](../../dbser/CtableRoute.cpp#L46))拼出 `USER_<userid%100>.t_pet_<(userid/100)%100>` 并 `select_db`;算出新值不越界后 `set_int_value` 发 `UPDATE`。
5. **执行 SQL**:`exec_update_sql`→`mysql_interface::execsql`([mysql_iface.cpp#L118](../../dbser/mysql_iface.cpp#L118))真正 `mysql_query`;`affected_rows==1` 才算成功,否则翻成 `USER_ID_NOFIND_ERR`。若配了 big_cache,写会同时让缓存该 key 失效([Cbig_cache.h#L58](../../dbser/Cbig_cache.h#L58))。
6. **(可选)流水**:经济相关操作会再调 `Citem_change_log::add()`([Citem_change_log.cpp#L88](../../dbser/Citem_change_log.cpp#L88))往当天的 `t_item_change_log_<YYYYMMDD>` 插一条审计记录。
7. **提交 + 组包回**:业务函数 `STD_RETURN`/`STD_RETURN_WITH_STRUCT`([Cfunc_route_cmd.h#L60/L111](../../dbser/Cfunc_route_cmd.h#L60))用 `set_std_return` 把头原样回、填新 `result` 和私有返回;`deal()` 收尾 `mysql_commit`([Cfunc_route_base.h#L84](../../dbser/Cfunc_route_base.h#L84))。dbsvr 把回包发回 gamesvr→客户端。

整条链:**cmd_id 决定库 + 函数 → userid 决定物理表 → 一次 UPDATE + 一条流水 + 一个 commit**。这就是摩尔庄园里每一个"数值变化"在服务端的真实生命周期。

---

## 复活实操提示(对 moleworld 复活)

**能直用的**:
- [proto_header.h](../../dbser/proto_header.h) 与 [db_error_base.h](../../dbser/db_error_base.h) 是纯头、无重依赖,直接 include 进自研 server 就能保证回包结构与 `result` 错误码和原版一致——抓包对照时极有用。
- `GET_ROUTE` 公式 + 整张 `proxy_route.h` enum:做协议解析器时,拿 cmd_id 就能反推"这条属于哪个子系统",不用猜。

**要自己补的**:
- 整套 `Cfunc_route_*`/`Ctable*`/`mysql_iface` 绑死 `libtaomee`/`libtaomee++`/老版 MySQL C API/glib,基本编不动(仓库里的 `Makefile`/`CMakeCache.txt` 是别人机器的产物,路径已失效)。建议**只借设计、重写实现**:用现代语言重做"cmd_id→handler 表 + 长度/频率校验 + 一命令一事务"这套骨架。
- 单机/离线场景**可以把分库压成单库**:令 `id%100→0`、`(id/100)%100→0`,`get_table_name` 退化成固定库表名,省掉整套 100×100 路由,数据兼容性不受影响(只是物理分布变了)。
- big_cache 是独立缓存进程,复活时可整段省略(`host_port` 传空即 `use_cache=false`,[Cbig_cache.h#L86](../../dbser/Cbig_cache.h#L86)),先跑通直连 MySQL。
- 按日分表的 `Citem_change_log` 这类:复活初期可合成单表,但若要还原"道具流水"玩法/客服回溯,保留 `opt_type` 三态(消耗/获得/米币购买)语义即可。

---

(以上路由含义均取自 `proxy_route.h` 源码注释,GBK 中文已用 iconv/grep -a 复核;命令号→具体玩法的精确映射需交叉对应各游戏模块,本文已标注属于推断之处。)
回总入口:[../../README.md](../../README.md)
