# UE 5.8 ReplicationGraph AOI 空间查询与通用索引设计

> 本文分析 UE 5.8 ReplicationGraph 如何确定“某个连接应该接收哪些 Actor”，并说明如何将其读取优先的空间查询思想应用到 StaticMesh、SmartObject 和自定义 Actor 的通用空间索引中。

相关文档：[UE5.8 Uniform Spatial Hash 空间查询系统设计](./UE5.8_UniformSpatialHash空间查询系统设计.md)

## 1. 核心结论

ReplicationGraph 的 `UReplicationGraphNode_GridSpatialization2D` 使用一种与普通空间查询方向相反的设计：

```text
普通 Uniform Spatial Hash：
    Actor 按自身 Bounds 放入 Cell
    Query 按查询范围枚举多个 Cell

ReplicationGraph GridSpatialization2D：
    Actor 按自身 NetCullDistance 覆盖范围放入多个 Cell
    Viewer 查询时只读取 Viewer 所在的一个 Cell
```

换句话说，ReplicationGraph 把查询半径的展开成本从查询阶段提前到了 Actor 注册和移动阶段。

这是一种“反向覆盖索引”或“Interest Coverage Grid”：

- Actor 的 Cell Membership 表示“Viewer 位于哪些 Cell 时可能对该 Actor 感兴趣”。
- Viewer 不需要枚举周围 Cell。
- Viewer 只取当前位置 Cell 中预先准备好的 Actor List。
- 最后再用真实距离做圆形 Cull，消除正方形 Cell 覆盖产生的误报。

对于服务器每帧有大量玩家 AOI 查询、Actor 的相关半径相对稳定、绝大多数 Actor 静态或低频移动的场景，这种设计可以显著降低查询侧成本。

但 ReplicationGraph 原生实现不能直接替代前一份通用 Uniform Spatial Hash：

- 它是 2D Grid，Z 只做连接高度门槛，不参与 Cell 定位。
- Grid 使用嵌套 `TArray`，不是稀疏 `TMap` Hash。
- 它只管理参与网络同步的 Actor。
- Actor 插入范围来自 `NetCullDistance`，不是 Actor Bounds。
- Dynamic Actor 默认在 `PrepareForReplication()` 中每帧全部扫描。
- Cell 中 Actor List 删除需要先线性查找 Actor。
- 查询语义围绕连接、Dormancy、复制频率和网络预算，而不是任意业务条件。

推荐做法不是直接拿 ReplicationGraph 内部 Cell 做 gameplay 查询，而是将其思想加入通用 Subsystem：

```text
稀疏 Uniform Hash
+ EntityId/Generation
+ Actor 双向索引
+ O(1) Membership 回链删除
+ Move DirtySet
+ RepGraph 风格的 Interest Coverage Grid
+ Static/Dynamic/Frequency 分层列表
```

最终形成两类互补索引：

| 索引 | Actor 如何插入 | Query 如何访问 | 适合场景 |
|---|---|---|---|
| Bounds Grid | 按 Actor Bounds 覆盖 Cell | Query 枚举范围覆盖 Cell | 任意 Box/Sphere、半径经常变化 |
| Interest Coverage Grid | 按 Actor 的 InterestRadius 覆盖 Cell | Query Point 只取一个 Cell | 高频固定语义 AOI、玩家可见性、感知候选 |

---

## 2. ReplicationGraph 的 AOI 目标

传统 Actor Replication 如果对每个连接遍历所有可复制 Actor，再逐一判断距离和相关性，成本接近：

```text
O(ConnectionCount * ReplicatedActorCount)
```

ReplicationGraph 的目标是先通过 Graph Node 将 Actor 分组，使每个连接只处理少量候选：

```text
所有网络 Actor
    |
    +-- AlwaysRelevant
    +-- AlwaysRelevantForConnection
    +-- OwnerOnly
    +-- Spatialized Grid
    +-- Dormancy
    +-- 自定义业务 Node
```

每个连接每帧执行 Gather：

```text
GlobalGraphNodes::GatherActorListsForConnection
+ ConnectionGraphNodes::GatherActorListsForConnection
    |
    v
FGatheredReplicationActorLists
    |
    v
Dormancy / ReplicationPeriod / Distance Cull
    |
    v
Priority 排序和网络预算
    |
    v
真正复制 Actor
```

入口位于：

```text
UReplicationGraph::ServerReplicateActors
    -> 构造 FConnectionGatherActorListParameters
    -> 遍历 GlobalGraphNodes
    -> 遍历 ConnectionGraphNodes
    -> ReplicateActorListsForConnections_Default
```

UE 5.8 源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:1267
```

---

## 3. BasicReplicationGraph 的 Actor 路由

`UBasicReplicationGraph` 是最直接的示例实现。

初始化全局节点：

```cpp
GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
GridNode->CellSize = 10000.f;
GridNode->SpatialBias = FVector2D(-UE_OLD_WORLD_MAX, -UE_OLD_WORLD_MAX);
AddGlobalGraphNode(GridNode);

AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_ActorList>();
AddGlobalGraphNode(AlwaysRelevantNode);
```

Actor 路由：

```cpp
if (Actor->bAlwaysRelevant)
{
    AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
}
else if (Actor->bOnlyRelevantToOwner)
{
    ActorsWithoutNetConnection.Add(Actor);
}
else
{
    GridNode->AddActor_Dormancy(ActorInfo, GlobalInfo);
}
```

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/BasicReplicationGraph.cpp:62
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/BasicReplicationGraph.cpp:91
```

实际项目通常会按 Class 建立 Routing Policy，例如：

- AlwaysRelevant。
- SpatializeStatic。
- SpatializeDynamic。
- SpatializeDormancy。
- OwnerOnly。
- 自定义 Team、Squad、Streaming、World Partition Node。

AOI Grid 只是 ReplicationGraph 的一个候选来源，不是完整相关性规则。

---

## 4. GridSpatialization2D 的底层数据结构

### 4.1 Grid 不是 Hash Map

原生结构：

```cpp
TArray<TArray<UReplicationGraphNode_GridCell*>> Grid;
```

Cell 访问：

```cpp
UReplicationGraphNode_GridCell*& GetCell(int32 X, int32 Y)
{
    TArray<UReplicationGraphNode_GridCell*>& GridX = GetGridX(X);
    return GetCell(GridX, Y);
}
```

当 X/Y 越界时使用 `SetNum()` 扩展数组。

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraph.h:725
```

优点：

- 已分配范围内 Cell 指针访问是真正的数组 `O(1)`。
- 没有 CellKey Hash 计算和哈希冲突。
- 相邻 X/Y 数据较容易预测。
- 适合边界明确、Grid 范围可控的网络世界。

缺点：

- 稀疏且坐标跨度很大的世界可能产生大量空指针槽位。
- 必须通过 `SpatialBias` 把坐标转换为非负 Index。
- Actor 出现在 Bias 左侧或下侧时可能触发整体 Rebuild。
- 不适合直接作为无限稀疏 World Partition gameplay 索引。

### 4.2 SpatialBias 和 GridBounds

坐标量化：

```text
CellX = (WorldX - SpatialBias.X) / CellSize
CellY = (WorldY - SpatialBias.Y) / CellSize
```

`SetBiasAndGridBounds()` 可以固定 Grid 的合法世界范围：

```cpp
SpatialBias = FVector2D(GridBox.Min);
GridBounds = GridBox;
```

Actor 和 Viewer 超出 GridBounds 时会 Clamp 到边界 Cell。

如果没有固定 GridBounds，而 Actor 位于当前 SpatialBias 更小的一侧，ReplicationGraph 可能调整 Bias 并设置 `bNeedsRebuild`，随后重建全部空间节点。

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5677
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5792
```

