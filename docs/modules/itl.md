# 模块 itl · 淘米内部运维监控平台(OA / ITL 监控系统)

> 价值:★  ·  游戏:基础设施(全游戏共用,与摩尔庄园玩法无关)  ·  源码量:约 8.5 万行
> 📖 深度功能解析(每个部分实际在干什么):[deep/itl.md](../deep/itl.md) · 彩蛋见 [easter-eggs.md](../easter-eggs.md)

## 是什么 / 在整套架构里的角色

`itl` 是淘米的**内部运维监控 + 资产管理平台**,源码内部代号同时叫 "OA 监控系统" / "ITL 后台"(见 [itl/doc/OA后台服务概要设计V2.doc](../../itl/doc/OA后台服务概要设计V2.doc)、[itl/doc/V3/ITL后台架构图v3.1.vsd](../../itl/doc/V3/ITL后台架构图v3.1.vsd))。它**不属于任何一款游戏**,而是支撑所有游戏(摩尔庄园、赛尔号、功夫派……)线上服务器的运维基础设施:采集主机/MySQL/交换机指标、写 RRD 画图、阈值告警(含短信)、CDN/IDC 测速、机房资产台账。

从作者头注释看,核心由 `mason(张龙龙)`(2010 年起,switch-monitor / itl-speed)和 `smyang`(2011–2012 年,head/node/alarm/db 重构)两批人写成,时间跨度 2010–2013,与整套泄漏源码的"大扩张期"吻合。

进程/角色关系(分布式 head → node → so 采集插件 → 落库/告警/画图):

| 进程 | 目录 | 作用 |
|------|------|------|
| **itl-head** | [itl/itl-head](../../itl/itl-head) | 前置网关/中枢,接 Web/OA 前端请求,转发给各 node,聚合 alarm/db/rrd 结果 |
| **itl-node** | [itl/itl-node](../../itl/itl-node) | 部署在每台被监控主机上的 agent,按配置调用采集插件、上报数据、管理本机 MySQL 实例 |
| **metric-so**(采集插件) | [itl/metric-so](../../itl/metric-so) | 一组 `.so` 采集器:主机通用指标、MySQL 性能、磁盘挂载、dmidecode 硬件信息 |
| **alarm** | [itl/alarm](../../itl/alarm) | 指标阈值判定 + 告警分发(含短信网关 + 工作时间段屏蔽) |
| **rrd** | [itl/rrd](../../itl/rrd) | 封装 RRDtool,把指标写入环形数据库供前端画趋势图 |
| **db**(DB 服务) | [itl/db](../../itl/db) | 统一 MySQL 访问层(资产/配置/告警全部落到 `db_itl`) |
| **switch-monitor** | [itl/switch-monitor](../../itl/switch-monitor) | 独立的 SNMP 交换机流量/接口监控采集器 |
| **itl-speed** | [itl/itl-speed](../../itl/itl-speed) | CDN / IDC 全网测速子系统(独立的 dbser + FCGI 采集 + 分析) |
| **control** | [itl/control](../../itl/control) | 配置变更下发/通知(node 配置 push、update notice) |

整体是经典的 "中心 head + 每机 node + 可插拔 so 采集器" 监控架构,所有元数据(节点、指标、告警策略、机房资产)集中在一个 MySQL 库 `db_itl`,并大量用**触发器**把配置表的变更同步到运行态表。

## 关键子目录与文件(每条 = 可跳转超链接 + 一句话作用)

