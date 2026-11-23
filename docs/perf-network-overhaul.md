# 暮色森林 1.21.1 性能与网络大修方案

- 基线分支：`1.21.1` @ `989245206`
- 起因：用户反馈 `twilightforest:travellers_wings_state` 每秒发包约 162 个，导致服务器卡顿。
- 审计方式：静态阅读代码，未做运行时测量。本文所有频率和字节数都是估算，必须用阶段 0 的埋点来验证。

标记约定：
- **✔** 已人工复核代码，确认问题存在。
- **⚠** 依赖对原版行为的记忆，或仅为推断，需要实测确认。

---

## 一、根因总览

这批问题可以归成六类，修复时按类处理比逐条打补丁更有效。

| # | 模式 | 典型案例 |
|---|------|----------|
| A | **全实体 tick 监听不做前置过滤**：对每个 `LivingEntity` 都执行装备逻辑，只有走到最后才发现它根本没穿装备 | 羽翼状态包、`getData` 附件膨胀 |
| B | **纯外观状态由服务端同步**：客户端本来就能自己推导，服务端却在抖动的阈值上反复发包 | 羽翼状态、锁链哥布林的链角度、恶魂攻击计时器、九头蛇嘴部开合 |
| C | **每 tick 重新施加短效果或重写状态**：每次都触发原版的效果包、属性包和成就检查 | 旅者背心、羽翼、潜行、怀表 |
| D | **一次操作发 N 个包**：粒子逐个打包、循环内重复广播、全量重发 | 时间核心、巫妖的魔法轨迹、多部件实体、结构保护框 |
| E | **C2S 包缺少校验和限流**：信任客户端传来的 UUID，没有冷却 | GradualGlide、GogglesZoom、Uncrafting、SwapHotbar |
| F | **服务端大范围扫描或同步加载区块** | Redcap 找 TNT、魔法地图、分类核心、时间核心、巫妖蜡烛 |

另外有一个**全局放大器**：所有暮色森林生物的追踪距离都设成了 `80`，但这个单位是**区块**，而不是方块（见 P0-6）。结果是上面每一类实体包的接收人数都被放大了 2 到 16 倍。

---

## 二、问题清单（按优先级）

### P0：持续性、随世界规模线性增长，必须优先修

#### P0-1 羽翼状态包对所有生物广播 ✔
- **位置**：
  - `events/TravellersGearEvents.java:231` 的 `updateOtherModifiers`，挂在 `EntityTickEvent.Post` 上，对**所有** LivingEntity 都会执行。
  - 由它调用 `item/travellers_gear/TravellersGearLogic.java:362` 的 `determineWingState`。
- **问题**：
  - 没有判断实体是否穿着 `TRAVELLERS_WINGS`。
  - IDLE、WALK、FALL_SLOW 等状态阈值会来回抖动（`walkAnimation.speed() > 0.1`、`fallDistance < 2.3`），每次变化都通过 `sendToPlayersTrackingEntityAndSelf` 广播。
  - 状态只给 `TravellersWingsModel` 渲染用。
  - 包体用了 3 个 `writeInt`。
- **修复**：
  1. 立即止血：在 `determineWingState` 开头加上 `if (!entity.getItemBySlot(LEGS).is(TRAVELLERS_WINGS)) return;`。
  2. 根治：IDLE、WALK、SPRINT、FALL、SWIM、RIDE 这几个状态改由**客户端在渲染前推导**。服务端只在二段跳和侧步这两个离散事件时发包。包改成 `VarInt entityId` 加上 `byte flags`。
  3. 卸下翅膀时调用 `removeData(TRAVELLERS_WINGS)`。
- **预期**：这个包从每秒百级降到"玩家做动作时才发"。

#### P0-2 旅者装备 tick 逻辑对所有生物执行，并创建附件 ✔
- **位置**：同样是 `updateOtherModifiers`，它会调用约 8 个 `TravellersGearLogic` 方法。
- **问题**：
  - 每只生物每 tick 都要做多次 `isModifierActive` 检查，每次都会查 `ConcurrentHashMap`、分配 `Optional`、读取组件。
  - 这些方法会对每只生物调用 `getData()`，涉及 `SLIMY_SOLES_BOUNCE_INFO`、`LAST_DAMAGE_ARMOR_TIME`、`TRAVELLERS_WINGS`。其中前两个是**可序列化**的附件，所以每只被保存的生物 NBT 里都多出这两项。