### 4.3 Actor 的缓存信息

Dynamic Actor：

```cpp
struct FCachedDynamicActorInfo
{
    FNewReplicatedActorInfo ActorInfo;
    FActorCellInfo CellInfo;
};

TMap<FActorRepListType, FCachedDynamicActorInfo> DynamicSpatializedActors;
```

Static Actor：

```cpp
struct FCachedStaticActorInfo
{
    FNewReplicatedActorInfo ActorInfo;
    bool bDormancyDriven;
};

TMap<FActorRepListType, FCachedStaticActorInfo> StaticSpatializedActors;
```

未初始化完成的 Static Actor 会进入：

```cpp
TArray<FPendingStaticActors> PendingStaticSpatializedActors;
```

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraph.h:665
```

### 4.4 GridCell 内部节点

每个 `UReplicationGraphNode_GridCell` 将 Actor 分成：

```text
GridCell
    |
    +-- 普通 Static Actor List
    +-- DynamicNode
    |       默认 ActorListFrequencyBuckets
    |
    +-- DormancyNode
            为不同连接维护 Dormancy 状态
```

DynamicNode 默认创建：

```cpp
UReplicationGraphNode_ActorListFrequencyBuckets
```

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraph.h:535
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5269
```

### 4.5 Actor List

ReplicationGraph 使用紧凑 Actor List：

```cpp
struct FActorRepList
{
    int32 RefCount;
    int32 Max;
    int32 Num;
    FBitReference UsedBitRef;
    FActorRepListType Data[];
};
```

它通过 `FActorRepListRefView` 增删和遍历 Actor。

列表优点：

- Actor 指针连续存储。
- Gather 和后续候选遍历 Cache 友好。
- 适合复制系统的大量顺序扫描。
- 内存由 RepList Pool 管理，减少普通 TArray 小分配。

需要注意：

```cpp
RemoveFast(Actor)
    -> RemoveSingleSwap(Actor)
```

虽然交换删除本身是 `O(1)`，但 `RemoveSingleSwap` 仍需要先在线性列表中找到 Actor，单 Cell 删除是 `O(Lcell)`。

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraphTypes.h:375
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraphTypes.h:413
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraphTypes.h:514
```

---

## 5. RepGraph 最关键的反向空间化算法

### 5.1 普通空间 Hash 的插入范围

普通 Bounds Grid 通常按 Actor Bounds 插入：

```text
ActorCellRange = Quantize(ActorBounds)
```

查询半径 `Rquery` 时：

```text
QueryCellRange = Quantize(QueryCenter +/- Rquery)
```

### 5.2 RepGraph 的插入范围

ReplicationGraph 不按 Actor Bounds，而是按 Actor 的 CullDistance：

```cpp
MinX = ActorLocation.X - CullDistance;
MinY = ActorLocation.Y - CullDistance;
MaxX = ActorLocation.X + CullDistance;
MaxY = ActorLocation.Y + CullDistance;

StartX = MinX / CellSize;
StartY = MinY / CellSize;
EndX   = MaxX / CellSize;
EndY   = MaxY / CellSize;
```

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5688
```

因此一个 Actor 会被放进其 CullDistance 正方形覆盖的所有 Cell：

```text
        Actor Cull Square

    +---+---+---+---+
    | A | A | A | A |
    +---+---+---+---+
    | A | A | A | A |
    +---+---+---+---+
    | A | A | A | A |
    +---+---+---+---+
    | A | A | A | A |
    +---+---+---+---+
```

这里 Cell 中的 `A` 表示：

> 如果 Viewer 位于这个 Cell，Actor A 有可能在其同步距离内。

### 5.3 查询只取 Viewer 所在 Cell

连接 Gather 时：

```cpp
CellX = (Viewer.ViewLocation.X - SpatialBias.X) / CellSize;
CellY = (Viewer.ViewLocation.Y - SpatialBias.Y) / CellSize;

if (UReplicationGraphNode_GridCell* CellNode = GetCell(CellX, CellY))
{
    CellNode->GatherActorListsForConnection(Params);
}
```

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:6238
```

单 Viewer 的 Grid 定位和 Cell 访问是数组 `O(1)`。

它不需要按最大 NetCullDistance 枚举 Viewer 周围所有 Cell，因为 Actor 已经提前复制到了所有可能看见它的 Viewer Cell。

### 5.4 为什么仍需精确距离 Cull

Actor 按正方形范围插入 Cell，但真正的 NetCullDistance 是圆形距离语义。正方形角落会产生误报。

后续 Prioritize 阶段计算：

```cpp
DistSq = (ActorWorldLocation - ViewerLocation).SizeSquared();

if (DistSq > CullDistanceSquared)
{
    SkipActor;
}
```

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:1495
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:1565
```

所以完整语义是：

```text
Cell Membership = AOI broad phase
真实距离平方   = AOI exact distance filter
```

### 5.5 多 Viewer

一个 NetConnection 可能有多个 Viewer，例如 Child Connection 或特殊 Replay Viewer。

ReplicationGraph 会：

1. 为每个 Viewer 计算 Cell。
2. 用 `UniqueCurrentGridCells` 避免同一连接重复 Gather 相同 Cell。
3. Gather 每个唯一 Viewer Cell。

默认 `REPGRAPH_VIEWERS_PER_CONNECTION` 为 4，数组使用 Inline Allocator，因此 `Contains` 的线性去重成本很小。

---

## 6. Static、Dynamic 与 Dormancy

### 6.1 Static Actor

Static Actor 注册时立即计算 Cell Range，并添加到全部覆盖 Cell。

```text
AddActorInternal_Static
    -> ActorRepInfo.WorldLocation = ActorLocation
    -> StaticSpatializedActors.Emplace
    -> PutStaticActorIntoCell
    -> GetGridNodesForActor
    -> 每个 CellNode::AddStaticActor
```

Static Actor 不在每帧位置更新循环中。

如果注册时 Actor 尚未初始化，先放入 `PendingStaticSpatializedActors`，下一帧初始化完成后再放入 Grid。

### 6.2 Dynamic Actor

Dynamic Actor 注册时只加入：

```cpp
DynamicSpatializedActors.Emplace(Actor, ActorInfo);
```

第一次 `PrepareForReplication()` 才计算 CellInfo 并加入 Grid。

每帧 `PrepareForReplication()` 遍历全部 DynamicSpatializedActors：