- [itl/itl-head/src/itl_head.cpp](../../itl/itl-head/src/itl_head.cpp) — head 进程主程序,异步网络框架 + 协议分发入口。
- [itl/itl-head/src/proto.h](../../itl/itl-head/src/proto.h) — head 协议绑定:聚合 head/db/alarm/rrd 四套命令字与结构体。
- [itl/itl-node/src/itl_node.cpp](../../itl/itl-node/src/itl_node.cpp) — node 进程主程序,处理配置变更通知、刷新采集与 MySQL 实例。
- [itl/itl-node/src/db_mgr/db_mgr.cpp](../../itl/itl-node/src/db_mgr/db_mgr.cpp) — 本机 MySQL 实例管理(本模块最大单文件之一,~3.6 万字节)。
- [itl/itl-node/src/collect](../../itl/itl-node/src/collect) — node 侧采集调度(加载 metric-so 插件并定时采集)。
- [itl/metric-so/src/libmysql/mysql_metric.cpp](../../itl/metric-so/src/libmysql/mysql_metric.cpp) — MySQL 性能指标采集插件(show status/innodb 等)。
- [itl/metric-so/src/libcommon/metrics.cpp](../../itl/metric-so/src/libcommon/metrics.cpp) — 主机通用指标采集(CPU/内存/负载等,最大单文件 ~4.7 万字节)。
- [itl/metric-so/src/libcommon/dmidecode.cpp](../../itl/metric-so/src/libcommon/dmidecode.cpp) — 解析 dmidecode 上报硬件资产信息。
- [itl/metric-so/src/libmount/mount_info_metrics.cpp](../../itl/metric-so/src/libmount/mount_info_metrics.cpp) — 磁盘挂载点/容量采集。
- [itl/alarm/src/alarm.cpp](../../itl/alarm/src/alarm.cpp) — 告警主逻辑:阈值判定、重试间隔、告警状态机。
- [itl/alarm/src/mobile_ban.cpp](../../itl/alarm/src/mobile_ban.cpp) + [itl/alarm/conf/mobile_ban.xml](../../itl/alarm/conf/mobile_ban.xml) — 短信告警的"工作时间段/节假日"屏蔽规则。
- [itl/rrd/src/c_rrd_handler.cpp](../../itl/rrd/src/c_rrd_handler.cpp) — RRDtool 封装,写入/创建环形库。
- [itl/db/src/db_interface.cpp](../../itl/db/src/db_interface.cpp) — DB 服务统一 SQL 接口层(本模块最大单文件 ~5.8 万字节)。
- [itl/switch-monitor/src/main.cpp](../../itl/switch-monitor/src/main.cpp) — SNMP 交换机监控进程入口(用 net-snmp)。
- [itl/switch-monitor/src/collect-process-thread/data_processer.cpp](../../itl/switch-monitor/src/collect-process-thread/data_processer.cpp) — 交换机采集数据处理线程。
- [itl/switch-monitor/src/proto.h](../../itl/switch-monitor/src/proto.h) — 交换机/接口/指标/告警的内存结构定义(无网络命令字,纯采集端)。
- [itl/itl-speed/dbser/src/CMakeLists.txt](../../itl/itl-speed/dbser/src/CMakeLists.txt) — CDN 测速入库服务(基于淘米 `dbser` 框架的 SO 插件)。
- [itl/itl-speed/tst_spd/analyze_server/process_packet.cpp](../../itl/itl-speed/tst_spd/analyze_server/process_packet.cpp) — 测速原始包分析。
- [itl/itl-common](../../itl/itl-common) — 本模块公共底座:异步 server 框架、协议绑定、MySQL 封装、XML/日志/timer 工具。
- [itl/script/db_itl.sql](../../itl/script/db_itl.sql) + [itl/script/db_itl_trigger_v3.sql](../../itl/script/db_itl_trigger_v3.sql) — 完整库表结构(40+ 张表)与触发器。
- [itl/doc/V3/OA后台服务概要设计V3.doc](../../itl/doc/V3/OA后台服务概要设计V3.doc) — 权威设计文档(中文需求/概要设计/Visio 流程图,共 12.7MB)。

## 协议 / 数据亮点

