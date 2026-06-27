# 内容与数据专题：游戏内容数据在哪、什么格式、能挖出什么

本文档盘点淘米泄漏服务端源码中的「游戏内容数据」——即不属于程序逻辑、而是描述游戏世界本身（物品、地图、答题、活动、养成数值等）的配置数据。重点服务于 **摩尔庄园离线复活**（moleworld-server / MoleWorld-offline）：哪些数据可以直接拿来喂给自建服务端，哪些只能参照，哪些根本不在仓库里。

相关文档：[模块总览 mole.md](./modules/mole.md) · [协议专题 protocol.md](./protocol.md) · [总入口 README](../README.md)

> 约定（据淘米自有 gameid 字典）：mole / mole2 = 摩尔庄园（mole 为 2007–2011 经典版，mole2 为后期改版/摩尔大冒险），pea = 赛尔号，monster = 淘米小怪兽（**非赛尔号**），gf = 功夫派（**非小花仙**），boke = 卜克/POP。本项目目标只针对 **mole（经典摩尔庄园）**。

---

## 0. 一句话结论

摩尔庄园的「世界设定数据」几乎全部以 **XML 配置文件**形式存在于 [mole/online/conf/](../mole/online/conf)（约 152 个文件），其中 [items.xml](../mole/online/conf/items.xml)（1.26 MB，9958 件物品）是绝对核心，配合 maps / swap / exchange / 答题 / 拉姆养成 / 挖宝活动等文件，**足以重建绝大部分游戏内容数值**。
而仓库里 42 个 `.sql` 文件**几乎不含游戏世界种子数据**——它们要么是建表 DDL，要么是监控系统（itl）的表，**真正的「世界数据」是 XML，不是 DB**；DB 在运行时由玩家行为填充，泄漏里没有玩家数据。

---

## 1. 内容数据主目录：`mole/online/conf/`（约 152 个文件）

这是 mole online 进程（游戏主逻辑服务）启动时加载的配置目录。所有面向玩家的游戏内容数值都在这里。下面逐个解剖关键文件。

---

### 1.1 `items.xml` —— 物品总表（核心中的核心，1.26 MB / 9958 件物品 / 40 个大类）

[items.xml](../mole/online/conf/items.xml) 定义了游戏里**一切「物品」**：服饰、道具、家园种子、宠物（拉姆）、卡牌、天使、地下城材料、海洋生物……以及抽象的「属性奖励」（摩尔豆、经验值等）。摩尔庄园的经济、商城、背包、奖励发放全部以此表的 `ID` 为唯一引用。

**根 / 节点结构**

```
<Items>
  <Kind ID Name Start End Max Kind [path] [Type] SpottedBean>   ← 物品大类（共 40 个 Kind）
     <Item ID Name Grade Price Layer Tradability ... />          ← 单个物品
     ...
  </Kind>
  ...
</Items>
```

**`<Kind>`（物品大类）字段表**

| 字段 | 含义 |
|---|---|
| `ID` | 大类序号（1..40） |
| `Name` | 大类中文名（如「服饰」「家园种子」） |
| `Start` / `End` | 该类物品 ID 的**取值区间**（如服饰 12001–15999） |
| `Max` | 该类物品在背包中默认的最大叠加数 |
| `Kind` | 位标志（power-of-two，如 1/2/4/8…，作位掩码用于「物品属于哪些类」判定） |
| `path` | 美术资源相对路径前缀（如 `resource/pet/icon/`），**指向客户端素材**（仓库不含） |
| `Type` | 子类型标记（部分类有，如家园相关 `Type="2"`） |
| `SpottedBean` | 是否「斑点豆」相关标记（普遍为 0） |

**`<Item>`（单个物品）常见字段表**（按出现频次，全表 9958 件统计）

| 字段 | 出现数 | 含义 |
|---|---|---|
| `ID` | 9958 | 物品唯一 ID（全局唯一，跨表引用就用它） |
| `Name` | 9958 | 物品中文名 |
| `Price` | 9958 | 售价（结合 `Mibi` 判断货币：摩尔豆 or 米币） |
| `Grade` | 9683 | 物品等级 / 品级 |
| `Tradability` | 9157 | 可交易性：`1`/`2`/`3` 等枚举（1=不可交易、2=可丢弃/特殊、3=可交易，具体语义待证，由 online 逻辑解释） |
| `Type` / `ItemType` | 5299 / 953 | 子类型标识（同字段在不同 Kind 下语义不同） |
| `Max` | 4438 | 单物品自身的最大叠加数（覆盖 Kind 默认） |
| `Layer` | 3964 | **服饰穿着层级 / Z 序**（5=鞋、15=衣、25=项链、35=眼镜、45=帽/发、55=手持……决定服饰叠穿渲染顺序） |
| `VipOnly` / `VipBuyable` / `VipLevel` | 1945 / 1321 / 256 | VIP（超级摩尔）专属 / VIP 可购买 / 所需 VIP 等级 |
| `Discount` | 1801 | 折扣系数（如 0.5 = 半价，常配活动） |
| `Mibi` | 137 | 货币类型标记：`Mibi="0"` 通常表示用**摩尔豆**而非米币购买（米币 = 充值货币） |
| `SellPrice` / `Shellprice` | 882 / 45 | 回收 / 卖出价 |
| `FromSeed` / `IsIntensive` | 1248 / 1248 | 来自种子 / 是否可强化 |
| `Food` / `food` / `Eat` / `EatType` | 投喂相关（宠物/牧场） | 该物品作为食物的属性、可被哪些宠物食用 |
| `Hungry`/`Thirsty`/`Sanitary`/`Spirit` | 各 100 | 喂养道具对宠物饥饿/口渴/清洁/精神值的影响 |
| `LevelNum`/`Fruit`/`PlantName`/`canGeneration` | 家园种子 | 各生长阶段所需时间、成熟产出物 ID、作物名、能否留种 |
| 其余 | — | 牧场/猪仔/天使战斗等子系统大量专有字段（`PorkUp`/`BeautyUp`/`MateCost`/`CardAttribute` 等），各由对应玩法逻辑解释 |