```cpp
for (auto& DynamicActor : DynamicSpatializedActors)
{
    Location = DynamicActor->GetActorLocation();
    NewCellInfo = GetCellInfoForActor(Location, CullDistance);
    UpdateCellMembership(PreviousCellInfo, NewCellInfo);
}
```

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5833
```

这意味着原生动态更新的基础成本是：

```text
O(Ndynamic)
```

即使本帧大多数 Dynamic Actor 没有移动，也会读取位置并计算新范围。

这是网络复制系统的务实选择：

- 不依赖所有移动路径都正确触发 Delegate。
- 在服务器每帧已有复制准备阶段集中更新。
- 能保证网络相关性不会因漏回调长期错误。

但对于“绝大多数 Actor 静止、只有少量对象偶尔移动”的通用索引，Move DirtySet 通常更合适。

### 6.3 矩形差集更新

ReplicationGraph 不总是全删全加。

如果新旧 Cell Rect 完全不相交：

```text
删除全部旧 Cell
+ 添加全部新 Cell
```

如果仍有重叠，它分别处理：

- 左侧丢失或新增的列。
- 右侧丢失或新增的列。
- 上侧丢失或新增的行。
- 下侧丢失或新增的行。

中间交集 Cell 不修改。

这是一种针对轴对齐整数 Cell Rect 的无临时 HashSet 差集算法。

定义：

```text
Cold = 旧 Cell Rect 面积
Cnew = 新 Cell Rect 面积
D    = 旧新 Rect 对称差 Cell 数
```

实际 Cell Add/Remove 数量是：

```text
O(D)
```

范围边界比较本身是固定数量操作。由于旧新 Membership 能由 Rect 隐式表达，不需要遍历全部 Membership 求差，因此对规则矩形来说，这比通用 CellKey Set 差集更高效。

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5890
```

### 6.4 Dormancy 转换

`AddActor_Dormancy()` 根据 `bWantsToBeDormant` 选择：

- Dormant：作为 Static Actor。
- Awake：作为 Dynamic Actor。

Dormancy 状态变化时：

```text
Dynamic -> Dormant
    RemoveDynamic
    AddStatic

Dormant -> Awake
    RemoveStatic
    AddDynamic
```

这把不需要持续位置检查的 Dormant Actor 移出 Dynamic 全量扫描。

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5352
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5634
```

---

## 7. Cell Gather 之后仍做什么

Grid Gather 只产生候选，不代表 Actor 一定在本帧发送。

后续过滤包括：

### 7.1 Streaming Level Visibility

Actor List 按 StreamingLevelName 分类。只有客户端声明该 Level 可见时才 Gather 对应列表：

```cpp
Params.CheckClientVisibilityForLevel(StreamingLevelName)
```

### 7.2 Connection Dormancy

如果 Actor 已经对该连接 Dormant，可以快速跳过。

### 7.3 Replication Period

Actor 可能不是每帧复制：

```cpp
ReadyForNextReplication(ConnectionData, GlobalData, FrameNum)
```

### 7.4 Distance Cull

对所有 Viewer 计算距离。如果没有任何 Viewer 位于 CullDistance 内，则跳过。

### 7.5 Priority

根据：

- 距离。
- 距离上次复制的帧数。
- Actor Priority 设置。
- Starvation。
- 网络预算。

计算优先级并排序。

### 7.6 Frequency Buckets

GridCell 的 DynamicNode 默认是 `ActorListFrequencyBuckets`。它把 Dynamic Actor 分散到多个 List，每帧只将一个 Bucket 加入 Default Replication Path，降低单帧候选量。

```cpp
BucketIndex = ReplicationFrameNum % BucketCount;
```

源码：

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:3889
```

这说明 ReplicationGraph 的性能并不只来自空间 Grid，还来自：

```text
空间分区
+ 静态/动态分离
+ Dormancy
+ Frequency Buckets
+ Per-Connection Cache
+ 距离与复制周期过滤
+ 网络预算
```

---

## 8. ReplicationGraph AOI 的复杂度

### 8.1 符号

| 符号 | 含义 |
|---|---|
| `N` | 全部 Spatialized Actor 数量 |
| `Nd` | Dynamic Spatialized Actor 数量 |
| `Ns` | Static Spatialized Actor 数量 |
| `V` | 一个连接的 Viewer 数量，通常很小 |
| `C(e)` | Actor `e` 的 CullDistance 覆盖 Cell 数 |
| `D(e)` | Actor 移动后旧新 Cell Rect 的对称差 Cell 数 |
| `Lcell` | 某 Cell 某 Actor List 的长度 |
| `B` | 连接 Gather 后追加的原始 Actor 候选条目数 |
| `K` | 后续实际检查的候选数，可能包含跨列表重复 |
| `R` | 最终参与复制的 Actor 数 |

### 8.2 Static 注册

Actor 覆盖：

```text
C(e) = (EndX - StartX + 1) * (EndY - StartY + 1)
```

每个 Cell：

- 数组 Cell 定位：`O(1)`。
- 首次创建 GridCell UObject：固定但较高成本。
- Actor List Add：摊还 `O(1)`。

所以：

```text
Static 注册平均/摊还：O(C(e))
```

如果 Grid 数组需要扩展到很大的 X/Y，`SetNum()` 会产生额外初始化和分配成本。这也是原生 dense grid 需要边界控制的原因。

### 8.3 Dynamic 每帧更新

即使 Actor 不移动，也要遍历全部 Dynamic Actor：

```text
基础位置检查：O(Nd)
```

跨 Cell 的 Actor 还要更新差集：

```text
Σ O(D(e) * RemoveOrAddCost)
```

原生 Cell Actor List 删除需要线性查找：

```text
RemoveOrAddCost：
    Add    摊还 O(1)
    Remove O(Lcell)
```

因此严格表达比 `O(Nd + ΣD)` 更复杂：

```text
O(Nd + Σ AddedCells(e) + Σ RemovedCellListLength(e))
```

在 Cell List 较短时可以近似为：

```text
O(Nd + ΣD(e))
```

### 8.4 删除

先通过 Actor 的位置和 CullDistance 重建其 Cell Range，覆盖 `C(e)` 个 Cell。

每个 Cell 内 `RemoveFast` 仍需线性寻找 Actor：

```text
O(Σ Lcell_i)，i 属于 Actor Membership Cells
```

这不是严格的 `O(C(e))`。

ReplicationGraph 可以接受这一点，因为：

- Actor List 顺序遍历很快。
- 通常 Cell List 长度受控。
- 删除频率远低于每连接每帧 Gather。
- 它优先优化读取和顺序复制扫描，而不是删除。

### 8.5 连接空间定位

每个 Viewer：

```text
世界坐标到 Cell：O(1)
Grid[X][Y]：O(1)
```

多个 Viewer 使用小数组线性去重：

```text
O(V^2)
```

但默认 V 约为 1 到 4，因此工程上可视为固定小常数。

### 8.6 Gather 候选

`GridCell::GatherActorListsForConnection` 最终调用：

```cpp
FGatheredReplicationActorLists::AddReplicationActorList
```

UE 5.8 的实现会把 Actor List 追加到连接的 `TArray`，因此复制 `B` 个 Actor 指针的成本为：

```text
O(B)
```

连接 AOI Gather 总复杂度近似：

```text
O(V + B)
```

后续每 Actor 距离、Dormancy、Period 和 Priority 检查：

```text
O(K * V)
```

如果最终 Priority 完整排序：

```text
O(Rcandidate log Rcandidate)
```

所以 RepGraph 并不是“整个 AOI 严格 O(1)”。它优化的是空间候选定位：

```text
从 O(枚举周围多个 Cell)
降低到 O(一个 Viewer Cell)
```

输出和检查候选仍至少是 `Omega(B)`。

### 8.7 内存复杂度

每个 Actor 被复制到 `C(e)` 个 Cell List：

```text
Actor Membership 总内存 = O(Σ C(e))
```

CullDistance 越大、CellSize 越小，`C(e)` 越大。

2D 下大致：

```text
C(e) 约为 (2 * CullDistance / CellSize + 1)^2
```

这是典型的“空间换查询时间”。

---

## 9. 为什么 RepGraph 的读取可能更快

假设普通 Bounds Grid 查询半径 `R`，CellSize 为 `S`。

普通查询需要访问：

```text
Q 约为 (2R / S + 1)^2
```

