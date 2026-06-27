# 协议罗塞塔石碑(Protocol Rosetta Stone)

> 本文是淘米(Taomee)泄漏服务端源码的**协议总导读**。目标:把这套 2007–2013 年的网页 Flash 页游服务端协议讲清楚,便于与我们抓到的**摩尔庄园 iOS 手游协议**做字段/命令字对照。
>
> 返回总入口:[../README.md](../README.md) · 模块文档:[mole.md](./modules/mole.md) / [mole2.md](./modules/mole2.md) / [gf.md](./modules/gf.md) / [monster.md](./modules/monster.md) · 内容专题:[content-data.md](./content-data.md)

业务对照(据淘米自有 gameid 字典 [common-tools/.../docs/gameid](../common-tools/linux/anticheat/trunk/docs/gameid)):`mole` = 摩尔庄园(id 1),`mole2` = 摩尔庄园2(id 8),`gf` = 功夫派(id 6,**非小花仙**),`pea` = 赛尔号(id 2),`monster` = 淘米小怪兽(id 13,**非赛尔号**),`boke` = 卜克 / POP 解密类(id 12)。所有服务都是给 **Flash 网页客户端**用的。

---

## 0. 一句话总览

整套协议是**命令字(cmdid)驱动**的二进制 RPC:每个请求包头里带一个 16-bit 的命令字,后端按命令字分发到对应的处理函数。命令字本身是一个**复合位域**——高位 6 bit 编码"这个请求要路由到哪个 DB",`0x0100` 这一位编码"这个请求要不要写库"。理解了命令字的位编码,就理解了大半套协议。

核心字典文件是 [mole/dbsvr/include/proto.h](../mole/dbsvr/include/proto.h)(约 15003 行,约 1300 个 `_CMD` 命令字),它是摩尔庄园 DB 服务端的协议总表,也是本文的主角。

---

## 1. 命令字(cmdid)的位编码机制

### 1.1 基本形态

命令字用 `#define` 定义,值是 **16-bit 十六进制**字面量,经常 OR 上一两个标志位。典型形态(均为真实片段):

```c
// 客户端私有数据,获取  —— 纯读,不带任何标志
#define ROOMINFO_CLIENT_DATA_GET_CMD (0xB469)

// 客户端私有数据,设置  —— 写库,OR 上 NEED_UPDATE
#define ROOMINFO_CLIENT_DATA_SET_CMD (0xB46A | NEED_UPDATE)
```

