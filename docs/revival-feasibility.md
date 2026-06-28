# 摩尔庄园(网页 Flash 版)复活可行性报告

> 研究日期 2026-06-28 · 总分析师综合稿
> 综合自 6 路研究代理:编译 / 运行 / 数据库 / 协议 / 客户端接线 / 前人经验
> 仓库(均在本地):
> `LEAK = /Users/xiaochoumao/Documents/github repo/taomee-server-leak`
> `RUFFLE = /Users/xiaochoumao/Documents/github repo/MoleRuffle`
> `WAYBACK = /Users/xiaochoumao/Documents/github repo/moleworld-wayback-archive`
> `MOBILE = /Users/xiaochoumao/Documents/github repo/moleworld-server`(iOS 手游线,仅作对照)

---

## 1. TL;DR / GO-NO-GO

**一句话结论:可以做,而且这是目前已知**起点最高**的一条复活路线——源码、协议、客户端运行时、客户端资产四大要件全部在手,GO。**

- **总体可行性评级:有条件可行(中等难度,★★★☆☆)。** 不是"缺东西"型的不可行,而是"考古环境 + 补构建胶水 + 重组 schema + 回填几个协议常量"型的工程活。每一块都有真实证据支撑、每一个缺口都有明确的填补办法。
- **GO-NO-GO:GO(分阶段)。** 建议以"登录进村庄走两步"为 M2 验收点先跑通端到端最小闭环,再谈完整玩法。
- **最可能成功的端到端路径(一句话):** 用 CentOS 6 era 老容器编出 `flashpolicy/login/online/cache_serv/dbsvr` 五件套 →按"合并后最终形态"裁剪式建好 `USER_INFO_*`/`USER_*` 两组分库表并手工 INSERT 一个 userid →由自建 LoginServer 实时签发 16 字节 DES session 并在选服列表里下发本机 IP →在 MoleRuffle 的 `connect_socket` 把 socket 目标收口到本机(spoof URL 仍保持 `mole.61.com` 以过域名守卫)→用 wayback 里 **2013-03~2013-06 版 Client.swf + 同期子模块** 连上验证进村。

---

## 2. 目标澄清

**本报告研究的是【网页 Flash 版】(mole.61.com)的复活**,三大要件:

1. **服务端** = `LEAK`(上海淘米 2007–2013 网页 Flash 版真源码,C/C++)。进程链:
   `flashpolicy(843 跨域策略)→ login(账号/选服/建角色)→ online(玩法核心,dispatch[cmd] 分发)→ cache_serv(内存缓存,1800s 回写)→ dbsvr(按 cmdid 高 6 位 0xFC00 路由 + userid%100 分 100 库)→ MySQL`。客户端↔在线服报文用 DES,密钥 `DES_KEY=">ta:mee<"`(`mole/online/proto.h:15`)。
2. **客户端资产** = `WAYBACK`(`data/CN/sites/mole.61.com/`,457 个 SWF,含 2012–2026 多版 Client.swf 与 2013/2014 Login.swf、version 清单)。
3. **客户端运行时** = `RUFFLE`(基于 Ruffle 的原生客户端,已实测裸 TCP + `with_spoofed_url` 连**真服**登录进城走路成功)。

**与 iOS 手游版严格区分:** `MOBILE`(moleworld-server,Rust)是 **iOS 手游 5.5.0 版**的私服,协议完全不同(24B 头 + MD5 盐 + cmd 1234 登录),与本 leak **无关**,仅作对照,切勿混为一谈。前人公开的"摩尔庄园协议分析"绝大多数也是手游版(52pojie / CSDN),对本项目无用——网页版协议的权威来源**就是手里这套源码本身**。

---

## 3. 端到端架构图

