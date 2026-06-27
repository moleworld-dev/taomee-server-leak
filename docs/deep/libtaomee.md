# libtaomee 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/libtaomee.md](../modules/libtaomee.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)

本文是概览的"深水区"补充。概览回答"它是什么";本文回答"它在运行时到底怎么跑、跟谁交换数据、算法怎么算"。

**取证方式**:本模块在仓库里是两个发布版 tar.gz——[Libtaomee-0.7.5.tar.gz](../../Libtaomee-0.7.5.tar.gz)(C 版)与 [Libtaomee++-0.7.2.tar.gz](../../Libtaomee++-0.7.2.tar.gz)(C++ 版)。tar 内文件无法直接 GitHub 链接,下文凡形如 `libtaomee/interprocess/shmq.c:L145` 的引用都来自**解包到临时目录后真实 Read 的行号**(只解到 /tmp,未释放进仓库)。凡能落到仓库工作树里(各游戏对框架的真实调用)的引用,用可点击的 `../../<路径>#L行号`。

---

## 一图看懂:进程/组件拓扑与数据流

libtaomee 不是一个跑起来的进程,而是被各游戏服务进程**编译进去**的工具箱。它的代码运行在调用方进程的地址空间里。真正"在多个进程之间流动数据"的只有一件:**shmq 共享内存队列**。

```
            ┌────────────────────────────────────────────────────────┐
            │  一个游戏服(如 mole/gamesvr、gf/gongfu_online)          │
            │  ── 进程内直接 #include + -ltaomee / -ltaomee++ 链接 ──  │
            │                                                          │
            │  收发包: proto_base(字节序) ─▶ 业务 ─▶ qdes(DES加密票据) │
            │  时间轴: timer(秒/毫秒级回调) + time(now/tm_cur 全局)    │
            │  配置:   conf_parser/config(C) · xmlparser/item_manager  │
            │          (C++,把 items.xml 解析成内存表)                 │
            │  日志:   log(旧) / tlog(新,带 gameid+svrtype+uid 追踪)   │
            │  合规:   tm_dirty(AC自动机脏词) · verification_image(验证码)│
            └───────────────┬──────────────────────────────────────────┘
                            │ shm_cq_push / shm_cq_pop
                            │ (POSIX shm_open + mmap 的环形队列)
                            ▼
            ┌────────────────────────────────────────────────────────┐
            │  另一进程(同机): dispatcher⇄worker、写库进程、统计 agent  │
            │  单读单写,一条队列只许一个进程 push、另一个进程 pop      │
            └────────────────────────────────────────────────────────┘
                            │ tlog 写盘  + tm_dirty/stat_agent 走 UDP
                            ▼
                     磁盘日志目录 / 运维上报服务器
```

要点:**框架的"数据流"基本只在 shmq 这一层是跨进程的**;其余(加密、时间、配置、日志)都是进程内的库函数调用。理解 libtaomee 的运行行为 = 理解 shmq 怎么在两个进程间安全传包 + qdes 怎么算加密 + 各游戏怎么把 XML 配置吃进内存表。

---

## 逐部分拆解(实际职责 + 关键函数 + 跟谁通信)

### 1. interprocess/shmq —— 进程间通信的命脉(单读单写环形队列)

这是整个框架里唯一真正"跨进程搬数据"的部件,也是最值得细读的。头文件开宗明义:`shmq.c:L4` "目前的 shm_cq 只适用于一个进程只读,另外一个进程只写"。

**它在共享内存里铺了什么(`shmq.c:L34-46` 的 `shm_cq` + `L20-23` 的 `shm_block`)**:
- 队头结构 `shm_cq` 含 `head`/`tail`(都是 `volatile uint32_t` 偏移量)、`shm_size`、`elem_max_sz`、`name[64]`。head/tail 用 volatile 是因为读写分属两个进程,要防编译器把它缓存进寄存器。
- 每条数据是一个 `#pragma pack(1)` 的 `shm_block`:4 字节 `len` + 变长 `data[]`。`len` 把"块头 4 字节"也算进去(`shmq.c:L149` `elem_len = len + sizeof(shm_block_t)`)。

