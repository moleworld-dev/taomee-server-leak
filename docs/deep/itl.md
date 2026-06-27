# itl 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/itl.md](../modules/itl.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)

本文件是 `itl`(淘米内部运维监控平台,源码代号 "OA 监控 / ITL 后台")的"深水区"补充。概览文档已经把"每个目录是什么"讲清楚了;这里只讲一件事:**进程跑起来之后,一条监控数据是怎么从被监控机器一路流到落库 / 画图 / 发短信的**,以及每个进程在这条链路上实际做的动作(带真实源码行号)。

---

## 一图看懂:进程/组件拓扑与数据流

```
                         ┌──────────────── MySQL: db_itl ────────────────┐
                         │ 资产/机房台账 + t_metric_info / t_alarm_*       │
                         │ + 触发器把"配置变更"写进 t_update_notice         │
                         └───────┬──────────────────────────────▲────────┘
                                 │ (db 进程是唯一访 db_itl 的人)  │
   被监控机 (每台一份)            ▼                               │ INSERT notice
 ┌─────────────────────┐   ┌──────────┐   配置/告警结果          │
 │ itl-node (agent)    │   │ itl-db   │◀──────────────────────── (各进程都连 db)
 │  ├ collect 定时器    │   │ 进程     │
 │  ├ dlopen metric-so │   └────▲─────┘
 │  │   ├ libcommon.so │        │ send_metric_to_db (类型&4)
 │  │   ├ libmysql.so  │        │
 │  │   └ libmount.so  │   ┌────┴───────────────┐
 │  └ db_mgr 管本机MySQL│   │     itl-head        │ ── send_metric_to_rrd ──▶ ┌────────┐
 └──────────┬──────────┘   │  (中枢/网关)         │   (类型&2)                  │ itl-rrd│─▶ *.rrd
            │ head_p_metric_data_cmd                │                            └────────┘
            │ (打包整组指标 TCP 上报)  │  dispatch_metric()  │ ── send_metric_to_alarm ─▶┌─────────┐
            └─────────────────────▶│  按 metric_type 分流 │   (类型&1 或采集失败)        │itl-alarm│
                                    └──────────┬──────────┘                            └────┬────┘
   ┌────────────┐  控制命令(重启/换so/        │ node 注册时                                  │ 阈值状态机
   │ itl-control│  下发配置/执行shell)         │ db_store_host_status                          │ 命中→HTTP POST
   │  轮询notice │◀────────────────────────────┘ alarm_node_up                                ▼
   │  push 到node│                                                              http://…/alarm-server/index.php
   └─────┬──────┘                                                              (cmd=10001 短信 / 10002 邮件 / 10003 RTX)
         │ notify_config_change_cmd
         ▼ (node 收到→refresh_collect)
   itl-node

 ── 独立子系统(不走上面这条 node→head 链)──
 switch-monitor : 自己多线程 SNMP 轮询交换机 → 自己写 RRD + 自己 HTTP POST 告警 + 自己写 db_itl
 itl-speed      : CDN/IDC 测速,FCGI 收数据 → analyze_server 按 IP/省运营商聚合 → dbser(SO)落库
```

一句话:**node 只负责"采集 + 上报",所有判断都在 head 这一侧的内存里做完,再扇出给 rrd/db/alarm 三个后端进程**。switch-monitor 和 itl-speed 是两个自带完整采集-处理-落库链路的独立子系统,只是复用了同一个 `db_itl` 库和告警网关。

---

## 逐部分拆解

### 1. itl-common —— 公共底座 + 唯一的真包头