### 命令字 / 协议头
ITL 自有一套二进制协议,通用包头见 [itl/itl-common/itl_proto.h#L31](../../itl/itl-common/itl_proto.h#L31)(`itl_proto_t`:`len/seq/cmd/ret/id/body[]`,`#pragma pack(1)`),head/node/db/rrd/alarm/control 复用同一包头(L48-54)。命令字与结构体由 `gen_proto` 工具从 XML **代码生成**到各模块 `src/proto/*_enum.h`(如 `find_alarm_cmd` / `find_db_cmd`,见 [itl/alarm/src/proto.h#L65](../../itl/alarm/src/proto.h#L65))——**这些生成产物在泄漏树里大多缺失**,需用 [itl/gen_proto](../../itl/gen_proto) 重新生成才能补全 cmdid 表。

唯一保留下来的真实命令字枚举在 CDN 测速子系统 [itl/itl-speed/dbser/src/proto/cdn_rate_enum.h](../../itl/itl-speed/dbser/src/proto/cdn_rate_enum.h),示例:
- `cdn_report_ip_cmd = 0x1001`(上报相同 IP 的分钟表数据)[#L6](../../itl/itl-speed/dbser/src/proto/cdn_rate_enum.h#L6)
- `cdn_report_prov_cmd = 0x1002`(按省市运营商上报 CDN 速度)[#L7](../../itl/itl-speed/dbser/src/proto/cdn_rate_enum.h#L7)
- `net_stat_ip_report_cmd = 0x1211`(替换旧 1003/1005/1007)[#L26](../../itl/itl-speed/dbser/src/proto/cdn_rate_enum.h#L26)

### 数据结构
指标三态:`metric_type` 1=写 RRD、2=报警、3=两者都做(见 [itl/switch-monitor/src/proto.h#L107](../../itl/switch-monitor/src/proto.h#L107));告警元信息 `alarm_info_t` 带 `warning_val/critical_val/op/normal_interval/retry_interval/max_atc`([itl/switch-monitor/src/proto.h#L94](../../itl/switch-monitor/src/proto.h#L94))——典型的 Nagios 式"警告/严重双阈值 + 重试退避"模型。

### SQL 数据
[itl/script/db_itl.sql](../../itl/script/db_itl.sql) 是一份完整的 `db_itl` MySQL dump(server 5.0.51a),40+ 张中文注释表,价值不在游戏数据而在**淘米运维台账模型**:
- 资产/机房:`t_idc_info`(机房)、`t_cabinet_info`(机柜)、`t_server_info`、`t_switch_info`、`t_storage_info`、`t_firewall_info`、`t_asset_type_info`、`t_supplier_info`、`t_company_info`、`t_department_info`。
- 监控核心:`t_node_info`、`t_metric_info`、`t_metric_group_info`、`t_alarm_info`、`t_so_info`(采集 so 注册)。
- 触发器([itl/script/db_itl_trigger_v3.sql](../../itl/script/db_itl_trigger_v3.sql)):`tr_update_metric_info`、`tr_insert_alarm_strategy_info` 等,把配置表变更同步到运行态,是这套系统"配置即生效"的关键设计。
- 时间段告警屏蔽配置 [itl/alarm/conf/mobile_ban.xml](../../itl/alarm/conf/mobile_ban.xml):工作日 09:30–18:30 默认屏蔽、按 2012/2013 国庆/元旦节假日开关全天短信告警——一个很具体的"别在下班时间狂发短信"运维巧思。

## 对我们有什么用(moleworld-server / MoleWorld-offline)

整体定位:**参照为主,基本不直用**。它监控的是淘米的真实生产机群,与摩尔庄园游戏逻辑/数据无任何重叠。

- **直用:几乎没有。** 没有任何游戏协议、玩法数据或摩尔庄园配置可搬。
- **参照(有限价值):**
  - [itl/itl-common](../../itl/itl-common) 的异步 server 框架、`itl_proto_t` 包头、`gen_proto` 由 XML 生成协议的范式,与其它游戏模块(mole/monster 等)同源,可当成"淘米后台 C++ 协议栈是怎么组织的"的旁证;但要看协议骨架,优先读 [c.md](./c.md) / [libtaomee.md](./libtaomee.md) 与 [protocol.md](../protocol.md),本模块只是它的一个应用样本。
  - 告警状态机(双阈值 + 重试退避 + 工作时间段屏蔽)和 RRD 画图思路,若 MoleWorld 复活后要做"私服运维监控",可作设计参考——但实现完全可以用现代工具(Prometheus/Grafana)替代,不必移植这套 2010 年的 C++/RRDtool/net-snmp 代码。
  - `db_itl` 的资产/机房建模可作"理解淘米当年机房规模与组织结构"的史料,对复活工程无直接技术用途。

结论:对 moleworld-server / MoleWorld-offline,**优先级最低**,了解即可,不需要投入移植。

## 注意 / 坑

- **生成代码大面积缺失:** 各模块 `proto.h` include 的 `src/proto/itl_*_enum.h`、`itl_*_func_def.h`、`itl_*.h` 结构体几乎都**不在泄漏树里**(它们由 [itl/gen_proto](../../itl/gen_proto) 从 XML 现生成)。因此除 itl-speed 的 `cdn_rate_enum.h` 外,**完整 cmdid 清单需自己跑 gen_proto 才能还原**,文档中其它命令字属"待证"。
- **依赖一堆外部库/旧环境:** `libtaomee` / `libtaomee++`(框架)、`glib`、`mysqlclient`(5.0.51a 时代)、`net-snmp`、`RRDtool`、CMake 2.6,且 dbser/itl-speed 依赖私有 `dbser` 框架——脱离淘米构建环境基本编不动。
- **二进制/文档为主的部分:** [itl/doc](../../itl/doc) 有 12.7MB 的 `.doc/.docx/.vsd/.xls`(需求、概要设计 V1/V2/V3、Visio 架构图、测试用例),信息量很大但都是二进制 Office 文件,无法在仓库内直接 diff/检索,需另存打开。
- **itl-speed 体量虚高:** 该子目录 408 文件 / 54MB,占了整个 itl 磁盘的绝大部分,但很多是测速二期的重复 `qdes.c` / `proto.h` 副本和数据,真正核心逻辑集中在 `analyze_server` 与 `dbser`。
- **年代局限:** MySQL 5.0、RRDtool、SNMP v2c 社区串明文(见 `switch_info_t.community`),都是 2010–2012 年的运维范式,安全/可观测性远不及现代方案。
- **评级说明:** 维持 baseline ★。代码工程质量不低(分布式 head/node/so 架构、触发器驱动配置、完整 SQL 与中文设计文档齐全),但对"摩尔庄园复活"这一目标几乎零直用价值,故不上调。