**创建/依附(`shmq.c:L40-114`)**:
- `shm_cq_create`(`L40`)= `shm_open(O_CREAT|O_EXCL)` + `ftruncate` 撑大 + `mmap(MAP_SHARED)`,然后把 head=tail=`sizeof(shm_cq_t)`(数据区从结构体之后开始)。写进程调它。
- `shm_cq_attach`(`L87`)有个巧妙两段式:先只 mmap 一个 `sizeof(shm_cq_t)` 拿到 `shm_size`,munmap 后再用真实大小 mmap 一次(`L95-106`)——因为 attach 方事先不知道队列多大。读进程调它。

**push(写进程,`shmq.c:L145-161`)实际怎么跑**:
1. 算 `elem_len`(数据+4字节块头)。
2. `shm_cq_align_tail`(`L187`):如果 tail 到缓冲区末尾的剩余空间够,直接进 `shm_cq_push_wait`;不够,就尝试**环回**——在末尾如果还塞得下 4 字节,写一个 `len=0xFFFFFFFF` 的"哨兵填充块"(`L196-199`)标记"此后到末尾是废料,读端请跳回开头",然后把 tail 重置回 `sizeof(shm_cq_t)`。
3. `shm_cq_push_wait`(`L222`):自旋判断 head 是否离 tail 够远(`head >= tail + len + elem_max_sz`,`L229`),多留一个 `elem_max_sz` 是"防止覆盖读端当前正引用的那块"。最多自旋 10 次、每次 `usleep(5)`,满了就返回 -1(push 失败)。
4. 真正写入:`shm_cq_tail()` 定位 tail,写 `len`、`memcpy(data)`、`tail += elem_len`(`L154-158`)。**注意:tail 的推进是最后一步**,保证读端只会看到完整写好的块。

**pop(读进程,`shmq.c:L121-143`)**:
1. `head == tail` 即空,返回 0。
2. `shm_cq_align_head`(`L174`):如果 head 走到了缓冲区尾部不足一个块头、或撞上 `0xFFFFFFFF` 哨兵(`L182`),就把 head 跳回 `sizeof(shm_cq_t)`(对应 push 端的环回)。
3. `*data = cur_mb->data`(直接给出**共享内存内的指针,零拷贝**)、`head += cur_mb->len`,返回 `len - 4`(去掉块头的纯数据长度)。

**它跟谁通信**:任意两个同机进程,只要约定同一个 `name`。摩尔庄园/功夫派等都把它当作 dispatcher↔worker、在线服↔写库进程之间传协议包的管道。各游戏 `bench.conf` 里大量出现 shmq 相关配置(如 [gf/gongfu_online/bench.conf](../../gf/gongfu_online/bench.conf)、[gf/trade_svr/bench.conf](../../gf/trade_svr/bench.conf))。

**坑(直接影响复活时移植)**:无锁、靠 volatile+操作顺序保证单读单写正确;`head==q->tail` 既表示空也是环回判断的基准,**一条队列绝不能两个进程同时 push 或同时 pop**;自旋 10 次失败即丢——上层必须处理 push 返回 -1。`samples` 里附了最小可复现的读写对:[push 示例](../../Libtaomee-0.7.5.tar.gz)(tar 内 `interprocess/shmq_push.c:L29`,`shm_cq_create("shmcq_test",1000,600)` 后 `while(shm_cq_push(...))` 死等)与 `shmq_pop.c:L28`(`shm_cq_attach` 后死循环 `shm_cq_pop`)。

### 2. crypt/qdes —— DES ECB,登录/积分票据的统一加密

`qdes.c` 是一份精简的 DES ECB 实现(从 OpenSSL 风格的 `PERM_OP`/`HPERM_OP` 宏 + IP/FP 置换表移植而来,`qdes.c:L16-19`)。对外只暴露 4 个函数:

- `des_encrypt(key, s_text, d_text)`(`qdes.c:L424`)= `des_set_key` 生成 16 轮子密钥 + `des_ecb_encrypt(...,1)`,加密**恰好 8 字节**一块。
- `des_decrypt`(`L431`)同上,最后参数传 0 即解密。
- `des_encrypt_n`/`des_decrypt_n`(`L438`/`L453`):对 8 字节的 n 倍数据循环跑 ECB(`for` 里 `s_text += 8; d_text += 8`,`L446-450`)。**纯 ECB、无 IV、无填充**——每个 8 字节块独立加密。