```mermaid
flowchart TD
    P([玩家]) --> MR

    subgraph 客户端运行时 RUFFLE
      MR[MoleRuffle<br/>原生 Ruffle 栈]
      MR -->|加载| CSWF[Client.swf 壳<br/>2013-03~06 版]
      CSWF -->|动态加载| SUB[LoginHome.swf / main_resource.swf<br/>module/* 子模块<br/>真正跑 DES/命令字/包头]
      MR -.spoof URL=mole.61.com.-> GUARD[过域名守卫]
      MR -.本地缓存改写.-> SXML[config/Server.xml<br/>指向本机]
    end

    MR -->|connect_socket host:port<br/>收口到本机| SELF

    subgraph 自建服务端 LEAK
      direction TB
      FP[flashpolicy :843<br/>MoleRuffle 下可省]
      LOGIN[login<br/>选服/建角色/签发 session]
      SWITCH[switch<br/>选服负载中心]
      ONLINE[online :1865<br/>玩法核心 dispatch]
      CACHE[cache_serv<br/>内存缓存 1800s 回写]
      DB[dbsvr<br/>cmd 0xFC00 路由 + 分库]
      LOGIN --- SWITCH
      LOGIN --> DB
      ONLINE --> CACHE --> DB
    end

    SELF[(本机入口)] --> LOGIN
    SELF --> ONLINE
    DB --> MYSQL[(MySQL<br/>USER_INFO_* / USER_* 分库)]
```

要点:
- Ruffle/MoleRuffle **完全不碰协议**,只做透明 TCP 管道;所有 DES/命令字/包头逻辑都在 Client.swf 动态加载的 AS3 子模块里跑。换服务器**不需要改协议**,只需改"地址注入"。
- 在线服 IP:port **由 LoginServer 的选服列表(`svr_info_t`)动态下发**(`mole/login/libtype.h:46-52`),不是 SWF 写死——所以"改地址"的正确入口在服务端下发逻辑 + Server.xml,而非改 SWF。

---

## 4. 六个维度逐项结论

### 维度一:编译 —— 有条件可行(中,★★★☆☆)

- **结论:端到端可编。** 五进程是标准"newbench 宿主 + 业务 .so 插件"模型(除 online 自带 `main.c` 外,login/cache_serv/flashpolicy/dbsvr 都编成 .so 由 newbench `dlopen + dlsym("handle_init")` 加载)。
- **关键证据:** `mole/login/bench.conf`(`dll_file ./bin/liblogin.so` / `lib_file ./lib/libmole.so`);`c/library/newbench-beta/.../dll.cpp:47` `dlopen`、`:57` `dlsym("handle_init")`、`main.cpp:372-374` 调 `handle_init`;`mole/flashpolicy/service.c` 实现该插件契约。位宽无 `-m32/-m64`(全仓 0 命中),newbench 用 `getconf LONG_BIT` 跟随宿主机,可直接 64 位编。lua 不需系统装(libtaomee 0.7.5 已聚合全套源码)。
- **缺口:**
  1. **login 的 `liblogin.so` —— 不是缺口(已核实修正)** —— `mole/new_login/Makefile:17` 明写 `BINS = liblogin.so`,即 **liblogin.so 的源码就在 `mole/new_login/`**。`mole/login/` 是旧版目录、`mole/new_login/` 是产出该 .so 的现版,login 进程的 `dll_file ./bin/liblogin.so` 直接由 new_login 编出即可。研究初稿曾误判为"源缺失硬缺口",此处更正:**login 进程无硬缺口**。
  2. cache_serv `-lproto/-lpub` 与 dbsvr `libsrc` 产物是否同源(头都叫 `Cclientproto.h`,疑同源,待证)。
  3. 时代假设强:`CMAKE_INSTALL_PREFIX=/usr/`、`-I/usr/include/libtaomee++/...`、`-I/home/mole`、`WORKDIR=$(HOME)/cache_svr`,假设"底座库已 install 到 /usr、源码在固定布局"。现代 gcc(13+)直编大概率失败,**用 CentOS 6 era 老容器规避**最省力。