见 [mole/dbsvr/include/proto.h#L32](../mole/dbsvr/include/proto.h#L32) 与 [#L35](../mole/dbsvr/include/proto.h#L35)。

### 1.2 两个关键宏(真实行号)

```c
//在cmdid 中数据库识别标志
#define RROTO_ROUTE_FIELD 				0xFC00 //使用前6个bit

//是否需要保存交易报文
//有涉及到更新数据库数据的协议，都要保存
#define NEED_UPDATE							0x0100
```

- `RROTO_ROUTE_FIELD = 0xFC00`(高 6 bit 作 DB 路由识别)—— 见 [mole/dbsvr/include/proto.h#L25](../mole/dbsvr/include/proto.h#L25)。
  注意宏名拼写就是源码里的 `RROTO`(疑似 `PROTO` 笔误,但全仓库一致沿用)。
- `NEED_UPDATE = 0x0100`(写库标志)—— 见 [mole/dbsvr/include/proto.h#L29](../mole/dbsvr/include/proto.h#L29)。

含义:
- **写库标志 `0x0100`**:凡涉及更新 DB 的协议都 OR 上它。服务端据此决定是否落盘、是否记录"交易报文"以便回放/审计。命令字名里通常带 `SET_` / `ADD_` / `DEL_` / `UPDATE_`,而 `GET_` 类几乎都不带。
- **路由位域 `0xFC00`**:取命令字高 6 bit 作为"打到哪台/哪类 DB"的识别。例如 `0xB469`、`0x10EE`、`0x3046` 落在不同号段,反映不同的 DB/表分区。

> 各游戏的路由位域宽度并不一样。摩尔 1 代用前 6 bit(`0xFC00`);摩尔 2 代用前 7 bit(`0xFE00`),见 [mole2/mole2_db/src/proto.h#L28](../mole2/mole2_db/src/proto.h#L28);但写库标志 `NEED_UPDATE = 0x0100` 各游戏一致,见 [mole2/mole2_db/src/proto.h#L32](../mole2/mole2_db/src/proto.h#L32)。

---

## 2. 报文头结构(packed 小端)

DB 服务端的包头定义在 [mole/dbsvr/include/proto_header.h#L28](../mole/dbsvr/include/proto_header.h#L28):

```c
typedef struct proto_header{
	uint32_t proto_length;  ///< 包的全部长度
	uint32_t proto_id;      ///< 协议版本号
	uint16_t cmd_id;        ///< 命令编号
	int32_t  result;        ///< 返回码(请求时一般为 0)
	userid_t id;            ///< 用户编号(userid_t = uint32_t)
} __attribute__((packed)) PROTO_HEADER;
```

要点:
- `__attribute__((packed))` —— **无对齐填充**,字段紧贴;x86 Linux 上是**小端**字节序。
- 头部固定 **18 字节**(4+4+2+4+4),正文紧随其后(`PROTO_HEADER_SIZE` 之后即私有数据区,见 [#L37–L41](../mole/dbsvr/include/proto_header.h#L37))。
- `proto_id` 是协议版本/魔数;`set_proto_buf` 里硬编码为 `0x12345678`,见 [proto_header.h#L83](../mole/dbsvr/include/proto_header.h#L83)。
- 回包用 `set_std_return` —— 直接把收到的头 `memcpy` 回去,只改 `proto_length` 与 `result`,见 [proto_header.h#L57](../mole/dbsvr/include/proto_header.h#L57)。所以**请求头与响应头同构**。
- `userid_t` 即 `uint32_t`,见 [proto_header.h#L25](../mole/dbsvr/include/proto_header.h#L25)。

### 2.1 其它层的包头(并非同一个头!)

DB 层之外还有几个**不同结构**的头,对照 iOS 抓包时要分清自己抓的是哪一层。它们定义在 [mole/online/protoheaders.h](../mole/online/protoheaders.h):

- **online ↔ DB / 中心服**:`server_proto_t` = `len / seq / cmd / ret / id / body[]`(`cmd` 是 `uint16_t`)。
- **online ↔ 广播器(Broadcaster)**:`communicator_header_t`,多了 `online_id / mapid / opid` 等场景字段。
- **消息日志**:`msglog_proto_t`,`len` / `cmd` 是 `uint16_t`,更紧凑。

客户端面向的"在线服务"(`mole/online`)还另有一套**业务事件命令字**(数值很小、十进制),例如学校吧事件,见 [mole/online/school_bar_cmd.h](../mole/online/school_bar_cmd.h):`LAHM_DIARY_CMD 1001`、`MOLE_LEVEL_UP_CMD 1002`、`TASK_FINI_CMD 1003`。这套和 DB 层的 `0xB469` 式命令字是**两个命名空间**,别混用。

> 加密:在线服务里出现 DES 密钥 `#define DES_KEY ">ta:mee<"`(台服另有 `DES_GAME_SCORE_KEY`),见 [mole/online/proto.h#L15](../mole/online/proto.h#L15)。即客户端↔在线服务这一段可能对(部分)报文做 DES;而 online↔DB 的内网段通常明文。对照 iOS 抓包时这点很关键(见第 6 节)。

---

## 3. 各主要 proto.h 命令字统计

下表为各文件中 `#define ... _cmd`(大小写不敏感)命令字总数,及其中带 `NEED_UPDATE`(写库)的数量。统计口径见下方"怎么自查"。

| proto.h 文件 | 业务 | 行数 | 命令字数 | 其中 NEED_UPDATE | 路由位域 |
|---|---|---:|---:|---:|---|
| [mole/dbsvr/include/proto.h](../mole/dbsvr/include/proto.h) | 摩尔庄园 DB 主表 | 15003 | 1300 | 698 | `0xFC00`(6bit) |
| [mole/cache_serv/pkg/proto.h](../mole/cache_serv/pkg/proto.h) | 摩尔缓存服 | 7945 | 634 | 381 | `0xFC00`(6bit) |
| [mole/register/pkg/proto.h](../mole/register/pkg/proto.h) | 摩尔注册/账号 | 4105 | 352 | 212 | `0xFC00`(6bit) |
| [gf/cache_svr/pkg/proto.h](../gf/cache_svr/pkg/proto.h) | 功夫派缓存服 | 3833 | 351 | 205 | (见文件内) |
| [gf/db/ser/src/proto.h](../gf/db/ser/src/proto.h) | 功夫派 DB | 4047 | 284 | 137 | (见文件内) |
| [mole2/mole2_db/src/proto.h](../mole2/mole2_db/src/proto.h) | 摩尔 2 代 DB | 2200 | 154 | 101 | `0xFE00`(7bit) |

补充说明(已 Read 复核):
- 上表"命令字数"统计的是**宏名以 `_cmd` 结尾**的 `#define`。mole 系用大写 `_CMD`,gf/mole2 的 DB 文件混用小写 `_cmd`,故用大小写不敏感口径。任务背景里提到的"约 1347" 是用更宽的"行内含 `_CMD`"口径(含个别非命令宏);本文采用更严格的"宏名以 `_cmd` 结尾"口径,得 1300,二者都对,只是口径不同。
- [mole/online/proto.h](../mole/online/proto.h)(2202 行)是**在线服务**头,主体是错误码 `enum`(如 `ERR_invalid_session = 10001`,见 [#L24](../mole/online/proto.h#L24))、DES 密钥与函数声明,**不**用 `_cmd` 宏,故不计入上表。客户端事件命令字另见 `school_bar_cmd.h`(十进制)。
- [monster/online/src/cli_proto.h](../monster/online/src/cli_proto.h)(466 行)是 **淘米小怪兽**(monster,gameid 13,**非赛尔号**)在线服务的**分发函数声明**(`dispatch` / `cache_a_pkg` 等),不含命令字宏。其命令字表在其它头里(待定位,本文未逐一统计 —— 推测/待证)。

---

## 4. 命令字分类与代表(以 mole 主表为主)

`mole/dbsvr/include/proto.h` 的命令字按**前缀**分类。前缀粗略统计(顶部几类):`USER_` 约 736、`ROOMINFO_` 约 119、`SYSARG_` 约 89、`SU_` 约 69、`USERINFO_` 约 58、`MSGBOARD_` 约 17 等。下面每类给少量真实代表(名称 + 0x 值 + 中文注释 + 是否写库 + 行号链接),**不**穷举,末尾给"怎么自查"的可复制命令。

### 4.1 `USERINFO_` —— 账号 / 登录 / 注册(账号体系)

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `USERINFO_GET_GAMEFLAG_CMD` | `0x0007` | 取用户游戏标志位 | 否 | [#L705](../mole/dbsvr/include/proto.h#L705) |
| `USERINFO_ADD_GAME_CMD` | `0x0008` | 给账号开通某游戏 | 是 | [#L708](../mole/dbsvr/include/proto.h#L708) |
| `USERINFO_SU_SET_PASSWD_CMD` | `0x000A` | 后台改密码 | 是 | [#L713](../mole/dbsvr/include/proto.h#L713) |
| `USERINFO_LOGIN_BY_MD5_TWO_CMD` | `0x005F` | MD5 二次校验登录 | 否 | [#L769](../mole/dbsvr/include/proto.h#L769) |
| `USERINFO_GET_LOGIN_INFO_CMD` | `0x0091` | 取登录信息 | 否 | [#L828](../mole/dbsvr/include/proto.h#L828) |

> 账号/注册的完整入口在 [mole/register/pkg/proto.h](../mole/register/pkg/proto.h) 与 [mole/cache_serv/pkg/proto.h#L33](../mole/cache_serv/pkg/proto.h#L33)(`USERINFO_LOGIN_CMD = 0x0001`、`USERINFO_REGISTER_CMD = 0x0002 | NEED_UPDATE`)。

### 4.2 `USER_GET_` / `USER_SET_` —— 用户主数据读写(最大类)

`USER_GET_` 约 128 个、`USER_SET_` 约 56 个。代表:

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `USER_GET_INFO_EX_CMD` | `0x1006` | 取用户扩展信息 | 否 | [#L875](../mole/dbsvr/include/proto.h#L875) |
| `USER_GET_XIAOMEE_CMD` | `0x100D` | 取小蜜(货币/资源) | 否 | [#L897](../mole/dbsvr/include/proto.h#L897) |
| `USER_GET_ATTIREID_AND_CNT_CMD` | `0x3046` | 取装扮 ID 与数量 | 否 | [#L252](../mole/dbsvr/include/proto.h#L252) |
| `USER_SET_TASK_CMD` | `0x1007` | 写任务状态 | 是 | [#L878](../mole/dbsvr/include/proto.h#L878) |
| `USER_SET_VIP_FLAG_CMD` | `0x1014` | 设置 VIP 标志 | 是 | [#L919](../mole/dbsvr/include/proto.h#L919) |
| `USER_SET_INT_VALUE_CMD` | `0x3050` | 通用整型字段写入 | 是 | [#L256](../mole/dbsvr/include/proto.h#L256) |

### 4.3 `USER_PET_` / `PET` —— 宠物(拉姆/小蜜)

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `USER_PET_GET_INFO_CMD` | `0x106B` | 取宠物信息 | 否 | [#L1124](../mole/dbsvr/include/proto.h#L1124) |
| `USER_ADD_USER_PET_CMD` | `0x1051` | 新增宠物 | 是 | [#L1063](../mole/dbsvr/include/proto.h#L1063) |
| `USER_DEL_USER_PET_CMD` | `0x1055` | 删除宠物 | 是 | [#L1069](../mole/dbsvr/include/proto.h#L1069) |
| `USER_SET_USER_PET_POSLIST_CMD` | `0x1054` | 设置宠物位置列表 | 是 | [#L1068](../mole/dbsvr/include/proto.h#L1068) |
| `USER_GET_PET_ATTIRE_LIST_CMD` | `0x1066` | 取宠物装扮列表 | 否 | [#L1107](../mole/dbsvr/include/proto.h#L1107) |
| `USER_USE_SKILL_ONE_CMD` | `0x10EE` | 宠物使用一次技能 | 是 | [#L41](../mole/dbsvr/include/proto.h#L41) |

### 4.4 `USER_TASK_` / `TASK` —— 任务系统

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `USER_PET_TASK_GET_TASKLIST_CMD` | `0x3053` | 取宠物任务列表 | 否 | [#L262](../mole/dbsvr/include/proto.h#L262) |
| `USER_TASK_PET_FIRST_STAGE_SET_CMD` | `0x1074` | 设置宠物任务首阶段 | 是 | [#L1145](../mole/dbsvr/include/proto.h#L1145) |
| `USER_TASK_TASKID_DEL_CMD` | `0x107F` | 删除指定任务 | 是 | [#L1182](../mole/dbsvr/include/proto.h#L1182) |

### 4.5 `USER_FARM_` / `FARM` —— 农场(种植)

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `USER_FARM_SET_COL_CMD` | `0x10AD` | 设置农田格子 | 是 | [#L1293](../mole/dbsvr/include/proto.h#L1293) |
| `USER_FARM_SET_WEB_CMD` | `0x10B7` | 设置农场网格 | 是 | [#L1303](../mole/dbsvr/include/proto.h#L1303) |

### 4.6 `USER_ATTIRE_` / `ATTIRE` —— 装扮 / 衣橱

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `USER_ATTIRE_SET_USED_CMD` | `0x103F` | 设置已穿戴装扮 | 是 | [#L1027](../mole/dbsvr/include/proto.h#L1027) |
| `USER_ATTIRE_PUT_CHEST_CMD` | `0x108A` | 装扮入箱 | 是 | [#L1211](../mole/dbsvr/include/proto.h#L1211) |

### 4.7 `FRIEND` / `*_FRIEND_BOX_*` —— 好友 / 好友礼物盒

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `GET_FRIEND_BOX_ITEMS_CMD` | `0x30B9` | 取好友礼物盒物品 | 否 | [#L572](../mole/dbsvr/include/proto.h#L572) |
| `PUT_ITEM_TO_FRIEND_BOX_CMD` | `0x30BB` | 放物品进好友盒 | 是 | [#L578](../mole/dbsvr/include/proto.h#L578) |
| `GET_ITEM_FROM_FRIEND_BOX_CMD` | `0x30BC` | 从好友盒取物品 | 是 | [#L581](../mole/dbsvr/include/proto.h#L581) |
| `USER_BATTLE_GET_FRIEND_EVENT_INFO_CMD` | `0x2220` | 取好友对战事件信息 | 否 | [#L385](../mole/dbsvr/include/proto.h#L385) |

### 4.8 `ROOMINFO_` —— 房间 / 场景 / 全局信息(约 119)

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `ROOMINFO_CLIENT_DATA_GET_CMD` | `0xB469` | 客户端私有数据读 | 否 | [#L32](../mole/dbsvr/include/proto.h#L32) |
| `ROOMINFO_SET_CLIENT_BUF_CMD` | `0xB480` | 设置前端 buf | 是 | [#L68](../mole/dbsvr/include/proto.h#L68) |
| `ROOMINFO_GET_CLIENT_BUF_CMD` | `0xB481` | 获取前端 buf | 否 | [#L71](../mole/dbsvr/include/proto.h#L71) |
| `ROOMINFO_GET_ITEM_SELL_CMD` | `0xB487` | 取道具寄售 | 否 | [#L89](../mole/dbsvr/include/proto.h#L89) |
| `ROOMINFO_ADD_ITEM_SELL_CMD` | `0xB488` | 上架寄售道具 | 是 | [#L92](../mole/dbsvr/include/proto.h#L92) |

### 4.9 `SPORTS_` / `ROOMINFO_SPORTS_` —— 拉姆运动会(活动)

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `ROOMINFO_SPORTS_MEET_SET_CMD` | `0xB478` | 设置运动会队 teamid | 是 | [#L59](../mole/dbsvr/include/proto.h#L59) |
| `ROOMINFO_SPORTS_MEET_MEDAL_SET_CMD` | `0xB47C` | 更新个人奖牌数 | 是 | [#L65](../mole/dbsvr/include/proto.h#L65) |
| `ROOMINFO_SPORTS_REWARD_GET_CMD` | `0xB484` | 取用户/拉姆奖励标志 | 否 | [#L80](../mole/dbsvr/include/proto.h#L80) |

### 4.10 `USER_LAMU_` / `LAMU_` —— 拉姆教室 / 超拉(社交玩法)

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `LAMU_CLASSROOM_GET_FRIEND_RANK_CMD` | `0x221A` | 教室好友排行 | 否 | [#L371](../mole/dbsvr/include/proto.h#L371) |
| `USER_LAMU_CLASSROOM_CREATE_CLASSROOM_CMD` | `0x30C1` | 创建拉姆教室 | 是 | [#L590](../mole/dbsvr/include/proto.h#L590) |
| `USER_LAMU_CLASSROOM_SET_INNER_STYLE_CMD` | `0x30C3` | 设置教室内饰 | 是 | [#L596](../mole/dbsvr/include/proto.h#L596) |

### 4.11 `MAP_` / `*_MAP_*` —— 地图 / 挖矿

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `USER_BATTLE_MAP_INFO_CMD` | `0x324C` | 取战斗地图信息 | 否 | [#L1748](../mole/dbsvr/include/proto.h#L1748) |
| `USER_FINISH_SOME_MAP_MINING_CMD` | `0x3615` | 完成某地图挖矿 | 否 | [#L2053](../mole/dbsvr/include/proto.h#L2053) |

### 4.12 `MSGBOARD_` / `EMAIL` —— 留言板 / 站内信

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `MSGBOARD_GET_MSGLIST_USERID_CMD` | `0x7008` | 取某用户留言列表 | 否 | [#L2249](../mole/dbsvr/include/proto.h#L2249) |
| `MSGBOARD_GET_MSG_NOCHECK_CMD` | `0x7009` | 取留言(不校验) | 否 | [#L2250](../mole/dbsvr/include/proto.h#L2250) |
| `USERINFO_SET_PASSWDEMAIL_CMD` | `0x0017` | 设置密保邮箱 | 是 | [#L727](../mole/dbsvr/include/proto.h#L727) |

### 4.13 `SU_` —— 后台 / 超级用户(运营工具,约 69)

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `SU_USER_GET_ALL_PIGLETS_CMD` | `0x32DB` | 后台:取某用户全部小猪 | 否 | [#L1933](../mole/dbsvr/include/proto.h#L1933) |
| `SU_USER_SWAP_LIST_CMD` | `0x32E5` | 后台:交换列表 | 是 | [#L1953](../mole/dbsvr/include/proto.h#L1953) |
| `SU_USER_GET_WORK_PIGLET_INFO_CMD` | `0x3648` | 后台:取打工小猪信息 | 否 | [#L2125](../mole/dbsvr/include/proto.h#L2125) |

### 4.14 `SYSARG_` —— 系统参数 / 每日清理(约 89)

| 命令字 | 值 | 含义 | 写库 | 链接 |
|---|---|---|:--:|---|
| `SYSARG_DAY_CLEAR_CMD` | `0xC004` | 每日清理 | 是 | [#L2329](../mole/dbsvr/include/proto.h#L2329) |
| `SYSARG_DAY_SET_CMD` | `0xC007` | 设置每日数据 | 是 | [#L2337](../mole/dbsvr/include/proto.h#L2337) |
| `SYSARG_DAY_GET_LIST_CMD` | `0xC00C` | 取每日列表 | 否 | [#L2347](../mole/dbsvr/include/proto.h#L2347) |

### 4.15 其它游戏的代表命令字

- **摩尔 2 代** [mole2/mole2_db/src/proto.h](../mole2/mole2_db/src/proto.h):`mole2_login_cmd = 0x0801`(否,[#L38](../mole2/mole2_db/src/proto.h#L38))、`mole2_user_user_create_role_cmd = 0x0802 | NEED_UPDATE`(是,[#L40](../mole2/mole2_db/src/proto.h#L40))、`mole2_user_pet_add_cmd`(宠物新增,写库)。
- **功夫派** [gf/db/ser/src/proto.h](../gf/db/ser/src/proto.h):`gf_get_user_info_cmd = 0x0601`(否,[#L34](../gf/db/ser/src/proto.h#L34))、`gf_set_child_count_cmd = 0x0609 | NEED_UPDATE`(是,[#L42](../gf/db/ser/src/proto.h#L42))、`gf_sell_attire_cmd = 0x0610 | NEED_UPDATE`(是,[#L51](../gf/db/ser/src/proto.h#L51))。注意功夫派沿用同名的写库位 `NEED_UPDATE`,但命令字号段从 `0x06xx` 起。

---

## 5. 怎么自查(可复制 grep)

> 路径含空格,务必整体加引号。

```bash
REPO="/Users/xiaochoumao/Documents/github repo/taomee-server-leak"
P="$REPO/mole/dbsvr/include/proto.h"

# 1) 这个文件里一共多少个命令字(宏名以 _cmd 结尾,大小写不敏感)
grep -ciE '^[[:space:]]*#[[:space:]]*define[[:space:]]+[A-Za-z0-9_]+_cmd[[:space:]]' "$P"

# 2) 其中多少个要写库(带 NEED_UPDATE)
grep -iE '^[[:space:]]*#[[:space:]]*define[[:space:]]+[A-Za-z0-9_]+_cmd[[:space:]]' "$P" | grep -ci NEED_UPDATE

# 3) 按一级前缀统计分布(USER_ / ROOMINFO_ / SU_ ...)
grep -oE '^#define ([A-Z0-9]+)_' "$P" | sed 's/^#define //' | sort | uniq -c | sort -rn | head -30

# 4) 按某前缀列出全部代表(例:好友盒 FRIEND_BOX),带行号
grep -nE '^#define [A-Z0-9_]*FRIEND_BOX[A-Z0-9_]*_CMD ' "$P"

# 5) 已知一个 16-bit 值(例 0x30bb),反查它是哪个命令字
grep -niE '0x30bb' "$P"

# 6) 把命令字名 -> 值 全量导出,便于和 iOS 抓包逐一对表
grep -oE '#define [A-Za-z0-9_]+_[Cc][Mm][Dd][[:space:]]*\(0x[0-9A-Fa-f]+' "$P"
```

---

## 6. 如何与 iOS 手游协议比对

我们抓到的是**摩尔庄园 iOS 手游**的实际流量;本仓库是 **2007–2013 年网页 Flash 版**的服务端真源码。二者**很可能同源但已演进**,把它当"字典"而非"逐字真值"来用。建议方法:

### 6.1 先对齐"层"

iOS 抓包通常是**客户端 ↔ 接入/在线服务**这一层,对应本仓库的 `mole/online`(`server_proto_t` / 广播器头 / DES),**不是** DB 层的 `PROTO_HEADER`。先判断你抓的包头长度与字段:
- 18 字节、`length/proto_id(0x12345678)/cmd16/result/uid` 形态 → 像 DB 层 `PROTO_HEADER`。
- 含 `seq` 字段、`len/seq/cmd/ret/id` → 像 `server_proto_t`(内网服务间)。
- 含场景字段(mapid/opid)→ 像广播器 `communicator_header_t`。
- 都不像 → iOS 版**自定义了新包头**(很可能),只能靠命令字数值/正文结构反推。

### 6.2 用 cmdid 当锚点

1. 从 iOS 抓包里取出疑似命令字的 16-bit 字段。
2. 在本仓库反查:`grep -niE '0xXXXX' "$P"`(注意大小写,值字面量大小写混用)。
3. 命中后,用命令字**名字 + 中文注释**推断该包语义,再回到对应模块源码看处理函数(参数结构、字段顺序)。

> 反查时记得**剥掉标志位**:iOS 抓到的可能是裸命令字(不含 `NEED_UPDATE`/路由位),也可能保留。对一个值 `v`,既查 `v`,也查 `v & 0x00FF`(去掉写库位)与 `v & 0x03FF`(去掉摩尔1代路由位),三种都试。

### 6.3 用字段语义当二次校验

光对上 cmdid 不够,还要看正文字段是否吻合:`PROTO_HEADER` 里的 `uid`、`USER_GET_*` 返回结构里的字段顺序(装扮 id、宠物 id、货币"小蜜"等)。字段定义散落在各模块,从 [mole.md](./modules/mole.md) 入手定位处理函数,内容/数值表见 [content-data.md](./content-data.md)。

### 6.4 注意事项(易踩坑)

- **包头不同**:iOS 版几乎肯定换了包头(可能加 token/session/设备号/版本号字段),不要假设 18 字节 `PROTO_HEADER` 原样保留。
- **字节序**:源码是 x86 **小端** packed。但 `mole/online/cache_svr.c` 里出现 `htonl(cmd)`(网络字节序=大端),说明**部分服务间字段走大端**。iOS 抓包要逐字段判断端序,别一刀切。
- **加密**:客户端段有 `DES_KEY ">ta:mee<"`(见第 2.1 节)。iOS 版很可能换了密钥/算法(AES?),抓到密文要先定位握手/密钥协商,别拿源码里的旧 DES 密钥硬解(推测/待证)。
- **命令字可能重排**:多年演进后,同一玩法的 cmdid 数值可能变,但**命名前缀体系(USER_/PET_/FARM_…)与玩法语义大概率保留**。优先按"玩法分类 + 字段结构"对齐,cmdid 数值作辅助证据。
- **新增/废弃**:iOS 版有手游独有功能(新活动/付费点),网页源码里查不到属正常;反之网页源码里大量后台 `SU_`/`SYSARG_` 命令不会出现在客户端流量里。

---

## 7. 小结

- 命令字 = 16-bit 复合位域:高 6 bit(`0xFC00`)路由到 DB,`0x0100` 标记写库,其余为业务序号。
- 报文头 packed 小端、18 字节、请求/响应同构;但 online/广播器/iOS 各有不同的头,务必分层。
- `mole/dbsvr/include/proto.h` 是命令字总字典(约 1300 条),按 `USER_`/`ROOMINFO_`/`SU_`/`SYSARG_`/`USERINFO_` 等前缀组织,用第 5 节的 grep 自查、第 6 节的方法与 iOS 抓包对表。

返回:[../README.md](../README.md) · [mole.md](./modules/mole.md) · [content-data.md](./content-data.md)