**key 只取前 8 字节**(头注释 `qdes.h:L20`),这正好对应淘米到处用的"8 字符串当 key"的习惯。

**它实际被谁用、用哪把 key——这里有一个概览里值得校正的细节**:

- 框架自带的"登录 session 加密 key"在 `libtaomee/project/constants.h:L35-39`:默认 `LOGIN_DES_KEY = "!tA:mEv,"`,台湾版(`TW_VER`)= `,.ta0me>`。这是**框架层面给登录 session 用的**约定 key。
- 但摩尔庄园 gamesvr **另有一把自己的 key 用于"积分票据(score session)"**:[mole/gamesvr/game.c #L324-L326](../../mole/gamesvr/game.c#L324) 里 `des_key = ">ta:mee<"`(`TW_VER` 时为 `,.ta0me>`),用在 [game.c #L320-L345](../../mole/gamesvr/game.c#L320) 的 `pack_score_session`——把"time+ip / time+score / userid+gameid"各凑成 8 字节,用 `des_encrypt` 加密成三段拼进 `gs->sess`(`#L335`/`#L339`/`#L344`)。

所以同一套 DES ECB 实现下,**至少存在两把不同的 8 字节 key:`!tA:mEv,`(框架登录 session)与 `>ta:mee<`(mole 小游戏积分票据)**,台湾版都切到 `,.ta0me>`。复活时要按"加密的是哪种票据"选对应 key,不能一把通吃。

**它跟谁通信**:不通信,纯计算函数。但它产出的密文(session/票据)会随协议包发给客户端 / 在服务间传递,是协议层能不能对得上的硬契约。

### 3. log / tlog —— 两代日志,tlog 带在线追踪

- **旧 log**(`libtaomee/log.h`/`log.c`):9 级、按时间自动轮转、每条 ≤8000 字节、可开多线程写。`c/` 框架的 README 注明其日志就是抄自老版 Libtaomee。
- **新 tlog**(`libtaomee/tlog/`,0.7.4+):核心是一条带"业务身份"的日志。初始化宏 `INIT_DEFAULT_LOGGER_SYSTEM(logdir, pfx, svc, gameid, svrtp)`(tar 内 `tlog/tlog.h:L10`)把 **服务名 / gameid / 服务器类型** 一起注册(`init_looger_svc_info`,`tlog.h:L28`)——这意味着日志里天然带"哪个游戏、哪类服务器"标签,是淘米多游戏统一运维的产物。
- 写日志走 `write_to_logger(lvl, logtype, uid, flag, fmt, ...)`(`tlog.h:L31`):**带 uid**,配合 `is_traced_uid`(`tlog.h:L38`)可只追踪被标记的玩家。
- 配套在线工具在 tar 内 `tlog/tools/`:`set_trace_uid`、`set_trace_addr`、`set_rate_limit`、`ctrl_tlog`——运行中不重启即可"打开对某 uid 的全量日志""按地址追踪""限流",是线上排障设施。

**它跟谁通信**:主要写本地磁盘;trace 工具通过控制通道(`ctrl_tlog`)动态改运行中进程的日志行为。

### 4. timer / time —— 全局时钟与秒/毫秒级回调

- `time/time.h` 维护两个**全局变量** `struct timeval now` 与 `struct tm tm_cur`(在 timer.h 里 `extern`,tar 内 `timer.h:L30-31`)。整套淘米服务都靠"主循环每帧刷一次 now"来避免频繁 `gettimeofday`——业务代码里到处直接读 `now.tv_sec`。
- `timer.h` 提供秒级(`timer_struct`,`L48`)与毫秒级(`micro_timer_struct`)两套定时器,都用 `libtaomee/list.h` 的内核风格侵入式链表挂接。回调签名 `int (*timer_cb_func_t)(void*, void*)`(`timer.h:L37`),**回调返回 0 表示到期后删除该定时器,非 0 表示保留**(`L35` 注释)——这是"一次性 vs 周期性"定时器的开关。
- 关键函数:`setup_timer`/`destroy_timer`(`L102`/`L107`)、`scan_seconds_timer`/`scan_microseconds_timer`(`L109`/`L110`,主循环每帧调用扫描到期项)、`register_timer_callback(nbr, cb)`(`L353`)按"类型编号"登记回调,定时器结构里只存 `func_indx`(`timer.h:L56`)而不存函数指针——这样定时器能放进共享内存/序列化而不怕地址失效。
- `timer_struct` 里有 `sprite_list`(`timer.h:L50`)字段,直白暴露了它服务于"精灵/玩家对象"的定时逻辑(摩尔庄园的 sprite)。

**它跟谁通信**:进程内主循环。是几乎所有游戏逻辑(buff 到期、采集冷却、定时刷怪)的时间底座。

### 5. inet —— TCP/组播/IP 工具

`libtaomee/inet/`:`tcp.c`(阻塞/非阻塞切换、收发超时、`accept`/`connect` 封装,`tcp.h:L24-71`)、`mcast.c`(组播 socket,mole 用它做跨服广播,见 [mole/gamesvr/mcast.c](../../mole/gamesvr/mcast.c#L11))、`inet_utils`(`is_legal_ip/port`)、`ipaddr_cat.h`(按网卡名取本机 IP)。都是薄封装,服务于"服务怎么把 socket 建起来"。

### 6. tm_dirty —— 脏词过滤(AC 自动机)+ 广告上报

`libtaomee/tm_dirty/`:核心是 `acsm.c`(Aho-Corasick 多模式串匹配自动机),一次扫描就能在文本里命中词库 `tm_dirty.dat` 中的所有敏感词;还处理 UTF-8 标点、支持在线更新词库;命中"广告"类时通过 **UDP 上报**到广告检测服务器。是聊天/昵称/UGC 合规链路的入口,各游戏 `bench.conf` 需配脏词服务器地址。

### 7. project —— 跨游戏账号常量与类型

- `constants.h`:`base_user_id = 50000`(起始米米号)、`base_guest_id = 2000000001`(起始游客号)(`constants.h:L21-26`),以及上文的 `LOGIN_DES_KEY`。这些是**所有淘米游戏共享的账号体系常量**。
- `types.h`:`typedef uint32_t userid_t`(`types.h:L26`)——全淘米用 32 位无符号整数当用户号。
- `stat_agent/msglog`:统计打点(走 hash_table + shm + UDP agent)。

### 8. libtaomee++ 的配置解析全家桶(C++ 侧主力)

- `conf_parser/xmlparser.hpp`:libxml2 的模板化封装。核心是一组 `get_xml_prop` 模板(`xmlparser.hpp:L55` 起):`get_xml_prop_arr` 把一个属性值按 dec/oct/hex 读进数组、`get_xml_prop`/`get_xml_prop_def`(带默认值)读单值,解析失败抛 `XmlParseError`(继承 `std::logic_error`)。这是各游戏读配置表的统一入口。
- `conf_parser/ini_parser.hpp`:`IniParser`,readme 示例用的就是它。
- `conf_parser/item_manager.hpp`:**把 items.xml 解析成内存二维表的公共模板**(详见下一节)。

---

## 关键机制实战之一:item_manager 怎么把 items.xml 变成 O(1) 内存表

`ItemManager<ITEM, MAX_CAT, MAX_ELEM>`(tar 内 `item_manager.hpp:L198`)是 C++ 侧最有料的设计,gf 把它当物品系统的地基。它实际怎么跑:

**数据布局**:成员是一个**编译期定死大小的二维数组** `ITEM items_[MAX_CAT][MAX_ELEM]`(`item_manager.hpp:L242`)。物品 ID 编码规则写死在 `item_base_item_id = 100000`(`L224`):

- 存:类别 cat 的物品基号 = `cat * 100000 + 1`(`L318`),物品落在 `items_[cat][id - base_id]`。
- 取:`get_item(id)`(`L260`)直接算 `idx1 = (id-1)/100000`、`idx2 = (id-1)%100000`,边界 + `id` 校验后返回指针——**纯下标寻址,O(1),不查哈希表**。这是用"ID 段编码 = 数组下标"换极致查表速度的经典做法。

**解析流程**(`reload_items`,`L272`):`xmlReadFile(..., XML_PARSE_NOBLANKS)` → 遍历根节点下的 `<Cat>`(读 `ID`/`DbCatID`/`Max` 属性,`L294-298`)→ `load_an_item`(`L315`)遍历每个 `<Item>`,读 `ID`/`Max`/`Price`/`SellPrice` 和两个位标志属性:

- `Tradability`(默认 3):bit1=全员可买、bit2=全员可卖。
- `VipTradability`(默认 3):bit1/bit2=VIP 可买/可卖。
- `VipOnly`(默认 0):仅 VIP 可用。
这些用 `bitmanip` 的 `set_bit_on/test_bit_on` 压进一个 `uint8_t flag_`(`Item` 类,`L82-102`、`L173`)。

**扩展点**:`load_an_item` 的最后会调 `virtual int load_item_extended_info(item, cur)`(`L240`、`L365`),默认返回 0(只解析公共字段)。**游戏要解析自己特有的物品字段,就继承 ITEM + 覆盖这个虚函数**——这是"公共部分统一、私有部分各游戏自定义"的契约。

**gf 的真实用法(落到仓库工作树,可点击)**:功夫派把这套模板**整份 vendoring 进自己的 [gf/kf_common/src/item_manager.hpp](../../gf/kf_common/src/item_manager.hpp)** 并大幅扩写(框架原版 379 行,gf 版超过 1261 行,加入了 `load_item_extended_info(GfItem*, ...)` 的实体实现,见 [#L829](../../gf/kf_common/src/item_manager.hpp#L829)、[#L988](../../gf/kf_common/src/item_manager.hpp#L988))。然后 [gf/kf_common/src/item_impl.hpp #L19](../../gf/kf_common/src/item_impl.hpp#L19) 定义:

```cpp
const int max_cat  = 30;
const int max_elem = 100000;
class GfItemManager : public ItemManager<GfItem, max_cat, max_elem> { ... };
```

即功夫派物品空间 = 30 类 × 10 万件的静态二维数组,实例化在各服:[gf/battle_svr/dll_interface.cpp #L98](../../gf/battle_svr/dll_interface.cpp#L98) `items = new GfItemManager("./conf/items_nor.xml", ...)`、[gf/trade_svr/item.cpp #L11](../../gf/trade_svr/item.cpp#L11) 全局 `GfItemManager* items`。**这本身是一个值得记的发现:gf 不是简单继承框架的 item_manager,而是把它 fork 出去自己魔改了** —— 说明框架的"公共部分"在落地时被各游戏当成可改的样板,而非锁死的库。

## 关键机制实战之二:shmq 的环回与"哨兵填充块"

复述上文最精妙的一处:环形队列在 tail 撞到缓冲区尾、剩余空间塞不下整块时,**不是简单从头覆盖**,而是:

1. 若末尾还能写下 4 字节块头,就写一个 `len = 0xFFFFFFFF` 的填充块作哨兵(`shmq.c:L196-199`);
2. tail 重置回数据区起点(`L200`)。
3. 读端 `shm_cq_align_head` 遇到这个 `0xFFFFFFFF` 或剩余不足一个块头,就把 head 也跳回起点(`shmq.c:L182-184`)。

配合 push 端"tail 最后才推进"和 `volatile` head/tail,实现了**无锁单读单写**的安全环回。`shm_cq_push_wait` 里多预留一个 `elem_max_sz` 的间距(`L229`),专门防止写端追上读端、覆盖掉读端正用 `*data` 引用着的那块零拷贝数据。

---

## 端到端调用流举例:摩尔庄园小游戏积分上报票据(mole)

追一条真实链路——玩家在摩尔庄园玩完一个小游戏,服务端要把"分数"封成防篡改票据回传:

1. **触发**:小游戏结束,gamesvr 进入 [mole/gamesvr/game.c](../../mole/gamesvr/game.c) 的积分处理,`p->waitcmd = proto_game_score`([#L317](../../mole/gamesvr/game.c#L317))。
2. **取 key**:`pack_score_session`([#L320](../../mole/gamesvr/game.c#L320))按编译宏选 key——非台湾版 `>ta:mee<`,台湾版 `,.ta0me>`([#L324-L326](../../mole/gamesvr/game.c#L324))。
3. **凑 8 字节明文块 + DES 加密三段**([#L329-L344](../../mole/gamesvr/game.c#L329)):
   - 块1 = `now(4字节时间) + remote_ip(4字节)` → `des_encrypt`;
   - 块2 = `now + score` → `des_encrypt`;
   - 块3 = `userid + gameid` → `des_encrypt`;
   每段 8 字节密文 `PKG_STR` 拼进 `gs->sess`。
4. **DES 实现**:三次 `des_encrypt` 全部落到框架 [Libtaomee-0.7.5.tar.gz](../../Libtaomee-0.7.5.tar.gz) 内 `crypt/qdes.c:L424` 的 ECB 实现(`des_set_key` + `des_ecb_encrypt`)。
5. **回包**:`gs->sess`(24 字节 = 3×8)随积分协议包发回客户端,字节序遵循协议约定(C++ 侧若参与则走 `proto_base.h` 的小端约定,见 [../protocol.md](../protocol.md))。
6. **校验闭环**:积分入库/积分服收到票据时,用同一把 key `des_decrypt` 还原出 time/ip/score/uid/gameid,核对一致性——time+ip 防重放、uid+gameid 防冒名,这就是"积分不可被客户端随意伪造"的实现。

**这条链路对复活的意义**:要让真实 Flash 客户端的积分上报对得上,复活服务端必须复刻"3 个 8 字节块的精确字段顺序 + 同一套 DES ECB + 正确的 key"。字段顺序、key、ECB 三者缺一,客户端票据就解不开。

---

## 复活实操提示(对 moleworld 复活)

**能直接用 / 必须复刻(硬契约)**:
- **DES ECB 算法**:`crypt/qdes.c` 是标准 DES ECB,任何语言都有等价实现(Node `crypto` 的 `des-ecb`、Rust `des` crate、Go `crypto/des`),**关键是无 IV、无填充、8 字节一块、key 只取前 8 字节**。
- **key 表(逐场景)**:登录 session = `!tA:mEv,`(台湾版 `,.ta0me>`,见 `project/constants.h:L35`);mole 小游戏积分票据 = `>ta:mee<`(台湾版 `,.ta0me>`,见 [game.c #L324](../../mole/gamesvr/game.c#L324))。**不要把概览里只提到的 `>ta:mee<` 当成唯一 key**——它只是积分票据那把。
- **积分票据字段布局**:严格照 [game.c #L329-L344](../../mole/gamesvr/game.c#L329) 的"time+ip / time+score / uid+gameid"三块顺序。
- **账号常量**:`base_user_id=50000`、`base_guest_id=2000000001`、`userid_t = uint32_t`,照搬即可与真实数据/客户端假设一致。

**要自己补 / 替换的**:
- **shmq**:复活若用单进程(Rust async / Node 单进程)就**不需要** shmq,它是淘米多进程架构的产物;若坚持多进程,直接用 POSIX shm 复刻这套单读单写环形队列即可,逻辑见本文第 2 节,注意自旋失败要丢/重试。
- **item_manager**:配置解析思路(ID 段编码 = 数组下标的 O(1) 物品表、`Tradability`/`VipTradability`/`VipOnly` 位标志、公共字段+扩展虚函数)非常值得照搬,但 XML 表本身是 gf 的,摩尔庄园物品表需另找(见 [../content-data.md](../content-data.md))。
- **timer / time**:"全局 now + 每帧扫描定时器链表 + 回调返回值决定一次性/周期"的模型可直接照抄到任意事件循环里。
- **tm_dirty / verification_image / tlog / inet**:属运维与合规设施,对"先把游戏跑起来"非必需,后置。

**坑**:整套面向 gcc4.1.2 / 32-64 位 Linux,大量 `__attribute__((packed))`、内核风格链表、`volatile`-而非原子-的并发假设;移植到现代环境/语言时,packed 结构的字节序与对齐、shmq 的内存序保证都要重新审视,不能假定原样可移植。

---
返回:[总入口 README](../../README.md) · 概览:[libtaomee 模块文档](../modules/libtaomee.md) · 相关:[c 底座库](../modules/c.md) · [mole 摩尔庄园](../modules/mole.md) · [gf 功夫派](../modules/gf.md) · [protocol 协议专题](../protocol.md) · [content-data 内容数据](../content-data.md)