- **编译顺序(无循环依赖,但有"装到 /usr 才被业务找到"的隐式耦合):** libtaomee 0.7.5 → libtaomee++ 0.7.2 → c/library 底座(net-io-server/ring-queue/ini-file)+ newbench 宿主 → dbsvr `libsrc`(产 libproto.a/libdbtable.a/libpub.a)→ cache_serv → login → online → flashpolicy(**最简单,只 `-ldl -lrt`,第一个练手目标**)。

### 维度二:运行拉起 —— 有条件可行(中)

- **结论:** 编出后用 newbench 宿主按各 `bench.conf` 拉起,online 独立可执行 `OnlineNew`。最小集 = flashpolicy(可省)+ login + online + cache_serv + dbsvr + MySQL,**整条链必须一起立起来**,单独跑 online 进不去(登录依赖 LoginServer 实时签发 session)。
- **关键证据:** 各模块 `bench.conf`/`bind.conf`(login 绑端口、flashpolicy `843 tcp`、switch `server_num=500`、`save_login_file=./data/USR_LOGIN_FILE`)。
- **缺口:** 进程编排无现成总入口/Dockerfile/部署文档(README 作者自评"几乎没有架构/编译/部署文档…开箱即编译跑不通")。需自己写启动脚本与配置对齐(端口、DB 凭据、号池文件)。

### 维度三:数据库自举 —— 有条件可行(中)

- **结论:** 建库脚本、表结构、分库分表算法、账号自助开号路径**全部齐全且自洽**。
- **关键证据:**
  - 建表机制:`*.sh` 用 heredoc 嵌 `CREATE TABLE` + bash 循环建库建表;运行时 C++ **不建表**(`grep "create table" ser/src/*.cpp` 无命中)。主脚本 `mole/dbsvr/ser/sql/create_db.sh`(t_user/attire/connect/pet/task)、`create_user_info.sh`(t_user_info + _ex,含 passwd BINARY16/gameflag/nick)。DB 凭据写死 `user=root password=ta0mee host=localhost`。
  - 分库算法:t_user 用 `CtableRoute100x10`(`Cuser.cpp:32`)= `USER_{id%100}.t_user_{(id%10000)/1000}`(100 库×10 表);t_user_info 用基类 `CtableRoute`(`Cuser_info.cpp:28`)= `USER_INFO_{id%100}.t_user_info_{(id/100)%100}`(100 库×100 表)。
  - 登录最小链路只要两组表:`USER_INFO_*`(存 passwd/gameflag,登录读)+ `USER_*`(进城加载 vip/exp/xiaomee/homemap)。
  - 账号:login 服**不查密码**(密码原属淘米外部 passport,不在 leak)。可绕过——自造 `DES(LOGIN_DES_KEY, ip‖now‖uid‖now)` 16B session 即可登录任意 uid。米米号分配有 `mole/register/` 号池工具(`tool/gen_uid.cpp`、`Cregact.cpp:43-69` mmap 弹号),私服更省事:手工 INSERT 几个 userid。
- **缺口/最大坑:** `Cuser_info::insert`(`Cuser_info.cpp:70-87`)是**位置式 INSERT**,列序必须等于"基础 create + 全部 ALTER 累加后的最终形态"(基础 `create_user_info.sh` 没有 mee_fans/gameflag,由后续 `20090326_alter_userinfo.sh` 等加上)。385 个零散 SQL 脚本无总入口,需自己排序/裁剪;只跑基础 create 会因列数对不上**静默失败**。**对策:照 INSERT 列序反推一份"合并后 schema",一次建好,别靠按序跑 ALTER 拼。** 裁剪式只建你要用的那两个分库,工时从小时级降到分钟级。

### 维度四:协议匹配 —— 高度兼容(协议格式层) / 有条件可行(端到端登录层)