- **修复**：
  - 方法入口统一前置判断：实体是玩家，或者任一装备槽带有 `IS_TRAVELLERS_GEAR`。更好的做法是在 `LivingEquipmentChangeEvent` 里维护一个"穿戴了暮色装备"的缓存标记。
  - 所有读取附件的地方改用 `getExistingDataOrNull`。
  - 可序列化附件加上 `serialize(codec, shouldSerialize)` 谓词，只在值非默认时才写盘。
  - `cancelSlimySolesJump`（约第 170 行）也要做同样处理。

#### P0-3 每 tick 施加 2 到 5 tick 的药水效果 ✔（触发原版发包的机制 ⚠）
- **位置**：
  - `TravellersGearLogic.java:177-189`：`travellersVestHaste` 施加急迫，`travellersWingsHighJump` 施加跳跃提升。
  - `TravellersGearEvents.java:209`：潜行时施加隐身。
  - `item/PocketWatchItem.java:29-34`：速度、跳跃、急迫，持续 5 tick，在 `inventoryTick` 里施加。
- **问题**：
  - 每次 `addEffect` 都会延长持续时间，于是触发 `onEffectUpdated(forced=true)`，进而：
    - 给本人发 `ClientboundUpdateMobEffectPacket`。
    - 触发一次 `EFFECTS_CHANGED` 成就检查。
    - 移除并重新添加属性修饰符，导致向本人**和所有追踪者**发送 `ClientboundUpdateAttributesPacket`。
  - 估算：穿背心加羽翼的玩家，每秒约 40 个效果包，再给每个追踪者每秒约 20 个属性包。怀表更多。
- **修复**：
  - 首选：改用 `addOrUpdateTransientModifier` 直接加属性修饰符。`travellersBootsStraightAhead` 已经是这么做的。
  - 不想改机制的话，用信标式续期：持续时间设为 220 tick，剩余不足 20 tick 时才续上。

#### P0-4 GradualGlide 客户端与服务端互相弹包，且信任客户端传来的 UUID ✔
- **位置**：
  - `client/event/TravellersClientEvents.java:174`：每个渲染帧计算一次状态，变化就发包。
  - `network/GradualGlidePacket.java:25`：服务端处理器。
  - `TravellersGearEvents.java:191`：服务端 tick 里的重置逻辑。
- **问题**：
  - 客户端不检查玩家是否拥有滑翔修饰符。没有该修饰符的玩家一边下落一边按潜行时，客户端发 `true`。
  - 服务端下一个 tick 发现修饰符未激活，就重置状态并广播 `false`。客户端收到后，下一帧又发 `true`。如此循环，每秒约 20 轮，每轮是 1 个 C2S 包加上 1+2×追踪者 个 S2C 包。
  - **安全问题**：服务端用 `getPlayerByUUID(packet.playerUUID)` 查找玩家，所以任何客户端都可以改写别人的滑翔状态。
- **修复**：
  - 客户端：只在修饰符激活时发包。
  - 服务端：改用 `ctx.player()`，并校验修饰符是否激活。
  - S2C 包：用 `VarInt entityId` 代替 UUID。
- `GogglesZoomPacket.java:28` 有同样的 UUID 信任问题，一起修。

#### P0-5 多部件实体几乎每 tick 全量重发所有部件 ⚠
- **位置**：`util/multiparts/MultipartEntityUtil.java:28`，由 tf-asm 的 `SendDirtyEntityDataTransformer` 注入。
- **问题**：
  - 每次 `sendDirtyEntityData` 都会把**所有** `TFPart` 的完整状态发一遍，每个部件大约 46 字节。
  - 九头蛇约有 46 个部件，每个包约 2.1 KB，每个追踪者每秒约 15 到 40 KB。娜迦、冰雪女王、锁链哥布林同样受影响。
  - 其它模组的多部件实体也会收到空包。
  - `HydraHead.setMouthOpen` 每 tick 写入两个 float，其中 `MOUTH_POSITION_LAST` 是冗余字段，加剧了这个问题。