RepGraph Coverage Grid 查询只访问：

```text
Q = 1 Cell / Viewer
```

例如：

```text
CellSize = 1000 cm
AOI Radius = 5000 cm
```

普通 2D Query 可能访问约：

```text
11 * 11 = 121 Cells
```

RepGraph 风格 Query：

```text
1 Cell
```

代价转移到 Actor：每个 Actor 可能被放入约 121 个 Cell。

如果：

- Actor 注册/移动少。
- Viewer 查询每帧发生。
- Viewer 数量多。
- AOI Radius 长期稳定。

那么预计算 121 个 Membership 很可能值得。

如果：

- Actor 大量高速移动。
- Query Radius 每次不同。
- 查询很少。
- InterestRadius 非常大。

那么普通 Bounds Grid 往往更好。

---

## 10. 不能直接复用 RepGraph 原生 Grid 的原因

### 10.1 ReplicationGraph 只存在于网络复制语境

它管理的是 NetDriver 认为需要复制的 Actor，不会自动包含：

- 非 Replicated StaticMesh Actor。
- Actorless SmartObject Runtime。
- 纯数据 Entity。
- 客户端本地对象。
- 未进入网络路由的业务对象。

### 10.2 内部数据结构不是稳定 gameplay API

GraphNode、Cell、DormancyNode 和 Actor List 是复制实现细节。业务直接读取会产生：

- 模块和服务器构建依赖。
- 生命周期耦合。
- Dormancy/Frequency Bucket 导致候选不是完整业务集合。
- 多 Connection 状态污染 gameplay 逻辑。
- 引擎升级维护成本。

### 10.3 Query 语义不同

RepGraph 回答：

> 此连接在本帧可能需要同步哪些 Actor？

通用空间索引回答：

> 此位置附近有哪些满足业务条件的 Entity？

前者还受到：

- bAlwaysRelevant。
- bOnlyRelevantToOwner。
- Dormancy。
- NetUpdateFrequency。
- Client Streaming Level Visibility。
- Connection Owner。
- 网络预算。

影响。

### 10.4 原生 Dense Grid 不适合无限稀疏大世界

World Partition 世界可能坐标跨度很大但当前只加载少量区域。使用 `TArray<TArray<Cell*>>` 会为中间空坐标保留指针槽位。

通用索引应继续使用：

```cpp
TMap<FSpatialCellCoord, FInterestCell>
```

以当前已加载 Cell 数而不是世界坐标跨度决定内存。

---

## 11. 推荐的 RepGraph 风格通用 AOI 索引

建议在前一份 `UUniformSpatialHashSubsystem` 中增加第二类 Grid：

```text
BoundsGrid
    按 Entity Bounds 建索引
    支持任意 Box/Sphere 查询

InterestCoverageGrid
    按 Entity Center +/- InterestRadius 建索引
    支持 Point/View AOI 查询
```

### 11.1 Entity Interest 配置

```cpp
USTRUCT(BlueprintType)
struct FSpatialInterestProfile
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    double InterestRadius = 0.0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    uint64 AudienceMask = MAX_uint64;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    uint8 FrequencyBucket = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bAlwaysRelevant = false;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bOwnerOnly = false;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bTrackMovement = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bTreatAsStatic = false;
};
```

这里的 InterestRadius 可以表示：

- 网络同步距离。
- AI 感知候选距离。
- SmartObject 最大检索距离。
- 交互提示显示距离。
- 资源点激活距离。
- 客户端表现激活距离。

### 11.2 Coverage Membership

```cpp
struct FInterestCellRange
{
    int64 StartX;
    int64 StartY;
    int64 EndX;
    int64 EndY;

    bool operator==(const FInterestCellRange&) const = default;
};
```

按 Entity 中心和 InterestRadius：

```cpp
FInterestCellRange CalculateInterestCellRange(
    const FVector& Location,
    double Radius,
    double InvCellSize)
{
    return {
        FMath::FloorToInt64((Location.X - Radius) * InvCellSize),
        FMath::FloorToInt64((Location.Y - Radius) * InvCellSize),
        FMath::FloorToInt64((Location.X + Radius) * InvCellSize),
        FMath::FloorToInt64((Location.Y + Radius) * InvCellSize)
    };
}
```

### 11.3 Cell 数据

推荐：

```cpp
struct FInterestCell
{
    TArray<FInterestCellItem> StaticEntities;

    TStaticArray<
        TArray<FInterestCellItem>,
        NumFrequencyBuckets> DynamicBuckets;

    TArray<FInterestCellItem> DormantOrDisabledEntities;

    uint64 CombinedCategoryMask = 0;
    uint64 CombinedAudienceMask = 0;
};
```

Cell Item：

```cpp
struct FInterestCellItem
{
    FSpatialEntityId EntityId;
    int32 MembershipIndex = INDEX_NONE;
};
```

Entity Membership：

```cpp
struct FInterestMembership
{
    FSpatialCellCoord Cell;
    int32 IndexInCell = INDEX_NONE;
    uint8 ListKind = 0;
    uint8 BucketIndex = 0;
};
```

与 RepGraph 原生实现不同，这里保留双向下标回链，使删除不需要在 Cell List 中线性寻找 Entity。

---

## 12. 改良版注册算法

### 12.1 流程

```text
Register Entity
    |
    v
分配 EntityId + Slot
    |
    v
写 Actor/SmartObject/Payload 双向索引
    |
    v
读取 InterestRadius、CategoryMask、AudienceMask、Bucket
    |
    v
计算 InterestCellRange
    |
    v
遍历 Range 内 CellKey
    |
    v
TMap::FindOrAdd Cell
    |
    v
追加到 Static / DynamicBucket / Dormant List
    |
    v
记录 IndexInCell + MembershipIndex
```

伪代码：

```cpp
FSpatialEntityId RegisterInterestEntity(const FSpatialInterestRegistration& Params)
{
    const FSpatialEntityId Id = AllocateEntitySlot();
    FSpatialEntitySlot& Slot = ResolveSlotChecked(Id);

    InitializeInterestData(Slot, Params);

    Slot.InterestCellRange = CalculateInterestCellRange(
        Params.Location,
        Params.Profile.InterestRadius,
        InterestGrid.InvCellSize);

    Slot.InterestCellRange.ForEachCell(
        [&](const FSpatialCellCoord& Coord)
        {
            FInterestCell& Cell = InterestGrid.Cells.FindOrAdd(Coord);
            TArray<FInterestCellItem>& List = SelectCellList(Cell, Slot);

            const int32 MembershipIndex = Slot.InterestMemberships.Num();
            const int32 IndexInCell = List.Num();

            List.Add({Id, MembershipIndex});
            Slot.InterestMemberships.Add({
                Coord,
                IndexInCell,
                Slot.ListKind,
                Slot.FrequencyBucket});
        });

    return Id;
}
```

复杂度：

```text
C(e) = InterestCellRange 覆盖 Cell 数
注册平均/摊还 = O(C(e))
内存 = O(C(e))
```

如果还同时加入 BoundsGrid：

```text
O(Cinterest(e) + Cbounds(e))
```

---

## 13. 改良版删除算法

利用上一份文档定义的两级回链：

```text
Membership.IndexInCell
CellItem.MembershipIndex
```

单 Membership 删除：