所有 itl 进程共用一个 `#pragma pack(1)` 的二进制包头 `itl_proto_t`,见 [itl/itl-common/itl_proto.h#L31](../../itl/itl-common/itl_proto.h#L31):`len / seq / cmd / ret / id / body[]`。head/node/db/rrd/alarm/control 全部 `typedef` 成同一个结构([#L48-L54](../../itl/itl-common/itl_proto.h#L48))。`seq` 注释里写明是 `(p->fd << 16) | p->waitcmd`,即把"哪个连接 + 在等哪个命令的回包"编码进序列号,用来做异步请求/回包配对。

head 处理类型(metric_type)是整套数据流的"开关",定义在 [itl/itl-common/define.h#L99](../../itl/itl-common/define.h#L99):`METRIC_TYPE_ALARM=1 / METRIC_TYPE_RRD=2 / METRIC_TYPE_MYSQL=4`,是**按位组合**的——一个指标可以同时既报警又画图。被监控对象类型 `ITL_TYPE_SERVER=1 / SWITCH=2 / MYSQL=3` 见 [#L90](../../itl/itl-common/define.h#L90)。

### 2. itl-node —— 部署在每台被监控机上的 agent

**实际职责:dlopen 采集 so → 定时调用 → 打包整组指标 TCP 推给 head。它自己完全不做阈值/画图判断。**

- 采集 so 的加载:`register_so()` 用 `dlopen(RTLD_NOW)` 打开 `.so`,再 `dlsym` 取出固定的四个符号 `handle_init / handle_fini / get_metric_info / metric_handler`(见 [itl/itl-node/src/collect/collect.cpp#L295-L335](../../itl/itl-node/src/collect/collect.cpp#L295))。哪些 so 要加载,是 db 进程根据本机 IP 下发的 `so_name_list`(分号分隔),so 不在本地就 `download_metric_so` 现下载([#L103-L122](../../itl/itl-node/src/collect/collect.cpp#L103))。
- 定时采集循环:每个 metric_group 一个定时器,`route_metric_group()`→`do_group_collect()`([#L343](../../itl/itl-node/src/collect/collect.cpp#L343))。它先检查两道闸:`g_collect_flag`(采集总开关)和 `g_head->is_connected()`(**没连上 head 就根本不采**,[#L346-L353](../../itl/itl-node/src/collect/collect.cpp#L346)),然后逐个指标 `do_metric_collect` 调 so 的 `metric_handler`,把整组打成一个 `head_p_metric_data_cmd` 包发给 head([#L414](../../itl/itl-node/src/collect/collect.cpp#L414))。注意它先发一次再注册定时器,即"启动即采一帧"。
- 配置热更新:head/control 下发 `node_p_notify_config_change`,node 收到只做一件事——`refresh_collect()` 重新向 db 拉本机配置([itl/itl-node/src/itl_node.cpp#L36-L54](../../itl/itl-node/src/itl_node.cpp#L36))。
- **node 是个被远程操控的 agent**,命令面很危险:`node_p_execute_command` 直接 `g_spawn_command_line_sync("sh -c '…'")` 跑任意 shell([#L273-L300](../../itl/itl-node/src/itl_node.cpp#L273));`node_p_update_so` / `node_p_update_metric_so` 从 URL 下载新 so + 校验 md5 后 `restart_node()` 热替换([#L186](../../itl/itl-node/src/itl_node.cpp#L186)、[#L234](../../itl/itl-node/src/itl_node.cpp#L234));还有 download/upload(ftp)、切采集开关等。这是典型的"中心可以让每台机器干任何事"的运维 agent。
- db_mgr:node 还兼管本机的 MySQL 实例(本模块最大单文件之一 [itl/itl-node/src/db_mgr/db_mgr.cpp](../../itl/itl-node/src/db_mgr/db_mgr.cpp)),`node_p_notify_mysql_instance_change` 触发 `refresh_mysql_instance`。

### 3. metric-so —— 真正"读 /proc"的采集插件

三个 so 都遵守同一个 ABI(node 靠 dlsym 这四个符号驱动):

- **libcommon.so** 主机通用指标。采集项是一张 X-macro 表 [itl/metric-so/src/libcommon/metric_define.h](../../itl/metric-so/src/libcommon/metric_define.h),45 个指标:网卡收发 bit/s、内存/swap、磁盘、负载 load_one/five/fifteen、CPU user/system/idle/wio…、内外网 IP、机型/OS/版本、系统时钟偏移等。X-macro 同时生成"指标元信息数组 + 枚举 + handler 分发"三份代码,见 [itl/metric-so/src/libcommon/common_proto.cpp#L31-L48](../../itl/metric-so/src/libcommon/common_proto.cpp#L31) 与 `metric_handler` 的 `case name: return name##_func(val)`([#L64-L172](../../itl/metric-so/src/libcommon/common_proto.cpp#L64))。底层都是读 `/proc/loadavg`、`/proc/meminfo`、`/proc/stat`、`/sys/.../scaling_max_freq` 等([itl/metric-so/src/libcommon/metrics.cpp#L118](../../itl/metric-so/src/libcommon/metrics.cpp#L118))。
- **libmysql.so** MySQL 性能指标(show status / innodb 等,[itl/metric-so/src/libmysql/mysql_metric.cpp](../../itl/metric-so/src/libmysql/mysql_metric.cpp));`handle_init` 收的参数是 node 传进来的 `mysql_so_param_t` 向量(端口 + sock + 要采的指标名),见 node 侧 [collect.cpp#L168-L215](../../itl/itl-node/src/collect/collect.cpp#L168)。
- **libmount.so** 磁盘挂载点容量([itl/metric-so/src/libmount/mount_info_metrics.cpp](../../itl/metric-so/src/libmount/mount_info_metrics.cpp));**libdmidecode** 解析硬件资产([itl/metric-so/src/libcommon/dmidecode.cpp](../../itl/metric-so/src/libcommon/dmidecode.cpp))。

`get_metric_info(index)` 返回每个指标的"名字/值类型/单位/格式/slope",node 据此知道该怎么解释和上报值——所以 head 那边收到的指标自带元信息(见下)。

### 4. itl-head —— 中枢,数据扇出的真正发生地

head 是无状态判断的"路由器 + 内存缓存":它把每个 node 的最新指标缓存在内存(`c_node→c_metric→c_metric_arg`),并按 metric_type 把数据扇给三个后端进程。

- 连接分发:`dispatch()` 先按 fd 判断是不是已注册 node([itl/itl-head/src/dispatch.cpp#L26](../../itl/itl-head/src/dispatch.cpp#L26)),是就走 `dispatch_node`,否则按 `find_head_cmd(cmd)` 查命令表回调([#L70-L87](../../itl/itl-head/src/dispatch.cpp#L70))。命令表 `cmd_proto_t` 由 gen_proto 生成。
- node 注册:`head_p_node_register` 里 `alloc_node` 建内存对象,并立刻 `db_store_host_status(STATUS_HOST_UP)` + `alarm_node_up`(通知 db 主机上线、通知 alarm 重置状态),见 [itl/itl-head/src/itl_head.cpp#L37-L88](../../itl/itl-head/src/itl_head.cpp#L37)。
- 收指标:`head_p_metric_data` 遍历包里的指标向量,刷新内存里的 `c_metric` 元信息和 `c_metric_arg` 值,然后对**每个 arg** 调 `dispatch_metric`([itl/itl-head/src/itl_head.cpp#L115-L169](../../itl/itl-head/src/itl_head.cpp#L115))。
- **扇出逻辑(最关键的 35 行)** [itl/itl-head/src/metric.cpp#L27-L64](../../itl/itl-head/src/metric.cpp#L27):
  - 采集失败(`m_fail`)→ 只发 alarm(让 alarm 发"采集失败"告警);
  - `type & ALARM` 或 是"相对指标" → `send_metric_to_alarm`;
  - `type & RRD` → `send_metric_to_rrd`;
  - `type & MYSQL` → `send_metric_to_db`。
- head 到 rrd/db/alarm 三个后端是**三条带自动重连的 TCP 长连**(`set_auto_reconnect`),IP/端口从配置读,见 [itl/itl-head/src/rrd.cpp#L32](../../itl/itl-head/src/rrd.cpp#L32)、[db.cpp#L30](../../itl/itl-head/src/db.cpp#L30)、[alarm.cpp#L33](../../itl/itl-head/src/alarm.cpp#L33)。三种扇出各有不同的打包函数 `pack_metric_to_rrd/alarm/db`([metric.cpp#L67/#L91/#L110](../../itl/itl-head/src/metric.cpp#L67))——比如发给 rrd 的只带 name/arg/value/slope/interval,发给 alarm 的额外带 value_type 和原始 c_value(供阈值比较)。

### 5. itl-alarm —— 双阈值 + 滞回 + 退避的告警状态机

alarm 进程收 head 转来的指标,在自己内存里跑状态机,命中阈值就 HTTP POST 给告警网关。

- 阈值配置很灵活:warning/critical 值可以写成 `0.5*$26`(系数 × 另一个指标 26 的当前值),`get_metric_and_coeff` 负责解析这种"相对阈值"([itl/alarm/src/metric_alarm.cpp#L519-L552](../../itl/alarm/src/metric_alarm.cpp#L519)),`init_alarm_value` 把它解析成系数 + 关联 metric_id([#L556](../../itl/alarm/src/metric_alarm.cpp#L556))。比较运算符支持 `= > < >= <=`(`compare_value`,[#L201](../../itl/alarm/src/metric_alarm.cpp#L201))。
- 两道"静默"门:`is_ban`(屏蔽时间窗,[#L121](../../itl/alarm/src/metric_alarm.cpp#L121))和 `is_alarm_off`(每天的免打扰时间段,默认 0–6 点,[#L169](../../itl/alarm/src/metric_alarm.cpp#L169),解析见 `init_alarm_off_time` [#L370](../../itl/alarm/src/metric_alarm.cpp#L370))。
- **状态机本体** `handle_alarm` [itl/alarm/src/alarm.cpp#L342-L454](../../itl/alarm/src/alarm.cpp#L342):
  - 正常态:连续异常达到 `m_abnormal_threshold` 次才真正翻成告警态并发第一条告警(滞回去抖,默认异常 2 次,[metric_alarm.cpp#L35](../../itl/alarm/src/metric_alarm.cpp#L35));
  - 告警态:连续正常达到 `m_normal_threshold` 次(默认 5 次,[#L34](../../itl/alarm/src/metric_alarm.cpp#L34))才发"恢复"并清零;
  - 持续告警时按 `get_span(m_alarm_count)` 做**退避重发**——告警次数越多间隔越大(配置 `次数区间:间隔分钟`,`init_alarm_span` [#L453](../../itl/alarm/src/metric_alarm.cpp#L453);没配就默认 1 小时,[#L106](../../itl/alarm/src/metric_alarm.cpp#L106))。
- 三种告警通道,按 warn/crit 各自的 `way` 位掩码选择 MOBILE/EMAIL/RTX([alarm.cpp#L573-L617](../../itl/alarm/src/alarm.cpp#L573));恢复时"先通知收到 critical 的人,再用异或排掉重复只通知额外收 warn 的人"([#L592-L598](../../itl/alarm/src/alarm.cpp#L592))。
- **告警出口其实是一个 PHP 网关**:三种 `send_*_alarm` 全部 `http_post` 到同一个 URL,只是 `cmd` 不同(10001 短信 / 10002 邮件 / 10003 RTX),默认地址 `http://10.1.1.63/itl-server/alarm-server/index.php`,见 [itl/alarm/src/alarm_interface.cpp#L24-L74](../../itl/alarm/src/alarm_interface.cpp#L24)。**真正接短信网关的是这个 PHP,不在泄漏树里**。
- 另外 alarm 还会 ping 探活:`check_host_alive` 直接 `/bin/ping -n -U -w%u -c5`([itl/alarm/src/check_host_alive.cpp#L31](../../itl/alarm/src/check_host_alive.cpp#L31))。

### 6. itl-rrd —— RRDtool 封装

收 head 的指标,`rrd_update` 写入对应 `.rrd`,文件不存在就 `rrd_create`。RRA 归档策略写死在宏里(server 与 switch 各一套),例如分钟粒度保 10080/25920 点、再按 6/24/288 倍降采样保更长周期,见 [itl/rrd/src/c_rrd_handler.cpp#L34-L57](../../itl/rrd/src/c_rrd_handler.cpp#L34)。这套粒度就是前端"近 1 天 / 近 1 周 / 近 1 月"趋势图的数据源。

### 7. itl-db —— 唯一访问 db_itl 的进程

统一 SQL 层(最大单文件 [itl/db/src/db_interface.cpp](../../itl/db/src/db_interface.cpp),~5.8 万字节)。head/node/alarm/control 都不直连 MySQL,而是发命令给 db 进程代办——既做配置下发(节点该采哪些 so、阈值多少),也做结果回写(主机状态、告警事件、db_mgr 执行结果)。

### 8. itl-control —— "配置即生效"的下发器

control 是把 DB 里的配置变更"推"给在线 node 的环节,**它和触发器配合实现热生效**:

- 定时轮询:`init_update_timely` 每 `GET_CONFIG_UPDATE_INTERVAL` 秒向 db 发 `db_p_get_update_notice_cmd` 拉一次待办([itl/control/src/update_notice.cpp#L21-L27](../../itl/control/src/update_notice.cpp#L21))。
- 回包处理 `db_p_get_update_notice_cb`:把待办按 `collect_update_flag` / `alarm_update_flag` 分成 switch/node/alarm 三组,对在线 node 调 `notify_config_change` + `notify_mysql_instance_change`,对 alarm 调 `alarm_change_node_alarm_config`([itl/control/src/db.cpp#L96-L150](../../itl/control/src/db.cpp#L96))。
- 它也接 node 注册、并能下发重启/换 so/执行命令等控制命令(`control_p_node_register` 等,[itl/control/src/control.cpp#L29](../../itl/control/src/control.cpp#L29))。

### 9. switch-monitor —— 独立的 SNMP 交换机监控(mason 写)

不走 node→head,自成一套多线程采集器([itl/switch-monitor/src/main.cpp](../../itl/switch-monitor/src/main.cpp)):从 `db_itl` 读交换机列表(IP + SNMP community,`SNMP_VERSION_2c` 明文社区串,[snmplib.cpp#L93](../../itl/switch-monitor/src/lib/snmplib.cpp#L93))→ 采集线程 `simple_snmp_walk/get` 轮询标准接口 MIB(`ifInOctets/ifOutOctets/ifSpeed/ifOperStatus` 等,OID 表见 [itl/switch-monitor/src/lib/snmplib.h#L9-L29](../../itl/switch-monitor/src/lib/snmplib.h#L9))+ 厂商私有 CPU/内存 OID(cisco / h3c 各一套,[#L23-L29](../../itl/switch-monitor/src/lib/snmplib.h#L23))→ `data_processer` 处理 → 自己写 RRD + 自己 HTTP POST 告警网关 + 自己回写 db_itl。默认告警/采集间隔(60/15s)、队列长、线程数、告警 URL、rrd 目录都在 main 里写死兜底([main.cpp#L319](../../itl/switch-monitor/src/main.cpp#L319))。安全降权:得到 rrd 目录后 `setuid nobody`,所以只能 access 检查不能创建([main.cpp#L176](../../itl/switch-monitor/src/main.cpp#L174))。

### 10. itl-speed —— CDN/IDC 全网测速(独立子系统)

体量虚高(408 文件 / 54MB,大量重复 `qdes.c`/`proto.h` 副本),真正核心是 FCGI 收数据 → `analyze_server` 聚合 → `dbser`(SO 插件)落库。命令字是泄漏树里**唯一完整保留**的 itl 协议枚举 [itl/itl-speed/dbser/src/proto/cdn_rate_enum.h](../../itl/itl-speed/dbser/src/proto/cdn_rate_enum.h):`cdn_report_ip_cmd=0x1001`、`cdn_report_prov_cmd=0x1002`(按省市运营商上报 CDN 速度)、`net_stat_ip_report_cmd=0x1211`(替换旧 1003/1005/1007 的延迟/丢包/跳数)等。`analyze_server` 用一个 IP 字典(mmap 的 IP→省/市/运营商映射)把原始 IP 数据归约成"按省市运营商"的聚合结果再上报([itl/itl-speed/tst_spd/analyze_server/process_packet.cpp#L193-L203](../../itl/itl-speed/tst_spd/analyze_server/process_packet.cpp#L193)),并用 cache 去重([#L117-L120](../../itl/itl-speed/tst_spd/analyze_server/process_packet.cpp#L117))。

---

## 关键机制实战

### 机制 A:metric_type 位掩码驱动的"一次采集、三路扇出"

整套系统的优雅之处:node 端**不知道**一个指标是用来画图还是报警的,它只管采集并带上 `metric_type`(这个类型来自 metric-so 的元信息 / DB 配置)。判断完全集中在 head 的 35 行 `dispatch_metric`([metric.cpp#L27](../../itl/itl-head/src/metric.cpp#L27)):因为 `ALARM=1/RRD=2/MYSQL=4` 是按位的,DB 里把一个指标配成 `3` 就同时画图 + 报警,配成 `7` 就三路全开。改监控行为只需改 DB 一个整数,无需动任何 node。这就是它能管几千台机器还保持简单的原因。

### 机制 B:触发器 + t_update_notice + control 轮询 = 配置热生效

这是"在 OA 网页上改个阈值,几十秒内全网生效"的实现链,横跨 SQL 和 C++:

1. 运维在网页改了 `t_metric_info` 的 `warning_val`/`critical_val`/`operation` 等;
2. **触发器** `tr_update_metric_info`([itl/script/db_itl_trigger_v3.sql#L7-L52](../../itl/script/db_itl_trigger_v3.sql#L7))`AFTER UPDATE` 触发,用游标找出所有受影响的 node,判断这次改动是影响"采集"还是"告警"(`@node_update`/`@alarm_update`),`INSERT … ON DUPLICATE KEY UPDATE` 写进 `t_update_notice`(用位或合并多次变更);
3. `tr_update_metric_group_info`([#L56](../../itl/script/db_itl_trigger_v3.sql#L56))同理处理采集间隔变更;还有 alarm_strategy / host_info / node_info 等一整套增删改触发器(共 40+ 个,见概览);
4. control 进程定时轮询取走 `t_update_notice`,推 `notify_config_change` 给在线 node([control/src/db.cpp#L96](../../itl/control/src/db.cpp#L96));
5. node 收到只是 `refresh_collect()` 重新拉配置([itl_node.cpp#L36](../../itl/itl-node/src/itl_node.cpp#L36))。

也就是说"业务逻辑的一半写在 MySQL 触发器里"——这是 2012 年很典型、但今天会被诟病的设计。

### 机制 C:告警去抖三件套(滞回 + 阈值次数 + 退避)

`handle_alarm`([alarm.cpp#L342](../../itl/alarm/src/alarm.cpp#L342))把"瞬时抖动不告警、恢复要稳、长时间不恢复别刷屏"做成了三层:连续异常 N 次才告(默认 2)、连续正常 M 次才恢复(默认 5)、持续告警按次数退避重发(`get_span`)。这是 Nagios 式 flapping 抑制的手写实现。

---

## 复活实操提示(对 moleworld)

- **直用:基本为零。** itl 监控的是淘米真实生产机群,不含任何摩尔庄园玩法/协议/配置。它和 mole 唯一的交集只是共用同一套 `itl_proto_t` 包头风格与 gen_proto 范式(要看协议骨架请优先读 [c.md](./c.md) / [protocol.md](../protocol.md))。
- **可借鉴的设计思路**(若 moleworld 复活后要做私服运维):
  - "中心 head + 每机 agent + dlopen so 采集"的可插拔架构、metric_type 位掩码一次采集多路扇出、告警三件套去抖、触发器驱动热配置——这些都是好范式,但**实现层完全可以用现代栈替代**(Prometheus node_exporter + Alertmanager + Grafana),不必移植这套 2010 年的 C++/RRDtool/net-snmp 代码。
  - itl-speed 的"IP→省/市/运营商映射 + 按运营商聚合测速"思路,若将来要做 CDN 选路可参考,但 IP 库本身已严重过期。
- **要自己补的**:告警出口的 PHP 网关(`alarm-server/index.php`)、gen_proto 生成的各模块 `*_enum.h` 命令字、整套 `db_itl` 配置数据,泄漏树里都没有或不全。
- **安全提示(仅作史料,勿照搬)**:node 的 `node_p_execute_command` 是远程任意 shell、`update_so` 是从 URL 热加载 so、switch-monitor 用明文 SNMPv2c community——这套 agent 一旦中心被攻破等于整个机群沦陷,现代复活务必不要复刻这种"全权 agent"模型。

> 评级仍为 ★:工程质量不低,但对"摩尔庄园复活"几乎零直用价值,了解架构即可。