- **修复**：
  - 只发送变化了的部件：比较位置、旋转、尺寸，以及 `packDirty()` 的结果。
  - 什么都没变就不发包。
  - 坐标改用相对父实体的 float 偏移，id 用 VarInt。
  - 新追踪者首次接收时用 `packAll`。这同时修复一个现存 bug：中途开始追踪的玩家收不到那些没被标脏的值。

#### P0-6 实体追踪距离单位写错 ✔（"放大了多少倍"⚠）
- **位置**：`init/TFEntities.java:119,128` 的 `makeBuilder(..., 80, 3)`，投射物用的是 150。
- **问题**：
  - `setTrackingRange` 的单位是**区块**，80 区块会被截断到服务器的视距上限。也就是说，所有暮色生物都按满视距追踪，而原版怪物只有 8 区块。
  - 同一个文件里的 `RISING_ZOMBIE` 用的是 `clientTrackingRange(16)`，可以作为对照。
- **修复**：
  - 普通生物：8。
  - Boss：10 到 16。九头蛇和 Ur-Ghast 体型大，可以取高值。
  - 投射物：4 到 6。
- **注意**：这一项影响玩家可见范围，需要在体型大的 Boss 上实测，确认不会突然消失。

#### P0-7 Redcap 每次评估目标都扫描 17³ 方块 ✔
- **位置**：`entity/ai/goal/RedcapBaseGoal.java:36` 的 `findBlockTNTNearby(8)`，由 `RedcapLightTNTGoal.canUse` 调用。
- **问题**：
  - 每次扫描要调用 4913 次 `getBlockState`，并且每个位置都 `new BlockPos`。
  - 只要开着 mobGriefing，每次评估目标都会扫描。
  - 20 只 Redcap 就是每 tick 约 5 万次方块读取。
- **修复**：
  - `canUse` 先做 `random.nextInt(20)` 门控。
  - 用 `LevelChunkSection.maybeHas(TNT)` 跳过不可能有 TNT 的整个区块段。
  - 改用 `MutableBlockPos`。

#### P0-8 魔法地图每 tick 全图处理，并同步加载区块 ⚠
- **位置**：`item/MagicMapItem.java:105-174`，以及 `util/landmarks/LandmarkUtil.java:67` 的 `isConquered`（`checkReady=false`）。
- **问题**：
  - 没有原版那种"每 tick 只处理每第 16 列"的节流。
  - 每 tick 都会对地标中心像素调用 `isConquered`，而它会同步执行 `getChunk(STRUCTURE_STARTS)`，导致加载或生成未加载的区块。
  - 新地图首次打开时要调用 26 万次 `getBiome`。
  - 静态 `CACHE` 永不淘汰，不区分维度，也不是线程安全的。
- **修复**：
  - 加上 `step & 15` 列节流。
  - `isConquered` 改用 `getChunkNow`，区块未加载就跳过；或者按地标缓存"已征服"状态。
  - 生物群系缓存改成按维度区分的有界 LRU。

#### P0-9 落叶方块：每个生物、每个落叶方块、每 tick 都可能发包 ✔
- **位置**：`block/FallenLeavesBlock.java:148-164`。
- **问题**：
  - 在服务端，所有移动中的生物进入落叶方块时，有 50% 概率给追踪者（含自己）发一个 `SpawnFallenLeafFromPacket`。
  - 暮色森林的生物群系里满地都是落叶方块。
  - 因为包含 "AndSelf"，移动的玩家自己会收到重复的粒子：客户端本地已经生成过一份了。
- **修复**：
  - 改用 `sendToPlayersTrackingEntity`，不再发给自己。
  - 每个实体每 4 tick 最多发一次。
  - 最终方案：客户端根据插值后的速度自行生成粒子，服务端不再发包。

### P1：频繁、包体过大或可被滥用