- **结论:** 协议格式高度兼容,真正缺口不在格式而在"登录闭环依赖"和"几个客户端协议常量数值缺失"。
- **关键证据:**
  - 报文头以 `parse_protocol`(`mole/online/proto.c:3346-3352`)为准:`len:u32(ntohl)` / `ver:u8(CRC8 滚动)` / `cmd:u32` / `id:u32` / `ret:u32` / `body`。
  - 版本握手:客户端首包 `PROTO_VERSION` → 服务端回 `"online-201206225"`(`proto.c:3309`,2012-06 版标识);之后每包 `calc_ver_crc` 滚动校验(`proto.c:3387-3392`),登录包要求 `pkg.ver==FIRST_VERSION`(`proto.c:3378`)——这是反篡改/版本绑定,客户端必须用同一算法。
  - DES 局部用于 session(非整包):session 明文 = `ip(4)+tm1(4)+uid(4)+tm2(4)`;`verify_session` 要求 `tm1==tm2` 且与服务器时间差 ≤±3600s(`login.c:319-327`)——**session 必须由配套 LoginServer 实时签发,不能离线伪造长期有效**。另有两道 MD5(访客 key `"bAbY mOlE Go tO heLl!"`、注册用户 key 拼 `"crAzy BAby "`)。
  - MoleRuffle 对协议零理解,纯管道:core 无任何 DES/cmd/parse 代码,`cache.rs:178 connect_socket` 原样转发,`lib.rs:195 confirm_socket→true`(`SocketMode::Allow`)。
- **缺口:**
  1. **客户端协议常量数值缺失:** `PROTO_LOGIN`/`PROTO_MAP_ENTER`/`FIRST_VERSION`/`LOGIN_DES_KEY` 的具体数值定义头不在泄漏包内(`mole/dbsvr/include/proto.h` 是 DB 侧 cmdid,与客户端 `PROTO_*` 不是同一套)。需从 Client.swf 子模块逆向回填或抓包比对(中等工作量)。
  2. **ver 滚动校验算法**客户端/服务端必须完全一致(同源应一致,待抓包验证)。

### 维度五:客户端接线 —— 有条件可行,且最省事路径已存在(易~中)

- **结论:** MoleRuffle 已把网络出口完全收口在 Rust `NavigatorBackend`(`navigate_to_url`/fetch/`connect_socket`),并用 `with_spoofed_url` 破域名守卫。**改指自建服务器根本不用碰 SWF、也不用改 hosts。**
- **关键证据:**
  - 连接发现三段式:静态 `config/Server.xml`(`WAYBACK/.../config/20130511215741__Server.xml`,含 `MOUrl/DirSer/GameSer/RegistSer/Visitor` 的 ip/port,2013 版与泄漏服同代)→ Login.swf 第一跳 HTTP 从 `login.mole.61.com:80` 拉 `ip.txt` → login 协议下发服务器列表。
  - 注入点:`RUFFLE/crates/moleruffle-core/src/lib.rs:35-38`(`GAME_SWF_URL/GAME_BASE_URL`)、`lib.rs:86-87`(`with_spoofed_url`/`with_page_url`)、`cache.rs:67`(`url.contains("mole.61.com")` 命中本地缓存)、`cache.rs:178-189`(`connect_socket(host,port)` 透传——**游戏服 socket 目标从这里出去**)。
  - 资产完整性:进游戏所需 Client + Login + 5 大 DLL(ClientAppDLL/ClientCommonDLL/ClientConfigDLL/ClientSocketDLL/TaomeeCoreDLL,均 >50KB 真 SWF)+ 核心 module + Server.xml **齐全**(`WAYBACK/INVENTORY.md` 第 3 节)。
  - flashpolicy(843):**MoleRuffle 用 `SocketMode::Allow`,根本不发 843 策略请求**,所以 MoleRuffle 路线下 **flashpolicy 可完全不部署**(仅真·浏览器 Flash 才需要)。