**真实样例（服饰类 Kind ID=1，[items.xml#L3](../mole/online/conf/items.xml#L3) 起）**

```xml
<Items>
  <Kind ID="1" Name="服饰" Start="12001" End="15999" Max="1" Kind="1" SpottedBean="0">
    <Item ID="12001" Name="斗士战袍" Grade="0" Price="800" Layer="15" Tradability="3"/>
    <Item ID="12007" Name="凯特发型" Grade="0" Price="300" Layer="45" Tradability="3" Mibi="0"/>
    <Item ID="12009" Name="急速蓝靴" Grade="0" Price="800" Layer="5" Tradability="1" Discount="0.5"/>
    <Item ID="12032" Name="黑色国王帽" Grade="0" Price="1000" Layer="45" Tradability="3" VipBuyable="1"/>
```

**特殊大类样例（说明跨表引用与「能挖什么」）**

```xml
<!-- 属性奖励：抽象「物品」，ID 0/1/2/3/4 是摩尔豆/经验/力量/智慧/魅力 —— 奖励发放就发这些 ID -->
<Kind ID="10" Name="属性奖励" Start="0" End="100" Max="0" ...>
  <Item ID="0" Name="摩尔豆"   Price="1" Max="4294967295"/>
  <Item ID="1" Name="经验值"   Price="1" Max="4294967295"/>
  <Item ID="4" Name="魅力值"   Price="1" Max="4294967295"/>

<!-- 家园种子：含完整种植数值，可直接重建农场 -->
<Kind ID="14" Name="家园种子" Start="1230001" End="1239999" path="resource/home/seed" Type="2" ...>
  <Item ID="1230004" Name="苹果种子" buyLevel="2" Price="60" Tradability="2" Class="AppleSeed"
        LevelNum="120,420,900,1800" Fruit="190141" PlantName="苹果" canGeneration="1"/>

<!-- 喂养道具：完整投喂数值 + 适用宠物 -->
<Kind ID="7" Name="喂养道具" Start="180001" End="189999" path="resource/pet/icon/" ...>
  <Item ID="180001" Name="巧克力豆" Price="10" Hungry="20" Thirsty="0" Eat="2,3,4,5,101,102" EatType="food"/>
```

**40 个大类清单（Kind ID → Name → ID 区间）**：服饰(12001–15999)、变色道具(16001–)、变身道具(17001–)、投掷道具(150001–)、小屋道具(160001–)、宠物种子(170001–)、喂养道具(180001–)、收集品(190001–199999)、邮件卡片(1000001–1100000)、属性奖励(0–100)、拉姆装扮、拉姆荣誉标志、家园装扮、家园种子、卡牌、宝盒物品、班级物品、牧场动物、牧场装扮、吉吉乐卡牌、交通工具、天使战斗道具、拉姆证书、摩尔助手、一条街道具、餐厅、龙坐骑、临时物品、新规则脚印、天使、地下城、天使战斗卡牌/天使/装扮/收集品、猪种/猪仔装扮/猪仔道具、海洋生物/海洋道具。
→ 该列表本身就是一份**摩尔庄园全玩法系统目录**：从「服饰 + 收集品」（最初核心）一直到「天使战斗 / 地下城 / 牧场 / 餐厅 / 龙坐骑」（后期扩展），可据此判断各玩法的内容规模。

**能挖出什么**
- **全套商城 / 经济数据**：每件物品的售价、货币（摩尔豆 vs 米币）、折扣、VIP 限制、是否可交易——足以重建商城与背包经济。
- **服饰渲染信息**：`Layer` 给出叠穿层级，配合 `path` 指向的客户端美术即可还原换装。
- **各子系统数值**：家园种植（生长时间/产出/留种）、宠物投喂（饱食/清洁等增减）、牧场/猪仔/天使战斗的成长公式参数。
- **ID 命名空间**：`Start/End` 区间是理解全仓所有「Item ID」引用的钥匙（maps、swap、exchange、奖励等全部引用这些 ID）。

---

### 1.2 `maps.xml` —— 地图 / 场景表（103 KB，244 个 Map 节点）

[maps.xml](../mole/online/conf/maps.xml) 定义游戏世界的所有场景：场景间的传送入口、每张图上的随机刷出物（采集物）、内嵌小游戏点位。

**结构与字段**（文件头部有作者自带注释，[maps.xml#L1](../mole/online/conf/maps.xml#L1) 起）

```
<Maps FirstMap TopMax>
  <Map ID Name Type PartyPeriod>
    <Entries> <Entry MapID PosX PosY/> ... </Entries>   ← 通往其它场景的传送门坐标
    <RandomItems> <Item ID MaxPoints StartPoint Max RebornTime Multiplicator AppearAnyTime/> </RandomItems>  ← 场景随机刷出采集物
    <Group GameID Type Action LG_PosX LG_PosY> <Item ID Direction Flag/> </Group>  ← 内嵌玩法/小游戏点
  </Map>
</Maps>
```

| 字段 | 含义 |
|---|---|
| `Map@ID` / `@Name` | 场景 ID（如 1=摩尔城堡、2=爱心教堂）/ 中文名 |
| `Map@Type` | 场景类型（注释：2=带游戏的图，4=场景游戏图） |
| `Entry@MapID/PosX/PosY` | 从本图走到目标图 `MapID`，落点坐标 |
| `RandomItems` 各字段 | `MaxPoints`=可刷点数上限、`Max`=同时存在上限、`RebornTime`=再生间隔(秒)、`Multiplicator`=派对加倍、`AppearAnyTime`=随时可刷 |

**真实样例**

```xml
<Map ID="1" Name="摩尔城堡" Type="1" PartyPeriod="1" >
    <Entries >
        <Entry MapID="2" PosX="130" PosY="340" />
        <Entry MapID="3" PosX="850" PosY="450" />
    </Entries>
    <Group GameID="47" Type="0" Action="0" LG_PosX="450" LG_PosY="450">
        <Item ID="1" Direction="0" Flag="0" />
    </Group>
</Map>
<Map ID="2" Name="爱心教堂" ... >
    <RandomItems>
        <Item ID="190026" MaxPoints="1" StartPoint="0" Max="1" RebornTime="30" Multiplicator="1" AppearAnyTime="1" />
    </RandomItems>
```

**能挖出什么**：整张摩尔庄园**世界地图拓扑**（哪张图连哪张图、传送落点）、每张场景的**采集物刷新规则**（采什么、刷多少、多久再生）、场景内小游戏点位。配合 items.xml 即可还原「逛地图捡道具」的核心循环。

---

### 1.3 `swap.xml` / `exchange.xml` / `gifts.xml` / `npc_tasks.xml` —— 交换 / 兑换 / 送礼经济

这一组都是 **「拿 N 个 A 物品换 M 个 B 物品」** 的配方表，是非充值经济的主干。

| 文件 | 大小 | Entry 数 | 用途 |
|---|---|---|---|
| [swap.xml](../mole/online/conf/swap.xml) | 275 KB | 1354 | 各 NPC 小摊 / 兑换点的物物交换配方（含概率 `Rand`、次数/数量限制 `LimitType`、每日/每周/每月上限） |
| [exchange.xml](../mole/online/conf/exchange.xml) | 91 KB | 380 | 任务式兑换（如「成为导游」收 4 个特定收集品 → 发服饰套装 + 摩尔豆） |
| [gifts.xml](../mole/online/conf/gifts.xml) | 8 KB | — | 送礼合成（投入若干物品 → 产出礼物物品） |
| [npc_tasks.xml](../mole/online/conf/npc_tasks.xml) | 8.5 KB | — | NPC 任务奖励（按 NPC、亲密度、每日/每周/每月限制发放物品） |

**统一结构**（swap/exchange/gifts 同构）

```xml
<Entry ID="..." [NOT_LIMIT] [QUIT_TASK] [LimitType] [Rand]>
    <ItemIn>  <Item ID="190727" Count="20" /> ... </ItemIn>   ← 消耗
    <ItemOut> <Item ID="1220055" Count="1" /> ... </ItemOut>  ← 产出
</Entry>
```

**真实样例（swap.xml，毛毛头酋长小摊）**

```xml
<Entry ID="1">
    <ItemIn>  <Item ID="190727" Count="20" /> </ItemIn>
    <ItemOut> <Item ID="1220055" Count="1" /> </ItemOut>
</Entry>
```

swap.xml 头部自带大量维护注释（哪些 `DayType`/`FlagIdx` 已被占用、台湾服占用了哪些 ID），对理解 ID 命名空间和跨服差异很有价值。

**能挖出什么**：完整的**非充值产出链路**——玩家通过采集/任务拿到收集品（190xxx），再经 swap/exchange/gifts 换成服饰、家园装扮、特殊道具。离线复活时这套配方可几乎原样保留游戏的「肝度」与产出节奏。

---

### 1.4 答题系统：`questions.xml` / `class_questions.xml` / `tv_question*.xml`

摩尔庄园有多处答题玩法（百科答题、班级答题、电视台有奖答题）。

| 文件 | 题数 | 用途 |
|---|---|---|
| [questions.xml](../mole/online/conf/questions.xml) | 100 | 通用知识答题题库（含难度 `Difficulty`） |
| [class_questions.xml](../mole/online/conf/class_questions.xml) | 402 | 班级 / 课堂答题题库 |
| [tv_question.xml](../mole/online/conf/tv_question.xml) ＋ tv_question1/2.xml | 84(+) | 摩尔电视台**按日期排期**的有奖答题（带奖品 `PrizeID`/`PrizeCnt`） |

**字段表（三者大体同构）**

| 字段 | 含义 |
|---|---|
| `ID` | 题目编号（tv 题要求按日期有序） |
| `Content` | 题干 |
| `OptionA/B/C/D` | 四个选项（空串表示该选项不存在） |
| `Answer` | 正确选项序号（1=A、2=B、3=C、4=D） |
| `Difficulty` | 难度（仅 questions.xml） |
| `Date` / `PrizeID` / `PrizeCnt` | 仅 tv_question：开放日期、奖品物品 ID、奖品数量 |

**真实样例**

```xml
<!-- questions.xml -->
<Question ID="1" Content="什么地方的温度最适合细菌生长?" Difficulty="2"
          OptionA="人体体温 " OptionB="床上" OptionC="冰箱中" OptionD="以上皆是" Answer ="1" />

<!-- tv_question.xml -->
<TV_question ID="1" Date="20110805" PrizeID="1351064" PrizeCnt="5"
    Content="乐乐侠的真正身份是？" OptionA="库拉" OptionB="摩乐乐" OptionC="菩提大伯" OptionD="" Answer="2"/>
```

**能挖出什么**：586+ 道**原版题目与标准答案**（含彩蛋型剧情题，如「乐乐侠的真正身份」），可直接重建答题玩法，附带的奖品配置（tv 题）连发奖逻辑都给齐了。

---

### 1.5 拉姆养成（学院/课堂）：`lahm_*.xml`

拉姆（玩家宠物）养成是摩尔庄园的招牌系统。一组 `lahm_*` 文件给出了完整的课程数值。

| 文件 | 用途 |
|---|---|
| [lahm_lessons.xml](../mole/online/conf/lahm_lessons.xml) | 课程表：每门课消耗课时/精力，加多少亲密度/潜力/德育/智力/体育/艺术/劳动，课程升级经验曲线 |
| [lahm_students.xml](../mole/online/conf/lahm_students.xml) | 学生（拉姆）初始属性表，按年级 `Grade` × 等级 `Level` 排布每个 `PetID` 的潜力/各项资质 |
| [lahm_lesson_questions.xml](../mole/online/conf/lahm_lesson_questions.xml) | 每门课对应的随堂题答案与加成值（最多 30 题/课） |
| lahm_task.xml / lahm_classroom_* / angry_lahm_task_info.xml | 拉姆任务、班级奖励/同窗情、愤怒拉姆活动等 |

**`lahm_lessons.xml` 字段表**（文件头自带注释，[lahm_lessons.xml#L1](../mole/online/conf/lahm_lessons.xml#L1)）

| 字段 | 含义 |
|---|---|
| `Class_hour` | 消耗课时数 |
| `Time` | 上课时长（秒） |
| `Channel` | 解锁渠道 |
| `Condition` | 解锁条件（累计经验门槛等） |
| `Energy` | 精力增减（负=消耗） |
| `Lovely` | 亲密度增减 |
| `Latent` | 潜力增加 |
| `Moral/IQ/Sport/Art/Labor` | 德育/智力/体育/艺术/劳动加成 |
| `Lesson_degree` | 熟练度增量 |
| `Level_exp` | 课程各级所需经验（逗号分隔列表） |
| `Attr` | 属性方向标记（如 `1,4`） |
| `Type` | 1=正常上课，2=活动课 |
| `VipOnly` | 是否超拉专属 |

**真实样例**

```xml
<Exam Num="0">
  <Lesson ID="1" Name="拉姆语言课" Class_hour="1" Time="300" Channel="1" Condition="0"
          Energy="-70" Lovely="20" Latent="100" Moral="0" IQ="0" Sport="0" Art="0" Labor="0"
          Lesson_degree="20" Level_exp="20,30,40,50,70" Attr="1,4" Type="1" VipOnly="0"/>
```

```xml
<!-- lahm_students.xml：年级1 等级1 的拉姆初始资质 -->
<Grade GradeID="1"><Level LevelID="1">
    <Item PetID="1" Latent="40" Moral="3" IQ="14" Sport="7" Art="9" Labor="9" Flag="0"/>
```

**能挖出什么**：拉姆养成的**全套成长公式**——上课的投入产出、属性曲线、随堂题答案、不同年级拉姆的资质分布。这是摩尔庄园数值策划最厚的一块，可几乎 1:1 复刻养成体验。

---

### 1.6 挖宝活动：`DigTreasure*.xml` ＋ 塔防活动：`TDInfo.xml` / `TDTask*.xml` / `TD2011*.xml`

两个大型限时活动，配置非常完整。

**挖宝（DigTreasure）**
- [DigTreasureMap_1.xml](../mole/online/conf/DigTreasureMap_1.xml) / _2 / _3：挖宝地图，定义各区域（矿/Boss/事件/机关 `Type`）的坐标与可行走点。
- [DigTreasureElement.xml](../mole/online/conf/DigTreasureElement.xml)：每个挖点的素材、动作（dig/Knock/brush）、血量/经验/产出，以及**按挖掘次数分段的掉落概率表**（`<Region Count>` → `<Award Rate>` → `<Item>`）。

```xml
<Mine ID="0" MaxCount="10" Hp="2" Exp="3" name="灵光魔方" action="Knock" sound="Cai_Kuang">
  <Region Count="1">
    <Award Rate="20"> <Item ID="1453000" Count="1"/> <Item ID="1453001" Count="1"/> </Award>
    <Award Rate="30"> <Item ID="1453002" Count="1"/> </Award>
```

**塔防 / 天使魔鬼（TD）**
- [TDInfo.xml](../mole/online/conf/TDInfo.xml)：134 个角色（魔鬼/守卫）定义——攻击力 `ATK`、攻速 `ASPD`、攻击范围 `ATR`、血 `life`、经验 `exp`、移速、台词 `talks` 等。
- TDTask1.xml … TDTask87.xml（**87 关**）：每关的波次脚本（`<round>` 内 `<monster id pathID time count>`）、星数、计分、音乐、胜负 swf。
- [TD20111223AwardInfo.xml](../mole/online/conf/TD20111223AwardInfo.xml) / [TDTask*.xml](../mole/online/conf/TDTask1.xml)：活动奖励。

**能挖出什么**：两个完整活动的**关卡/掉落/敌人数值**，连台词和音效文件名都在。挖宝的分段掉落概率表尤其难得——离线复活若要还原活动，几乎无需再设计数值。

---

### 1.7 其它内容文件（同目录，按用途归类）

| 文件 | 用途要点 |
|---|---|
| [levelinfo.xml](../mole/online/conf/levelinfo.xml) | PVE 关卡表（47 关）：金钱/经验奖励、出怪 `Monster`、翻牌奖励、掉落 `Drop` ID 列表、进入条件（前置关卡/等级/能量）。头部注释枚举了关卡类型 `LT_NORMAL/LT_BOSS/...` |
| [games.xml](../mole/online/conf/games.xml) | 小游戏注册表（100 项）：HappyCook/Fishing/Skating/Kart… 每个小游戏的经验/魅力/游戏币上限、奖励物品 `ItemBonus` 与概率 |
| [games_bonus.xml](../mole/online/conf/games_bonus.xml) | 小游戏按分数区间的奖牌与发奖配置 |
| [skill.xml](../mole/online/conf/skill.xml) | 变身/技能解锁表（所需 VIP/玩家等级、消耗、变身道具 ID） |
| [npc.xml](../mole/online/conf/npc.xml) | NPC 行为脚本（进场/行走/动作/说话的指令序列与坐标） |
| [tasks.xml](../mole/online/conf/tasks.xml) | 任务表 |
| [cards.xml](../mole/online/conf/cards.xml) / [new_cards.xml](../mole/online/conf/new_cards.xml) | 卡牌系统 |
| [candy.xml](../mole/online/conf/candy.xml) / [cutepig_*.xml](../mole/online/conf/cutepig_explor_mine.xml) / [animal_*.xml](../mole/online/conf/animal_skill.xml) | 各子玩法（糖果/猪仔挖矿/牧场动物）数值 |
| [angel_compose.xml](../mole/online/conf/angel_compose.xml) / [cp_*.xml](../mole/online/conf/cp_task_info.xml) / [af_*.xml](../mole/online/conf/af_prize.xml) | 天使合成、CP（情侣）系统、天使战斗（af_prize/af_task/af_skills）奖励与任务 |
| [mails.xml](../mole/online/conf/mails.xml) / [mails_tw.xml](../mole/online/conf/mails_tw.xml) | 系统邮件模板（含台服版本） |
| [holidays.xml](../mole/online/conf/holidays.xml) / [events.xml](../mole/online/conf/events.xml) / [weekend_bonus.xml](../mole/online/conf/weekend_bonus.xml) / [month_task.xml](../mole/online/conf/month_task.xml) | 节日/活动/周末/月度奖励排期 |
| [dirtyword.conf](../mole/online/conf/dirtyword.conf) | 敏感词过滤表（非 XML，纯文本） |
| [rand.xml](../mole/online/conf/rand.xml) / [rand_type.xml](../mole/online/conf/rand_type.xml) / [exclu_things.xml](../mole/online/conf/exclu_things.xml) | 随机掉落 / 互斥物品配置 |

---

## 2. 其它游戏的内容数据分布（gf / mole2 / monster·pea）

本项目不需要它们，但盘点一下可佐证「内容数据 = 各游戏自己的 conf/ XML」这一规律，且其中 mole2 是摩尔庄园改版、可作旁证。

| 游戏 | 主要内容数据目录 | 文件数 | 代表文件 |
|---|---|---|---|
| **功夫派 (gf)** | [gf/gongfu_online/conf/](../gf/gongfu_online/conf) | 46 | 技能、装备、关卡… |
| | [gf/battle_svr/conf/](../gf/battle_svr/conf) | 35 | [skills.xml](../gf/battle_svr/conf/skills.xml)（含 `<Effect>` 技能分解步骤、攻击方向 `Dir`、MP 消耗 `BaseMpPer/MpConsumed`，注释极详细） |
| **摩尔庄园改版 (mole2)** | [mole2/mole2_online/conf/](../mole2/mole2_online/conf) | 31 | [clothes.xml](../mole2/mole2_online/conf/clothes.xml)（装备品阶 `EquipColor`、部位 `EquipPart`、职业 `Prof`、品阶加成 `AddValue`——比 mole 的 items.xml 更「装备 RPG」化） |
| **淘米小怪兽 (monster)** | [monster/online/conf/](../monster/online/conf) | 18 | [pet.xml](../monster/online/conf/pet.xml)（怪兽/宠物基础表：id/name/type/三段进化 seed 与配色；养成社区类，**非赛尔号**） |
| **赛尔号 (pea)** | [pea/online/conf/](../pea/online/conf) | 14 | [pet.xml](../pea/online/conf/pet.xml)（精灵战斗数值：hp/magic/agility/技能槽/每星每级成长与经验曲线，非常细；马龙赠送的精灵=赛尔号铁证） |

**规律**：每个游戏的 online（主逻辑）与 battle/cache/home 等子服务各自带 `conf/` 目录，内容数据全是 XML，结构思路一致（一张「总表 + 大类 + 单项」），都引用各自的全局物品/技能 ID。mole2 的 clothes.xml 印证了后期摩尔庄园往「装备品阶/职业」方向演进，可作 mole 经典版的对照参考（但 ID 与玩法不通用，不能直接混用）。

---

## 3. SQL 文件盘点（全仓 42 个 `.sql`）

`find . -name '*.sql'` 得 42 个文件。逐个用 `grep -c` 统计 `CREATE TABLE` / `INSERT` / `CREATE TRIGGER`：

| 路径 | CREATE TABLE | INSERT | CREATE TRIGGER | 性质 |
|---|---:|---:|---:|---|
| [common-tools/.../gf/.../create_db_gf.sql](../common-tools/linux/anticheat/trunk/deploy/gf/db_agent/db/create_db_gf.sql) | 4 | 0 | 0 | 反作弊建表 |
| [common-tools/.../mole/.../create_db_mole.sql](../common-tools/linux/anticheat/trunk/deploy/mole/db_agent/db/create_db_mole.sql) | 4 | 0 | 0 | 反作弊建表（摩尔上报表，**非游戏数据**） |
| [common-tools/.../seer/.../create_db_seer.sql](../common-tools/linux/anticheat/trunk/deploy/seer/db_agent/db/create_db_seer.sql) | 4 | 0 | 0 | 反作弊建表 |
| [itl/script/alarm_info.sql](../itl/script/alarm_info.sql) | 2 | 0 | 0 | itl 监控 |
| [itl/script/alert_alarm.sql](../itl/script/alert_alarm.sql) | 0 | 0 | 0 | itl 监控 |
| [itl/script/alter_t_switch_interface_info.sql](../itl/script/alter_t_switch_interface_info.sql) | 0 | 0 | 0 | itl ALTER |
| [itl/script/db.sql](../itl/script/db.sql) | 2 | 0 | 0 | itl 监控 |
| [itl/script/db_alert.sql](../itl/script/db_alert.sql) | 0 | 2 | 1 | itl 监控 |
| [itl/script/db_init.sql](../itl/script/db_init.sql) | 0 | 7 | 0 | itl 初始化种子 |
| [itl/script/db_itl.sql](../itl/script/db_itl.sql) | 46 | 0 | 0 | itl 监控（最大建表脚本） |
| [itl/script/db_itl_trigger.sql](../itl/script/db_itl_trigger.sql) | 0 | 51 | 25 | itl 触发器+种子 |
| [itl/script/db_itl_trigger_v3.sql](../itl/script/db_itl_trigger_v3.sql) | 0 | 56 | 27 | itl 触发器+种子 |
| [itl/script/db_itl_v2.sql](../itl/script/db_itl_v2.sql) | 1 | 10 | 8 | itl 监控 |
| [itl/script/db_modify.sql](../itl/script/db_modify.sql) | 2 | 4 | 12 | itl 监控 |
| [itl/script/grant.sql](../itl/script/grant.sql) | 0 | 3 | 0 | itl 授权 |
| [itl/script/init.sql](../itl/script/init.sql) | 3 | 16 | 13 | itl 初始化 |
| [itl/script/metric_alarm_interval_reinit.sql](../itl/script/metric_alarm_interval_reinit.sql) | 0 | 0 | 0 | itl |
| [itl/script/mysql_metrics.sql](../itl/script/mysql_metrics.sql) | 0 | 17 | 0 | itl 指标种子 |
| [itl/script/mysql_metrics_1.sql](../itl/script/mysql_metrics_1.sql) | 0 | 9 | 0 | itl 指标种子 |
| [itl/script/reinit.sql](../itl/script/reinit.sql) | 4 | 7 | 14 | itl |
| [itl/script/strategy_triggers.sql](../itl/script/strategy_triggers.sql) | 0 | 2 | 6 | itl 触发器 |
| [itl/script/switch_interface_trigger.sql](../itl/script/switch_interface_trigger.sql) | 0 | 5 | 12 | itl 触发器 |
| [itl/script/t_alarm_log_info.sql](../itl/script/t_alarm_log_info.sql) | 3 | 0 | 0 | itl |
| [itl/script/t_host_info.sql](../itl/script/t_host_info.sql) | 4 | 0 | 0 | itl |
| [itl/script/t_host_to_mysql.sql](../itl/script/t_host_to_mysql.sql) | 0 | 7 | 2 | itl |
| [itl/script/t_mysql_error_log.sql](../itl/script/t_mysql_error_log.sql) | 3 | 3 | 0 | itl |
| [itl/script/t_mysql_info.sql](../itl/script/t_mysql_info.sql) | 1 | 0 | 0 | itl |
| [itl/script/t_network_segment.sql](../itl/script/t_network_segment.sql) | 0 | 1 | 0 | itl |
| [itl/script/t_pic_metrics.sql](../itl/script/t_pic_metrics.sql) | 0 | 2 | 0 | itl |
| [itl/script/t_switch_interface_info.sql](../itl/script/t_switch_interface_info.sql) | 1 | 3 | 9 | itl |
| [mole/dbsvr/libsrc/clientproto/table.sql](../mole/dbsvr/libsrc/clientproto/table.sql) | 0 | 0 | 0 | mole DB（空/片段） |
| [mole/dbsvr/logser/sql/a.sql](../mole/dbsvr/logser/sql/a.sql) | 0 | 1 | 0 | mole 日志库 |
| [mole/dbsvr/logser/sql/table.sql](../mole/dbsvr/logser/sql/table.sql) | 1 | 0 | 0 | mole 日志表（`t_user_opt_*` 按日分表） |
| [mole/dbsvr/payser/src/table.sql](../mole/dbsvr/payser/src/table.sql) | 0 | 0 | 0 | mole 充值（只有一条 select 注释） |
| [mole/dbsvr/payser/tools/deal_month/tmp.sql](../mole/dbsvr/payser/tools/deal_month/tmp.sql) | 0 | 0 | 0 | mole 充值临时脚本 |
| [mole/dbsvr/ser/sql/20100603/table.sql](../mole/dbsvr/ser/sql/20100603/table.sql) | 0 | 0 | 0 | mole DB 变更（片段） |
| [mole/dbsvr/ser/sql/20100610/table.sql](../mole/dbsvr/ser/sql/20100610/table.sql) | 1 | 0 | 0 | mole DB（`t_sysarg_type_buf`） |
| [mole/dbsvr/ser/sql/appeal_ex.sql](../mole/dbsvr/ser/sql/appeal_ex.sql) | 1 | 0 | 0 | mole 申诉表 |
| [mole/dbsvr/ser/sql/get_card_grade.sql](../mole/dbsvr/ser/sql/get_card_grade.sql) | 0 | 0 | 0 | mole 查询片段 |
| [mole/dbsvr/ser/sql/get_grade.sql](../mole/dbsvr/ser/sql/get_grade.sql) | 0 | 0 | 0 | mole 查询片段 |
| [mole/dbsvr/ser/tools/mkregisterserial/table.sql](../mole/dbsvr/ser/tools/mkregisterserial/table.sql) | 0 | 0 | 0 | mole 注册序列号工具 |
| [mole/dbsvr/tools/validationcode/table.sql](../mole/dbsvr/tools/validationcode/table.sql) | 1 | 0 | 0 | mole 验证码表（memory 引擎） |

**汇总统计（42 个 `.sql` 合计）**：`CREATE TABLE ≈ 99`、`INSERT ≈ 232`、`CREATE TRIGGER ≈ 173`。

**关键现实**
1. **绝大多数 SQL 与摩尔庄园游戏数据无关**：itl/script/（共 27 个）是「淘米基础设施监控系统（IT 运维/MySQL 指标/交换机/告警）」的库，common-tools 的 3 个是**反作弊上报库**——这两类占了所有 `INSERT` 的几乎全部，且 INSERT 的是监控指标/告警规则，**不是游戏世界数据**。
2. **mole/dbsvr 下的 `.sql` 只是零碎的 DDL 片段/工具**：真正的摩尔庄园 DB 建表语句**并不在 `.sql` 文件里**，而是以 C `sprintf` 模板**硬编码在 dbsvr 源码**中（如 `CREATE TABLE IF NOT EXISTS t_user_attire_$1 (...)`，`$1` 是分库分表的分片后缀）。grep 整个 mole/dbsvr 能看到上百张表的模板：`t_user_$1`(玩家)、`t_user_attire_$1`(穿戴)、`t_user_pet_$1`(拉姆)、`t_user_farm_$1`(农场)、`t_user_card_$1`(卡牌)、`t_user_dig_treasure_$1`(挖宝)、`t_user_lamu_classroom_*`(拉姆课堂)、`t_user_battle_*`(天使战斗) 等——这是「玩家存档」的表结构。
3. **CREATE 多、游戏侧 INSERT 几乎为零 → 缺世界数据 / 玩家数据**：DB 在泄漏里只有**表结构（且多为运行时由源码动态建表）**，没有任何玩家档案、没有运营配置种子数据。游戏的「内容数据」本来就不放 DB——它在 §1 的 XML 里；DB 只存「玩家运行时状态」，而玩家数据未泄漏（也不应存在）。

---

## 4. 对摩尔庄园复活怎么用

把上面的数据按「可用度」分三档，给 moleworld-server / MoleWorld-offline 落地参考。

### 4.1 可直接喂给自建服务端（直用 / 几乎原样可用）

只要自建服务端复刻对应玩法逻辑，这些 XML 可**原样加载**作为权威数据源：

| 数据 | 文件 | 直接用作 |
|---|---|---|
| 物品 / 经济 / 商城 | [items.xml](../mole/online/conf/items.xml) | 物品总表、定价、货币、交易/VIP 规则、服饰层级 |
| 世界地图 / 采集 | [maps.xml](../mole/online/conf/maps.xml) | 场景拓扑、传送门、采集物刷新 |
| 交换 / 兑换 / 送礼 / NPC 任务 | [swap.xml](../mole/online/conf/swap.xml) · [exchange.xml](../mole/online/conf/exchange.xml) · [gifts.xml](../mole/online/conf/gifts.xml) · [npc_tasks.xml](../mole/online/conf/npc_tasks.xml) | 非充值产出配方 |
| 答题 | [questions.xml](../mole/online/conf/questions.xml) · [class_questions.xml](../mole/online/conf/class_questions.xml) · [tv_question.xml](../mole/online/conf/tv_question.xml) | 题库 + 答案 + 奖品 |
| 拉姆养成 | [lahm_lessons.xml](../mole/online/conf/lahm_lessons.xml) · [lahm_students.xml](../mole/online/conf/lahm_students.xml) · [lahm_lesson_questions.xml](../mole/online/conf/lahm_lesson_questions.xml) | 课程数值/资质/随堂题 |
| PVE 关卡 / 小游戏 | [levelinfo.xml](../mole/online/conf/levelinfo.xml) · [games.xml](../mole/online/conf/games.xml) · [games_bonus.xml](../mole/online/conf/games_bonus.xml) | 关卡掉落、出怪、小游戏奖励 |
| 活动 | [DigTreasureElement.xml](../mole/online/conf/DigTreasureElement.xml) ＋ DigTreasureMap_*.xml · [TDInfo.xml](../mole/online/conf/TDInfo.xml) ＋ TDTask*.xml | 挖宝/塔防的完整关卡与掉落概率 |
| 等级 / NPC / 技能 / 邮件 / 敏感词 | [levelinfo.xml](../mole/online/conf/levelinfo.xml) · [npc.xml](../mole/online/conf/npc.xml) · [skill.xml](../mole/online/conf/skill.xml) · [mails.xml](../mole/online/conf/mails.xml) · [dirtyword.conf](../mole/online/conf/dirtyword.conf) | 同名系统 |

> 落地建议：自建服务端直接以 [mole/online/conf/](../mole/online/conf) 为「权威配置目录」按原 schema 解析，能省掉绝大部分数值策划工作。各字段的精确语义以 online 源码的解析逻辑为准（见 [mole.md](./modules/mole.md)、协议见 [protocol.md](./protocol.md)）。

### 4.2 只能参照（需移植 / 需改造）

- **DB 表结构**：dbsvr 源码里的 `CREATE TABLE ... t_user_*_$1` 模板可作为「玩家存档该存哪些字段」的**蓝图**，但它们是分库分表、用 C 拼接的，自建服务端需重新设计自己的存储（不必照搬分片）。
- **mole2 / gf 的 conf**：[mole2 clothes.xml](../mole2/mole2_online/conf/clothes.xml) 等可作摩尔庄园后期演进的参考，但 ID/玩法与经典 mole 不通用，**不能直接混入**。
- **itl / anticheat 的 SQL**：与游戏无关，复活时**忽略**。

### 4.3 明确缺失（仓库里没有，需另想办法）

1. **DB 实际数据少**：没有任何玩家档案、没有运营配置种子；DB 只有表结构（且多为运行时动态建表）。复活后是「空世界」，玩家数据需从零生成。
2. **无客户端美术 / SWF**：items/maps 里的 `path`、`File`、`losturl`/`winurl`/`musicurl` 指向的是**客户端资源**（图标、SWF、音乐），这些**不在服务端泄漏里**。要能玩，仍需配套原版 Flash 客户端资源（另行获取/归档）。
3. **部分字段语义需反推**：如 `Tradability` 的 1/2/3、各子系统专有字段的精确含义，XML 里没有完整图例，需结合 online 源码确认（本文已标注「待证」处）。

---

## 5. 速查表

| 想要 | 看这里 |
|---|---|
| 所有物品/服饰/道具及定价 | [items.xml](../mole/online/conf/items.xml)（9958 件 / 40 类） |
| 地图与采集 | [maps.xml](../mole/online/conf/maps.xml)（244 场景） |
| 交换/兑换经济 | [swap.xml](../mole/online/conf/swap.xml)(1354) · [exchange.xml](../mole/online/conf/exchange.xml)(380) |
| 答题题库 | [questions](../mole/online/conf/questions.xml)/[class](../mole/online/conf/class_questions.xml)/[tv](../mole/online/conf/tv_question.xml)（586+ 题） |
| 拉姆养成 | [lahm_lessons](../mole/online/conf/lahm_lessons.xml) · [lahm_students](../mole/online/conf/lahm_students.xml) |
| 活动 | [挖宝](../mole/online/conf/DigTreasureElement.xml) · [塔防](../mole/online/conf/TDInfo.xml)(87 关) |
| 关卡/小游戏 | [levelinfo](../mole/online/conf/levelinfo.xml) · [games](../mole/online/conf/games.xml) |
| 玩家存档表结构 | dbsvr 源码里的 `t_user_*_$1` 模板（非 `.sql`）— 见 [mole.md](./modules/mole.md) |

返回：[模块总览](./modules/mole.md) · [协议专题](./protocol.md) · [README](../README.md)
