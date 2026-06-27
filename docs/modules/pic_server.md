# 模块 pic_server · 淘米全平台图片服(上传/缩略图/存储/CDN 回源)

> 价值:★★  ·  游戏:基础设施(全游戏共用,含摩尔庄园相册/头像)  ·  源码量:约 2.5 万行
> 📖 深度功能解析(每个部分实际在干什么):[deep/pic_server.md](../deep/pic_server.md) · 彩蛋见 [easter-eggs.md](../easter-eggs.md)

## 是什么 / 在整套架构里的角色

`pic_server` 是淘米所有页游共用的**图片基础设施**:负责把 Flash 客户端上传的图片(相册照片、用户头像 logo)落盘存储、生成缩略图、分配全局唯一定位码、并提供浏览/回源服务。它不属于某一款游戏,是横向支撑——摩尔庄园的相册、个人头像,赛尔号/功夫派等的图片需求都可走这套服务。

它**不是单进程**,而是一组协作的 C 服务进程 + FastCGI/CGI 前端,典型链路如下:

```
Flash 客户端
   │ HTTP(multipart 上传 / GET 浏览)
   ▼
pic_cgi / pic_cgi_v2 (CGI)  或  FCGI_PIC (FastCGI)   ← Web 前端,解析上传/鉴权
   │ 自定义 TCP 二进制协议(protocol_t 包头)
   ▼
FileServ(fileserv)  ← 原图落盘 + 分配 lloccode + 派生缩略图任务(GThreadPool)
   │
   ├──► ThumbServ(thumbserv)  ← 缩略图用 libgd 生成并存储(mmap 大文件)
   ├──► AdminServ(admin_serv) ← 删除/转移/改属性等运维型批处理
   └──► KeyServ(key_serv)     ← 防刷:按 userid 限制每日上传次数(mmap 计数)
   │
2pic/webproxy ← 第二代:把 lloccode 回 POST 给 web 后端(curl + json_parser)
```