- **最省事方案 = A + C 组合:** (A) 把 `config/Server.xml` 各 ip/port 改本机放进 MoleRuffle 本地缓存(命中 `mole.61.com/config/Server.xml`),spoof URL 保持 `mole.61.com` 不变;(C) 自建 login/switch 让选服列表回包本机 IP;`connect_socket` 加一层 host 重写兜底。改 hosts(B)对 GameSer 的**纯 IP 连接无效**,改 SWF(D)最脏无必要。

### 维度六:前人经验 —— 起点更高,但无现成成果可抄

- **结论:** **没有公开证据表明有人直接用这套泄漏 C/C++ 源码端到端跑通过网页 Flash 版私服。** 本项目相对前人**起点更高、更容易**。
- **关键证据:**
  - 上游 `chenbk85/taomee`(star 21 / fork 31,最后 push 2013-01-25)的 31 个 fork 几乎全是原样镜像(`gh api compare` 抽查 ahead=0 behind=0),**无任何 build 脚本/Dockerfile/部署补丁**——若有人跑通过大概率会留痕。
  - 前人路线一 **RecMole**(`Miigon/taomee`,Lua 98.5%):靠**抓包逆向**重写服务端,**非**编译这套源码;作者自述"极初期……可能永远无法完成",唯一里程碑"可以看到地图了"。**关键反证:** RecMole 作者本人也 fork 了 chenbk85/taomee,但最终仍选"纯逆向重写"——侧证"直接编译这套老 C/C++"当时不是被走通的路。
  - 前人路线二 **mole.sundayx.cn**:真实在运营的怀旧服(有客服群/ICP 备案/场景更新),但**技术实现完全不透明**,是否用本 leak 无法证实——仅作"网页版私服确能长期运营"的**存在性背书**。
- **我们的信息优势:** 协议是"已知"而非"待猜"——手里有真源码的 DES 密钥、~1300 cmdid 表、dispatch 逻辑、DB 路由规则,跳过了 RecMole 卡了十多年的整段逆向。
- **缺口:** 无前人编译/部署成果可抄,2013 年老工具链、多进程链 + 分库、客户端版本↔源码版本对齐都得自己搞。

---

## 5. 最小可玩路径(MVP runbook):从 0 到"登录进村庄走两步"

> 目标:MoleRuffle 客户端连自建服务端,用某个 userid 登录、进村、走两步。

| 步 | 动作 | 难度/风险 |
|---|---|---|
| **0. 老容器** | `docker run centos:6`(或 vault 源)/Debian 7,装 gcc4.x、glib2-devel、libxml2-devel、openssl-devel、mysql-devel、cmake2.6。复刻 `/home/mole` 目录布局。 | 易 / 低(关键是规避现代 gcc 报错) |
| **1. 编底座** | 按序编:libtaomee 0.7.5 → libtaomee++ 0.7.2 → c/library(net-io-server/ring-queue/ini-file)→ newbench 宿主。`make install` 到 /usr。 | 中 / 中(头路径/位宽一致性) |
| **2. 编 flashpolicy** | 第一个练手目标(只 `-ldl -lrt`),验证宿主链路通。**MoleRuffle 路线可跳过部署,但用它验证编译链最省心。** | 易 / 低 |
| **3. 编 dbsvr** | `make -C mole/dbsvr/libsrc`(产 libproto.a/libdbtable.a/libpub.a)→ `make -C ser/src` 出 libser.so。 | 中 / 中 |
| **4. 编 cache_serv + online** | cache_serv 依赖 `-lproto/-lpub`(取自 libsrc);online 独立 `OnlineNew`。 | 中 / 中 |
| **5. 编 login** | 直接 `make -C mole/new_login`(它就是 `liblogin.so` 的源)→ 供 login 进程用。 | 中 / 中 |
| **6. 建库(裁剪式)** | 装 MySQL(root/ta0mee),**照 `Cuser_info.cpp:70` 位置式 INSERT 列序反推"合并后 schema"一次建好**,只建你选的 userid 命中的那一个 USER_INFO 分库 + 一个 USER 分库。 | 中 / 高(列序错位静默失败) |
| **7. 造账号** | 手工 `INSERT` 一个 userid 到 t_user_info(gameflag bit1=摩尔庄园,`Cuser_info.cpp:776`)+ 对应 t_user 行(默认 vip/exp/homemap)。 | 易 / 中 |
| **8. 配置编排** | 写启动脚本:对齐端口、DB 凭据、号池/data 文件,按 dbsvr→cache_serv→online→login 顺序起。 | 中 / 中(无现成总入口) |
| **9. 客户端接线** | 改 `config/Server.xml` 各 ip/port 指本机放进 MoleRuffle 缓存;spoof URL 保持 `mole.61.com`;`connect_socket` 加 host 重写兜底;选 2013-03~06 版 Client.swf + 同期子模块。 | 易 / 低(已有验证经验) |
| **10. 登录票据** | 由自建 LoginServer 实时签发 `DES(LOGIN_DES_KEY, ip‖now‖uid‖now)` 16B session,时间须在 ±3600s 内;若先单测可用小工具造。 | 中 / 中(常量数值需回填) |
| **11. 验证** | MoleRuffle 连上 → 抓首几包对照 `parse_protocol` 的 len/ver/cmd/id → 登录 → 进村 → 走两步。 | — / —(M2 验收点) |