| ID | 位置 | 问题 | 修复 |
|----|------|------|------|
| P1-1 | `block/TimeLogCoreBlock.java:39-70` | 每秒最多发约 480 个**单粒子**包，发给 64 格内的所有玩家。随机读取方块时不检查是否已加载，可能同步加载区块 | 整轮只用一个 `ParticlePacket`，并设上限；读取前先 `isLoaded` |
| P1-2 | `block/TransLogCoreBlock.java:52,73` | 每秒 `resendBiomesForChunks`，即使什么都没改 ⚠；范围超过 16 时 `getChunk` 会加载区块 | 只在确有改动时才重发；先用 `hasChunk` 守卫 |
| P1-3 | `block/SortLogCoreBlock.java:26,42` | 每秒约 3.6 万次 `getBlockEntity`，配置上限 128 时约 1670 万次；能力缓存放在 Block 单例上，不区分维度，会泄漏 `ServerLevel` | 遍历范围内区块的 `getBlockEntities()`；缓存移到按维度区分的 WeakHashMap |
| P1-4 | `entity/boss/Lich.java:693-711` | `makeMagicTrail` 在遍历玩家的循环里对追踪者广播，产生 N² 份重复包 | 循环外构造一次，只发一次 |
| P1-5 | `network/ParticlePacket.java:32-45` | 每个粒子约 55 字节：类型 id 重复写了一次，坐标用 6 个 double | 去掉冗余 int，坐标改成 float 或相对基准点的偏移，计数用 VarInt；更进一步可以只发"种子加效果 id"，由客户端生成 |
| P1-6 | `network/UncraftingGuiPacket.java:34`, `inventory/UncraftingMenu.java:286` | 滚轮每滚一格发一个包，每个包都要遍历全部配方。恶意客户端可以借此消耗服务端 CPU | 每个玩家每 tick 最多处理 1 个；按输入物品缓存配方列表 |
| P1-7 | `events/ProgressionEvents.java:84-157` | 在受保护区域右键会触发整背包同步，并向 64 格内所有人广播 `AreaProtectionPacket`。持续按住右键时每秒 5 次 | 只发给操作者，按玩家和区域节流；只同步手持槽 |
| P1-8 | `events/ProgressionEvents.java:195-219` | 每 100 tick 无条件重发整份结构保护框列表，大型结构可达几 KB | 缓存每个玩家上次发送的内容，只在变化时发送 |
| P1-9 | `network/SwapHotbarPacket.java:24` | 没有冷却，每个包交换 9 个槽并重写容器组件；`broadcastToPlayer` 这行调用实际不起作用 | 加 5 tick 冷却，删掉无效调用 |
| P1-10 | `entity/monster/BlockChainGoblin.java:154` ✔ | 链角度每 tick 写入 `SynchedEntityData`，每只哥布林每 tick 发 1 个包 | 客户端已经在模拟这个角度，服务端只需每 20 tick 同步一次或在状态变化时同步 |
| P1-11 | `UrGhast.java:283`, `CarminiteGhastguard.java:129` | 战斗中每 tick 写入两个计时器字段 | 只同步攻击状态，计时器在客户端运行；删掉 `PREVTIMER` |
| P1-12 | Firefly、Cicada、Moonworm、Brazier、Trophy、SkullCandle、ChiseledCanopyShelf 的 `getTicker` ✔ | 服务端注册的 ticker 什么都不做；萤火虫在世界生成中数量极多 | 改成 `level.isClientSide ? createTickerHelper(...) : null` |
| P1-13 | `world/components/structures/util/ProgressionStructure.java:56` | 生成提示怪失败时不记录时间，所以每 20 tick 重试 20 次实体构造和射线检测 | 失败也要记录时间；只构造一个实体，在各次尝试间移动它 |
| P1-14 | `entity/monster/Kobold.java:318` | `SeekBreadGoal` 每 tick 重新寻路 | 每 10 到 20 tick 寻路一次，或在目标变化时寻路 |
| P1-15 | `events/CapabilityEvents.java:57` | 每只受伤生物都被创建 `FORTIFICATION_SHIELDS` 附件，从此永久 tick | 改用 `getExistingDataOrNull`；护盾为 0 时 `removeData` |
| P1-16 | `client/renderer/TFWeatherRenderer.java:372`（客户端 FPS） | 每帧调用 961 次 `getBiome` 并做进度判断 | 在玩家移动到新方块或新区块时才重算缓存 |

### P2：次要和收尾

