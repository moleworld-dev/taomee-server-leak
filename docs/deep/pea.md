# pea 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/pea.md](../modules/pea.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)
>
> 本文是「深水区」补充:概览讲“pea 是什么、有哪些目录”,本文讲 **运行时它实际怎么算一场赛尔号对战** —— 弹道怎么飞、伤害/暴击公式、回合怎么轮转、online↔battle_server↕dbserver 的数据怎么流、XML 协议怎么变成 C++ 处理函数。所有结论均带真实行号。

---

## 一图看懂:进程/组件拓扑与数据流

赛尔号对战是 **多进程拆分** 的:大厅逻辑、房间撮合、真实战斗结算、落库各自一个进程,客户端的战斗指令要 **经 online 中转** 才到 battle_server。

```
                 [Flash 客户端]
                      │ RC4 加密 TCP,cli_proto_t 包头
                      ▼
   ┌──────────┐   票据   ┌────────────────────────────────┐
   │  login   │────────▶│            online (.so)         │  世界/大厅主逻辑
   │ login_   │         │  cli_proto.cpp 入包分发          │  精灵/背包/任务/邮件/聊天
   │ impl.cpp │         │  battle.cpp  ←→ 转发战斗指令     │
   └──────────┘         └───┬───────────────┬─────────────┘
                            │ btlsw 11001+  │ btl_proto_t (转输)
                            ▼               ▼
              ┌───────────────────┐   ┌────────────────────────┐
              │ new_battle_switch │   │      battle_server      │  真正跑战斗
              │ PVP 房间/座位/开战 │   │ battle_impl.cpp 回合机   │
              └───────────────────┘   │ attack_obj/bullet 弹道  │
                            ▲          │ physical_engine 物理积分 │
                            │ 开战通知  └────────────────────────┘
                            │
                            └──────── online 通知房主 → btlsw 广播“加入战斗服”
                            
   online ──db_proto_*──▶ dbproxy ──▶ dbserver ──mysql_iface──▶ MySQL
                                       Croute_func.cpp 路由
```