```cpp
void RemoveInterestMembership(
    FSpatialEntityId Id,
    int32 MembershipIndex)
{
    FSpatialEntitySlot& Slot = ResolveSlotChecked(Id);
    const FInterestMembership Membership =
        Slot.InterestMemberships[MembershipIndex];

    FInterestCell& Cell = InterestGrid.Cells.FindChecked(Membership.Cell);
    TArray<FInterestCellItem>& List = SelectCellList(Cell, Membership);

    const int32 RemoveIndex = Membership.IndexInCell;
    const int32 LastIndex = List.Num() - 1;

    if (RemoveIndex != LastIndex)
    {
        const FInterestCellItem MovedItem = List[LastIndex];
        List[RemoveIndex] = MovedItem;

        FSpatialEntitySlot& MovedSlot = ResolveSlotChecked(MovedItem.EntityId);
        MovedSlot.InterestMemberships[MovedItem.MembershipIndex].IndexInCell =
            RemoveIndex;
    }

    List.Pop(EAllowShrinking::No);
}
```

删除全部 Membership：

```text
O(C(e))
```

相比 RepGraph 原生列表：

```text
RepGraph 原生：O(Σ Lcell_i)
改良回链版： O(C(e))
```

代价是每个 Cell Item 和 Entity Membership 多保存下标。

---

## 14. 改良版移动算法

### 14.1 不扫描全部 Dynamic Entity

沿用前一方案：

```cpp
RootComponent->TransformUpdated
    -> DirtyEntities.Add(EntityId)
```

每帧只 Flush 不同的 Dirty Entity。

定义：

- `U`：本帧 Move Callback 次数。
- `M`：本帧不同 Dirty Entity 数量。
- `D(e)`：Entity 新旧 Cell Rect 的对称差 Cell 数。

成本：

```text
回调收集：平均 O(U)
Cell 更新：Σ O(D(e))，e 属于 M
```

不再有 RepGraph 原生的固定：

```text
O(Ndynamic)
```

### 14.2 Rect 差集算法

Interest Coverage 一定是轴对齐 Cell Rect，因此可以直接复用 RepGraph 的边条更新思想。

旧范围：

```text
[OldStartX, OldEndX] x [OldStartY, OldEndY]
```

新范围：

```text
[NewStartX, NewEndX] x [NewStartY, NewEndY]
```

若完全不相交：

```text
RemoveRect(Old)
AddRect(New)
```

若有交集：

```text
Remove 旧 Rect 左侧多出的列
Add    新 Rect 左侧新增的列
Remove 旧 Rect 右侧多出的列
Add    新 Rect 右侧新增的列

在 X 交集范围内：
Remove 旧 Rect 上侧多出的行
Add    新 Rect 上侧新增的行
Remove 旧 Rect 下侧多出的行
Add    新 Rect 下侧新增的行
```

实际 Hash 修改次数：

```text
O(D(e))
```

Range 完全相同：

```text
O(1)
```

只更新 Slot.Location，不修改任何 Cell List。

### 14.3 InterestRadius 改变

InterestRadius 改变等价于 Cell Rect Resize：

```cpp
UpdateInterestRadius(EntityId, NewRadius)
    -> CalculateNewRange
    -> ApplyRectDifference
```

不需要全删全加。

### 14.4 Static 与 Dynamic 分离

建议不完全依赖 Actor Mobility，而由注册参数明确：

```cpp
Profile.bTreatAsStatic
```

Static：

- 不绑定 Move Delegate，或者只绑定用于检测违规移动。
- 注册后不进入 DirtySet。
- 交互导致移动时，业务显式 `PromoteToDynamic(EntityId)`。

Dynamic：

- 绑定 Root TransformUpdated。
- 进入 Dynamic Frequency Bucket。

Dormant/Disabled：

- 可保留在独立列表。
- Query 默认跳过。
- 状态恢复时不重新计算 Cell Range，只在 Cell 内切换列表。

---

## 15. 改良版 AOI 查询算法

### 15.1 Query 参数

```cpp
struct FInterestQuery
{
    FVector ViewerLocation;

    uint64 RequiredAnyCategory = 0;
    uint64 RequiredAllCategory = 0;
    uint64 ExcludedCategory = 0;

    uint64 ViewerAudienceMask = MAX_uint64;

    uint32 FrameNumber = 0;
    bool bUseFrequencyBuckets = false;
    bool bRequireExactDistance = true;
    int32 MaxResults = 0;
};
```

### 15.2 单 Viewer 查询

```cpp
template<typename VisitorType>
void QueryInterestAtPoint(
    const FInterestQuery& Query,
    VisitorType&& Visitor)
{
    const FSpatialCellCoord CellCoord =
        InterestGrid.ToCell2D(Query.ViewerLocation);

    const FInterestCell* Cell = InterestGrid.Cells.Find(CellCoord);
    if (!Cell)
    {
        return;
    }

    VisitInterestList(Cell->StaticEntities, Query, Visitor);

    if (Query.bUseFrequencyBuckets)
    {
        const int32 Bucket = Query.FrameNumber % NumFrequencyBuckets;
        VisitInterestList(Cell->DynamicBuckets[Bucket], Query, Visitor);
    }
    else
    {
        for (const auto& Bucket : Cell->DynamicBuckets)
        {
            VisitInterestList(Bucket, Query, Visitor);
        }
    }
}
```

### 15.3 候选过滤顺序

Cell List 中每个 Entity：

```text
1. EntityId Generation 校验
2. AudienceMask
3. CategoryMask
4. Owner/Team/Connection 条件
5. 精确距离平方
6. 可选 Class/Tag/Predicate
7. Visitor/输出
```

精确距离：

```cpp
const double DistSq = FVector::DistSquared2D(
    Query.ViewerLocation,
    Slot.Location);

if (DistSq > FMath::Square(Slot.InterestRadius))
{
    continue;
}
```

如果使用 3D AOI：

```cpp
FVector::DistSquared
```

### 15.4 查询复杂度

单 Viewer：

```text
CellKey 计算：O(1)
TMap Cell 查找：平均 O(1)
扫描 Cell 候选：O(B)
固定 Filter：O(B)
输出：O(R)
```

总计：

```text
平均 O(1 + B + ΣP(e) + R)
```

固定 Predicate 时：

```text
O(1 + B)
```

这就是 RepGraph 思想相对普通范围查询的核心优势：没有 `Q` 个查询 Cell，`Q` 被预计算为 1。

### 15.5 多 Viewer 查询

```cpp
void QueryInterestForViewers(
    TConstArrayView<FVector> ViewerLocations,
    const FInterestQueryFilter& Filter,
    TArray<FSpatialEntityId>& OutResults);
```

流程：

1. 每个 Viewer 转 CellKey。
2. 对少量 Viewer 使用 Inline Array 去重 CellKey。
3. 每个唯一 Cell 访问一次。
4. QueryStamp 去重跨 Cell 重复 Entity。
5. 精确距离使用所有 Viewer 中的最小距离。

复杂度：

```text
O(V^2 + B + K * V)
```

当 `V <= 4` 时可以视为：

```text
O(B + K)
```

Viewer 数量较大时使用本地 `TSet<CellKey>`，将 Cell 去重改为平均 `O(V)`。

---

## 16. StaticMesh、SmartObject 与自定义 Actor 的接入

### 16.1 StaticMesh Actor

StaticMesh 白名单注册时同时指定 InterestProfile：

```cpp
FSpatialActorRegistrationParams Params;
Params.CategoryMask = SpatialCategory::StaticMesh
                    | SpatialCategory::Interactable;

Params.InterestProfile.InterestRadius = 3000.0;
Params.InterestProfile.bTreatAsStatic = true;
Params.InterestProfile.FrequencyBucket = 0;
```