- `BaseTFBoss.java:79` 和 `KnightPhantom.java:120`：有新玩家开始追踪时，`UpdateDeathTimePacket` 发给了**所有**追踪者。应改为只发给这个新玩家。
- `KnightPhantom.java:186`：6 个骑士每 tick 各做一次 129³ 范围的实体查询。改成只由 0 号骑士每 10 tick 做一次。
- `GhastTrapBlockEntity.java:44`：每 tick 一次 21×33×21 范围的实体查询。改成 5 tick 一次。
- `MasonJarBlockEntity.java:120`：每次 `setChanged` 都向整个区块广播完整物品，漏斗操作也会触发。改为只在内容变化时发送，并用原版的 `getUpdatePacket`。
- `ChiseledCanopyShelfBlockEntity.java:87`：update tag 里带着书的完整 NBT。从 tag 里去掉 `Items`。
- `TFMagicMapData.java:122`：`MagicMapPacket` 直接传递服务端的活动 `ArrayList`，有 CME 风险，并且每次都重发。改为 `List.copyOf`，只在变化时发送。
- `Lich.java:745`：`lightNearbyCandles` 在实体移除时扫描约 28.8 万个方块。改为生成时记录蜡烛坐标。
- `MiscEvents.java:81`：先做廉价的槽位和物品判断，再查 Curios。
- `LandmarkUtil.java:37`：每次调用都重新构建结构 Set。按注册表缓存，在 `TagsUpdatedEvent` 时失效。
- `OreMeterItem.java:52`：扫描期间每 tick 修改组件。改成 5 tick 一次。
- Snow Queen 悬停相关目标每 tick 调用 `push()`，会触发 `hasImpulse`。改用 `setDeltaMovement`。
- 所有自定义包统一做精简：
  - 用 VarInt 代替 `writeInt`：TravellersWingsState、UpdateThrown、UpdateDeathTime、LifedrainParticle、CreateMovingCicadaSound 等。
  - 用 float 代替 double：MovePlayer、LifedrainParticle、SpawnFallenLeafFrom。
- 世界生成：
  - `UndergroundPlantFeature.java:71`：把只依赖 x/z 的地标查询提到 Y 循环外面。
  - `FeaturePlacers.java:323`：最近一次提交后，根系会走完整条路径，`FeatureUtil.anyBelowMatch` 里的 Stream 改成普通循环。
- 顺带发现的逻辑 bug（与性能无关）：
  - `Enforcements.java:46`：酸雨的 `%5` 判断嵌在外层 `%20` 里面，实际频率变成了 20 tick 一次。
  - `SortLogCoreBlock.java:93`：`outputsByCount` 以数量为键，数量相同的输出会互相覆盖。
  - `EnforceProgressionStatusPacket` 在玩家登录时不发送。

### 已复核、无需改动
- `PerformDoubleJump`、`PerformSidestep`、`CycleMapSlot`、`WipeOreMeter`：已有校验。
- TFBossBar：原版已做变更检测。
- `SyncUncraftingTableConfig`、`MissingAdvancementToast`：低频。
- 所有包处理器都用了 `enqueueWork`。

---

## 三、分阶段实施

每个阶段单独提交 PR，可以独立回滚。完成标准都以**阶段 0 的埋点数据**为准。

### 阶段 0：测量基线（约 0.5 天）
- 加一个 debug 开关（配置项或 JVM 参数），统计每个 `CustomPacketPayload` 类型每秒的发送次数和字节数，按包类型和接收人数汇总，每 10 秒输出到日志。
- 用 spark 采集一份服务端 profile 作为基线。
- 固定测试场景，保证前后可比：
  1. 暮色森林主群系，附近有 50 只生物，玩家穿全套旅者装备跑动 60 秒。
  2. 九头蛇 Boss 战。
  3. 1 个时间核心加 1 个分类核心，同时有 3 只 Redcap。

### 阶段 0、1 实施记录（分支 `perf/network-overhaul`）

**已完成**，编译已通过，尚未做运行时验证。

阶段 0：发包统计
- 新增 `network/debug/PacketStats`，默认关闭。
- 启动时加上 JVM 参数 `-Dtwilightforest.packetStats=true` 即可开启。统计周期默认 200 tick，可用 `-Dtwilightforest.packetStats.interval=<tick数>` 修改。
- 每个周期在日志里按包类型输出每秒包数和 KB/s。字节数是压缩前的大小。
- 只统计连到专用服务器的远程玩家。单人游戏用的是内存连接，不经过编码器，所以不会被统计。