**两条关键数据流**(下文“端到端”详解):
1. **撮合流**:客户端→online→new_battle_switch 建房/坐下→online 发起开战→btlsw 通知所有人连同一个 battle_server。
2. **战斗流**:客户端的移动/瞄准/开火→online `btlsvr_*`(见 [online/battle.cpp](../../pea/online/battle.cpp))把指令打成 `btl_proto_t` 转给 battle_server→battle_server 结算→`online_proto_transmit_only_cmd` 把结果原样回传给 online,online 再 `send_to_player` 给客户端(见 [online/battle.cpp#L434](../../pea/online/battle.cpp#L434))。

---

## 逐部分拆解

### login —— 通用框架 + 业务两层
概览已述:通用 `login.cpp`(头注释“不许更改”)+ 业务 `login_impl.cpp`。运行时只负责发票据,不参与战斗,这里不展开。

### online —— 世界主逻辑服(挂在 AsynServ 框架的 .so)
**入包分发**:所有客户端包先进 [online/cli_proto.cpp](../../pea/online/cli_proto.cpp) 的 `dispatch`:
- 登录包特判 + 防重登:`cli_proto_login_cmd` 时若已有同 uid 在线,给旧连接发 `ONLINE_ERR_MULTI_LOGIN` 并踢掉,见 [cli_proto.cpp#L116](../../pea/online/cli_proto.cpp#L116)。
- 命令派发:`find_cli_cmd_bind(cmd)` 在命令表里查处理函数,查不到回 `ONLINE_ERR_CMD_ID`,见 [cli_proto.cpp#L135](../../pea/online/cli_proto.cpp#L135)。
- **串行化机制**:每个玩家有 `waitcmd`,处理一个包期间后续包被 `cache_a_pkg` 缓存进 `pkg_queue`,等 `clear_waitcmd()` 后由 `proc_cached_pkgs` 重放,见 [cli_proto.cpp#L175](../../pea/online/cli_proto.cpp#L175)。这是淘米所有页游统一的“一个玩家一次只处理一条指令”模型。

**战斗转发层** [online/battle.cpp](../../pea/online/battle.cpp):online 自己不算战斗,它维护一张 battle_server 注册表(`battle_svrs[]`,从 `btlsvr.xml` 读 `SvrId`,见 [battle.cpp#L267](../../pea/online/battle.cpp#L267)),把客户端战斗指令打成 `btl_proto_t` 发给对应 battle_server:
- `btlsvr_player_attack` / `btlsvr_player_move` / `btlsvr_select_skill` 等一组转发函数,见 [battle.cpp#L599 起](../../pea/online/battle.cpp#L599)。
- 反向:battle_server 用 `online_proto_transmit_only_cmd` 回包,online 的 `btlsvr_transmit_only_callback` 把内层 `cli_proto_t` 拆出来直接转给客户端;**唯一特判**是公屏聊天 `cli_proto_chat_public_msg_cmd`——要解出发送者 uid 查黑名单,命中黑名单则不转发,见 [battle.cpp#L434](../../pea/online/battle.cpp#L434)。
- 战斗结束 `btlsvr_battle_end_callback`:收到 battle_server 的结算包后,向 btlsw 发 `btlsw_pvp_end_battle_cmd` 释放房间,再把每个玩家的命中率/伤害/经验拆包回传客户端,见 [battle.cpp#L456](../../pea/online/battle.cpp#L456)。

### new_battle_switch —— PVP 房间状态机
负责“开打前”的一切:房间列表、创建/加入/座位/队伍、发起开战。命令字是全模块注释最全的明文枚举:
- `btlsw_online_register_cmd = 11001`、`btlsw_pvp_create_room_cmd = 11003`、`btlsw_pvp_room_start_cmd = 11009`(通知 online 房间进入战斗态)、`btlsw_pvp_room_start_battle_cmd = 11015`(online 通知房主、btlsw 转发给全房玩家“加入对应 battle_server 正式开战”),见 [pea_btlsw_enum.h#L13](../../pea/new_battle_switch/proto/pea_btlsw_enum.h#L13)。
- 错误码带淘米黑话:`不能自宫!`、`你不是房间的主子/堂口/话事人/扛把子`,见 [pea_btlsw_enum.h#L33 起](../../pea/new_battle_switch/proto/pea_btlsw_enum.h#L33)。

### battle_server —— 真正跑战斗的进程(本模块技术核心)
这是赛尔号弹道对战的全部计算所在,核心类:
- `Battle` / `single_pvp_battle` —— 一场战斗的状态机(回合、玩家队列、胜负),见 [battle_impl.cpp](../../pea/battle_server/battle_impl.cpp)、[single_pvp_battle.cpp](../../pea/battle_server/single_pvp_battle.cpp)。
- `Player` —— 战斗内玩家对象,持 HP/攻防/暴击等属性 + 出战精灵 `pet`,见 [player.cpp](../../pea/battle_server/player.cpp)。
- `attack_obj` —— 一次开火产生的“攻击体”,按技能配置裂变出多波、多发子弹,见 [attack_obj.cpp](../../pea/battle_server/attack_obj.cpp)。
- `bullet` —— 单发子弹:飞行、碰撞、伤害/暴击结算,见 [bullet.cpp](../../pea/battle_server/bullet.cpp)。
- `phy_container` / `phy_model` —— 弹道物理引擎(带阻尼的运动积分),见 [pea_common/physical_engine.cpp](../../pea/pea_common/physical_engine.cpp)。
- 战斗内客户端协议处理 [battle_server/battle.cpp](../../pea/battle_server/battle.cpp):`online_proto_player_attack` / `_player_move` / `_select_skill` 等,这里才是真正改 HP、判生死的地方。

### dbserver / dbproxy —— 落库
- `Croute_func` 是 DB 路由总表:`g_db_cmd_list[]` 用 `BIND_PROTO_CMD` 宏从 4 个 `pea_db_bind_*.h` 拼出 cmd→成员函数映射,`find_db_cmd_bind(cmd)` 派发,见 [Croute_func.cpp#L39](../../pea/dbserver/Croute_func.cpp#L39)。
- 实际 DB 动作按 handler 拆分(`handle_user/handle_pet/handle_item/handle_mail/handle_task/...`),构造时全部注入同一个 `mysql_interface*`,见 [Croute_func.cpp#L80](../../pea/dbserver/Croute_func.cpp#L80)。例如取玩家 `db_proto_get_player` → `handle_user.query_user_info(...)`,见 [Croute_func.cpp#L98](../../pea/dbserver/Croute_func.cpp#L98)。
- 精灵相关 DB 动作很丰富:`db_proto_get_pet` / `db_proto_get_fight_pet` / `db_proto_set_pet_iq` / `db_proto_try_pet_train` / `db_proto_pet_merge`(精灵合成),见 [Croute_func.cpp#L308 起](../../pea/dbserver/Croute_func.cpp#L308)。这正是赛尔号“养精灵”玩法的落库面。

---

## 关键机制/算法实战

### 机制 1 ·【最有料】弹道物理:不是标准抛物线,是“带空气阻力的终端速度模型”
直觉以为子弹走 `x=v·t`、`y=v·t-½g t²` 的标准抛物线,**但 pea 不是**。[physical_engine.cpp#L75](../../pea/pea_common/physical_engine.cpp#L75) 的 `simulate_models` 用的是带阻尼系数 `b=0.05`、质量 `m=2.0` 的解析积分:

```
second = simulate_time / 33.0           // 33ms 一帧
coeff  = 1 - e^(-b·second/m)            // 指数衰减(空气阻力)
txl    = m·v_x / b                      // 水平“终端位移”
tyl    = m²/(v_a·b²) + m·v_y/b
x = init_x + txl·coeff                  // 水平随时间逼近 txl(终端速度)
y = init_y - (tyl·coeff - m·v_a·second/b)
```

含义:水平方向不是匀速,而是因阻力**逐渐逼近一个终端位移**;竖直方向是“阻力 + 重力 `v_a`”的复合。`GRAVITY_ACCELERATION` 在 `Battle::init_phy_container` 里设进容器,见 [battle_impl.cpp#L103](../../pea/battle_server/battle_impl.cpp#L103)。开火时把 `muzzle_velocity` 按炮口角拆成 `x_speed=v·cos θ`、`y_speed=v·sin θ` 灌进子弹,见 [attack_obj.cpp#L31](../../pea/battle_server/attack_obj.cpp#L31)。这就是为什么协议里要传 `gun_angle`(炮口角)+`muzzle_velocity`(蓄力出膛速度)——经典“百战天虫/弹弹堂”式蓄力瞄准手感。

**碰撞是逐帧步进 + 线段补判**:`process_attack_obj` 每 33ms `simulate_models` 推进一帧,然后判:出图(`out_of_area`)即销毁;撞地形(`process_collision(map_data)`)即炸;遍历对手判中(`process_collision(Player)`),见 [attack_obj.cpp#L87](../../pea/battle_server/attack_obj.cpp#L87)。由于一帧位移可能很大穿过薄墙,`bullet::line_collision` 还沿旧→新位置做**逐像素线段插值**补判命中,见 [bullet.cpp#L98](../../pea/battle_server/bullet.cpp#L98)。命中地形会用弹坑数据 `merge_bullet_crater_to_map` **破坏地形并触发所有人坠落判定**(`all_fall_down`),见 [bullet.cpp#L78](../../pea/battle_server/bullet.cpp#L78) + [battle_impl.cpp#L420](../../pea/battle_server/battle_impl.cpp#L420)——可破坏地形玩法实锤。

### 机制 2 · 伤害 / 暴击公式(逐行还原)
命中后走 `Player::on_hit` → `bullet::calc_damage`,见 [bullet.cpp#L176](../../pea/battle_server/bullet.cpp#L176):

```
atl_ex   = random(0,20)                                   // 攻击浮动
real_atk = ATK · (90 + atl_ex) / 100                     // 实际攻击 = 攻击×(90%~110%)
tmp      = max(real_atk - target_DEF, 0)                  // 减防,有保底 0
happy    = min(HAPPY, 100)                                // 亲密度/心情上限 100
damage   = tmp · happy / 100                              // 心情折算
if 暴击:  damage ·= (1 + CRIT_DAMAGE)                     // 暴伤倍率
if 技能:  damage  = damage · skill.damage_percent / 100   // 技能威力%(来自 skill.xml)
damage  ·= player_state_mrg->damage_trim()               // buff/debuff 修正系数
```

**暴击判定** `bullet::is_crit`(分等级两段),见 [bullet.cpp#L149](../../pea/battle_server/bullet.cpp#L149):

```
lucky_crit_rate = LUCK · CRIT · 5.4 / (target_lv + 4)         // 目标 ≤60 级
lucky_crit_rate = LUCK · CRIT · 5.4 / (target_lv+2·(lv-59))   // 目标 >60 级
暴击 ⇔ random(0,100) ≤ lucky_crit_rate
```

即:攻方 **幸运×暴击** 决定暴击率,被目标等级稀释(高级目标更难被暴击)。结算完 `dec_hp(damage)`,再用 `notify_all_obj_attr_chg_delay` 把新 HP + 暴击标志广播给全房客户端做表现,死亡则 `notify_all_player_dead_delay`,见 [player.cpp#L406](../../pea/battle_server/player.cpp#L406)。**带 `bullet_times`/`syn_number` 的 “delay” 系列协议** 是为了让客户端把“第几波第几发子弹命中”和动画对上(战斗驱动在服务端,客户端只是回放)。

### 机制 3 · 回合机:出手队列 + 50% 连击 + 双计时器
`Battle::on_battle_timer` 是心脏(每 tick 调用),见 [battle_impl.cpp#L571](../../pea/battle_server/battle_impl.cpp#L571):
- **出手顺序**:`calc_player_turn` 把所有活人推进 `btl_turn_queue`,按 `turn_value` 排序(速度/敏捷决定先手),队列空了就开新回合 `inc_round_num`,见 [battle_impl.cpp#L677](../../pea/battle_server/battle_impl.cpp#L677)。
- **连击**:每次轮空时先 `calc_player_turn_again` 掷骰——注释写“50% 几率连击”,但实际代码是 `random(1,100) >= 200` **永远为假**(疑似留了个写死的开关 / 上线前关掉),见 [battle_impl.cpp#L665](../../pea/battle_server/battle_impl.cpp#L665)。**(此为代码事实:连击当前不可能触发)**
- **双计时器逼迫出手**:轮到某人后先给 `ready_attack_timer`(准备攻击),客户端发 `ready_attack` 才转 `attack_timer`(可开火窗口);任一超时即 `goto_next_round` 强制换人,见 [battle_impl.cpp#L625](../../pea/battle_server/battle_impl.cpp#L625) 与开火校验 [battle.cpp#L288](../../pea/battle_server/battle.cpp#L288)。
- **胜负**:`single_pvp_battle::check_battle_end` 按红蓝队 `team` 全灭判定 + 设 `win_team`,见 [single_pvp_battle.cpp#L9](../../pea/battle_server/single_pvp_battle.cpp#L9)。

### 机制 4 · attack_obj 弹幕裂变 + 数据驱动
一次开火不只一发。`attack_obj::init_attack_obj` 按技能配置 `bullet_count_per_time`(每波几发)以 `bullet_excursion_angle`(扇形间隔角)裂变,`bullet_times` 控制连续几波,见 [attack_obj.cpp#L21](../../pea/battle_server/attack_obj.cpp#L21)。这些数全来自 XML 三件套:
- [skill.xml](../../pea/battle_server/conf/skill.xml):`damage_percent`(技能威力%)、`consume_anger`(消耗怒气)、`consume_strength`、`target`(1自己/2队友/3目标/4全体)、`recove_hp`(吸血%),载入逻辑见 [skill.cpp#L52](../../pea/battle_server/skill.cpp#L52)。
- [bullet.xml](../../pea/battle_server/conf/bullet.xml):`width/height`(碰撞盒)、`damage_radius`(溅射半径)、`bullet_crater_id`(弹坑→破地形),载入见 [bullet.cpp#L234](../../pea/battle_server/bullet.cpp#L234)。
- [buff.xml](../../pea/battle_server/conf/buff.xml)+[effect.xml](../../pea/battle_server/conf/effect.xml):buff/debuff(优先级互斥、触发次数、维持回合)。
**怒气(anger)**:作为玩家属性 `OBJ_ATTR_ANGER` 由 online 在 `syn_player_info` 阶段灌入 battle_server(见 [player.cpp#L136](../../pea/battle_server/player.cpp#L136)),技能 `consume_anger` 是放大招的消耗——这是赛尔号“蓄怒放必杀”的服务端依据。

### 机制 5 · gen_proto:XML → C++/Python/PHP 协议代码自动生成
pea 的协议**不是手写常量**,是构建期生成的。流程在 [gen_proto/getxml.sh](../../pea/gen_proto/getxml.sh):
1. `wget http://10.1.1.5/su/proto_test/download.php?project=<名>` 从淘米内网协议管理站拉 **协议定义 XML**(命令号/字段/结构体在那台 10.1.1.5 上集中维护,泄漏源码里没有 XML 本体——这就是概览说“online cmdid 未入库”的根因)。
2. `cpp_proto` 解析 XML,套插件 `--plugin_name_list=gen_db_bind_func.py,gen_online.py` 生成 C++:每条命令产出 `int <name>(DEFAULT_ARG){ ... p_in=P_IN; ... }` 处理函数骨架,见生成器 [plugin/gen_online.py#L8](../../pea/gen_proto/plugin/gen_online.py#L8);DB 侧并生成 `BIND_PROTO_CMD` 绑定表(被 `Croute_func.cpp` include)。
3. `py_proto`/`php_proto` 同源产出 Python/PHP 版,供测试与 Web 端。
生成的 `*_func_def.h`/`*_bind.h`/`*.h`(结构体)最终被 [online/proto.hpp](../../pea/online/proto.hpp) 用 `PROTO_FUNC_DEF` 宏三遍展开:声明客户端/btlsw 处理函数、声明 db `_callback`、定义结构体,见 [proto.hpp#L44](../../pea/online/proto.hpp#L44)。**所以 pea/mole2 共用 `mole2_online_proto.py` 生成器,本质是同一套协议工具链。**

---

## 端到端调用流举例:一次 PVP「瞄准开火击杀对手」

追一发子弹从客户端按钮到对手掉血的完整链路:

1. **撮合**(略):客户端在 new_battle_switch 建房/坐下,online 发起 `btlsw_pvp_room_start_battle_cmd=11015`,btlsw 通知全房玩家连同一 battle_server。
2. **同步入场**:每人发 `syn_player_info`,online `btlsvr_syn_player_info` 把该玩家 HP/经验/等级/魔法/敏捷 + 出战精灵打成 `btl_proto_t` 发给 battle_server,见 [online/battle.cpp#L511](../../pea/online/battle.cpp#L511);battle_server `online_proto_syn_player_info` 建 `Player`、灌属性/精灵/技能,人齐(`check_syn_players`)就广播 `load_resource`,见 [battle_server/battle.cpp#L111](../../pea/battle_server/battle.cpp#L111)。全员 `load_resource` 完成 → `notify_all_battle_start` 开打,见 [battle_server/battle.cpp#L147](../../pea/battle_server/battle.cpp#L147)。
3. **轮到我**:`on_battle_timer` 从出手队列取到我,广播 `player_turn`,起 `ready_attack_timer`,见 [battle_impl.cpp#L596](../../pea/battle_server/battle_impl.cpp#L596)。
4. **瞄准/蓄力开火**:客户端发 attack(带 `gun_angle`、`muzzle_velocity`)。online `btlsvr_player_attack` 转给 battle_server,`online_proto_player_attack` 校验(角度≤360、回合/状态/计时器),取当前技能,广播 `player_attack`,然后 **`create_attack_obj` + `process_attack_obj` 在服务端把整个弹道一次性算完**,见 [battle_server/battle.cpp#L270](../../pea/battle_server/battle.cpp#L270)。
5. **弹道结算**:子弹按阻尼模型逐帧飞(机制1),命中对手 → `bullet::after_collision` → `target->on_hit` → `calc_damage`(机制2)算出伤害+暴击 → `dec_hp` → `notify_all_obj_attr_chg_delay` 广播掉血,见 [bullet.cpp#L51](../../pea/battle_server/bullet.cpp#L51)+[player.cpp#L406](../../pea/battle_server/player.cpp#L406)。HP 归零 → `notify_all_player_dead_delay`。
6. **判胜负 + 换人**:开火后 `check_battle_end`,全灭则置 `battle_end_state`;否则 `goto_next_round`,见 [battle_server/battle.cpp#L327](../../pea/battle_server/battle.cpp#L327)。
7. **结算落库**:战斗终态 → `notify_all_battle_end` 把每人命中率/总伤害/经验回传 online,见 [battle_impl.cpp#L932](../../pea/battle_server/battle_impl.cpp#L932);online `btlsvr_battle_end_callback` 拆包发客户端 + 向 btlsw 释放房间,经验/等级变更再经 `db_proto_change_exp`/`db_proto_level_up` 走 dbserver 落 MySQL,见 [online/battle.cpp#L456](../../pea/online/battle.cpp#L456) + [Croute_func.cpp#L271](../../pea/dbserver/Croute_func.cpp#L271)。

**一句话**:战斗是 **服务端权威**(弹道/伤害全在 battle_server 算,客户端只传角度速度并回放结果),online 是“战斗指令的邮局 + 落库网关”,btlsw 只管开打前的房间。

---

## 复活实操提示(对 moleworld 复活)

定位仍是 **架构参照,不直用**(pea 是赛尔号,玩法/内容数据与摩尔庄园无关)。但本次深挖出几条可迁移的硬货:

**可直接借鉴的运行时模型**(跨游戏通用,摩尔庄园同框架):
- **入包串行化 + 缓存重放**(`waitcmd` + `pkg_queue` + `proc_cached_pkgs`,[cli_proto.cpp#L175](../../pea/online/cli_proto.cpp#L175)):摩尔庄园 online 同样需要“一玩家一次一指令”,这是可照抄的并发模型。
- **命令绑定表 + find_*_cmd_bind 派发**([proto.hpp#L44](../../pea/online/proto.hpp#L44)、[Croute_func.cpp#L39](../../pea/dbserver/Croute_func.cpp#L39)):还原摩尔庄园协议派发骨架的最佳模板。
- **online↔dbserver 的 handler 分库模式**(一个 `mysql_interface` 注入多个 `handle_*`,[Croute_func.cpp#L80](../../pea/dbserver/Croute_func.cpp#L80)):复活 DB 层时直接套这个分模块结构。

**协议工具链**:[gen_proto/getxml.sh](../../pea/gen_proto/getxml.sh) 揭示了淘米协议从 `10.1.1.5` 协议站 XML 自动生成 C++/Py/PHP 的全流程——**复活摩尔庄园若拿不到协议站 XML,只能从生成出来的 `.h`/`.cpp` 反推**;`mole2_online_proto.py` 残留的 mole2 定义对还原摩尔庄园 2 客户端协议比 pea 本体更值钱,值得单独翻。

**需要自己补的**:
- battle_server 的弹道/伤害是赛尔号专属,摩尔庄园用不上(摩尔庄园无 PVP 弹道战斗)。
- 战斗的 `online_proto_transmit_only_cmd` 双层包封装(外 `btl_proto_t` 内 `cli_proto_t`,[battle.cpp#L434](../../pea/online/battle.cpp#L434))是“跨进程透传客户端协议”的范式,若 moleworld 将来要做多进程副本可参考,但当前单服复活不需要。
- **无 DB schema**:`.sql` 不在库,建表结构要从 dbserver 各 handler 的 SQL 字符串反推(`Croute_func.cpp` + `dbserver/pet.cpp`/`user.cpp`/`item.cpp`)。

---

回总入口:[../../README.md](../../README.md) · 概览:[../modules/pea.md](../modules/pea.md) · 协议:[../protocol.md](../protocol.md) · 同框架兄弟深档:[mole2.md](./mole2.md)
