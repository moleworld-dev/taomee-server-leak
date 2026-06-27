# 模块 dbser · 淘米全平台共用的 MySQL 数据库访问 / 分库分表路由库

> 价值:★★  ·  游戏:基础设施(被摩尔庄园/功夫派/赛尔号等所有 gamesvr 复用)  ·  源码量:约 0.4 万行(cpp/c/h 实测 3972 行,54 个文件)
> 📖 深度功能解析(每个部分实际在干什么):[deep/dbser.md](../deep/dbser.md) · 彩蛋见 [easter-eggs.md](../easter-eggs.md)

## 是什么 / 在整套架构里的角色

dbser **不是一个独立进程**,而是一个用 CMake 构建、安装成 `libdbser` 静态库 + 头文件(`include/dbser/`)的 **C++ 数据库访问框架库**。它把"如何把游戏请求落到 MySQL"的通用逻辑抽象出来,供各游戏的 dbsvr / gamesvr 进程链接复用。

证据:
- [CMakeLists.txt](../../dbser/CMakeLists.txt#L61) `ADD_LIBRARY(dbser ...)`、[L63-L64](../../dbser/CMakeLists.txt#L63) `INSTALL(TARGETS dbser DESTINATION lib)` 与 `INSTALL(FILES ... DESTINATION include/dbser/)` —— 产物是库不是可执行文件。
- 依赖链:[CMakeLists.txt#L7-L10](../../dbser/CMakeLists.txt#L7) `find_package(glib/mysql/taomee/taomee++)`,即它建立在内部框架库 [libtaomee.md](./libtaomee.md) 之上、再往下接 MySQL C API。
- 作者头注释一致为 **Author: xcwen / jim@taomee.com,Company: TAOMEE**,创建时间从 **2007 年 11 月**(`proto_header.h`、`Ctable.h`)延续到 **2010–2011 年**(`db_macro.h`、`Cbig_cache.h`、`Citem_change_log.h`),说明它是淘米从摩尔庄园起家就在用、并持续迭代的"地基"代码。

在整套架构里的分层(推测,基于代码形态):

```
Flash 客户端 → gamesvr(各游戏逻辑) → dbsvr(链接 libdbser) → MySQL 集群(分 100 库 × 100 表)
                                          ↘ Cbig_cache(可选的 SQL 缓存代理进程)
```

dbser 提供三件核心能力:
1. **协议头与命令分发**(`proto_header.h` / `Cfunc_route_*` / `Ccmdmap*`)——定义 `PROTO_HEADER`、按 `cmd_id` 查表调用处理函数、做频率限制和报文长度校验。
2. **分库分表路由**(`proxy_route.h` / `CtableRoute*`)——按 userid(米米号)做 `id%100` 选库、`(id/100)%100` 选表;并用 `cmd_id` 高位 bit 解码出"这条命令属于哪个业务库"。
3. **SQL 执行与结果集→C 结构体的宏工厂**(`mysql_iface.*` / `db_macro.h`)——把"取一行 / 取列表 / escape / 事务"封装成宏,业务代码几乎不写裸 MySQL API。

## 关键子目录与文件

- [proto_header.h](../../dbser/proto_header.h#L32) —— 定义全平台通用报文头 `PROTO_HEADER`(proto_length / proto_id / cmd_id / result / id),及 `set_std_return` 等组包内联函数;`GF_PROTO_HEADER` 宏给功夫派多一个 `role_tm` 字段。
- [proxy_route.h](../../dbser/proxy_route.h#L29) —— **整套平台的"分库地图"**:用 `enum_route_type` 把每个 `cmd_id` 区段映射到一个业务库(用户库/小花仙/功夫/MOLE2/摩尔小屋/会员……),是全仓库最有信息量的单文件之一。
- [db_macro.h](../../dbser/db_macro.h#L24) —— 路由位运算 `GET_ROUTE`、`set_mysql_string`(escape)、`STD_QUERY_ONE_BEGIN/STD_QUERY_WHILE_BEGIN` 结果集遍历宏、日期/哈希工具;依赖 `libtaomee/conf_parser/config.h` 读配置。
- [mysql_iface.h](../../dbser/mysql_iface.h#L13) / [mysql_iface.cpp](../../dbser/mysql_iface.cpp) —— `mysql_interface` 类:封装连接、`select_db`、`exec_query_sql`、`exec_update_sql`、错误日志、测试环境开关。
- [Ctable.h](../../dbser/Ctable.h#L31) / [Ctable.cpp](../../dbser/Ctable.cpp) —— 单表基类:`select_data` / `exec_insert/update/delete_sql` / 取自增 id,带 `dbser_return_buf` 返回缓冲。
- [CtableRoute.h](../../dbser/CtableRoute.h#L29) / [CtableRoute.cpp](../../dbser/CtableRoute.cpp#L46) —— 分库分表基类:`get_table_name(id)` 生成 `USER_00.t_user_pet_05` 形式表名;提供 `set/get/change_int_value`、按位设 flag 等"对某用户某字段"的通用读写。
- [CtableRoute10.cpp](../../dbser/CtableRoute10.cpp) / [CtableRoute100.cpp](../../dbser/CtableRoute100.cpp) / [CtableRoute10x10.cpp](../../dbser/CtableRoute10x10.cpp) / [CtableRoute100x10.cpp](../../dbser/CtableRoute100x10.cpp) / [CtableRoute100x1.cpp](../../dbser/CtableRoute100x1.cpp) —— 不同分片粒度的路由变体(10 库、100 库、库×表二维等)。
- [CtableWithKey.cpp](../../dbser/CtableWithKey.cpp) / [CtableString.cpp](../../dbser/CtableString.cpp) —— 带二级 key、字符串字段的表封装。
- [CtableDate.cpp](../../dbser/CtableDate.cpp) / [CtableDate_100.cpp](../../dbser/CtableDate_100.cpp) / [CtableMonth.cpp](../../dbser/CtableMonth.cpp) —— 按日期/按月分表(日志、活动等时间序列数据用)。
- [Cfunc_route_cmd.h](../../dbser/Cfunc_route_cmd.h#L161) —— `Cfunc_route_cmd` 命令分发基类:`initlist` 把 `cmd_id→处理函数指针` 填进表,`deal()` 做频率/长度校验后调用对应 `Croute_func` 成员函数。
- [Cfunc_route_base.h](../../dbser/Cfunc_route_base.h#L31) —— 在 `Cfunc_route_cmd` 之上加 MySQL 事务 commit、在线数据同步(`Csync_user_data`)的派生基类。
- [Ccmdmaplist.h](../../dbser/Ccmdmaplist.h#L78) —— `Ccmdmaplist` 模板:用一个 `reserve(65536)` 的 vector 做 `cmd_id→私有结构` 的 O(1) 查表。
- [Ccmdmap_private_checklen.h](../../dbser/Ccmdmap_private_checklen.h) —— 每条命令的私有结构:期望长度、`MUST_EQ/MUST_GE` 校验模式、每分钟频率上限(`exec_cmd_limit`)。
- [Cfunc_route_cmd_v2.h](../../dbser/Cfunc_route_cmd_v2.h) / [Cfunc_route_db_v2.h](../../dbser/Cfunc_route_db_v2.h) —— 上述分发框架的 v2 版本(后期演进)。
- [Cbig_cache.h](../../dbser/Cbig_cache.h#L47) —— `Cbig_cache`:走 TCP 连一个外部 "big_cache" SQL 缓存进程的客户端;自带二进制协议 `proto_head_t`,事务中会发 `cmd=2` 让缓存清数据。
- [Csync_user_data.h](../../dbser/Csync_user_data.h) / [Csync_user_data.cpp](../../dbser/Csync_user_data.cpp) —— "在线分裂/同步用户数据":扩容换库时把老库数据搬到新库的逻辑。
- [Citem_change_log.h](../../dbser/Citem_change_log.h#L34) / [Citem_change_log.cpp](../../dbser/Citem_change_log.cpp) —— 道具变动流水:按 userid+时间分表记录"消耗/获得/米币购买"(`item_change_log_opt_type`)。
- [db_error_base.h](../../dbser/db_error_base.h#L22) —— 全平台统一错误码(`SUCC=0`、`DB_ERR=1002`、`USER_ID_NOFIND_ERR=1105`……),gamesvr 也靠它判错。
- [proxy_dll_interface.h](../../dbser/proxy_dll_interface.h) / [benchapi.h](../../dbser/benchapi.h) —— 代理 DLL 接口约定 / 压测 API 声明。
- [cmake/](../../dbser/cmake/Findtaomee.cmake) —— `Findmysql/glib/taomee/taomee++` 四个查找脚本,反映构建依赖。

## 协议 / 数据亮点

### 报文头(全平台共享)

[proto_header.h#L32-L43](../../dbser/proto_header.h#L32) 的 `PROTO_HEADER` 是淘米所有游戏服间通信的统一头:

```c
uint32_t proto_length; // 报文总长度
uint32_t proto_id;     // 序列号,原样返回
uint16_t cmd_id;       // 命令号
int32_t  result;       // 返回值
userid_t id;           // 一般是米米号(userid_t = uint32_t)
```

要分析任何一个 gamesvr 的协议,这个头都是起点。详见专题 [../protocol.md](../protocol.md)。

### cmd_id 即路由:一行位运算解码业务库

dbser 最精彩的设计是**把"路由信息"编码进 cmd_id 本身**。见 [proxy_route.h#L25](../../dbser/proxy_route.h#L25) / [db_macro.h#L24](../../dbser/db_macro.h#L24):

```c
#define GET_ROUTE(cmdid) ((cmdid&0x8000)?(70+((cmdid&0x7E00)>>6)+((cmdid&0x00E0)>>5)):(cmdid>>9))
```

- 最高 bit=0:`cmdid>>9` → 6 位库标示(512 个库,每库 256 个命令号);
- 最高 bit=1:另一套编码(64 个库,每库 32 个命令号)。

由此得到的 `enum_route_type` 就是整张分库地图,真实命令字区段例:
- [proxy_route.h#L31](../../dbser/proxy_route.h#L31) `ROUTE_USERINFO = GET_ROUTE(0x0000)` —— 用户基础信息库
- [proxy_route.h#L40](../../dbser/proxy_route.h#L40) `ROUTE_USER = GET_ROUTE(0x1000)` —— **摩尔庄园**用户数据库
- [proxy_route.h#L38](../../dbser/proxy_route.h#L38) `ROUTE_GF = GET_ROUTE(0x0600)` —— **功夫(功夫派)**库
- 另有 `ROUTE_MOLE2 / ROUTE_MOLE_DINING(餐厅)/ ROUTE_ROOM(小屋)/ ROUTE_SERIAL(神奇密码)/ ROUTE_PARTY` 等大量摩尔庄园子系统(L35–L99)。

> 关于游戏名:本文件里 `ROUTE_XHX` 注释为"小花仙",`ROUTE_GF` 注释为"功夫"。结合 [gf.md](./gf.md) 的实证(功夫豆/木人/铜人/battle),**gf 的真实游戏名是"功夫派"(KungFu Pai)**;请注意早期 baseline 曾把 `gf` 误标为"小花仙"——小花仙在本 enum 中是另一个独立区段 `ROUTE_XHX`,与功夫派不是一回事,此处一并修正。

### 分库分表落地公式

[CtableRoute.cpp#L46-L55](../../dbser/CtableRoute.cpp#L46) `get_table_name(id)`:

```c
sprintf(db_name, "%s_%02d", db_name_pre, id%100);                       // 选库:id%100
sprintf(db_table_name, "%s_%02d.%s_%02d",
        db_name_pre, id%100, table_name_pre, (id/100)%100);             // 选表:(id/100)%100
```

即同一个 userid 的数据稳定落在 `PREFIX_<id%100>.t_xxx_<(id/100)%100>`,这是理解淘米所有用户数据物理分布的钥匙。

### 表与错误码

- 真实表名前缀散见于各 `CtableRoute` 子类构造,如道具流水表 `t_item_change_log`([Citem_change_log.h#L44](../../dbser/Citem_change_log.h#L44))。
- 统一错误码见 [db_error_base.h](../../dbser/db_error_base.h#L22)(`USER_ID_EXISTED_ERR=1104`、`LIST_ID_MAX=1115` 等),返回到 `PROTO_HEADER.result`。
- 本模块**没有 .sql 建表脚本、没有 conf XML、没有 README**(实测 `find` 为空):它只是访问层框架,真实表结构与配置在各业务 dbsvr 模块里。

## 对我们有什么用(moleworld-server / MoleWorld-offline)

定位:**强参照,弱直用**。dbser 是理解"摩尔庄园服务端如何存数据"的最佳总览,但作为代码很难直接拿来用(纯 2007 风格 C++ 宏 + libtaomee + MySQL C API + 100×100 分库)。

- **参照(高价值)**:
  - [proxy_route.h](../../dbser/proxy_route.h) 是复活摩尔庄园的"数据目录":告诉你用户数据、小屋、餐厅、Party、神奇密码、邮件等分别是独立库,以及它们的 cmd_id 区段——重建 server 时按这张表组织数据模型即可一一对应。
  - `cmd_id` 高位编码路由的设计、`PROTO_HEADER` 结构,直接用于解析摩尔庄园抓包/协议(配合 [../protocol.md](../protocol.md))。
  - `get_table_name` 的 `id%100 / (id/100)%100` 公式:离线版若想还原"哪个账号数据在哪",照此即可。
- **直用(有限)**:
  - [db_error_base.h](../../dbser/db_error_base.h) 错误码、[proto_header.h](../../dbser/proto_header.h) 组包/拆包内联函数,是纯头文件、无重依赖,可直接 include 到自研 moleworld-server 里保持协议返回值一致。
  - 离线版若仍打算用 MySQL,`mysql_interface` / `Ctable` 的接口形状值得照搬;但单机离线场景完全可以把分库压成单库(令 `id%100→0`),省去整套路由。

## 注意 / 坑

- **不是可执行进程**:dbser 编不出 server,只是库;真正的 dbsvr/gamesvr 进程在别的模块,需结合 [c.md](./c.md)、[libtaomee.md](./libtaomee.md) 与各游戏模块([mole](./mole.md) 等)一起看。
- **缺依赖,基本编不动**:依赖私有 `libtaomee`/`libtaomee++`(见 [CMakeLists.txt#L9-L10](../../dbser/CMakeLists.txt#L9))、old MySQL client、glib;仓库里 `Makefile`/`CMakeCache.txt`/`mtags` 是别人机器上 CMake 生成的产物,路径已失效,别指望直接 `make`。
- **年代局限**:大量 `sprintf` 拼 SQL(虽有 `mysql_real_escape_string` 包装,但调用点未必都走),`memcpy(this,&amap,...)` 这类 C 式写法、固定大小栈缓冲(`char sqlstr[8192*2]`)——做安全/移植参考可,直接上生产不可。
- **部分头注释错位**:不少 `.h` 的 Filename 字段写着别的名字(如 `proxy_route.h` 头写 `common.h`、`Cfunc_route_base.h` 与 `Cfunc_route_cmd.h` 都写 `func_route_base.h`),是早期复制模板没改,以实际文件名为准。
- **中文乱码**:`CtableRoute.h/.cpp`、`Ccmdmaplist.h` 等文件的中文注释是 GBK 存储,在 UTF-8 下显示为乱码(不影响代码语义)。
- 回总入口:[../../README.md](../../README.md)。

(以上路由含义均取自源码注释,个别业务区段语义为注释直译,如需精确请交叉核对对应游戏模块。)