阶段 1：止血热修
- **羽翼状态**：`determineWingState` 只处理腿部穿着翅膀的实体。非玩家卸下翅膀时移除附件；玩家保留附件，因为里面存着侧步冷却。刚穿上翅膀时强制同步一次状态。**包格式没有改**，VarInt 化推迟到阶段 3。
- **装备 tick**：`updateOtherModifiers` 先检查实体的四个护甲槽，全空就直接返回。服务端此时顺带清理残留的 straight-ahead 速度修饰符。自动修复、粘液鞋底的弹跳、下落和跳跃逻辑都改为不再主动创建附件。
- **GradualGlide 和 GogglesZoom**：
  - 服务端只对发送者本人生效，改用 `ctx.player()`，不再根据客户端传来的 UUID 找人。
  - 两个包都只在状态确实变化时才处理和转发。滑翔包额外要求修饰符处于激活状态。
  - 客户端在没有滑翔修饰符时不再发包，消除了来回弹包的循环。
  - UUID 字段仍保留在包里，以保持新旧版本兼容。
- **Redcap**：`findBlockTNTNearby` 改为按区块段扫描。先用调色板判断这个段里可能不可能有 TNT，不可能就整段跳过；未加载的区块也直接跳过。
- **落叶**：去掉 "AndSelf"。服务端只在偶数 tick 发包，所以远端看到的粒子数约为原来的一半。
- **时间核心**：
  - 每轮所有粒子合并成一个包，每轮最多 128 个粒子。
  - 接收范围从"每个方块周围 64 格"改为"核心周围 64 格加上核心的作用范围"。
  - 读取随机位置前先判断区块是否已加载。
- **巫妖魔法轨迹**：循环里改为 `sendToPlayer`，消除 N² 份重复包。
- **空的服务端 ticker**：Firefly、Cicada、Moonworm、Brazier、Trophy、SkullCandle 改为只在客户端注册 ticker。ChiseledCanopyShelf 只有在服务端且是刷怪笼书架时才注册。
- **护盾附件**：`absorbShieldHits` 改用 `getExistingDataOrNull`，不再给每只受伤的生物都创建这个附件。

**还没做**：按"验收"一节的三个场景，用 `runServer` 加真实客户端实测前后对比。

### 阶段 1：止血热修（约 1 天，可以单独发一个小版本）
只做改动小、风险低、收益最大的部分：
- P0-1：前置装备判断，并改用 VarInt。
- P0-2：前置装备判断，并改用 `getExistingDataOrNull`。
- P0-4：GradualGlide 和 GogglesZoom 改用 `ctx.player()`，补上校验。**这是安全修复。**
- P0-7：Redcap 扫描加门控。
- P0-9：落叶包去掉 "AndSelf"，并加节流。
- P1-1：时间核心粒子合并成一个包。
- P1-4：修掉巫妖轨迹的 N² 重复。
- P1-12：服务端空 ticker 改为返回 null。
- P1-15：护盾附件改用 `getExistingDataOrNull`。
- **验收**：场景 1 中 `travellers_wings_state` 接近 0；当玩家没有滑翔修饰符时，`gradual_glide_packet` 为 0；场景 3 的时间核心包数下降 90% 以上。

### 阶段 2 实施记录

**已完成**，编译已通过，尚未做运行时验证。这一阶段**没有修改任何包格式**，新旧版本仍可混连。

- **羽翼状态（P0-1 根治）**：
  - `determineWingState` 改为只在客户端运行。
  - 下落距离改用插值后的位置变化（`getY() - yo`）自行累计，不再依赖服务端字段。远端实体在客户端上没有 `fallDistance` 和 `deltaMovement` 的准确值。
  - 服务端不再计算这个状态，只在二段跳和侧步时发 `TravellersWingsStatePacket`。
- **每 tick 施加药水效果（P0-3）**：新增 `util/EffectUtil.maintainHiddenEffect`。
  - 背心的急迫、翅膀的跳跃提升、怀表的速度、跳跃和急迫：持续时间改为 40 tick，剩余不超过 20 tick 时才续上。效果包和属性包从每秒 20 次降到约每秒 1 次。
  - 潜行隐身：持续时间 20 tick，剩余不超过 10 tick 时续上。站起时服务端主动移除这个效果，但只移除隐藏、无图标、短时长、看起来是我们加的那一个，不会误删药水或信标效果。
  - **行为变化**：卸下装备或放下怀表后，效果最多还会残留 2 秒。
  - 没有改用属性修饰符，这样其他依赖 `hasEffect` 的模组仍然兼容。