---

## 6. 风险登记册

| 级别 | 风险 | 缓解办法 |
|---|---|---|
| **致命** | **协议版本漂移**:wayback SWF 子模块年份与 2013 源码 cmd/ver 算法对不上,第二个包就被 `calc_ver_crc` 踢。 | 首选 **2013-03~2013-06 版 Client.swf + 同期 version 清单 + 同期子模块**(`20130302165622__Client...swf` 等);用 JPEXS/ffdec 反编译 2013 LoginHome.swf 核对 DES key/ver 算法/PROTO_* 数值与源码一致;抓 MoleRuffle 连真服的首包逐字段比对。 |
| **致命** | **DB schema 还原失败**:385 个零散脚本无总入口,位置式 INSERT 列序与最终表不一致 → 静默写坏。 | 不按序拼 ALTER,直接照 `Cuser_info.cpp:70` INSERT 列序反推"合并后 schema"一次建好;`mysqldump --no-data` 对照核零偏差;裁剪式只建必需分库。 |
| ~~高~~ → 低 | ~~login `liblogin.so` 源缺失~~(**已核实修正:不缺**)——`mole/new_login/Makefile` 就产 `liblogin.so`。 | 直接编 `mole/new_login/` 得到 `liblogin.so` 供 login 用即可;无需自写壳。 |
| **高** | **客户端协议常量数值缺失**(PROTO_LOGIN/MAP_ENTER/FIRST_VERSION/LOGIN_DES_KEY)。 | 从 2013 子模块 SWF 逆向回填,或抓 MoleRuffle↔真服包比对数值;源码有全部算法,只缺枚举常量,工作量可控。 |
| **高** | **老工具链编不过**(现代 gcc/glib 语法收紧、隐式声明)。 | 用 CentOS 6 era 老容器(与 2009–2013 代码假设最匹配);逐库 `-fpermissive`、补 `-include` 缺头作兜底。 |
| **中** | **登录闭环依赖整条链**,单跑 online 进不去(session ±3600s 实时签发 + uid/ip 绑定 + 两道 MD5)。 | 整套 login+online+cache_serv+dbsvr+MySQL 一起立;先把 session 签发跑通再谈玩法。 |
| **中** | **在线服地址下发入口待确证**(改 Server.xml vs 改 login 选服列表 `svr_info_t`)。 | 两条都喂:Server.xml 指本机 + login 回包 IP 指本机 + `connect_socket` host 重写兜底,三保险。 |
| **中** | **进城后玩法触发更多 Cxxx 表**(任务/宠物/装扮),缺表报 SQL 错。 | 最小登录进村只要 USER_INFO+USER;"完整可玩"再把绝大多数 create_*.sh 跑全,分阶段补。 |