查询者在任意位置只取所在 Interest Cell，即可获得 3000 cm 内可能相关的 StaticMesh Actor。

### 16.2 SmartObject

SmartObject 的 `USmartObjectSpacePartition` 适配器可以选择：

#### Bounds Query 模式

保持上一份文档设计：

```text
SmartObject Add(Bounds)
    -> BoundsGrid

Find(QueryBox)
    -> QueryBox 覆盖多个 Cell
```

优点：兼容任意 `FSmartObjectRequest.QueryBox`。

#### Coverage 模式

如果项目中 SmartObject 查询半径是少数固定档位，可以按最大业务半径建立 Coverage Grid：

```text
SmartObject Location +/- MaxSearchRadius
    -> InterestCoverageGrid

AI Location
    -> 单 Cell Query
```

然后仍用原始 `Request.QueryBox` 做精确检查。

限制：如果每次 SmartObject QueryBox 尺寸任意变化，单一 Coverage Radius 无法精确覆盖所有查询。可以：

- 使用最大允许查询半径，牺牲内存和误报。
- 按 500/1500/5000 cm 建立多个 Interest Profile Grid。
- 继续使用 BoundsGrid。

因此 SmartObject 默认更适合 BoundsGrid；只有项目查询半径高度标准化时，Coverage Grid 才明显占优。

### 16.3 自定义 Actor

```cpp
FSpatialActorRegistrationParams Params;
Params.CategoryMask = SpatialCategory::Enemy;
Params.bTrackMovement = true;
Params.InterestProfile.InterestRadius = EnemyReplicationRadius;
Params.InterestProfile.AudienceMask = Audience::Players;
Params.InterestProfile.FrequencyBucket = EntityId.Index % NumBuckets;

FSpatialEntityId Id = SpatialIndex->RegisterActor(EnemyActor, Params);
```

### 16.4 一个 Entity 同时进入两种 Grid

推荐允许：

```text
Entity Slot
    +-- BoundsMemberships
    +-- InterestMemberships
```

例如一个资源点：

- 玩家执行任意半径搜索时走 BoundsGrid。
- 每帧激活附近资源表现时走 InterestCoverageGrid。

EntityId、Actor 映射、状态和 Payload 共用，不重复创建 Entity Slot。

---

## 17. Frequency Bucket 思想

ReplicationGraph 不要求所有候选每帧都执行完整复制。

通用系统也可以让低优先级查询按帧分摊：

```cpp
BucketIndex = GetTypeHash(EntityId) % NumBuckets;
ActiveBucket = FrameNumber % NumBuckets;
```

适合：

- 环境交互提示刷新。
- 远距离 AI 感知更新。
- 资源点激活检查。
- 非关键 LOD/表现状态。

不适合：

- 即时攻击判定。
- 精确拾取。
- 关键碰撞。
- 必须同帧响应的交互。

API 应明确：

```cpp
QueryAllBuckets()
QueryCurrentFrequencyBucket(FrameNumber)
```

不能让普通调用者不知情地漏掉其他 Bucket。

---

## 18. AlwaysRelevant 与 Per-Audience List

RepGraph 不把所有 Actor 都塞进空间 Grid。通用 AOI 也应支持：

```cpp
TArray<FSpatialEntityId> AlwaysRelevantEntities;
TMap<FAudienceId, TArray<FSpatialEntityId>> AudienceRelevantEntities;
TMap<FObjectKey, TArray<FSpatialEntityId>> OwnerRelevantEntities;
```

AOI Query：

```text
Interest Cell Candidates
+ AlwaysRelevant
+ AudienceRelevant(ViewerAudience)
+ OwnerRelevant(ViewerOwner)
```

典型用途：

- 玩家自己的 Pawn、Controller、背包对象。
- Team 全局目标。
- 世界事件管理器。
- 始终需要客户端表现的少量对象。

AlwaysRelevant 数量必须保持很小，否则空间索引的收益会被常驻列表抵消。

---

## 19. Dormancy 思想在通用查询中的应用

网络 Dormancy 表示 Actor 状态暂时不需要重复发送。

通用索引可以对应为：

- Static：位置和索引范围不变。
- Dormant：空间位置不变，业务状态很少变化。
- Disabled：保留 Membership，但默认 Query 跳过。
- ActiveDynamic：需要 Move 更新和高频 Query。

状态转换：

```text
ActiveDynamic -> Dormant
    从 Dynamic Bucket 移到 Dormant/Static List
    解绑或暂停 Move 监听（按业务保证）

Dormant -> ActiveDynamic
    移回 Dynamic Bucket
    重新计算当前位置和 Cell Range
    恢复 Move 监听

Enabled -> Disabled
    从活跃 Cell List 移除或直接注销 Entity
    可由业务保留在独立 Disabled List
```

如果 Disabled 时间很长且数量巨大，可以彻底从 Coverage Grid 移除；如果频繁启停，保留 Membership 只切换列表更便宜。

---

## 20. World Partition 适配

### 20.1 不复用 RepGraph Dense Grid

仍使用：

```cpp
TMap<FSpatialCellCoord, FInterestCell>
```

仅为当前已加载 Actor 创建 Cell。

### 20.2 生命周期

沿用前一份文档：

- Actor Spawn：延迟注册。
- LevelAddedToWorld：补充扫描。
- Actor OnEndPlay：注销。
- PreLevelRemovedFromWorld：提前批量注销。
- TWeakObjectPtr：不阻止卸载。
- PostWorldOriginOffset：重建 Cell Key。

### 20.3 Streaming 可见性

如果 AOI 结果用于客户端表现或网络扩展，可以仿照 RepGraph 在 Entity Slot 保存：

```cpp
FName StreamingLevelName;
FName RuntimeDataLayerName;
```

Query 参数携带 Viewer 可见集合：

```cpp
const TSet<FName>* VisibleStreamingLevels;
const TSet<FName>* ActiveDataLayers;
```

在精确距离前做可见性过滤。

### 20.4 未加载 Actor

Coverage Grid 默认只表示服务器当前加载对象。如果业务需要查询未加载对象，必须建立基于 Actor Descriptor 或离线烘焙数据的独立索引，不能让运行时 Actor 索引保存强引用来阻止 World Partition 卸载。

---

## 21. 与前一份 Uniform Spatial Hash 的对比

| 项目 | Bounds Uniform Hash | RepGraph Coverage Grid |
|---|---|---|
| Actor 插入依据 | Actor Bounds | Location +/- InterestRadius |
| Query 输入 | Box/Sphere/任意范围 | Viewer Point 或少量 Viewer |
| Query Cell 数 | `Q` 个 | 每 Viewer 1 个 |
| Actor Membership | 通常少 | Radius 大时可能很多 |
| Query Radius | 可任意变化 | 预先编码在 Entity Profile |
| 静态对象 | 很适合 | AOI 高频时更适合 |
| 高速移动对象 | Membership 少，更新较轻 | 大 Radius 时更新差集较重 |
| 内存 | 较低 | 较高 |
| 查询延迟 | 受 Q 和局部密度影响 | 受单 Cell 候选 B 影响 |
| SmartObject 任意 QueryBox | 适合 | 不完全适合 |
| 玩家网络 AOI | 可用 | 通常更优 |
| 大世界 | 稀疏 TMap 适合 | 改良为稀疏 TMap 后适合 |

---

## 22. 选择规则

### 使用 BoundsGrid