- **多部件实体增量同步（P0-5）**：
  - `TFPart.writeDataIfChanged` 只在部件的位置变化超过 1/4096、旋转变化超过 0.01°、尺寸变化或有脏数据时才写入。
  - `UpdateTFMultipartPacket.delta` 只包含变化了的部件；一个都没变时不发包。
  - 新追踪者通过 `PlayerEvent.StartTracking` 收到一份包含全部非默认数据的完整快照，同时修复了中途开始追踪时缺数据的问题。
  - 客户端本来就会忽略包里不存在的部件，所以包格式不变。
- **追踪距离（P0-6）**：
  - 普通生物：8 区块。
  - 宽度不小于 2 的生物：10 区块，包括九头蛇、Ur-Ghast、Naga、恶魂守卫、巨人矿工。
  - 原来是 80 或 150 的投射物和杂项实体：8 区块。
  - CharmEffect：4 区块。
  - **需要实测**：大型 Boss 的可见距离。
- **魔法地图（P0-8）**：
  - 按原版的方式每 tick 只刷新 1/16 的列。
  - 判断地标是否已征服时，如果区块未加载，就保留地图上已有的状态，不再同步加载区块。
  - 生物群系缓存的 LRU 化尚未做，留到阶段 4。
- **锁链哥布林（P1-10）**：
  - 链角度改为客户端自行模拟，服务端每 20 tick 同步一次 `DATA_CHAINPOS`。
  - 客户端通过 `onSyncedDataUpdated` 收到新值时直接校正。

**推迟到阶段 3**：
- 恶魂计时器（P1-11）：删除 `ATTACK_PREVTIMER` 会改变实体数据的 id，破坏新旧版本兼容，需要和协议版本号的提升一起做。
- 九头蛇嘴部的 `MOUTH_POSITION_LAST`：理由相同。

### 阶段 3（兼容部分）与阶段 4 实施记录

**已完成**，编译已通过，尚未做运行时验证。这一批**都没有修改包格式**。

阶段 3 中不破坏新旧版本兼容的部分：
- 新增 `network/PacketRateLimiter`，按玩家限流：
  - 反合成台操作：每 tick 最多 2 次。
  - 热键栏交换：每 5 tick 1 次。
  - 地图槽切换：每 2 tick 1 次。
  - 同时删掉了 `SwapHotbarPacket` 里那行无效的 `broadcastToPlayer` 调用。
- `AreaProtectionPacket`：只发给触发保护的玩家本人，每个玩家每 10 tick 最多发 1 次。放置被拦截时，只重同步主手和副手两个槽位，不再整背包同步。
- `StructureProtectionPacket`：记录每个玩家上次收到的内容，只在内容变化时发送，包括"全部解除"的情况。
- MasonJar：只在显示的物品或旋转角度变化时发包；只在发光强度变化时才重新计算光照。
- `MagicMapPacket`：发送 `conqueredStructures` 的副本，避免并发修改异常。
- `UpdateDeathTimePacket`：只发给新开始追踪的玩家（BaseTFBoss、KnightPhantom）。
- 修复 bug：`EnforceProgressionStatusPacket` 现在会在玩家登录、重生和切换维度时补发一次。

阶段 4：
- 分类核心：
  - 改为遍历范围内已加载区块的方块实体列表。
  - 去掉了原来的能力缓存。这个缓存是一处**内存泄漏**：它以 `betweenClosed` 返回的可变 `BlockPos` 作键，键在插入后会被改掉，导致缓存只增不减；而且它放在共享的 Block 单例上，不区分维度。
  - 修复数量相同的输出互相覆盖的问题。
- 变化核心：
  - 用 `getChunkNow` 读取区块，不再加载未加载的区块。
  - 只有确实写入了生物群系，才重发生物群系并把区块标记为待保存。