**最大的未知数:协议版本漂移 + DB schema 还原。** 两者都是"致命"级,但都有明确的、证据支撑的填补路径——不是无解,而是需要细致的对齐与核对工作。

---

## 7. 里程碑路线图

- **M0 编出来:** 老容器里五件套(flashpolicy/login/online/cache_serv/dbsvr)全部编出二进制(`liblogin.so` 由 `mole/new_login/` 产出,非缺口)。卡点 = 老工具链对齐(旧 gcc/glib/mysql client)。
- **M1 单进程起来:** newbench 宿主能拉起各 .so,online `OnlineNew` 能跑;flashpolicy 回固定 XML、online 回 `online-201206225`。
- **M2 登录通:** 建好两组分库表 + 造账号 + 自建 login 签发 session,MoleRuffle 连上完成 `PROTO_LOGIN` 鉴权握手(**端到端最小闭环验收点**)。
- **M3 进村庄:** `PROTO_MAP_ENTER → enter_map_cmd`(`proto.c:2147`)走通,默认地图集合 `{1,3,2,8,10,37,9,41,6,7}`(`login.c:29`)加载,角色出现、能走两步。
- **M4 核心玩法:** 补齐 cache_serv 回写、任务/宠物/装扮等 Cxxx 表与对应 cmd,聊天/好友/小屋可用。
- **M5 完整:** 跑全 create_*.sh、号池注册模块、各功能 module,接近原版体验;可选部署真·flashpolicy 以支持浏览器 Flash 栈。

---

## 8. 诚实的反方观点(为什么可能失败)

1. **老工具链是真考古坑。** 2009–2013 的硬编码 `/usr`、`/home/mole`、cmake 2.6、glib2、旧 mysql client,现代环境直编几乎必败;即便上老容器,CentOS 6 vault 源、依赖版本、静态库链接仍可能层层踩雷。README 作者亲口"开箱即编译跑不通"。
2. **缺数据 + 缺常量。** `liblogin.so` 源、客户端 `PROTO_*` 数值常量头、密码库(passport)都不在 leak 里。前两者可补但要逆向/抓包,后者虽可绕过(无密码登录)但说明这套源码本就不是"自包含可跑"的完整快照。
3. **协议可能对不上。** `calc_ver_crc` 滚动校验是反篡改机制——只要客户端子模块版本与 2013 源码差一代,第二个包就被踢。wayback 虽有 2013 版 Client.swf 壳,但**壳几乎一字未变(2013→2026)**,真正的协议差异在子模块,必须逐一核对同期子模块是否齐全、算法是否同源,这一步可能反复碰壁。
4. **客户端版本与服务端不同代的风险被低估。** MoleRuffle 实测连的是**第三方私服 123.206.131.236**(已知能跑的版本组合),并非用我们这套 2013 源码验证过;把 SWF 换到精确匹配 2013 服务端、再把服务端真编出来,是两件都没人做过的事叠加,组合风险高于任一单项。
5. **无前人成果兜底。** 31 个 fork 全无 build 产物,RecMole 走了十多年只到"能看地图",怀旧服不开源——这条"编译真源码"的路**确实没有公开先例**,意味着每一个坑都得自己第一次踩。

**结论复述:** 以上反方观点都是真实风险,但**没有一条是不可逾越的死局**——每个缺口都有证据支撑的填补路径。本项目相对所有前人拥有"源码 + 协议已知 + 客户端运行时已解决 + 客户端资产齐全"四重优势,是已知最有希望的一条网页 Flash 版复活路线。建议按 M0→M2 分阶段推进,以"登录进村走两步"作为第一个硬验收,再决定是否投入完整玩法。