- QueryBox 或 Radius 每次不同。
- 用户需要任意范围搜索。
- Actor InterestRadius 不明确。
- Actor 移动频繁。
- 内存比查询延迟更重要。
- SmartObject 保持原生 Request 语义。

### 使用 InterestCoverageGrid

- 每帧对大量 Viewer 做相同语义 AOI。
- 每个 Entity 有明确、稳定的 InterestRadius。
- Actor 多数静态、Dormant 或低频移动。
- Query 是 Point/View Location。
- 愿意用 Membership 内存换取单 Cell 查询。

### 两者同时使用

- 同一 Entity 既参与玩家 AOI，也参与任意半径 gameplay 查询。
- 网络同步和 AI 查询需要不同空间语义。
- 需要以真实 Profile 数据决定哪条路径更快。

---

## 23. 性能模型与分界点

### 23.1 BoundsGrid 每帧成本

假设：

- `V` 个 Viewer。
- 每个 Query 访问 `Q` 个 Cell。
- 每个 Query 扫描 `Bbounds` 个 Cell Item。

查询成本：

```text
O(V * (Q + Bbounds))
```

### 23.2 CoverageGrid 每帧成本

查询：

```text
O(V * (1 + Bcoverage))
```

更新：

```text
O(U + ΣD(e))
```

内存：

```text
O(ΣCinterest(e))
```

### 23.3 何时 Coverage 更值

粗略比较：

```text
Bounds 查询节省空间，但每 Viewer 多访问 Q-1 个 Cell。
Coverage 查询节省读取，但每 Entity 多维护 Cinterest-1 个 Membership。
```

当：

```text
ViewerQueriesPerSecond * QueryCellSavings
    >>
EntityMembershipUpdatesPerSecond
```

Coverage Grid 更可能占优。

不要只比较一次查询。ReplicationGraph 的优势来自同一 Grid 被所有连接每帧重复读取。

---

## 24. 数值示例

配置：

```text
CellSize = 2000 cm
InterestRadius = 6000 cm
```

Actor Coverage Rect 每轴大约覆盖：

```text
2 * 6000 / 2000 + 1 = 7 Cells
```

Membership：

```text
C(e) 约为 7 * 7 = 49
```

10 万个 Actor 全部使用同一 Profile，理论 Membership 约：

```text
4,900,000 个 Cell Item
```

假设每个 Item 保存：

```text
EntityId 8 bytes
MembershipIndex 4 bytes
对齐后约 16 bytes
```

仅 Cell Item 粗略内存：

```text
4,900,000 * 16 约为 78.4 MB
```

还未计入：

- Entity Membership 回链。
- TArray Slack。
- Cell Hash。
- Entity Slot。
- UObject/Actor。

因此不能无条件给所有 Entity 设置大 InterestRadius。

另一方面，单 Viewer Query 只访问一个 Cell。假设该 Cell 300 个候选，精确距离后 180 个结果：

```text
1 次 Cell Hash Find
+ 300 次连续 Cell Item 扫描
+ 300 次 Mask/距离判断
+ 180 次输出或 Visitor
```

对比 BoundsGrid 需要访问 49 个 Cell，并处理跨 Cell 去重。大量 Viewer 高频查询时，Coverage Grid 的读取路径更稳定。

---

## 25. CellSize 调优

Coverage Grid 的 CellSize 影响与 BoundsGrid 不完全相同。

CellSize 变小：

- Viewer Cell 中候选更精确，`Bcoverage` 下降。
- Actor 覆盖 Cell 数 `C(e)` 快速上升。
- 注册、删除、移动和内存成本上升。

CellSize 变大：

- `C(e)` 下降。
- 单 Cell 候选变多。
- 精确距离误报增加。
- Viewer Query 虽然仍只访问一个 Cell，但扫描 B 增大。

需要记录：

- 每 Profile 平均/P95 `C(e)`。
- 每 Cell Static/Dynamic/Dormant 数量。
- 每 Viewer 原始候选 B。
- 精确距离后的结果 R。
- `B/R` 误报比例。
- 每帧 Dirty Entity 和 `ΣD(e)`。
- Membership 总内存。

可以按 Profile 使用不同 Grid：

```text
NearInterestGrid     CellSize 1000
MediumInterestGrid   CellSize 4000
FarInterestGrid      CellSize 16000
```

但 Grid 数量越多，查询和维护常数也越大。只在 Profile 尺寸跨度显著时采用。

---

## 26. 推荐 API

```cpp
class UUniformSpatialHashSubsystem : public UTickableWorldSubsystem
{
public:
    FSpatialEntityId RegisterActor(
        AActor* Actor,
        const FSpatialActorRegistrationParams& Params);

    FSpatialEntityId RegisterSmartObject(
        FSmartObjectHandle Handle,
        const FBox& Bounds,
        const FSpatialInterestProfile* OptionalInterestProfile);

    bool UnregisterEntity(FSpatialEntityId Id);

    void UpdateEntityBounds(FSpatialEntityId Id, const FBox& Bounds);
    void UpdateInterestRadius(FSpatialEntityId Id, double Radius);
    void MarkEntityDirty(FSpatialEntityId Id);

    void QueryBox(
        const FBox& Bounds,
        const FSpatialQueryFilter& Filter,
        TArray<FSpatialEntityId>& OutResults);

    void QuerySphere(
        const FVector& Center,
        double Radius,
        const FSpatialQueryFilter& Filter,
        TArray<FSpatialEntityId>& OutResults);

    void QueryInterestAtPoint(
        const FInterestQuery& Query,
        TArray<FSpatialEntityId>& OutResults);

    void QueryInterestForViewers(
        TConstArrayView<FVector> ViewerLocations,
        const FInterestQueryFilter& Filter,
        TArray<FSpatialEntityId>& OutResults);
};
```

---

## 27. 网络 AOI 中如何直接使用 ReplicationGraph

如果目标本身就是网络同步，不需要重复实现一套 AOI。应创建项目自己的 `UReplicationGraph` 子类。

### 27.1 启用 ReplicationGraph

项目 NetDriver 配置指定 ReplicationDriverClassName，具体 Section 取决于项目 NetDriver 名称：

```ini
[/Script/OnlineSubsystemUtils.IpNetDriver]
ReplicationDriverClassName="/Script/YourGame.YourReplicationGraph"
```

项目需启用 ReplicationGraph Plugin，并在模块依赖中加入：

```csharp
"ReplicationGraph"
```

### 27.2 创建全局 Grid

```cpp
void UYourReplicationGraph::InitGlobalGraphNodes()
{
    Super::InitGlobalGraphNodes();

    GridNode = CreateNewNode<UReplicationGraphNode_GridSpatialization2D>();
    GridNode->CellSize = 10000.f;
    GridNode->SetBiasAndGridBounds(ProjectWorldBounds);
    AddGlobalGraphNode(GridNode);

    AlwaysRelevantNode = CreateNewNode<UReplicationGraphNode_ActorList>();
    AddGlobalGraphNode(AlwaysRelevantNode);
}
```

### 27.3 按 Class 路由

```cpp
void UYourReplicationGraph::RouteAddNetworkActorToNodes(
    const FNewReplicatedActorInfo& ActorInfo,
    FGlobalActorReplicationInfo& GlobalInfo)
{
    AActor* Actor = ActorInfo.Actor;

    if (Actor->bAlwaysRelevant)
    {
        AlwaysRelevantNode->NotifyAddNetworkActor(ActorInfo);
    }
    else if (IsProjectStaticReplicatedActor(Actor))
    {
        GridNode->AddActor_Static(ActorInfo, GlobalInfo);
    }
    else
    {
        GridNode->AddActor_Dormancy(ActorInfo, GlobalInfo);
    }
}
```