- Knight Phantom：只由 0 号骑士每 10 tick 计算一次 Boss 血条。
- 恶魂陷阱：服务端每 5 tick 扫描一次。
- 进度提示怪：一轮生成全部失败后，下一轮至少间隔 100 tick。
- Kobold：寻找面包时每 10 tick 寻路一次，目标消失时立即重新寻路。
- `LandmarkUtil`：判断结构是否在标签内时直接遍历标签，不再每次调用都构建一个 HashSet。
- 魔法地图的生物群系缓存：
  - 按维度对象用弱引用分开存放，每个维度最多保留 16 个区域（LRU）。
  - 不同世界之间不再错误复用缓存。
  - 加锁保证线程安全。
- 世界生成：
  - `FeatureUtil.isAnyMatchInArea` 改用普通循环。
  - `UndergroundPlantFeature` 的地标查询按区块缓存。

**尚未做**：
- 巫妖蜡烛扫描（P2）。
- 客户端天气渲染缓存（P1-16）。
- 阶段 3 中需要修改包格式的部分：VarInt 和 float 编码、精简 `ParticlePacket`、恶魂计时器、九头蛇嘴部字段。这部分需要升级协议版本号。
- **需要设计决定**：酸雨的伤害判断 `tickCount % 5` 嵌在外层每 20 tick 一次的检查里，实际每秒只造成一次伤害。改正后伤害频率会变成原来的 4 倍，属于平衡性改动，所以没有改。

### 阶段 2：机制重构（约 3 到 4 天）
- P0-1 根治：羽翼外观状态改由客户端推导，服务端只发事件。
- P0-3：效果改为属性修饰符（背心、羽翼、潜行、怀表）。
- P0-5：多部件实体做增量同步，并修复新追踪者首包问题。
- P0-6：调整追踪距离。需要逐个 Boss 做可见性测试。
- P0-8：魔法地图节流，并去掉同步加载区块。
- P1-10、P1-11：哥布林链角度和恶魂计时器改由客户端模拟。
- **验收**：场景 2 中每个追踪者的平均流量下降 70% 以上；场景 1 的 `ClientboundUpdateAttributesPacket` 降到个位数每秒。

### 阶段 3：协议精简与防滥用（约 2 天）
- 重写 `ParticlePacket` 编码（P1-5），并统一把所有包的 int 改成 VarInt、double 改成 float（P2）。
- C2S 限流：Uncrafting、SwapHotbar、CycleMapSlot、GogglesZoom（P1-6、P1-9）。
- 结构保护框和区域保护包改为增量发送并定向给相关玩家（P1-7、P1-8）。
- MasonJar、ChiseledCanopyShelf、MagicMap 的同步问题（P2）。
- **注意**：这一阶段改了包格式，会破坏客户端与服务端的版本兼容。需要提升网络协议版本号，在 `RegistrationEvents` 的 registrar 上加 `.versioned(...)`。

### 阶段 4：服务端 CPU 与世界生成（约 2 天）
- 分类核心和变化核心的扫描方式（P1-2、P1-3），以及分类核心的缓存泄漏。
- 提示怪重试（P1-13）、Kobold 寻路（P1-14）、Knight Phantom 和 Ghast Trap 的实体查询、巫妖蜡烛扫描、`LandmarkUtil` 缓存。
- 世界生成的两处热点。
- 客户端天气渲染缓存（P1-16）。
- 修复顺带发现的逻辑 bug（单独提交）。

---

## 四、风险与注意事项
- **协议兼容**：阶段 1、2 尽量保持包格式不变，让新旧版本可以混连。阶段 3 统一做破坏性改动，并升级协议版本号。
- **羽翼状态改由客户端推导**：远端玩家的 `walkAnimation` 和 `onGround` 在客户端是插值出来的值，动画可能和以前有细微差别，需要录屏对比。
- **追踪距离**：调小以后，大型 Boss 在远处可能突然消失。九头蛇、Ur-Ghast、Snow Queen 要单独调参。
- **属性修饰符代替效果**：玩家背包里不再显示急迫、跳跃提升的图标。这正好和现在 `showIcon=false` 的行为一致。另外要确认与其它模组对 `MobEffect` 的依赖，比如检测急迫的模组。
- **上游关系**：每个阶段都建议拆成独立 PR，提交回 TeamTwilight 上游，降低维护 fork 的分叉成本。
