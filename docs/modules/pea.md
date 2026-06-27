# 模块 pea · 赛尔号「精灵对战」页游服务端(精灵 PVP/PVE 弹道战斗)

> 价值:★ · 游戏:赛尔号(Seer) · 源码量:约 4.4 万行(C/C++,411 个文件)
> 📖 深度功能解析(每个部分实际在干什么):[deep/pea.md](../deep/pea.md) · 彩蛋见 [easter-eggs.md](../easter-eggs.md)

## 是什么 / 在整套架构里的角色

`pea` 是淘米另一款页游 **赛尔号(Seer)对战服**的完整服务端,核心数据是“精灵”(pet):
[pea/online/conf/pet.xml](../../pea/online/conf/pet.xml) 里每个 `<pet>` 带 `name`(如“小八”)、`skill1~3`、`uni_skill`、星级成长(`star`)与逐级经验曲线 —— 这是赛尔号精灵养成的标志性数据结构,与摩尔庄园(mole)、功夫派(gf)都不同。

战斗是**带物理弹道的对战**:玩家移动/瞄准/开火协议里带 `angle`(角度)、`muzzle_velocity`(出膛速度)、`gun_angle`(炮口角度),见 [pea/online/online.cpp](../../pea/online/online.cpp#L11) 的 `cli_proto_btl_player_move` / `cli_proto_player_attack`,并有独立物理引擎 [pea/pea_common/physical_engine.hpp](../../pea/pea_common/physical_engine.hpp) 和子弹系统 [pea/battle_server/bullet.cpp](../../pea/battle_server/bullet.cpp)。这是“回合制瞄准射击 + 精灵属性”的混合玩法。

它沿用了淘米典型的**多进程分服架构**(与 mole2 同源,proto 生成器甚至直接复用 `mole2_online_proto.py`):

| 进程 | 目录 | 角色 |
|------|------|------|
| login | [pea/login/](../../pea/login/) | 通用登录框架(`login.cpp` 头注释自称“不许更改”),具体逻辑在 `login_impl.cpp` |
| online | [pea/online/](../../pea/online/) | 大厅/世界主逻辑服:精灵、背包、任务、邮件、好友、聊天、商店,作为 `.so` 挂在 `bin/AsynServ` 框架上 |
| battle_switch / new_battle_switch | [pea/battle_switch/](../../pea/battle_switch/) · [pea/new_battle_switch/](../../pea/new_battle_switch/) | 对战“房间/大厅”分配服(PVP 房间列表、创建/加入/座位/开战转发);new_ 是重写版 |
| battle_server | [pea/battle_server/](../../pea/battle_server/) | 真正跑战斗的进程:回合、子弹、技能、buff、地图、命中判定 |
| dbserver / dbproxy | [pea/dbserver/](../../pea/dbserver/) · [pea/dbproxy/](../../pea/dbproxy/) | 落地 MySQL 的 DB 服(`service.cpp` 直接 `#include "mysql_iface.h"`)+ online 侧的 db 访问代理 |
| dirty | [pea/dirty/](../../pea/dirty/) | 脏字/敏感词过滤 agent |

进程关系:客户端先连 login 拿票据 → 连 online 进世界 → 在 battle_switch 里组房 → online 通知房主转发,所有玩家加入对应的 battle_server 正式开打(见下文 `btlsw_pvp_room_start_battle_cmd` 注释)。

作者头注释实证:文件统一署名 `Copyright: TaoMee, Inc. ShangHai CN.`,日期集中在 **2011-12 ~ 2012-06**(见 [pea/online/proto.hpp](../../pea/online/proto.hpp#L6) `@date 2011-12-26` / `Modify $Date: 2012-06-13 ... $Author: zog`),pet_data 为 2012-04;dbserver 的 `Croute_func.cpp` 可追溯到 2009,说明 DB 层骨架更老、上层战斗逻辑为 2012 新写。

## 关键子目录与文件

- [pea/online/proto.hpp](../../pea/online/proto.hpp) — online 协议总入口:用宏 `PROTO_FUNC_DEF` 把客户端/btlsw/db 三类命令注册成处理函数表 `bind_proto_cmd_t`。
- [pea/online/cli_proto.cpp](../../pea/online/cli_proto.cpp#L135) — 客户端入包分发器:校验 seqno、防重登(`ONLINE_ERR_MULTI_LOGIN`)、`find_cli_cmd_bind(cmd)` 派发。
- [pea/online/online.cpp](../../pea/online/online.cpp) — 战斗转发型协议实现(移动/攻击/进度/拉取战斗玩家列表),实际转交 `btlsvr_*`。
- [pea/online/pet.cpp](../../pea/online/pet.cpp) — 精灵核心逻辑(34KB):玩家精灵列表、出战精灵 `fight_pet`、助战 `assist_pet`。
- [pea/online/task_funcs.cpp](../../pea/online/task_funcs.cpp) / [task.cpp](../../pea/online/task.cpp) — 任务系统(主线/支线/日常,前置依赖与/或判定)。
- [pea/online/db_pet.cpp](../../pea/online/db_pet.cpp) · [db_item.cpp](../../pea/online/db_item.cpp) — online 侧精灵/道具的 DB 读写封装。
- [pea/battle_server/battle_impl.cpp](../../pea/battle_server/battle_impl.cpp) — 战斗主实现(26KB):回合推进与命中计算。
- [pea/battle_server/bullet.cpp](../../pea/battle_server/bullet.cpp) — 子弹对象:多波攻击 `bullet_times`、每波子弹数、间隔角度。
- [pea/battle_server/skill.cpp](../../pea/battle_server/skill.cpp) · [buff.cpp](../../pea/battle_server/buff.cpp) — 技能与增益/减益效果。
- [pea/pea_common/pet_data.cpp](../../pea/pea_common/pet_data.cpp) — 精灵静态配置加载器(解析 pet.xml,被 online 与 battle_server 共用)。
- [pea/pea_common/physical_engine.hpp](../../pea/pea_common/physical_engine.hpp) — 弹道物理引擎(角度/速度/重力计算)。
- [pea/pea_common/rc4.cpp](../../pea/pea_common/rc4.cpp) — RC4 包体加密(淘米通用)。
- [pea/new_battle_switch/proto/pea_btlsw_enum.h](../../pea/new_battle_switch/proto/pea_btlsw_enum.h) — btlsw 命令字与错误码枚举(最完整、注释最详尽的协议定义,见下节)。
- [pea/dbserver/service.cpp](../../pea/dbserver/service.cpp) — DB 服主循环,直连 MySQL。
- [pea/dbserver/Croute_func.cpp](../../pea/dbserver/Croute_func.cpp) — DB 请求路由分发(2009 老骨架)。
- [pea/gen_proto/tools/cli/mole2_online_proto.py](../../pea/gen_proto/tools/cli/mole2_online_proto.py) — 客户端协议生成源(直接复用 mole2 的生成器,证明 pea 与 mole2 同框架)。
- [pea/bin/all_server.sh](../../pea/bin/all_server.sh) — 全套进程的启停脚本(区分 `.so` 业务服与独立可执行)。
- [pea/login/conf/server.xml](../../pea/login/conf/server.xml) · [pea/online/conf/btlsvr.xml](../../pea/online/conf/btlsvr.xml) — 分服 IP/端口与 battle server 注册表(`pea.btlsvr` SvrId=33)。

## 协议 / 数据亮点

**命令字(真实 cmdid)** —— online 客户端协议号是构建期由 Python 生成器产出、未入库;最完整的明文枚举在 btlsw:

- `btlsw_online_register_cmd = 11001` —— online 向 btlsw 注册,见 [pea_btlsw_enum.h#L6](../../pea/new_battle_switch/proto/pea_btlsw_enum.h#L6)
- `btlsw_pvp_create_room_cmd = 11003` —— 建立 PVP 房间
- `btlsw_pvp_room_start_battle_cmd = 11015` —— online 通知房主、btlsw 转发给房内所有玩家加入对应 battle_server 正式开战,见 [pea_btlsw_enum.h#L16](../../pea/new_battle_switch/proto/pea_btlsw_enum.h#L16)

错误码段落很有“淘米味”的中文吐槽(`400018 不能自宫!`、`400016 你不是房间的主子/堂口/话事人/扛把子`),见 [pea_btlsw_enum.h#L33 起](../../pea/new_battle_switch/proto/pea_btlsw_enum.h#L33)。整套协议总览见 [../protocol.md](../protocol.md)。

**conf XML(内容数据)** —— pea 的配置数据信息量很大(~1MB):

| 文件 | 内容 |
|------|------|
| [online/conf/pet.xml](../../pea/online/conf/pet.xml)(388KB) | 全精灵:技能、星级成长系数、逐级经验表(精灵养成核心) |
| [online/conf/pet_culture.xml](../../pea/online/conf/pet_culture.xml)(268KB) | 精灵培养数据 |
| [battle_server/conf/skill.xml](../../pea/battle_server/conf/skill.xml) | 技能弹道(波数/每波子弹数/间隔角度/立即生效类型) |
| [online/conf/player_attribute.xml](../../pea/online/conf/player_attribute.xml) | 玩家等级→属性曲线(hp/atk/crit/速度等) |
| [online/conf/item.xml](../../pea/online/conf/item.xml) | 道具(含 `end_time="201305..."` 时效物品 → 内容更新到 2013) |
| [online/conf/maps.xml](../../pea/online/conf/maps.xml) | 地图/出生点(XML 头残留 XMLSpy 编辑器水印) |

**SQL**:本模块仓库内**未找到 `.sql` 建表脚本**(DB schema 隐含在 dbserver/dbproxy 的 C++ 代码与 `gen_sql` 工具里),建表语义需从 [pea/dbserver/](../../pea/dbserver/) 各 `*.cpp` 反推。

## 对我们有什么用(摩尔庄园离线复活)

定位:**整体参照,不直用**。pea 是赛尔号、不是摩尔庄园,精灵/弹道战斗与摩尔庄园玩法无关,内容数据(pet.xml 等)对 moleworld-server 没有直接价值。

但它与 mole2 是同一套框架同一套 proto 生成器,因此可作为**架构参照**:

- **直用(框架层,跨游戏通用)**:[pea/pea_common/rc4.cpp](../../pea/pea_common/rc4.cpp)(包加密)、[pea/bin/all_server.sh](../../pea/bin/all_server.sh)(进程编排模板)、login 框架的“通用 login.cpp + 业务 login_impl.cpp”分层模式 —— 这些与摩尔庄园的实现思路一致,可直接对照补全 moleworld-server 缺失的会话/分包/防重登逻辑。
- **参照**:`bind_proto_cmd_t` 命令注册表 + `find_cli_cmd_bind` 分发(见 cli_proto.cpp#L135)是淘米所有页游统一的入包模型,照此可还原摩尔庄园的协议派发骨架;PVP 房间/座位/开战转发(btlsw 11001~11028)是淘米对战类玩法的标准状态机,若 moleworld 要做对战副本可借鉴。
- **协议生成器**:[gen_proto/tools/cli/mole2_online_proto.py](../../pea/gen_proto/tools/cli/mole2_online_proto.py) 里残留了 mole2 的协议定义,对还原摩尔庄园 2 的客户端协议比 pea 本体更有价值,值得单独翻。

## 注意 / 坑

- **纯二进制无源**:[pea/bin/AsynServ](../../pea/bin/AsynServ)(异步框架可执行)、`ServerBench`、`flashpolicyd.tar` 均为编译产物/打包,无对应源码;online/battle_switch 业务是 `.so`,必须挂在这个二进制框架上才能跑,缺它无法独立启动。
- **缺外部依赖**:大量 `#include <libtaomee/...>` / `<libtaomee++/...>` / `<async_serv/...>` / `<glib.h>` / `<main_login/...>` —— 这些是淘米内部框架库(见 [libtaomee.md](./libtaomee.md) / [c.md](./c.md)),仓库内以两个 `.tar.gz` 形式存在,需先解包才能编译。
- **协议号不全**:online 的客户端 cmdid 由构建期 Python 生成、源码树里没有最终常量值,只能从 gen_proto 的 `.py` 推断;只有 btlsw 段(11001+)是明文。
- **无 DB schema**:无 `.sql`,建表结构需从 dbserver 代码反推。
- **新旧两套 battle_switch 并存**:`battle_switch/` 与 `new_battle_switch/` 同时在库,以 new_ 为准;老版本可能含废弃协议,阅读时别混。
- **年代局限**:代码为 gcc 4.1.2 / 32 位时代风格(头注释明示 `compiler gcc4.1.2`),`bswap`、网络序手工处理、裸 `Cmessage*` 反射,迁移到现代工具链需适配。

---

回总入口:[../../README.md](../../README.md) · 相关:[../protocol.md](../protocol.md) · [../content-data.md](../content-data.md) · 同框架兄弟:[mole2.md](./mole2.md) · [monster.md](./monster.md)
