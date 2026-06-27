# pic_server 深度档案 · 每个部分实际在干什么

> 配套概览见 [../modules/pic_server.md](../modules/pic_server.md) · 协议 [../protocol.md](../protocol.md) · 总入口 [../../README.md](../../README.md)

本档案不重复"是什么",只追"运行时它到底怎么跑"。所有断言都带真实行号,跨进程数据流逐字段对齐。**编码核对**:本模块源文件多为 GBK 注释,文中所引中文注释均已用 `iconv -f gbk` 复核。

---

## 一图看懂:进程/组件拓扑与数据流

pic_server 是一组**协作进程**,不是单体。它们之间走同一个二进制包头 `protocol_t`(见 [include/proto.h#L5](../../pic_server/include/proto.h#L5)),但**包头字段读的是主机字节序**(`pp->len`、`pp->cmd` 直接取,无 `ntohl`,见 [fileserv/dll_if.c#L72-L74](../../pic_server/fileserv/dll_if.c#L72));只有 **body 内的字段**才用 `PKG_*`/`UNPKG_*` 宏做大端/主机序转换。一句话:**头是本机序,身随宏定**——这是阅读时最容易踩的坑。

```
Flash 客户端(摩尔庄园相册/头像)
   │  ① HTTP GET ?session_key=<DES密文>         ② HTTP multipart 上传(file + session_key)
   ▼                                              ▼
 FCGI_PIC(只读浏览,FastCGI 常驻)            pic_cgi_v2 / pic_cgi(上传,CGI 每请求 fork)
   │  DES 解密 session_key 取 cmd+lloccode/参数      │  DES 解密 session_key 取 userid/albumid/限宽高
   │                                                │  gd 验真图(魔数+能否解码)
   ├─ 原图: 直接 fopen 磁盘 /opt/taomee/file_root  ├──TCP 6999──► FileServ(fileserv.so)
   │        (不经任何后端,见 fcgi_get_image.c)     │
   └─ 缩略图: TCP 7000 问 ThumbServ 拿字节 ────┐    │  FileServ:① 落原图(GThreadPool pool1)
                                              │    │            ② gd 生成缩略图(pool2)
                                              ▼    │            ③ 缩略图 ──TCP 7000──► ThumbServ
                                         ThumbServ(thumbserv.so) ◄────────────────────┘
                                              │  按 albumid 分 1000 目录 + 变长块文件存储
                                              │  返回每张缩略图的 thumb_id(偏移编码)
                                              ▼
   FileServ 收齐 thumb_id → 拼进 lloccode → 回 CGI → CGI 输出 JSON {lloc, albumid, photoid}

旁路服务:
  KeyServ(keyserv.so, 端口见 bench)  → 分配全局 albumid(自增) + 限每人每天 100 次
  AdminServ(adminserv.so, 6998)      → 删除/批量跨服迁移图片+缩略图
  webproxy(2pic 二代, curl)          → 把 lloccode 回 POST 给业务后端 bus.album.addPhoto
```

宿主框架:FileServ/ThumbServ/AdminServ/webproxy 都**不是独立 main**,而是 `.so`,由外部 `iter_serv`(`IterServer`)宿主 dlopen,回调约定为 `init_service` / `get_pkg_len` / `on_pkg_received` / `proc_events` / `on_conn_closed`(见 [fileserv/dll_if.c#L16-L117](../../pic_server/fileserv/dll_if.c#L16))。端口:FileServ=6999、ThumbServ=7000、AdminServ=6998(见 [include/fs_list.h#L9-L10](../../pic_server/include/fs_list.h#L9) 与各 `bench.conf`)。

---

## 逐部分拆解

### 1. FileServ —— 全模块的中枢(fileserv/)

**实际职责**:接收 CGI 上传的原图字节,落盘并分配 `lloccode`,同时驱动缩略图生成与跨进程交付。它是唯一同时与 CGI、ThumbServ 双向通信的进程。

启动时一次性做完所有准备([dll_if.c#L16-L48](../../pic_server/fileserv/dll_if.c#L16)):
- 加载 `file_serv_list.xml` / `thumb_serv_list.xml` 集群表,并用本机 `bind_ip` 反查出自己的 `hostid`(`get_fs_id`,[fs_list.c#L44](../../pic_server/fileserv/fs_list.c#L44))——这个 hostid 后面会写进每个 lloccode 的头 2 字节。
- `init_dir_index` mmap 一个 40960 字节的索引文件([save_file.c#L59-L88](../../pic_server/fileserv/save_file.c#L59)),里面按 `file_type` 存"今天写到第几个目录/第几张图"。
- 建两个 glib 线程池:`pool1=save_file`(默认 20 线程,落盘)、`pool2=create_thumbs`(默认 10 线程,生成缩略图),见 [dll_if.c#L40-L45](../../pic_server/fileserv/dll_if.c#L40)。
- 注册 1 秒一次的扫描定时器 `scan_all_work_file`,负责收尾(见机制章节)。

**命令分发**([dll_if.c#L82-L112](../../pic_server/fileserv/dll_if.c#L82)):

| cmd | 值 | 处理函数 | 干什么 |
|---|---|---|---|
| `proto_fs_cgi_upload_file` | 1 | `upload_file_cmd` | CGI 上传相册原图 |
| `proto_fs_cgi_chg_thumb` | 2 | `change_picture_thumbs_cmd` | 改某图的缩略图裁剪 |
| `imp_proto_upload_file` | 2000 | `imp_upload_file` | 历史数据导入(强制 2010-10~2011-01) |
| `imp_proto_upload_logo` | 2001 | `imp_upload_logo` | 头像导入(缩略图字节直传) |
| `proto_ts_upload_thumb` | 1001 | `upload_thumb_callback` | **ThumbServ 回包**:缩略图存好了,带回 thumb_id |
| `proto_fs_delete_file` | 1011 | `delete_file_cmd` | 删原图 |

**上传主路径 `upload_file_cmd`**([save_file.c#L319-L432](../../pic_server/fileserv/save_file.c#L319)),逐字段拆 body:

```
[albumid/key(4)] [file_type(4)] [branch_type(4)] [w_limit(4)] [h_limit(4)] [thumb_cnt(4)]
  ×thumb_cnt 个 { clip_w clip_h start_x start_y thumb_w thumb_h }(各 4 字节,共 24)
[file_len(4)] [file 原始字节...]
```
关键运行行为:
- **防重**:用 `albumid` 当 key 查 `upload_maps` 哈希,若同一相册有未完成的上传,直接回 `err_upload_busy_now`([L340-L344](../../pic_server/fileserv/save_file.c#L340))——这是它"一次只让一张图在飞"的并发闸。
- **分配路径与 lloccode**:`create_picpath_llocc_logpath`([L148-L195](../../pic_server/fileserv/save_file.c#L148))在 mutex 内推进当天计数器(满 999 张换目录,见 `MAX_PIC_ID`),拼出磁盘路径 `/opt/taomee/file_root/{type}/{YYYYMMDD}/{dir:03}/{pic:03}.{JPEG|PNG|GIF}` 与半成品 lloccode(此时缩略图段还是 0)。
- 落盘任务 push 进 `pool1`;若 `thumb_cnt>0` 再 push 进 `pool2` 生成缩略图([L418-L425](../../pic_server/fileserv/save_file.c#L418))。**注意**:函数 `return 0` 后并不立即回 CGI——它把 work 挂进全局队列 `gqlist`,由定时器收尾(见机制章节)。只有"无缩略图"分支会当场回包并 `return -1` 关连接([L419-L423](../../pic_server/fileserv/save_file.c#L419))。

**落盘 `save_file`**([L279-L317](../../pic_server/fileserv/save_file.c#L279)):线程池工作函数。建目录(递归 `file_make_dir`)、若设了 `w_limit/h_limit` 先调 `limit_picture` 用 gd 等比缩到限制内([thumb.c#L140-L184](../../pic_server/fileserv/thumb.c#L140)),再 `write` 原图,置 `save_file_tag=1`。

**与 ThumbServ 通信**:`send_pic_to_thumbserv`([thumb.c#L256-L271](../../pic_server/fileserv/thumb.c#L256))把生成好的缩略图字节打成 `proto_ts_upload_thumb` 包,经 `send_to_thumbserv` 按 albumid 路由发出([fs_list.c#L97-L117](../../pic_server/fileserv/fs_list.c#L97))。ThumbServ 存完回 thumb_id,`upload_thumb_callback`([thumb.c#L345-L406](../../pic_server/fileserv/thumb.c#L345))在 `gqlist` 里按 `unique_key` 找回对应 work,把 4 个 thumb_id `create_lloccode` 写进 lloccode 末尾并算校验和,然后 `send_result_to_cgi`。

### 2. ThumbServ —— 缩略图的自研变长块存储引擎(thumbserv/)

**实际职责**:这是整个模块技术含量最高的部分。它**不是把缩略图当独立小文件存**,而是把同一相册(albumid)的所有缩略图打包进**少量大文件**做变长块分配——本质是一个手写的 mini 文件系统,目的是避免海量小文件拖垮 inode 与磁盘寻道。

存储布局([thumb.c#L18-L94](../../pic_server/thumbserv/thumb.c#L18)):
- 根目录 `/opt/taomee/photofs/data`,按 `albumid % 1000` 分成 000~999 共 **1000 个目录**([ALBUM_DIR_SIZE=1000](../../pic_server/thumbserv/thumb.c#L23))。
- 每目录下有索引文件 `photofs` + 数据文件 `photofs0`、`photofs1`...(`ifileno` 代表块规格档位,块大小 `128KB × 2^ifileno`)。
- 索引 mmap 进内存,`pmaps_offset[dirid][albumid/1000]` 一个 int 同时编码"块偏移(高 24 位)+ 文件档位(低 8 位)"([L109-L111](../../pic_server/thumbserv/thumb.c#L109))。

**thumb_id 是一个自编码的定位整数**(不是数据库主键):`photosid = (缩略图字节长度 << 15) | 块内偏移`([save_photo L121](../../pic_server/thumbserv/thumb.c#L121)、[read_photo L538-L539](../../pic_server/thumbserv/thumb.c#L538))。所以拿到 thumb_id 就能反算出"读多少字节、从哪个块偏移读",无需再查表——这就是为什么 lloccode 里塞 4 个 thumb_id 就够定位 4 张缩略图。

`save_photo`([L96-L364](../../pic_server/thumbserv/thumb.c#L96))核心:
- 首张图走"分配新块、写 1KB 对齐头 `photos_head_t`(size/photoscount)+ `photos_info_t`(photosid/stime)+ 图字节"分支。
- 后续图先扫块内 free 列表找够大的空洞复用,找不到就在块尾 append,块满则升档(`photos_block_len * 2`)([L228-L240](../../pic_server/thumbserv/thumb.c#L228))。free 块用魔数 `"FREE"` 标记([L25](../../pic_server/thumbserv/thumb.c#L25)),删除时回收进链表(`del_photo`,[L366](../../pic_server/thumbserv/thumb.c#L366))。

`read_photo`([L523-L564](../../pic_server/thumbserv/thumb.c#L523)):从 thumb_id 拆出 size/offset,`pread64` 读出 `photos_info_t` + 图字节,**校验 `photo_info->photosid == 入参 photosid && stime>0`** 才认有效([L554](../../pic_server/thumbserv/thumb.c#L554))——这是它防"偏移撞库读到别人图"的最后一道闸。

命令分发([thumbserv/dll_if.c#L49-L84](../../pic_server/thumbserv/dll_if.c#L49)):`upload_thumb_cmd`(1001,存)、`get_thumb_cmd`(1003,读,供 FCGI 浏览)、`del_thumbs_cmd`/`del_album_cmd`(删)、`change_thumb_cmd`(改)、`trans_multi_*`(批量迁移)。注意 1003 处理后宿主返回 -1 关连接([dll_if.c#L64-L66](../../pic_server/thumbserv/dll_if.c#L64))。

### 3. FCGI_PIC —— 只读浏览前端(FastCGI 常驻)

**实际职责**:对外的图片 URL 后端,只服务 GET。`main` 是 `while(FCGI_Accept())` 常驻循环([fcgi_pic.c#L49-L201](../../pic_server/FCGI_PIC/trunk/src/fcgi_pic.c#L49))。

请求格式:`GET ...?session_key=<hex(DES密文)>`。`parser_session_key`([L28-L48](../../pic_server/FCGI_PIC/trunk/src/fcgi_pic.c#L28))把十六进制串还原成字节、用**硬编码 DES 密钥 `"!tA:mEv,"`**([fcgi_proto.h#L10](../../pic_server/FCGI_PIC/trunk/src/fcgi_proto.h#L10))解密,头 4 字节是 cmd,后面是参数体。按 cmd 分四路:

- `proto_get_image`(1004,看原图):取出 lloccode → `FCGI_GET_IMAG_Main`([fcgi_get_image.c#L81](../../pic_server/FCGI_PIC/trunk/src/fcgi_get_image.c#L81))。它**先验 lloccode 校验和**([L107-L115](../../pic_server/FCGI_PIC/trunk/src/fcgi_get_image.c#L107)),再 `Fcgi_analyse_lloccode` 解析出磁盘路径,**直接 `FCGI_fopen` 读盘并吐字节**——原图浏览完全不经任何后端进程,纯静态文件直读。
- `proto_get_thumb`(1003,看缩略图):`FCGI_get_thumb`([fcgi_get_thumb.c#L32](../../pic_server/FCGI_PIC/trunk/src/fcgi_get_thumb.c#L32))建 TCP 连到 ThumbServ,发 `(albumid, thumb_id)` 取回字节再吐给浏览器。
- `proto_change_thumb`(1005,改缩略图):解析裁剪参数转给 fileserver。
- 所有响应带 `Cache-Control: max-age=94608000`(3 年,[fcgi_proto.h#L81](../../pic_server/FCGI_PIC/trunk/src/fcgi_proto.h#L81)),且对 `If-Modified-Since` 一律回 304——**靠浏览器缓存扛读流量**。

### 4. pic_cgi / pic_cgi_v2 —— 上传前端(CGI 每请求一进程)

**实际职责**:接 Flash 的 multipart 上传,做"验真→转发 FileServ→回 JSON"。二代 [pic_cgi_v2/trunk/src/cgi_upload.c](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c) 的 `cgiMain`([L208](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L208))运行流:

1. 取表单 `session_key`,`parse_web_data` DES 解密([L161-L179](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L161))出 `userid/channel/albumid/ip/time/width_limit/height_limit`。
2. **两道鉴权**:① 解出的 `ip` 必须等于本机 `bind_ip`([L263-L271](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L263))——即只信任指定调度机签发的票;② `now - time > 5 分钟`即超时拒绝([L274-L280](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L274))。这是它防止 session_key 被重放的机制。
3. **验真图**:读文件头 8 字节 `parse_uploadfile_type` 按魔数(0xFFD8=jpg、PNG 签名、GIF87a/89a)判类型([L29-L65](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L29)),再用 `gd` 真去解码一遍确认不是伪装文件([L293-L303](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L293))。
4. `connect_with_fileserver`([L124-L159](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L124))打 `proto_fs_cgi_upload_file` 包,同步 `do_net_io` 等 FileServ 回 lloccode。
5. `cgi_print_result` 用 json-c 输出 `{result, albumid, photoid, lloc, hostid, len}`([L180-L207](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L180))。错误码统一在 4000+([cgi_proto.h#L21-L34](../../pic_server/pic_cgi_v2/trunk/src/cgi_proto.h#L21))。

一代 [pic_cgi](../../pic_server/pic_cgi/trunk/src/cgi_upload.c) 同构,但上传头像、移图分了独立 CGI(`cgi_upload_logo.c`、`cgi_mv_logo.c`、`cgi_mv_pic.c`)。

### 5. KeyServ —— albumid 发号 + 防刷(key_serv/)

**实际职责**:不存任何图片,只做两件事(`on_pkg_received`,[service.cpp#L82-L119](../../pic_server/key_serv/service.cpp#L82))。它**只认一条命令 `proto_create_album`(1001)**:
- **发号**:`g_mmap->get_index()`([mmap.cpp#L54-L58](../../pic_server/key_serv/mmap.cpp#L54))对 mmap 文件 `records` 里的 `index` 做 `++` 原子返回——这就是全局唯一、单调递增的 albumid(也是 ThumbServ 的分桶 key)。
- **防刷**:`map<int,int> g_user_cnt_map` 按 userid 累计当日建相册次数,超 `MAX_COUNT_DAY=100`([key_proto.h#L48](../../pic_server/key_serv/key_proto.h#L48))就回 `err_get_too_many_keys`([service.cpp#L98-L105](../../pic_server/key_serv/service.cpp#L98))。计数器靠定时器在**每天 23:54** 清零([service.cpp#L25-L43](../../pic_server/key_serv/service.cpp#L25))。

### 6. AdminServ —— 运维与跨服迁移(admin_serv/)

**实际职责**:删除与批量迁移。`on_pkg_received`([admin_serv/dll_if.c#L70](../../pic_server/admin_serv/dll_if.c#L70))分发删单文件(3001)、删多文件(3002)、改属性(3004)、跨服批量迁移(`proto_as_trans_multi_file`=3003)。迁移是个三段握手([trans_multi_pics.c](../../pic_server/admin_serv/trans_multi_pics.c)):`trans_multi_file_get`(从源拉缩略图)→`trans_multi_file_add`(写到目标 ThumbServ)→`trans_multi_file_del`(删源),用哈希表 `trans_multi_maps` 跟踪在途任务([trans_multi_pics.c#L31-L36](../../pic_server/admin_serv/trans_multi_pics.c#L31))。这是当年机房扩容/换 hostid 时搬图的工具。

### 7. webproxy —— 二代新增的回源桥(2pic/webproxy/)

**实际职责**:一代里 lloccode 是 CGI 直接回给 Flash 由前端再入库;二代加了 webproxy 主动把结果 HTTP 回 POST 给业务后端。`post_llocc_cmd`([worker.c#L7-L75](../../pic_server/2pic/webproxy/worker.c#L7))收 `proto_wp_post_llocc`(5001),拼出 URL `?method=bus.album.addPhoto&uid=..&albumid=..&llocc=..&file_name=..`([worker.c#L34-L41](../../pic_server/2pic/webproxy/worker.c#L34)),用 libcurl 请求业务总线,自带 `json_parser.c` 解析返回([json_callback](../../pic_server/2pic/webproxy/json_parser.c)),把后端分配的 `photoid` 回给调用方。这是二代"图片服自己负责把元数据写进业务库"的设计转向。

---

## 关键机制/算法实战

### 机制一:lloccode —— 64 字节自描述定位码

lloccode 是本模块的"硬通货",**不查任何表就能从中算出原图磁盘路径 + 4 张缩略图位置 + 自校验**。生成分两阶段:

1. FileServ 落盘时先写前 54 字节(`create_picpath_llocc_logpath`,[save_file.c#L189-L192](../../pic_server/fileserv/save_file.c#L189)):
   ```
   %02X  hostid(本机集群id)
   %u    type(1=图片)
   %u    branch_type(1=jpg 2=png 3=gif)
   %4u%02u%02u  日期 YYYYMMDD
   %03u  dir_id
   %03u  pic_id
   %010u userid
   ```
2. 缩略图存完拿到 thumb_id 后,`create_lloccode`([lloccode.h#L53-L66](../../pic_server/include/lloccode.h#L53))把 4 个 `%08X` thumb_id 续写进去,最后 4 字节是校验和 `%02X%02u`(`lloccode_checksum`,[lloccode.h#L41-L51](../../pic_server/include/lloccode.h#L41))。

反解 `analyse_lloccode_ex`([lloccode.h#L117-L144](../../pic_server/include/lloccode.h#L117))用一条 `sscanf` 拆回所有字段,拼出 `/opt/taomee/file_root/{type}/{date}/{dir}/{pic}{.JPEG}`。浏览端 FCGI 则**先验校验和再读盘**,校验和不符直接 404——这让 URL 不能被随意构造去遍历他人图片。

### 机制二:异步上传的"队列 + 1 秒扫描"收尾状态机

FileServ 不在请求线程里同步等落盘和缩略图,而是把每个上传封成 `work_data_t` 挂进全局 `gqlist`,用 4 个 tag 跟踪进度:`save_file_tag`(原图写完)、`make_thumb_tag`(gd 缩略图做完)、`send_thumb_tag`(已发 ThumbServ)、`done_thumb_tag`(收到 thumb_id)。

1 秒定时器 `scan_all_work_file`([thumb.c#L323-L342](../../pic_server/fileserv/thumb.c#L323))遍历队列,对每个 work 调 `need_remove_or_send`([thumb.c#L300-L321](../../pic_server/fileserv/thumb.c#L300)):
- `save_file_tag && done_thumb_tag` 都齐 → 收工、删日志、回 CGI、释放([delete_work L285-L298](../../pic_server/fileserv/thumb.c#L285))。
- 缩略图做好但还没发 → 发给 ThumbServ。
- **超过 2 分钟还没齐** → 判定失败,`unlink` 已落的原图、回 `err_invalid_file` 给 CGI([thumb.c#L311-L320](../../pic_server/fileserv/thumb.c#L311))。

这套设计让单个 FileServ 进程能在不阻塞网络线程的前提下吞高并发上传,代价是逻辑分散在回调里、且依赖那个 2 分钟兜底超时清理孤儿文件。

### 机制三:thumb_id 自编码偏移 —— 无主键的缩略图寻址

见 ThumbServ 章节:`thumb_id = (len<<15) | offset`。好处是**省掉一张 id→位置 的映射表**(海量缩略图的映射表本身就是负担),坏处是块文件一旦升档(`ifileno` 变),老 thumb_id 里隐含的偏移就失效——所以迁移/升档必须由 AdminServ 重新读写而非简单拷文件。

---

## 端到端调用流举例:摩尔庄园玩家上传一张相册照片

追一条真实链路(mole 相册玩法,gameid=mole):

1. **建相册**:Flash 先请求业务后端建相册,业务后端向 **KeyServ** 发 `proto_create_album`,KeyServ `++index` 返回全局 `albumid`([service.cpp#L109](../../pic_server/key_serv/service.cpp#L109)),并把该 userid 当日计数 +1(超 100 拒)。
2. **签票**:业务后端用 DES 密钥 `"!tA:mEv,"` 把 `{userid, channel, albumid, 调度机ip, 当前时间, 限宽, 限高}` 加密成 `session_key`。
3. **上传**:Flash POST multipart 到 `pic_cgi_v2`,带 `file` 与 `session_key`。CGI `cgiMain` 解密、验 ip==bind_ip、验未超 5 分钟、用 gd 验真图([cgi_upload.c#L230-L303](../../pic_server/pic_cgi_v2/trunk/src/cgi_upload.c#L230))。
4. **落盘**:CGI 打 `proto_fs_cgi_upload_file`(cmd=1) TCP 到 **FileServ:6999**。FileServ `upload_file_cmd` 防重→分配路径与半成品 lloccode→`pool1` 把原图写到 `/opt/taomee/file_root/1/20110815/001/001.JPEG`,`pool2` 用 gd 按裁剪参数生成缩略图([save_file.c#L406-L425](../../pic_server/fileserv/save_file.c#L406))。
5. **存缩略图**:FileServ 把缩略图字节经 `send_to_thumbserv` 按 `albumid` 路由到 **ThumbServ:7000**(albumid<1亿走 1 号机,见 [thumb_serv_list.xml](../../pic_server/fileserv/thumb_serv_list.xml))。ThumbServ `save_photo` 写进 `photofs` 块文件,回 `proto_ts_upload_thumb` 带 4 个 thumb_id([thumb.c#L687-L724](../../pic_server/thumbserv/thumb.c#L687))。
6. **拼码回包**:FileServ `upload_thumb_callback` 把 thumb_id 续写进 lloccode、算校验和、`send_result_to_cgi`([fileserv/thumb.c#L386-L388](../../pic_server/fileserv/thumb.c#L386))。
7. **回 JSON**:CGI 输出 `{"result":"success","albumid":..,"photoid":..,"lloc":"<64字节>","hostid":..}`。Flash 拿 lloccode 入业务库(二代则由 webproxy `bus.album.addPhoto` 代写)。
8. **后续浏览**:页面拼出 `http://pic...61.com/...?session_key=<DES(get_image,lloccode)>`,**FCGI_PIC** 验校验和后直接 `fopen` 原图返回([fcgi_get_image.c#L156-L185](../../pic_server/FCGI_PIC/trunk/src/fcgi_get_image.c#L156));缩略图则经 FCGI→ThumbServ 取字节。

---

## 复活实操提示(对 moleworld 复活)

定位仍是**整体参照、零件直用**,但深挖后可给出更精确的取舍:

**能直接拿走的纯函数零件**:
- [include/lloccode.h](../../pic_server/include/lloccode.h) 的 `create_lloccode`/`analyse_lloccode_ex`/`lloccode_checksum` 是无依赖纯函数,可原样抄进离线服当"图片定位码"工具;只要复活端用同一格式,老 URL 里的 lloccode 就能继续解析定位。
- [include/proto.h](../../pic_server/include/proto.h) 的 `PKG_*`/`UNPKG_*` 宏同样可直接复用。
- DES 密钥 `"!tA:mEv,"`(三处:[fcgi_proto.h#L10](../../pic_server/FCGI_PIC/trunk/src/fcgi_proto.h#L10)、[pic_cgi_v2 cgi_proto.h#L14](../../pic_server/pic_cgi_v2/trunk/src/cgi_proto.h#L14))——若要解读抓包里的真实老 session_key,这是钥匙;但**切勿照搬上线**。

**必须自己补 / 不建议照搬**:
- FileServ/ThumbServ/AdminServ 都依赖外部 `iter_serv` 宿主(本仓只有 `.so` 侧无 main),无法独立 `make` 跑起来;ThumbServ 的变长块存储引擎(`save_photo`)虽精巧,但对单机离线复活是过度设计——直接"一张图存一个文件 + 一张 `lloccode→本地路径` 映射表"即可平替全部存储逻辑。
- 鉴权链(ip==bind_ip + 5 分钟时效 + DES 票)是为多机调度设计,离线单机可大幅简化或去掉。
- 集群路由表 [file_serv_list.xml](../../pic_server/fileserv/file_serv_list.xml) / [thumb_serv_list.xml](../../pic_server/fileserv/thumb_serv_list.xml) 里的真实内外网 IP(`114.80.173.91`/`192.168.9.91`)、硬编码根目录 `/opt/taomee/file_root`、`/opt/taomee/photofs/data` 全部要改。

**结论**:对"登录进游戏看世界"最小目标,pic_server 非必需路径,相册/头像可后置或占位实现;真要复活上传链,照"CGI 验真→落盘分配 lloccode→生成缩略图→回 lloccode"的**设计**走即可,实现用现代静态文件服务平替这套多进程+mmap+gd+iter_serv 的重型架构。