必须实现对称的 Remove 路由，并确保 Add/Remove 使用同一种 Policy。

### 27.4 配置 CullDistance

Actor 的 `NetCullDistanceSquared` 决定 Coverage Rect。项目应按 Class 设置合理距离，避免所有 Actor 使用过大的统一值。

典型策略：

- 玩家和关键单位：较大距离。
- 小交互物：较小距离。
- 投射物：较小距离且禁止触发 Grid Bias Rebuild。
- AlwaysRelevant：不要进入空间 Grid。
- OwnerOnly：进入 PerConnection Node。

### 27.5 不把 gameplay 语义塞进网络 Node

ReplicationGraph Node 应只解决同步相关性。如果 AI、SmartObject 和交互查询也需要相同空间思想，应使用独立 Subsystem，并共享配置或 Profile，而不是让 gameplay 系统直接读取网络 Graph 内部数组。

---

## 28. 与 Chaos Overlap 的关系

ReplicationGraph Coverage Grid 与前一份 Uniform Hash 一样属于语义 broad phase。

它比 Chaos Overlap 更适合：

- 玩家附近应激活哪些业务对象。
- 哪些 Actor 可能需要同步。
- 哪些 AI/SmartObject 是距离候选。
- 哪些表现对象需要更新。

它不能替代：

- 精确物理碰撞。
- Shape Overlap。
- Sweep/Raycast。
- Slot 入口胶囊有效性。

推荐：

```text
Coverage Grid 单 Cell AOI
    -> Mask/State/Distance
    -> 少量候选
    -> 必要时 Chaos 精确查询
```

---

## 29. 推荐实施方案

### 第一阶段

- 保留现有 BoundsGrid。
- 增加 `FSpatialInterestProfile`。
- 增加稀疏 InterestCoverageGrid。
- 单 Viewer Point Query。
- Entity Membership 双向回链。

### 第二阶段

- Rect 差集移动算法。
- Move DirtySet。
- Static/Dynamic List 分离。
- AudienceMask。
- AlwaysRelevant/OwnerRelevant。

### 第三阶段

- Frequency Buckets。
- 多 Viewer Query。
- Per-Viewer LastRelevantSet，用于 AOI Enter/Leave。
- World Partition/Data Layer 可见性过滤。

### 第四阶段

- 使用真实项目数据对比 BoundsGrid 与 CoverageGrid。
- 按 Profile 选择 Grid。
- 评估多层 Grid。
- 异步只读 Snapshot。

---

## 30. AOI Enter/Leave 集合

若用户需要的不只是“当前附近对象”，而是：

- 哪些 Entity 本帧刚进入 AOI。
- 哪些 Entity 本帧离开 AOI。
- 哪些 Entity 保持可见。

为每个 Viewer 保存：

```cpp
TSet<FSpatialEntityId> PreviousRelevant;
TSet<FSpatialEntityId> CurrentRelevant;
```

查询后：

```text
Entered = Current - Previous
Left    = Previous - Current
Stayed  = Current ∩ Previous
```

复杂度平均：

```text
O(|Current| + |Previous|)
```

然后 Swap 集合复用内存。

高性能版本可使用 Entity Index BitSet 加 Generation 校验，避免大量 TSet Node 分配。

这与 ReplicationGraph 的 PerConnection ActorInfo、Dormancy 和 Channel 生命周期思想一致，但通用 Subsystem 不应复制全部网络状态。

---

## 31. Debug 与 Profile

建议命令：

```text
SpatialInterest.DrawCells 0/1
SpatialInterest.DrawEntity EntityId
SpatialInterest.DrawViewer ActorName
SpatialInterest.DumpCell x y
SpatialInterest.DumpProfiles
SpatialInterest.DumpMemory
SpatialInterest.Validate
SpatialInterest.CompareBoundsGrid
```

统计：

```text
TotalEntities
StaticEntities
DynamicEntities
DormantEntities
TotalCells
TotalMemberships
AverageMembershipsPerEntity
P95MembershipsPerEntity
AverageCandidatesPerViewer
P95/P99CandidatesPerViewer
DistanceRejectRatio
MaskRejectRatio
DirtyEntitiesPerFrame
CellAddsPerFrame
CellRemovesPerFrame
QueryTime P50/P95/P99
```

使用 RepGraph 自带调试命令和 Insights Scope 对照：

```text
Net.RepGraph.Spatial.SetCellSize
Net.RepGraph.Spatial.DebugDynamic
NET_ReplicateActors_GatherForConnection
NET_ReplicateActors_PrioritizeForConnection
UReplicationGraphNode_GridSpatialization2D_PrepareForReplication
```

---

## 32. UE 5.8 源码索引

### ReplicationGraph 主类型

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraph.h
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraphTypes.h
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp
```

### BasicReplicationGraph

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/BasicReplicationGraph.cpp:62
```

### GridSpatialization2D 声明

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraph.h:578
```

### Grid、Dynamic/Static Cache

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraph.h:665
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraph.h:725
```

### Actor CullDistance 到 Cell Range

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5688
```

### Dynamic Actor 每帧更新和 Rect 差集

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5833
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5890
```

### Connection Viewer 单 Cell Gather

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:6238
```

### Actor List 数据结构

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraphTypes.h:375
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraphTypes.h:481
```

### Cell Static/Dynamic/Dormancy Node

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraph.h:534
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5159
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:5269
```

### Gather 参数

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Public/ReplicationGraphTypes.h:1719
```

### Gather 后距离 Cull 与 Priority

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:1449
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:1565
```

### Frequency Buckets

```text
Engine/Plugins/Runtime/ReplicationGraph/Source/Private/ReplicationGraph.cpp:3889
```

---

## 33. 最终建议

ReplicationGraph 的空间查询思想确实可以让 AOI 查询比普通范围 Hash 更快，但它并不是无条件更优。

最值得复用的设计是：

1. 将 Actor 的 InterestRadius 预先展开到 Cell Membership。
2. Viewer 查询只读取所在 Cell。
3. Static、Dynamic、Dormant 分离。
4. 使用 Frequency Bucket 分摊非关键更新。
5. 空间候选后仍执行真实距离过滤。
6. AlwaysRelevant、OwnerRelevant 不进入普通空间查询。
7. 多 Viewer 先去重 Cell。
8. 对规则 Cell Rect 使用边条差集更新。

不建议原样复用的部分是：

1. 大世界使用嵌套 Dense `TArray` Grid。
2. Dynamic Actor 每帧全量扫描。
3. Cell List 删除前线性搜索 Actor。
4. gameplay 直接依赖网络 Graph 内部列表。

推荐最终结构：

```text
UUniformSpatialHashSubsystem
    |
    +-- BoundsGrid
    |       任意范围查询
    |
    +-- InterestCoverageGrid
    |       RepGraph 风格单 Cell AOI
    |
    +-- Entity Slot Pool
    +-- Actor/Entity 双向索引
    +-- Static/Dynamic/Dormant/Frequency Lists
    +-- World Partition Lifecycle
    +-- SmartObject SpacePartition Adapter
```

网络同步本身继续使用项目自定义 ReplicationGraph；AI、SmartObject、交互和表现查询使用独立 Subsystem。两者可以共享 CellSize、Interest Profile 和 Class Policy 配置，但不要共享内部可变容器。