- **FileServ / ThumbServ / AdminServ / KeyServ / webproxy** 都不是独立 `main`,而是编译成 `.so`(`fileserv.so`/`keyserv.so` 等),由共享框架 `iter_serv`(见 [c.md](./c.md) / [libtaomee.md](./libtaomee.md))的 `IterServer` 宿主进程 dlopen 装载,通过回调 `init_service / on_pkg_received / proc_events` 接入。证据见 [fileserv/dll_if.c](../../pic_server/fileserv/dll_if.c#L82) 的 `on_pkg_received` 分发与 [fileserv/Makefile](../../pic_server/fileserv/Makefile) 的 `TARGETS = fileserv.so`。
- **bench.conf** 是宿主框架的统一配置(log/绑定端口/dll 文件),如 [fileserv/bench.conf](../../pic_server/fileserv/bench.conf) 绑 `6999`、[key_serv/bench.conf](../../pic_server/key_serv/bench.conf) 绑 `6482`。
- 仓库里同时存在第一代(顶层 `fileserv/` `thumbserv/`)与**第二代 `2pic/`**(摩尔庄园2/跨服时期的图片服重构,加了 webproxy 回源)。作者署名 maxwell (zdl)@taomee,关键头注明 `Created: 11/25/2010`(见 [key_serv/key_proto.h](../../pic_server/key_serv/key_proto.h#L9)),与"2010–2011 大扩张期"吻合。

## 关键子目录与文件(每条可跳转 + 一句话作用)

- [pic_server/include/common.h](../../pic_server/include/common.h) — 全套服务间 TCP 命令字总表(ts/as/fs 三组 enum),最该先读。
- [pic_server/include/proto.h](../../pic_server/include/proto.h) — 通用包头 `protocol_t`(len/cmd/id/seq/ret + 柔性 body)与 PKG/UNPKG 大端序打包宏。
- [pic_server/include/lloccode.h](../../pic_server/include/lloccode.h) — **核心数据格式**:64 字节"原图定位码 lloccode"的生成/解析(hostid+type+日期+目录+图号+缩略图 id+校验和),决定图片落盘路径。
- [pic_server/fileserv/dll_if.c](../../pic_server/fileserv/dll_if.c) — FileServ 的框架入口与命令分发(`init_service`/`on_pkg_received`)。
- [pic_server/fileserv/save_file.c](../../pic_server/fileserv/save_file.c) — 原图落盘主逻辑:mmap 目录索引、按日期/目录分桶、GThreadPool 异步写盘(最大单文件)。
- [pic_server/fileserv/thumb.c](../../pic_server/fileserv/thumb.c) — FileServ 侧派生缩略图、与 ThumbServ 交互的回调。
- [pic_server/fileserv/fs_list.c](../../pic_server/fileserv/fs_list.c) / [include/fs_list.h](../../pic_server/include/fs_list.h) — fileserv/thumbserv 集群列表(按 id 路由、负载权重)。
- [pic_server/thumbserv/thumb.c](../../pic_server/thumbserv/thumb.c) — ThumbServ 主逻辑:libgd 缩放生成缩略图、mmap 大文件存储(最大源文件,~4.4 万字节)。
- [pic_server/thumbserv/thumb.h](../../pic_server/thumbserv/thumb.h) — 缩略图存储结构 `thumb_t`/`up_thumb_t` 与命令处理函数声明。
- [pic_server/key_serv/service.cpp](../../pic_server/key_serv/service.cpp) — KeyServ:`map<int,int>` 按 userid 计当日上传次数,每天 23:54 定时清零(防刷)。
- [pic_server/key_serv/key_proto.h](../../pic_server/key_serv/key_proto.h) — KeyServ 协议头(含作者/日期署名)与 `proto_create_album`/`proto_upload_pic` 等命令字。
- [pic_server/admin_serv/admin.c](../../pic_server/admin_serv/admin.c) — AdminServ:删除/批量转移/改属性等运维命令(`proto_as_*`)。
- [pic_server/admin_serv/trans_multi_pics.c](../../pic_server/admin_serv/trans_multi_pics.c) — 批量迁移多张图片(跨相册/跨服迁移)。
- [pic_server/FCGI_PIC/trunk/src/fcgi_pic.c](../../pic_server/FCGI_PIC/trunk/src/fcgi_pic.c) — FastCGI 前端入口(浏览原图/缩略图、改缩略图)。
- [pic_server/FCGI_PIC/trunk/src/fcgi_proto.h](../../pic_server/FCGI_PIC/trunk/src/fcgi_proto.h) — FastCGI 与 fileserver 的收发结构 + HTTP 响应头宏(Cache-Control: max-age=94608000)。
- [pic_server/pic_cgi/trunk/src/cgi_upload.c](../../pic_server/pic_cgi/trunk/src/cgi_upload.c) — 第一代 CGI 上传相册图;[cgi_upload_logo.c](../../pic_server/pic_cgi/trunk/src/cgi_upload_logo.c) 上传头像。
- [pic_server/pic_cgi_v2/trunk/src/cgi_proto.h](../../pic_server/pic_cgi_v2/trunk/src/cgi_proto.h) — 第二代 CGI 协议:图片类型枚举、`recv_fileserver_t`/`snd_fileserer_t` 上传交互结构、错误码 4000+。
- [pic_server/2pic/webproxy/json_parser.c](../../pic_server/2pic/webproxy/json_parser.c) + [service.c](../../pic_server/2pic/webproxy/service.c) — 第二代 webproxy:用 curl 把 lloccode 回 POST 给 web 后端,自带 JSON 解析。
- [pic_server/pic_shell/client.c](../../pic_server/pic_shell/client.c) — 运维/测试用命令行客户端,直连服务发包。
- [pic_server/cgi_conf/crossdomain.xml](../../pic_server/cgi_conf/crossdomain.xml) — Flash 跨域策略(`domain="*"` 全开,见"坑")。

## 协议 / 数据亮点

**统一包头**(所有服务共用,见 [include/proto.h#L5](../../pic_server/include/proto.h#L5)):
```c
struct protocol { uint32_t len; uint16_t cmd; uint32_t id; uint32_t seq; uint32_t ret; uint8_t body[]; }
```
整数字段大端序,通过 `PKG_UINT*/UNPKG_UINT*` 宏读写;`id` 通常承载 userid 或 albumid。

**命令字三大组**(权威表在 [include/common.h](../../pic_server/include/common.h)):
- ThumbServ:`proto_ts_upload_thumb = 1001`([common.h#L14](../../pic_server/include/common.h#L14))、`proto_ts_cgi_get_thumb = 1003`、`proto_ts_trans_multi_add_thumbs = 1016`。
- AdminServ:`proto_as_delete_file = 3001`([common.h#L28](../../pic_server/include/common.h#L28))、`proto_as_trans_multi_file = 3003`。
- FileServ:`proto_fs_cgi_upload_file = 1`([common.h#L36](../../pic_server/include/common.h#L36))、`proto_fs_delete_file = 1011`、`imp_proto_upload_file = 2000`(导入路径)。
- KeyServ 业务侧:`proto_upload_pic = 1002`、`proto_get_thumb = 1003`([key_serv/key_proto.h#L32](../../pic_server/key_serv/key_proto.h#L32))。
- 第二代 webproxy:`proto_wp_post_llocc = 5001`(回源 POST,见 [2pic/include/common.h#L50](../../pic_server/2pic/include/common.h#L50))。

注意:命令字在不同子模块的 enum 里**会重号/改值**(例如 `proto_fs_delete_file` 在一代是 1011、二代 [2pic/include/common.h](../../pic_server/2pic/include/common.h) 改成 5),跨代对接需以各自 `common.h` 为准。

**lloccode(图片定位码)** 是本模块最值钱的数据格式(见 [include/lloccode.h](../../pic_server/include/lloccode.h)):64 字节定长字符串,字段拼接为 `hostid(2) + type(1) + branchtype(1) + 日期(8) + dir(3) + pic(3) + uid(10) + 4×thumbid(8) + 校验和`,解析函数 `analyse_lloccode_ex` 据此还原磁盘路径 `/opt/taomee/file_root/{type}/{date}/{dir}/{pic}.{JPEG|PNG|GIF}`。图片只支持 jpg/png/gif(见 [cgi_proto.h#L38](../../pic_server/pic_cgi_v2/trunk/src/cgi_proto.h#L38))。

**配置/集群**:[fileserv/file_serv_list.xml](../../pic_server/fileserv/file_serv_list.xml) 与 [thumb_serv_list.xml](../../pic_server/fileserv/thumb_serv_list.xml) 暴露真实内外网 IP(如 `114.80.173.91` / `192.168.9.91`)和 thumb id 分段(1~1亿 / 1亿~2亿),揭示当年是按 id 区间水平分片的多机部署。存储根目录硬编码 `/opt/taomee/file_root/`、缩略图根 `/opt/taomee/photofs/data`。

本模块无 `.sql`——它只管二进制文件存储,图片元数据(相册归属等)由各游戏自己的 DB 服管理,pic_server 只返回 lloccode 给上层入库。

## 对我们有什么用(moleworld-server / MoleWorld-offline)

定位:**整体参照,极少直用**。

- **直用(可拿走的小零件)**:[include/lloccode.h](../../pic_server/include/lloccode.h) 的 lloccode 编解码、[include/proto.h](../../pic_server/include/proto.h) 的 PKG/UNPKG 大端宏是纯函数,可直接抄进离线服当工具。
- **参照(设计层面)**:如果 MoleWorld 要复活"摩尔庄园相册/上传头像"功能,这套"CGI 解析上传 → FileServ 落盘分配 lloccode → ThumbServ 生成缩略图 → 回 lloccode 给业务库"的链路是完整可借鉴的参考实现;但实现重度依赖 `iter_serv` 宿主框架、glib `GThreadPool`、libgd、mmap 索引,移植成本高。对单机离线复活,直接用一个现代静态文件服务 + 一张 `lloccode→本地路径` 映射表即可平替,**不建议照搬这套多进程架构**。
- 对"登录进游戏看世界"这一最小目标,pic_server **不是必需路径**——相册/头像是次要玩法,可后置或用占位实现。

## 注意 / 坑

- **二进制无源**:仓库里有编译产物 `FileServ`/`ThumbServ`/`KeyServ`/`fileserv.so` 等,但 FileServ/ThumbServ 的宿主 `main` 在外部框架 `iter_serv`(只给了 `.so` 侧),无法独立 `make` 出可跑进程;`KeyServ` 等是 ELF 二进制,非源码。
- **缺依赖**:编译需 glib-2.0/gthread、libgd、libxml2、`-ltaomee`(libtaomee 内部库,见 [libtaomee.md](./libtaomee.md)),且 include 路径指向 `iter_serv/net_if.h` 这类外部头,本仓未必齐全。
- **硬编码路径与 IP**:`/opt/taomee/file_root`、内网 `192.168.9.x`、`pic1-bus.61.com` 等写死,迁移须改 [include/fs_list.h](../../pic_server/include/fs_list.h) 和各 `*_serv_list.xml`。
- **安全**:[crossdomain.xml](../../pic_server/cgi_conf/crossdomain.xml) 跨域 `domain="*"` 全开、上传鉴权 KEY 在头文件里明文 `"!tA:mEv,"`(见 [fcgi_proto.h#L10](../../pic_server/FCGI_PIC/trunk/src/fcgi_proto.h#L10))——这是 2010 年代写法,复活时切勿照抄上线。
- **年代局限**:一代/二代(`2pic/`)并存且命令字重号,阅读时务必锁定子目录对应的 `common.h`;FastCGI 走的是已淘汰的 `fcgi_stdio.h` 模型。
- `lloccode.h` 中第一版 `create_lloccode`/`analyse_lloccode` 用 `#if 0` 注掉,实际走的是 `analyse_lloccode_ex`,读码时别看错分支。
