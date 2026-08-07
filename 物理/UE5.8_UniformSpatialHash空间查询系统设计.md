# UE 5.8 Uniform Spatial Hash 空间查询系统设计

> 目标：在不向 Actor 挂载额外组件的前提下，建立一个由 `UWorldSubsystem` 集中维护的高性能语义空间索引，统一支持指定 StaticMesh 资产生成的 Actor、SmartObject 和业务主动注册的 Actor，并适配 World Partition 的动态加载与卸载。

> 当前实现更新：StaticMesh 资产白名单自动注册已取消，StaticMeshActor 只保留手动和批量注册。FastGeo 使用本文件第 27 节的 Level/Container 手动注册方案。

相关文档：[UE5.8 ReplicationGraph AOI 空间查询与通用索引设计](./UE5.8_ReplicationGraph_AOI空间查询思想与通用索引设计.md)

## 1. 设计结论

对于“查找周围满足业务条件的对象”这一类查询，推荐使用独立的 Uniform Spatial Hash，而不是直接把 Chaos `OverlapMulti` 当作通用对象检索系统。

推荐系统具备以下特征：

- 使用 `UWorldSubsystem` 或 `UTickableWorldSubsystem` 集中维护，不给被索引 Actor 添加组件。
- 使用固定 CellSize 的稀疏 Uniform Spatial Hash，Cell 通过哈希表按需创建。
- 使用 `EntityId = Index + Generation`，避免数组槽位复用后旧 ID 错误命中新对象。
- 同时维护 `EntityId -> Actor` 和 `Actor -> EntityId` 反向索引。
- 支持一个 Actor 对应多个 Entity，例如一个 Actor 同时包含 StaticMesh 记录和多个 SmartObject Handle。
- StaticMesh 自动注册由资产白名单决定。
- SmartObject 通过自定义 `USmartObjectSpacePartition` 适配器接入同一个 Hash，而不是修改 SmartObject 查询主流程。
- 自定义 Actor 通过 Subsystem API 主动注册，不要求组件。
- 通过 Actor Spawn、EndPlay、Level 加载/卸载和 RootComponent `TransformUpdated` 更新生命周期。
- 移动回调只写入 DirtySet，同一 Actor 一帧内移动多次也只更新一次 Hash。
- 对规则尺寸对象采用多 Cell 插入；超大对象进入独立的粗粒度 Uniform Grid。
- 空间查询先做 Cell 和位掩码过滤，再做距离、AABB、Class、Tag 或自定义 Predicate 精确过滤。
- 默认只在游戏线程修改索引。工作线程查询使用只读快照，不直接访问 Actor。

预期复杂度：

| 操作                |            平均复杂度 | 说明                                           |
| ----------------- | ---------------: | -------------------------------------------- |
| EntityId 校验       |           `O(1)` | 数组下标加 Generation 校验                          |
| Actor -> EntityId |           `O(1)` | `TMap<FObjectKey, TArray<FSpatialEntityId>>` |
| EntityId -> Actor |           `O(1)` | Entity Slot 中保存 `TWeakObjectPtr<AActor>`     |
| 注册                |           `O(C)` | `C` 为对象覆盖 Cell 数，通常是常数                       |
| 删除                |           `O(C)` | Cell 内 `RemoveAtSwap`，不线性搜索                  |
| 移动                | `O(Cold + Cnew)` | 删除旧 Cell Membership，再添加新 Membership          |
| 附近查询              |       `O(Q + K)` | `Q` 为覆盖 Cell 数，`K` 为候选数量                     |

当查询半径、CellSize 和对象密度相对稳定时，附近查询可以近似看作 `O(1 + K)`，但不能笼统宣称所有场景都是严格 `O(1)`。

---

## 2. 为什么不直接使用 Chaos Overlap

### 2.1 Chaos Overlap 的定位

Chaos 查询适合回答物理问题，例如：

- 一个球体、胶囊体或盒体是否与世界碰撞几何重叠。
- AI 胶囊放到某个 Slot 位置后是否会卡进墙体。
- 哪些物理 Shape 满足 Collision Channel、Profile 或 ObjectType 条件。
- 是否存在阻挡、触碰或复杂几何相交。

Chaos 的 broad phase 本身也有空间加速结构，因此不能简单描述成“Chaos 每次线性遍历全部 Actor”。它的性能成本主要来自查询语义比业务对象检索更重。

一次通用 `OverlapMulti` 可能涉及：

1. 构造查询 Shape、Transform 和碰撞过滤参数。
2. 获取或同步 Physics Scene 查询数据。
3. 遍历物理 broad phase 候选。
4. 对 Particle、Shape、Collision Filter 做判断。
5. 必要时执行几何 narrow phase overlap。
6. 从 Physics Proxy/Shape 映射回 Component 和 Actor。
7. 生成 `FOverlapResult` 数组。
8. 用户再对 Actor Class、GameplayTag、资产类型和业务状态进行二次过滤。

对于“附近有哪些可交互物”“附近有哪些资源点”“附近有哪些指定 StaticMesh Actor”“附近有哪些 SmartObject”来说，这些工作大部分不是查询真正需要的。

### 2.2 Uniform Spatial Hash 的优势

Uniform Spatial Hash 只存储业务需要查询的 Entity：

- 不遍历无关碰撞体。
- 不做几何 narrow phase。
- 不构造 `FOverlapResult`。
- 不需要先从碰撞 Shape 映射回 Actor 再判断业务类型。
- Category、状态和业务类型可直接通过整数位掩码过滤。
- Entity 数据连续存储，更有利于 CPU Cache。
- 可以只返回 EntityId，在真正需要时才解析 Actor。
- 查询成本主要由空间覆盖范围和局部对象密度决定，而不是整个物理世界的复杂程度。

### 2.3 不能替代 Chaos 的场景

Uniform Spatial Hash 是 broad phase 和语义索引，不是精确碰撞系统。以下情况仍应使用 Chaos：

- 判断胶囊是否实际与墙、地形、复杂碰撞体相交。
- 需要 Collision Channel/Profile 的真实物理语义。
- 需要精确 Shape Overlap、Sweep 或 Raycast。
- 需要考虑旋转凸体、三角网格或复杂碰撞。

推荐组合方式：

```text
Uniform Spatial Hash
    -> 找到少量业务候选
    -> 位掩码、距离、状态、Tag 过滤
    -> 仅对最终少量候选执行 Chaos 精确检查
```

SmartObject 的默认实现正是类似思路：附近 SmartObject 使用 HashGrid，Slot/Entrance 的占用碰撞验证才使用 `OverlapMulti`。

---

## 3. 系统边界

系统负责：

- 已加载、已注册 Entity 的空间 broad phase。
- StaticMesh 白名单 Actor 自动识别。
- SmartObject 空间候选查询。
- 自定义 Actor 注册和更新。
- Actor 与 EntityId 双向映射。
- World Partition Cell 加载和卸载后的增删。
- Category 位掩码过滤。

系统不负责：

- 加载尚未进入 World Partition Runtime Cell 的 Actor。
- 替代 Chaos 精确几何查询。
- 自动保证任意 Component 内部形状变化都能被 RootComponent 移动回调捕获。
- 在工作线程安全地解引用 `AActor*`。
- 自动选择适合所有玩法和密度的 CellSize。

World Partition 支持的含义是：索引始终准确表示当前 World 中已经加载的对象；卸载对象不会残留，未加载对象不会被索引强行引用而阻止卸载。

---

## 4. 总体架构

建议模块划分：

```text
UGameSpatialIndexSettings
    - StaticMesh 资产白名单
    - CellSize / LargeCellSize
    - 最大单 Entity Cell 数
    - 默认查询配置

UUniformSpatialHashSubsystem : UTickableWorldSubsystem
    - 生命周期监听
    - Entity Slot Pool
    - Actor 双向索引
    - RegularGrid
    - LargeGrid
    - PendingRegisterActors
    - DirtyEntities
    - 查询 API

FUniformSpatialHashGrid
    - TMap<FSpatialCellCoord, FSpatialCell>
    - Cell 坐标计算
    - Add / Remove / Update / Query

UProjectSmartObjectSpacePartition : USmartObjectSpacePartition
    - 将 SmartObject Add/Remove/Find 转发给 UUniformSpatialHashSubsystem

业务代码
    - RegisterActor
    - UnregisterEntity/Actor
    - SetEntityMask
    - QuerySphere/Box
```

推荐由 `UTickableWorldSubsystem` 承担更新，而不是每帧扫描全部 Actor。Tick 只处理：

- 本帧新 Spawn 或新加载的候选 Actor。
- 本帧被移动回调标记为 Dirty 的 Entity。
- 延迟删除队列。
- 可选的只读查询快照发布。

没有注册、移动、删除时，Tick 应接近空操作。

---

## 5. EntityId 与 Slot Pool

### 5.1 EntityId

不要直接把数组 Index 暴露成永久 ID。数组槽位删除后会被复用，旧 Index 可能错误指向新对象。

推荐：

```cpp
USTRUCT(BlueprintType)
struct FSpatialEntityId
{
    GENERATED_BODY()

    UPROPERTY()
    uint32 Index = MAX_uint32;

    UPROPERTY()
    uint32 Generation = 0;

    bool IsSet() const
    {
        return Index != MAX_uint32;
    }

    friend bool operator==(const FSpatialEntityId& A, const FSpatialEntityId& B)
    {
        return A.Index == B.Index && A.Generation == B.Generation;
    }
};

FORCEINLINE uint32 GetTypeHash(const FSpatialEntityId& Id)
{
    return HashCombineFast(::GetTypeHash(Id.Index), ::GetTypeHash(Id.Generation));
}
```

也可以打包成 `uint64`：

```cpp
uint64 Packed = (uint64(Generation) << 32) | uint64(Index);
```

### 5.2 Entity Slot

```cpp
enum class ESpatialEntitySource : uint8
{
    StaticMeshActor,
    SmartObject,
    CustomActor
};

struct FSpatialEntitySlot
{
    uint32 Generation = 1;
    bool bAllocated = false;

    TWeakObjectPtr<AActor> Actor;
    TWeakObjectPtr<USceneComponent> BoundRootComponent;
    int32 ActorEntityListIndex = INDEX_NONE;

    ESpatialEntitySource Source = ESpatialEntitySource::CustomActor;
    FSmartObjectHandle SmartObjectHandle;

    FBox Bounds = FBox(ForceInit);
    uint64 CategoryMask = 0;

    TArray<FSpatialCellMembership, TInlineAllocator<8>> Memberships;

    uint32 LastQueryStamp = 0;
};
```

Slot Pool：

```cpp
TArray<FSpatialEntitySlot> EntitySlots;
TArray<uint32> FreeSlotIndices;
```

分配：

1. 优先从 `FreeSlotIndices` 取 Index。
2. 设置 `bAllocated = true`。
3. 返回当前 `Index + Generation`。

释放：

1. 从 Grid 和 Actor 反向索引移除。
2. 清理弱引用、Payload 和 Membership。
3. `bAllocated = false`。
4. `Generation++`，避免旧 ID 有效。
5. Index 放回 Free List。

校验：

```cpp
bool IsValidEntity(FSpatialEntityId Id) const
{
    return EntitySlots.IsValidIndex(Id.Index)
        && EntitySlots[Id.Index].bAllocated
        && EntitySlots[Id.Index].Generation == Id.Generation;
}
```

---

## 6. Actor 双向索引

### 6.1 EntityId 到 Actor

直接存放在 Entity Slot：

```cpp
TWeakObjectPtr<AActor> Actor;
```

必须使用弱引用，不能因为空间索引而阻止 World Partition Actor 被卸载和 GC。

解析 API：

```cpp
AActor* ResolveActor(FSpatialEntityId Id) const;
```

解析时先校验 Generation，再调用 `Slot.Actor.Get()`。

### 6.2 Actor 到 EntityId

一个 Actor 可能：

- 命中 StaticMesh 白名单。
- 同时拥有一个或多个 SmartObject。
- 又被业务主动注册为另一种 Category。

因此推荐一对多：

```cpp
TMap<FObjectKey, TArray<FSpatialEntityId, TInlineAllocator<2>>> ActorToEntities;
```

为了让 Entity 从 Actor 反向数组中删除时也是 `O(1)`，Entity Slot 应保存自己在该数组中的位置：

```cpp
int32 ActorEntityListIndex = INDEX_NONE;
```

删除时对 `ActorToEntities[ActorKey]` 使用 `RemoveAtSwap`，再把被交换 Entity Slot 的 `ActorEntityListIndex` 修正为新位置。否则直接在数组中搜索 EntityId，会额外产生 `O(A)`，其中 `A` 是该 Actor 关联的 Entity 数量。

API：

```cpp
TConstArrayView<FSpatialEntityId> FindEntitiesForActor(const AActor* Actor) const;
bool FindPrimaryEntityForActor(const AActor* Actor, FSpatialEntityId& OutId) const;
```

如果项目能严格保证一 Actor 一 Entity，才可简化为：

```cpp
TMap<FObjectKey, FSpatialEntityId> ActorToEntity;
```

但统一支持 SmartObject 后，一对多更稳妥。

---

## 7. Uniform Spatial Hash 数据结构

### 7.1 Cell 坐标

UE 5 Large World Coordinates 下 `FVector` 使用双精度。Cell 坐标建议使用 `int64`，避免对世界范围做不必要限制：

```cpp
struct FSpatialCellCoord
{
    int64 X = 0;
    int64 Y = 0;
    int64 Z = 0;

    friend bool operator==(const FSpatialCellCoord&, const FSpatialCellCoord&) = default;
};

FORCEINLINE uint32 GetTypeHash(const FSpatialCellCoord& C)
{
    uint32 H = ::GetTypeHash(C.X);
    H = HashCombineFast(H, ::GetTypeHash(C.Y));
    H = HashCombineFast(H, ::GetTypeHash(C.Z));
    return H;
}
```

坐标换算：

```cpp
FSpatialCellCoord ToCell(const FVector& P) const
{
    return {
        FMath::FloorToInt64(P.X * InvCellSize),
        FMath::FloorToInt64(P.Y * InvCellSize),
        FMath::FloorToInt64(P.Z * InvCellSize)
    };
}
```

如果玩法本质上是平面世界，可以配置 `bUse2D = true`，令 `Z = 0`。多层建筑、空战、地下空间应使用 3D。

### 7.2 Cell 和 Membership

为了让删除不在 Cell 中线性查找，每个 Cell Item 需要反向指回 Entity Membership：

```cpp
struct FSpatialCellItem
{
    FSpatialEntityId EntityId;
    int32 MembershipIndex = INDEX_NONE;
};

struct FSpatialCell
{
    TArray<FSpatialCellItem> Items;
    uint64 CombinedCategoryMask = 0;
};

struct FSpatialCellMembership
{
    FSpatialCellCoord Cell;
    int32 IndexInCell = INDEX_NONE;
    bool bLargeGrid = false;
};

TMap<FSpatialCellCoord, FSpatialCell> Cells;
```

`CombinedCategoryMask` 是可选优化。若：

```cpp
(Cell.CombinedCategoryMask & Query.RequiredAnyMask) == 0
```

可跳过整个 Cell。删除或修改 Category 时需要重新维护它；如果维护成本高，可先不做。

### 7.3 多 Cell 插入

对 Entity 的 `FBox Bounds` 计算 MinCell/MaxCell，并插入所有覆盖 Cell：

```cpp
for (int64 Z = Min.Z; Z <= Max.Z; ++Z)
for (int64 Y = Min.Y; Y <= Max.Y; ++Y)
for (int64 X = Min.X; X <= Max.X; ++X)
{
    AddToCell(EntityId, {X, Y, Z});
}
```

优点：

- 查询不需要根据全局最大物体尺寸扩展 QueryBox。
- 不会遗漏跨 Cell 的实体。
- 适合尺寸接近 CellSize、覆盖 Cell 数很少的对象。

缺点：

- 大对象可能插入大量 Cell。
- 查询结果需要去重。

### 7.4 超大对象

配置：

```cpp
int32 MaxCellsPerEntity = 64;
double RegularCellSize = 1000.0;
double LargeCellSize = 8000.0;
```

如果对象覆盖 Cell 数超过阈值，不要继续写入大量普通 Cell。将其放入第二个固定 CellSize 的 `LargeGrid`。

查询时同时查询：

```text
RegularGrid(QueryBounds)
LargeGrid(QueryBounds)
```

这仍然是 Uniform Spatial Hash，只是针对两种尺寸分布使用两个固定分辨率。相比把超大对象放进每次都遍历的 SpillList，它在超大对象较多时更稳定。

### 7.5 O(1) Cell 删除

删除 Membership：

```cpp
FSpatialCell& Cell = Cells.FindChecked(Membership.Cell);
const int32 RemoveIndex = Membership.IndexInCell;
const int32 LastIndex = Cell.Items.Num() - 1;

if (RemoveIndex != LastIndex)
{
    const FSpatialCellItem MovedItem = Cell.Items[LastIndex];
    Cell.Items[RemoveIndex] = MovedItem;

    FSpatialEntitySlot& MovedSlot = ResolveSlotChecked(MovedItem.EntityId);
    MovedSlot.Memberships[MovedItem.MembershipIndex].IndexInCell = RemoveIndex;
}

Cell.Items.Pop(EAllowShrinking::No);
```

Cell 为空时从 `TMap` 删除。这样每个 Membership 的删除是平均 `O(1)`。

---

## 8. 核心操作算法与复杂度推导

本节详细说明注册、删除、移动和查询到底执行了哪些步骤，以及复杂度中的每一项来自哪里。

### 8.1 符号定义

先定义后续分析使用的变量：

| 符号 | 含义 |
|---|---|
| `N` | 当前已注册 Entity 总数 |
| `H` | 当前实际存在的非空 Hash Cell 数量 |
| `S` | 注册或更新一个 Actor 时需要检查的组件数量 |
| `C(e)` | Entity `e` 的 Bounds 覆盖并加入的 Cell 数量 |
| `C_old` | Entity 移动前的 Membership 数量 |
| `C_new` | Entity 移动后的 Membership 数量 |
| `D` | 移动前后真正需要删除或新增的 Cell Membership 数量 |
| `Q` | 一次查询范围覆盖、需要访问的 Cell 数量 |
| `B` | 查询过程中从这些 Cell 中实际扫描的 Cell Item 总数，包含重复 Entity |
| `K` | 去重后进入 Entity 级过滤的候选数量，`K <= B` |
| `R` | 最终输出结果数量，`R <= K` |
| `A` | 一个 Actor 关联的 Entity 数量 |
| `P(e)` | 对候选 `e` 执行业务 Predicate 的成本 |
| `Cmax` | 配置允许 Entity 在 RegularGrid 中覆盖的最大 Cell 数量 |

3D 查询覆盖 Cell 数：

```text
Q = Qx * Qy * Qz
```

其中：

```text
Qx = MaxCellX - MinCellX + 1
Qy = MaxCellY - MinCellY + 1
Qz = MaxCellZ - MinCellZ + 1
```

2D Grid 中固定 `Qz = 1`。

Entity 覆盖 Cell 数同理：

```text
C(e) = Cx(e) * Cy(e) * Cz(e)
```

### 8.2 基础容器操作的复杂度

算法建立在以下容器操作之上：

| 操作 | 平均/摊还复杂度 | 最坏复杂度 | 说明 |
|---|---:|---:|---|
| `TMap::Find` | 平均 `O(1)` | `O(H)` | 极端哈希碰撞时可能退化 |
| `TMap::FindOrAdd` | 平均 `O(1)` | `O(H)` | 可能触发哈希表扩容 |
| `TMap::Remove` | 平均 `O(1)` | `O(H)` | 已知 Cell Key |
| `TArray::Add` | 摊还 `O(1)` | 单次扩容 `O(M)` | 多次 Add 的总成本仍是线性 |
| `TArray::Pop` | `O(1)` | `O(1)` | 使用 `EAllowShrinking::No` |
| `TArray::RemoveAtSwap` | `O(1)` | `O(1)` | 已知删除下标，不保持顺序 |
| Entity Slot 校验 | `O(1)` | `O(1)` | 数组下标和 Generation 比较 |
| 位掩码过滤 | `O(1)` | `O(1)` | 固定数量整数按位运算 |
| AABB 相交/点到 AABB 距离 | `O(1)` | `O(1)` | 固定数量坐标运算 |

文档中出现的 `O(1)` Cell 操作，默认指哈希分布正常情况下的平均复杂度。若所有 Cell Key 被恶意或异常地哈希到同一 Bucket，`TMap` 查找理论上会退化，但正常世界坐标和合适的 `GetTypeHash` 下不应以这种最坏情况估算日常性能。

`TArray::Add` 的摊还 `O(1)` 来自容量按比例增长：某一次扩容需要搬移现有元素，但连续插入 `M` 个元素的总搬移成本仍为 `O(M)`，所以平均到每次 Add 是常数级。

### 8.3 注册算法

#### 8.3.1 注册流程

Actor 注册的完整逻辑：

```text
输入 Actor + RegistrationParams
    |
    v
校验 World、Actor 生命周期和重复注册状态
    |
    v
StaticMesh 自动注册时扫描组件和资产白名单
    |
    v
计算 Entity World Bounds
    |
    v
从 FreeList 分配 Entity Slot，生成 Index + Generation
    |
    v
写入 EntityId -> Actor 和 Actor -> EntityIds
    |
    v
根据 Bounds 计算 MinCell、MaxCell、CellCount
    |
    +-- CellCount <= Cmax --> RegularGrid
    |
    +-- CellCount >  Cmax --> LargeGrid 重新量化
    |
    v
遍历覆盖 Cell：FindOrAdd Cell，追加 CellItem，记录 Membership 回链
    |
    v
绑定 OnEndPlay 和可选 RootComponent::TransformUpdated
    |
    v
返回 EntityId
```

伪代码：

```cpp
FSpatialEntityId RegisterActor(AActor* Actor, const FSpatialActorRegistrationParams& Params)
{
    if (!CanRegister(Actor))
    {
        return {};
    }

    const FSpatialEntityId Existing = FindExistingEntity(Actor, Params.Source);
    if (Existing.IsSet())
    {
        return Existing;
    }

    // StaticMesh Actor 可能需要遍历 S 个组件。
    const FBox Bounds = CalculateRegistrationBounds(Actor, Params);
    if (!Bounds.IsValid)
    {
        return {};
    }

    const FSpatialEntityId Id = AllocateEntitySlot();
    FSpatialEntitySlot& Slot = ResolveSlotChecked(Id);
    InitializeSlot(Slot, Actor, Bounds, Params);

    LinkActorToEntity(Actor, Id);

    FUniformSpatialHashGrid* Grid = SelectGrid(Bounds);
    const FCellRange Range = Grid->CalculateCellRange(Bounds);

    Slot.Memberships.Reserve(Range.CellCount());
    Range.ForEachCell([&](const FSpatialCellCoord& Coord)
    {
        FSpatialCell& Cell = Grid->Cells.FindOrAdd(Coord);

        const int32 MembershipIndex = Slot.Memberships.Num();
        const int32 IndexInCell = Cell.Items.Num();

        Cell.Items.Add({Id, MembershipIndex});
        Slot.Memberships.Add({Coord, IndexInCell, Grid->IsLargeGrid()});
    });

    BindActorLifecycle(Actor, Slot, Params.bTrackMovement);
    return Id;
}
```

#### 8.3.2 Bounds 计算成本

自定义 Bounds 已经由调用者提供时：

```text
CalculateBounds = O(1)
```

使用 Actor Bounds 或聚合命中 StaticMeshComponent 时，需要访问 `S` 个相关组件：

```text
CalculateBounds = O(S)
```

StaticMesh 白名单查找使用 `TSet<FSoftObjectPath>`，每个组件平均 `O(1)`，扫描全部组件仍为 `O(S)`。

#### 8.3.3 Slot 与反向索引成本

- 从 FreeList Pop 或向 Slot Array Add：摊还 `O(1)`。
- `ActorToEntities.FindOrAdd(FObjectKey)`：平均 `O(1)`。
- Actor Entity 数组 Add：摊还 `O(1)`。
- 记录 `ActorEntityListIndex`：`O(1)`。
- Delegate 绑定：按一次固定操作计为 `O(1)`。

#### 8.3.4 Cell 插入成本

Entity 覆盖 `C(e)` 个 Cell。每个 Cell 执行：

1. `TMap::FindOrAdd`：平均 `O(1)`。
2. `Cell.Items.Add`：摊还 `O(1)`。
3. `Slot.Memberships.Add`：摊还 `O(1)`。

因此 Cell 插入总成本：

```text
O(C(e))
```

Actor 注册总复杂度：

```text
平均/摊还：O(S + C(e))
```

如果业务直接提供 Bounds，或者 Actor 只有固定少量组件：

```text
O(C(e))
```

当 `C(e)` 被 `Cmax` 限制，并且超过阈值的对象转入 LargeGrid 后，在配置上可将一次注册的 Grid 成本约束为常数上限。此时常规对象注册在业务分布下可近似为摊还 `O(1)`，但严格表达仍应保留 `C(e)`。

理论上的异常哈希碰撞最坏情况可达到：

```text
O(S + C(e) * H)
```

这不是正常数据分布下的预期复杂度。

### 8.4 删除算法

#### 8.4.1 为什么普通删除可能是 O(M)

如果 Cell 只保存：

```cpp
TArray<FSpatialEntityId> Items;
```

删除时需要先找到 EntityId 在数组中的位置。Cell 中有 `M` 个条目时：

```text
Search + Remove = O(M)
```

即使之后调用 `RemoveAtSwap`，前面的线性搜索已经是 `O(M)`。

本方案通过 Membership 保存 `IndexInCell`，直接定位删除位置，并通过 `MembershipIndex` 修正交换元素的回链，因此单个 Cell 删除为平均 `O(1)`。

#### 8.4.2 完整删除流程

```text
输入 EntityId
    |
    v
Index + Generation 校验
    |
    v
从 Dirty/Pending 队列移除或标记无效
    |
    v
遍历 C(e) 个 Membership
    |
    v
根据 CellKey 找到 Cell
    |
    v
按 IndexInCell 执行 RemoveAtSwap
    |
    v
修正被交换 Entity 的 Membership.IndexInCell
    |
    v
Cell 为空则从 TMap 删除
    |
    v
从 ActorToEntities 按 ActorEntityListIndex 执行 RemoveAtSwap
    |
    v
解除 Actor Delegate，清理 Slot，Generation++，Index 放回 FreeList
```

伪代码：

```cpp
bool UnregisterEntity(FSpatialEntityId Id)
{
    FSpatialEntitySlot* Slot = ResolveSlot(Id);
    if (!Slot)
    {
        return false;
    }

    UnbindActorLifecycle(*Slot);

    // 倒序删除，便于 Slot.Memberships 自身使用 Pop/RemoveAtSwap。
    while (!Slot->Memberships.IsEmpty())
    {
        const int32 MembershipIndex = Slot->Memberships.Num() - 1;
        const FSpatialCellMembership Membership =
            Slot->Memberships[MembershipIndex];

        FUniformSpatialHashGrid& Grid =
            Membership.bLargeGrid ? LargeGrid : RegularGrid;

        FSpatialCell& Cell = Grid.Cells.FindChecked(Membership.Cell);
        const int32 RemoveIndex = Membership.IndexInCell;
        const int32 LastIndex = Cell.Items.Num() - 1;

        if (RemoveIndex != LastIndex)
        {
            const FSpatialCellItem MovedItem = Cell.Items[LastIndex];
            Cell.Items[RemoveIndex] = MovedItem;

            FSpatialEntitySlot& MovedSlot = ResolveSlotChecked(MovedItem.EntityId);
            MovedSlot.Memberships[MovedItem.MembershipIndex].IndexInCell = RemoveIndex;
        }

        Cell.Items.Pop(EAllowShrinking::No);
        Slot->Memberships.Pop(EAllowShrinking::No);

        if (Cell.Items.IsEmpty())
        {
            Grid.Cells.Remove(Membership.Cell);
        }
    }

    UnlinkActorWithRemoveAtSwap(Id, *Slot);
    ReleaseEntitySlot(Id);
    return true;
}
```

#### 8.4.3 删除复杂度

Entity 有 `C(e)` 个 Membership，每个 Membership 的 Cell 查找、交换删除、回链修正和可选空 Cell 删除平均都是 `O(1)`：

```text
Grid 删除 = O(C(e))
```

Actor 反向索引使用保存的 `ActorEntityListIndex`：

```text
Actor 反向索引删除 = O(1)
```

Slot 清理和 FreeList Push：

```text
O(1)
```

所以总复杂度：

```text
平均：O(C(e))
```

如果没有 `ActorEntityListIndex`，需要在 Actor 的 Entity 数组中搜索：

```text
O(C(e) + A)
```

如果没有 `IndexInCell` 回链，则可能退化为：

```text
O(所有 Membership 所在 Cell 的条目数之和)
```

这正是设计两级回链的原因。

### 8.5 移动算法

#### 8.5.1 Move 回调阶段

RootComponent `TransformUpdated` 回调只做：

```cpp
DirtyEntities.Add(EntityId);
```

`TSet::Add` 平均 `O(1)`。同一 Entity 一帧内触发多次回调，只有第一次真正增加集合元素，后续只做重复查找。

假设一帧有 `U` 次 TransformUpdated，但只有 `M` 个不同 Entity 移动：

```text
回调收集平均成本 = O(U)
实际 Hash 更新次数 = M，而不是 U
```

通常 `M <= U`，频繁移动组件的合并收益会很明显。

#### 8.5.2 Flush 阶段

每个 Dirty Entity：

1. 校验 EntityId：`O(1)`。
2. 检查 Actor/Root 是否仍有效：`O(1)`。
3. 重新计算 Bounds：自定义 Bounds 为 `O(1)`，组件聚合为 `O(S)`。
4. 计算新 Cell Range：`O(1)`。
5. 比较新旧 Grid 和 Cell 范围。
6. 若范围未变化，只写 `Slot.Bounds`。
7. 若范围变化，更新 Membership。

#### 8.5.3 最简单可靠的更新算法

先删除全部旧 Membership，再插入全部新 Membership：

```cpp
void UpdateEntityBounds(FSpatialEntityId Id, const FBox& NewBounds)
{
    FSpatialEntitySlot& Slot = ResolveSlotChecked(Id);
    const FCellRange NewRange = SelectGrid(NewBounds).CalculateCellRange(NewBounds);

    if (IsSameGridAndCellRange(Slot, NewRange))
    {
        Slot.Bounds = NewBounds;
        return;
    }

    RemoveAllMemberships(Id, Slot); // O(C_old)
    Slot.Bounds = NewBounds;
    AddAllMemberships(Id, Slot);    // O(C_new)
}
```

复杂度：

```text
Bounds 计算                 O(S)
范围未变化                 O(1)，总计 O(S)
范围发生变化               O(C_old + C_new)，总计 O(S + C_old + C_new)
```

对于绝大多数“在当前 Cell 内小幅移动”的 Entity，Cell Range 不变，不需要操作 Hash Cell。

#### 8.5.4 差集更新算法

当 Entity 常跨 Cell 移动且覆盖多个 Cell时，可以只处理新旧 Cell 集合的差集：

```text
RemovedCells = OldCells - NewCells
AddedCells   = NewCells - OldCells
KeptCells    = OldCells ∩ NewCells
```

定义：

```text
D = |RemovedCells| + |AddedCells|
```

真正的 Hash 修改成本是：

```text
O(D)
```

但必须注意，构造或比较新旧 Cell 集合本身仍需要访问这些 Cell：

- 将较小集合放入临时 HashSet：平均 `O(C_old + C_new)`。
- 排序后双指针求差：`O(C_old log C_old + C_new log C_new)`。
- 规则长方体 Cell Range 使用坐标范围专用算法，可减少临时内存，但实现更复杂。

因此差集版本的严格总复杂度通常仍写作：

```text
O(S + C_old + C_new)
```

它优化的是 Hash Cell 实际修改次数和常数，而不是一般情况下的渐进阶数。

#### 8.5.5 一帧移动更新总成本

一帧有 `M` 个不同 Dirty Entity：

```text
O(U) + Σ[ O(S_i + C_old_i + C_new_i) ]
```

若其中大多数 Entity 未跨 Cell：

```text
约为 O(U) + Σ O(S_i)
```

若 Bounds 可由缓存 Extent 加 Actor Transform 直接计算，`S_i` 也可降为常数。

### 8.6 查询算法

#### 8.6.1 Box 查询完整流程

```text
输入 QueryBox + Filter
    |
    v
QueryBox 量化为 RegularGrid Cell Range
    |
    v
遍历 Qregular 个 CellKey
    |
    v
TMap 查找非空 Cell
    |
    v
遍历 Cell.Items，QueryStamp 去重
    |
    v
校验 EntityId Generation
    |
    v
CategoryMask 过滤
    |
    v
Entity Bounds 与 QueryBox 精确 AABB 相交
    |
    v
可选 Actor、Class、Tag、Predicate 过滤
    |
    v
输出 EntityId 或调用 Visitor
    |
    v
以同样流程查询 LargeGrid
```

伪代码：

```cpp
template<typename VisitorType>
void QueryBoxInternal(
    const FBox& QueryBox,
    const FSpatialQueryFilter& Filter,
    VisitorType&& Visitor)
{
    const uint32 QueryStamp = AcquireQueryStamp();

    auto QueryOneGrid = [&](const FUniformSpatialHashGrid& Grid)
    {
        const FCellRange Range = Grid.CalculateCellRange(QueryBox);

        Range.ForEachCell([&](const FSpatialCellCoord& Coord)
        {
            const FSpatialCell* Cell = Grid.Cells.Find(Coord);
            if (!Cell)
            {
                return;
            }

            if (!CellCanMatchMask(*Cell, Filter))
            {
                return;
            }

            for (const FSpatialCellItem& Item : Cell->Items)
            {
                FSpatialEntitySlot* Slot = ResolveSlot(Item.EntityId);
                if (!Slot)
                {
                    continue;
                }

                if (Slot->LastQueryStamp == QueryStamp)
                {
                    continue;
                }
                Slot->LastQueryStamp = QueryStamp;

                if (!PassMasks(*Slot, Filter))
                {
                    continue;
                }

                if (!Slot->Bounds.Intersect(QueryBox))
                {
                    continue;
                }

                if (!PassSourceLifetime(*Slot))
                {
                    continue;
                }

                if (!PassOptionalPredicate(*Slot, Filter))
                {
                    continue;
                }

                if (Visitor(MakeEntityView(Item.EntityId, *Slot))
                    == ESpatialQueryControl::Stop)
                {
                    return;
                }
            }
        });
    };

    QueryOneGrid(RegularGrid);
    QueryOneGrid(LargeGrid);
}
```

实际实现中，Visitor 返回 Stop 后需要让停止状态传播出 Cell 和 Grid 的嵌套遍历。

#### 8.6.2 Cell 遍历成本 Q

查询范围转换为 Cell Range 是固定数量的除法、Floor 和减法：

```text
O(1)
```

之后必须访问范围内的每个 CellKey：

```text
O(Q)
```

即使某些 Cell 不存在，也需要进行一次平均 `O(1)` 的 `TMap::Find`。因此空旷区域的大范围查询仍可能受 `Q` 影响。

两套 Grid 时：

```text
Q = Qregular + Qlarge
```

Grid 数量固定为 2，所以不会额外引入随 `N` 増长的阶数。

#### 8.6.3 Cell Item 扫描成本 B

设查询访问的每个非空 Cell 分别包含：

```text
b1, b2, ..., bQ
```

实际扫描条目总数：

```text
B = b1 + b2 + ... + bQ
```

遍历所有 Cell Item 必然需要：

```text
O(B)
```

同一个 Entity 插入多个 Cell 时，它可能在 `B` 中出现多次。QueryStamp 让每次重复判断保持 `O(1)`，但不能消除读取重复 Cell Item 的成本。

在 `Cmax` 限制下，一个 Entity 最多在 RegularGrid 中出现固定次数，因此正常小范围查询中通常可以认为：

```text
B <= 常数 * K + 邻接 Cell 中的空间误报
```

但密度极高、对象大量重叠或查询范围很大时，`B` 仍可能接近 `N * Cmax`。

#### 8.6.4 去重、Mask 和精确空间过滤成本

对 `B` 个 Cell Item：

- Entity Slot 下标与 Generation 校验：`O(1)`。
- QueryStamp 比较：`O(1)`。

去重后只有 `K` 个 Entity 执行：

- Category/State 位掩码：`O(1)`。
- AABB 相交：`O(1)`。
- Actor 弱引用校验：`O(1)`。

所以固定过滤总成本：

```text
O(B + K)
```

由于 `K <= B`，可合并写成：

```text
O(B)
```

#### 8.6.5 业务 Predicate 成本

如果 Predicate 包含 GameplayTag、Class、接口或业务数据访问，不能一概视为纯常数。严格写法：

```text
Σ P(e)，e 属于通过便宜过滤后的候选集合
```

例如：

- 固定 Class 指针比较：通常近似 `O(1)`。
- 固定数量 Tag 位图判断：近似 `O(1)`。
- 遍历 Actor Component：`O(S_e)`。
- 对候选执行寻路或 Chaos 查询：成本远大于普通 Hash 查询。

因此 Predicate 必须放在 Cell、Mask 和 Bounds 过滤之后。

#### 8.6.6 输出成本

向预留容量的结果数组追加 `R` 个 EntityId：

```text
摊还 O(R)
```

如果使用 Visitor 并在找到第一个结果后停止，实际成本只包含停止前访问的 Cell 和候选。

如果要求排序：

```text
完整排序：O(R log R)
```

只找最近一个：

```text
单次扫描：O(R)
```

找最近 `TopK`：

```text
固定大小 Heap：O(R log TopK)
```

#### 8.6.7 Box 查询总复杂度

不含结果排序：

```text
平均：O(Q + B + ΣP(e) + R)
```

由于 `R <= K <= B`，如果 Predicate 是固定成本：

```text
平均：O(Q + B)
```

在对象尺寸受控、重复次数受 `Cmax` 限制时，常简化描述为：

```text
O(Q + K)
```

但严格实现分析应使用 `B`，因为实际 CPU 确实会读取重复 Cell Item 和空间误报。

#### 8.6.8 Sphere 查询复杂度

Sphere 查询先构造包围盒，再复用 Box broad phase。对去重后的候选计算点到 AABB 距离平方：

```text
每个候选 O(1)
```

避免 `sqrt`，直接比较：

```cpp
DistanceSquared <= RadiusSquared
```

总复杂度与 Box 查询相同：

```text
平均：O(Q + B + ΣP(e) + R)
```

只是每个候选多出固定数量的浮点运算。

#### 8.6.9 为什么有时称为 O(1 + K)

只有同时满足以下条件时，才可以从业务角度近似为 `O(1 + K)`：

1. Query Radius/Extent 有固定上限。
2. CellSize 固定，因此 `Q` 有固定上限。
3. 每 Cell 对象密度有合理上限。
4. `Cmax` 限制单 Entity 的重复条目数量。
5. 不执行随世界规模增长的 Predicate。
6. 不对结果完整排序。

此时随着世界总 Entity 数 `N` 从 1 万增长到 100 万，只要查询附近的局部密度不变，`Q` 和 `B` 不会跟着 `N` 线性增长，因此查询时间基本不受全世界总对象数影响。

这并不意味着：

- 查询整个世界仍是 `O(1)`。
- 所有对象集中在一个 Cell 时仍是 `O(1)`。
- 返回 10 万个结果时仍是 `O(1)`。

任何需要输出 `R` 个结果的算法，至少有：

```text
Omega(R)
```

因为仅写出这些结果就需要 `R` 次操作。

### 8.7 最坏情况与退化条件

#### 查询整个世界

如果 QueryBox 覆盖所有 Cell：

```text
Q = H
B 约等于全部 Cell Item 数
```

复杂度接近：

```text
O(H + N * 平均Membership数)
```

#### 所有对象集中在一个 Cell

小范围查询命中该 Cell：

```text
Q = O(1)
B = N
```

总复杂度：

```text
O(N)
```

这说明 CellSize 过大或对象热点过密时，空间 Hash 无法消除局部线性扫描。

#### CellSize 过小

- `Q` 增大。
- `C(e)` 增大。
- 注册、删除、移动成本上升。
- 多 Cell 重复条目增多。

#### CellSize 过大

- `Q` 下降。
- 每 Cell Item 数量增加。
- `B` 增大。
- broad phase 误报增加。

CellSize 调优本质上是在 `Q` 与 `B/C(e)` 之间寻找平衡。

#### 哈希碰撞

若 `TMap` Hash 极端退化，单次 Cell 查找最坏可达 `O(H)`。查询理论最坏可能写成：

```text
O(Q * H + B)
```

正常实现应使用质量足够且计算便宜的 64 位坐标 Hash，并通过统计实际 Bucket 分布验证，不应把这一理论最坏情况当作日常预期。

### 8.8 数值示例

假设配置：

```text
RegularCellSize = 1000 cm
LargeCellSize   = 8000 cm
Cmax            = 64
```

执行一个半径 `1500 cm` 的 3D Sphere 查询。其 broad phase AABB 每条边长度为 `3000 cm`。Cell 坐标使用：

```text
MinCell = floor(QueryMin / CellSize)
MaxCell = floor(QueryMax / CellSize)
```

根据球心相对 Cell 边界的位置，每个轴通常覆盖 4 个 Cell，因此 RegularGrid 大约访问：

```text
Qregular = 4 * 4 * 4 = 64
```

如果使用 2D Grid：

```text
Qregular = 4 * 4 = 16
```

LargeGrid 的同一查询通常每轴覆盖 1 到 2 个 Cell。假设覆盖 `2 * 2 * 2`：

```text
Qlarge = 8
Qtotal = 64 + 8 = 72
```

假设这 72 个 Cell 中实际只有 20 个非空，每个非空 Cell 平均有 12 个 Item：

```text
B = 20 * 12 = 240
```

由于 Entity 可能跨 Cell 重复出现，QueryStamp 去重后剩余：

```text
K = 150
```

Mask 和精确 Sphere/AABB 过滤后最终命中：

```text
R = 18
```

那么这次查询主要工作量是：

```text
72 次 Cell Hash 查找
+ 240 次 Cell Item 读取和 QueryStamp 判断
+ 150 次 Entity Mask/Bounds 判断
+ 18 次结果写入
```

如果 Predicate 是固定成本，可以近似记为：

```text
O(72 + 240) = O(Q + B)
```

世界中总共有 1 万还是 100 万 Entity 并没有直接出现在这组计算中。只要这块局部区域的 Cell 密度不变，查询工作量基本不变。

再看一个删除例子。某 Entity 覆盖 6 个 Cell：

```text
C(e) = 6
```

删除时执行 6 次已知下标的 Cell Item 交换删除，以及一次 Actor 反向索引交换删除：

```text
6 * O(1) + O(1) = O(C(e))
```

不会因为每个 Cell 中有 10 个还是 1000 个其他 Item 而进行线性查找。这一点依赖 `IndexInCell` 和 `MembershipIndex` 两个回链保持正确。

移动例子：Entity 移动前覆盖 4 个 Cell，移动后覆盖 6 个 Cell：

```text
C_old = 4
C_new = 6
```

全删全加版本最多执行：

```text
4 次删除 + 6 次插入 = 10 次 Cell Membership 修改
```

复杂度：

```text
O(C_old + C_new) = O(10)
```

如果新旧集合有 3 个 Cell 相同，差集版本只有：

```text
Removed = 1
Added   = 3
D       = 4
```

实际 Hash 修改从 10 次下降为 4 次，但为了求差集仍需比较新旧 Membership，因此总体渐进复杂度仍写作 `O(C_old + C_new)`。

### 8.9 操作复杂度汇总

| 操作 | 平均/摊还复杂度 | 业务受控后的常见近似 | 主要决定因素 |
|---|---:|---:|---|
| 自定义 Bounds 注册 | `O(C(e))` | `O(1)` | Entity 覆盖 Cell 数 |
| StaticMesh Actor 注册 | `O(S + C(e))` | `O(S)` 或近似 `O(1)` | 组件数和 Cell 数 |
| 删除 Entity | `O(C(e))` | `O(1)` | Membership 数；依赖双向回链 |
| Move 回调标脏 | 平均 `O(1)` | `O(1)` | DirtySet 哈希插入 |
| 移动但未跨 Cell | `O(S)` | `O(1)` | Bounds 计算方式 |
| 移动并跨 Cell | `O(S + C_old + C_new)` | `O(1)` | 旧/新 Membership 数 |
| Box/Sphere 查询 | `O(Q + B + ΣP(e) + R)` | `O(1 + K)` | 查询 Cell、候选密度、Predicate |
| 找最近一个 | 查询成本加 `O(R)` | `O(1 + K)` | 不需要排序 |
| 返回并完整排序 | 查询成本加 `O(R log R)` | 不应称为 `O(1)` | 结果数量 |

这里的“常见近似 `O(1)`”依赖 CellSize、查询范围、对象尺寸和局部密度都被玩法约束为固定上限。工程文档、Profile 报告和 API 注释中，建议优先写完整公式，避免误导。

### 8.10 正确性不变量

算法要保持以下不变量，否则 `O(1)` 删除和查询正确性无法成立：

1. 每个有效 Entity Slot 的 Generation 与其公开 EntityId 一致。
2. 每个 Cell Item 都指向一个有效 EntityId。
3. `Cell.Items[i].MembershipIndex` 指向对应 Entity Slot 的 Membership。
4. 该 Membership 的 `IndexInCell` 必须等于 `i`。
5. Membership 的 CellKey 必须指回当前 Cell。
6. ActorToEntities 中的每个 ID 必须能反查同一 Actor。
7. Entity Slot 的 `ActorEntityListIndex` 必须指向 ActorToEntities 数组中的自己。
8. Entity 的 Bounds 变化并跨 Cell 后，旧 Membership 必须全部移除。
9. World Partition Actor EndPlay 后不能残留有效 Cell Item。
10. 查询输出前必须执行精确 Bounds/距离判断，不能把 Hash Cell 命中直接视为最终空间命中。

开发版本的 `SpatialHash.Validate` 应逐项验证这些条件。

---

### 8.11 去重策略

多 Cell 插入意味着一个 Entity 可能被多个查询 Cell 找到。

高频游戏线程查询推荐 QueryStamp：

```cpp
uint32 CurrentQueryStamp = 1;
```

每次查询：

```cpp
const uint32 Stamp = ++CurrentQueryStamp;

if (Slot.LastQueryStamp == Stamp)
{
    continue;
}
Slot.LastQueryStamp = Stamp;
```

溢出到 0 时清空所有 `LastQueryStamp` 并重新从 1 开始。

注意：这种写 Slot 的方式不是并发只读。工作线程查询应使用：

- 每个查询独立的 BitSet/Scratch Set；或
- 不可变查询快照中的局部去重缓冲区。

不要为了去重在热路径频繁创建临时 `TSet<FSpatialEntityId>`。可以使用可复用 Scratch Buffer。

---

## 9. 分类位掩码

高频 Gameplay 分类使用一个 `uint64`：

```cpp
uint64 CategoryMask;
```

示例：

```cpp
namespace SpatialCategory
{
    constexpr uint64 StaticMesh   = 1ull << 0;
    constexpr uint64 SmartObject  = 1ull << 1;
    constexpr uint64 Interactable = 1ull << 2;
    constexpr uint64 Resource     = 1ull << 3;
    constexpr uint64 Cover        = 1ull << 4;
    constexpr uint64 Enemy        = 1ull << 5;
}

```

过滤：

```cpp
if (Query.RequiredAnyCategory != 0
    && (Slot.CategoryMask & Query.RequiredAnyCategory) == 0)
{
    continue;
}

if ((Slot.CategoryMask & Query.RequiredAllCategory)
    != Query.RequiredAllCategory)
{
    continue;
}

if ((Slot.CategoryMask & Query.ExcludedCategory) != 0)
{
    continue;
}
```

GameplayTag、Class、接口判断和任意 Predicate 应放在空间和位掩码之后，避免对大量候选执行 UObject 层逻辑。

---

## 10. StaticMesh 白名单自动注册

### 10.1 配置

```cpp
UCLASS(Config=Game, DefaultConfig)
class UGameSpatialIndexSettings : public UDeveloperSettings
{
    GENERATED_BODY()

public:
    UPROPERTY(Config, EditAnywhere, Category="Static Mesh")
    TArray<TSoftObjectPtr<UStaticMesh>> IndexedStaticMeshes;

    UPROPERTY(Config, EditAnywhere, Category="Grid", meta=(ClampMin="1.0"))
    double CellSize = 1000.0;

    UPROPERTY(Config, EditAnywhere, Category="Grid", meta=(ClampMin="1.0"))
    double LargeCellSize = 8000.0;

    UPROPERTY(Config, EditAnywhere, Category="Grid", meta=(ClampMin="1"))
    int32 MaxCellsPerEntity = 64;
};
```

初始化时将数组转换为快速查找集合：

```cpp
TSet<FSoftObjectPath> IndexedStaticMeshPaths;
```

判断组件资产：

```cpp
const UStaticMesh* Mesh = StaticMeshComponent->GetStaticMesh();
const bool bIndexed = Mesh
    && IndexedStaticMeshPaths.Contains(FSoftObjectPath(Mesh));
```

使用 `TSoftObjectPtr` 的原因是配置本身不应强制加载所有 StaticMesh 资产。Actor 已经加载后，它引用的 Mesh 通常已经可用。

### 10.2 Actor 判定

收到新 Actor 时：

1. 忽略 CDO、Template、PendingKill 和其他 World 的 Actor。
2. 收集 `UStaticMeshComponent`。
3. 判断组件的 Mesh 是否位于白名单。
4. 计算所有命中组件的 World Bounds 并集。
5. 以 `StaticMeshActor` Source 注册 Entity。

默认推荐“一 Actor 一 Entity，Bounds 为命中组件并集”，优点是条目少，查询直接返回 Actor。

如果一个 Actor 中组件分布范围很大，改为“一命中组件一 Entity”。此时 Actor 反向索引天然是一对多。

### 10.3 不使用附加组件

索引系统不会给 Actor 创建 Marker Component。识别条件只来自：

- StaticMesh 资产白名单。
- SmartObjectSubsystem 的空间分区 Add/Remove。
- 业务主动调用 `RegisterActor()`。

索引元数据全部位于 Subsystem 的连续数组中。

---

## 11. SmartObject 接入

UE 5.8 SmartObject 默认已经使用：

```cpp
THierarchicalHashGrid2D<2, 4, FSmartObjectHandle>
```

其查询入口是：

```cpp
USmartObjectSubsystem::FindSmartObjects
    -> SpacePartition->Find(QueryBox)
    -> SmartObject Handle 候选
    -> 精确 QueryBox 检查
    -> Slot、Tag、Behavior、Condition、Claim 过滤
```

若希望 SmartObject 与 StaticMesh、自定义 Actor 共用本方案，不应修改 `FindSmartObjects()` 主流程，而应实现空间分区适配器。

### 11.1 适配器

```cpp
USTRUCT()
struct FProjectSmartObjectSpatialEntryData
    : public FSmartObjectSpatialEntryData
{
    GENERATED_BODY()

    FSpatialEntityId EntityId;
};

UCLASS()
class UProjectSmartObjectSpacePartition
    : public USmartObjectSpacePartition
{
    GENERATED_BODY()

protected:
    virtual void SetBounds(const FBox& Bounds) override;

    virtual void Add(
        FSmartObjectHandle Handle,
        const FBox& Bounds,
        FInstancedStruct& OutEntryData) override;

    virtual void Remove(
        FSmartObjectHandle Handle,
        FStructView EntryData) override;

    virtual void Find(
        const FBox& QueryBox,
        TArray<FSmartObjectHandle>& OutResults) override;
};
```

Add：

```cpp
void UProjectSmartObjectSpacePartition::Add(
    FSmartObjectHandle Handle,
    const FBox& Bounds,
    FInstancedStruct& OutEntryData)
{
    UUniformSpatialHashSubsystem* Index =
        GetWorld()->GetSubsystem<UUniformSpatialHashSubsystem>();

    FSpatialRegistrationParams Params;
    Params.Source = ESpatialEntitySource::SmartObject;
    Params.CategoryMask = SpatialCategory::SmartObject;
    Params.Bounds = Bounds;
    Params.SmartObjectHandle = Handle;

    FProjectSmartObjectSpatialEntryData Data;
    Data.EntityId = Index->RegisterEntity(Params);
    OutEntryData.InitializeAs(Data);
}
```

Remove：

```cpp
void UProjectSmartObjectSpacePartition::Remove(
    FSmartObjectHandle Handle,
    FStructView EntryData)
{
    const FProjectSmartObjectSpatialEntryData& Data =
        EntryData.Get<FProjectSmartObjectSpatialEntryData>();

    GetWorld()
        ->GetSubsystem<UUniformSpatialHashSubsystem>()
        ->UnregisterEntity(Data.EntityId);
}
```

Find：

```cpp
void UProjectSmartObjectSpacePartition::Find(
    const FBox& QueryBox,
    TArray<FSmartObjectHandle>& OutResults)
{
    FSpatialQuery Query;
    Query.Bounds = QueryBox;
    Query.RequiredAnyCategory = SpatialCategory::SmartObject;

    GetWorld()->GetSubsystem<UUniformSpatialHashSubsystem>()
        ->QueryEntities(Query,
            [&OutResults](const FSpatialEntityView& Entity)
            {
                OutResults.Add(Entity.SmartObjectHandle);
                return ESpatialQueryControl::Continue;
            });
}
```

项目配置：

```ini
[/Script/SmartObjectsModule.SmartObjectSubsystem]
SpacePartitionClassName=/Script/YourGame.ProjectSmartObjectSpacePartition
```

### 11.2 SmartObject Actor 映射

`USmartObjectSpacePartition::Add()` 只保证提供 Handle 和 Bounds，不保证直接提供 Owner Actor。因此 SmartObject Entity 的核心身份应是 `FSmartObjectHandle`，Actor 弱引用是可选关联数据。

如果能够从 SmartObject Runtime/Component 得到 Owner Actor，可以调用：

```cpp
Index->AttachActor(EntityId, OwnerActor);
```

无法得到时也不影响 `FindSmartObjects()`，因为其后续过滤根据 Handle 访问 SmartObject Runtime。

不要为了建立 Actor 映射而强制 Spawn World Partition 中 dehydrated 或未加载的 SmartObject Actor。

### 11.3 SmartObject 移动

SmartObjectSubsystem 更新 Transform 时本身会执行：

```text
SpacePartition->Remove(old entry)
SpacePartition->Add(new bounds)
```

因此通过 SpacePartition 适配器接入后，不需要再对同一个 SmartObject Entity 绑定 Actor Move 回调，否则会产生重复更新。

### 11.4 SmartObject 静态点快速通道

本实现默认把 SmartObject 表示为 Bounds 中心处的无体积静态点：

```text
Point = Bounds.GetCenter()
bIsPointEntity = true
bTrackMovement = false
```

点实体只进入一个 Regular Cell，不分配多 Cell Membership，不进入 Dirty 更新队列。由于点不会重复出现在多个 Cell，查询时也跳过 `LastQueryStamp` 的读写；Box 查询执行 `QueryBox.IsInsideOrOn(Point)`，Sphere 查询执行一次点距离平方比较。注册和删除平均为 `O(1)`，存续期间没有移动更新成本。

该模式的语义是“中心点位于查询范围”，而不是“SmartObject Bounds 与查询范围相交”。必须保留 Bounds 相交语义时开启：

```ini
[/Script/GMS_Demo.UniformSpatialHashSettings]
bCalculateSmartObjectBounds=True
```

SpacePartition 没有单独的移动接口；SmartObjectSubsystem 发生运行时变换时，通过 Remove 旧条目并 Add 新 Bounds 建立新点，无需绑定 Actor Move 回调。

---

## 12. 自定义 Actor 注册

用户侧 API：

```cpp
USTRUCT(BlueprintType)
struct FSpatialActorRegistrationParams
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    uint64 CategoryMask = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bTrackMovement = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    bool bUseActorBounds = true;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    FVector CustomExtent = FVector::ZeroVector;
};
```

注册：

```cpp
UUniformSpatialHashSubsystem* SpatialIndex =
    World->GetSubsystem<UUniformSpatialHashSubsystem>();

FSpatialActorRegistrationParams Params;
Params.CategoryMask = SpatialCategory::Interactable
                    | SpatialCategory::Resource;

const FSpatialEntityId EntityId =
    SpatialIndex->RegisterActor(Actor, Params);
```

注销：

```cpp
SpatialIndex->UnregisterEntity(EntityId);
```

或者注销 Actor 对应的所有 Entity：

```cpp
SpatialIndex->UnregisterActor(Actor);
```

Bounds 由业务决定时：

```cpp
SpatialIndex->UpdateEntityBounds(EntityId, NewBounds);
```

---

## 13. Actor 生命周期监听

### 13.1 Subsystem 初始化

```cpp
void UUniformSpatialHashSubsystem::Initialize(FSubsystemCollectionBase& Collection)
{
    Super::Initialize(Collection);

    UWorld* World = GetWorld();
    ActorSpawnedHandle = World->AddOnActorSpawnedHandler(
        FOnActorSpawned::FDelegate::CreateUObject(
            this,
            &ThisClass::HandleActorSpawned));

    LevelAddedHandle = FWorldDelegates::LevelAddedToWorld.AddUObject(
        this,
        &ThisClass::HandleLevelAdded);

    PreLevelRemovedHandle = FWorldDelegates::PreLevelRemovedFromWorld.AddUObject(
        this,
        &ThisClass::HandlePreLevelRemoved);

    WorldOriginOffsetHandle = FCoreDelegates::PostWorldOriginOffset.AddUObject(
        this,
        &ThisClass::HandlePostWorldOriginOffset);

    BootstrapExistingActors();
}
```

`BootstrapExistingActors()` 用于处理 Subsystem 初始化前已经存在的 Actor。只在初始化和 Level 加载时扫描，不每帧调用。

### 13.2 Spawn

`UWorld::AddOnActorSpawnedHandler` 能捕获运行时 Spawn 的 Actor。回调中不必立即扫描组件和写 Hash，可以先排队：

```cpp
void HandleActorSpawned(AActor* Actor)
{
    if (Actor && Actor->GetWorld() == GetWorld())
    {
        PendingRegisterActors.Add(Actor);
    }
}
```

下一次 Tick 统一处理，优点是：

- 合并短时间大量 Spawn。
- 避免在 Actor 初始化链中做较重工作。
- Actor 如果同帧被销毁，可以直接跳过。

### 13.3 EndPlay 和 Destroyed

只监听 `OnDestroyed` 不够。World Partition 卸载、关卡切换和 PIE 结束会触发 EndPlay，但语义不一定等同于业务主动 Destroy。

对已索引 Actor 绑定：

```cpp
Actor->OnEndPlay.AddDynamic(this, &ThisClass::HandleActorEndPlay);
```

处理所有 EndPlayReason：

```cpp
void HandleActorEndPlay(AActor* Actor, EEndPlayReason::Type Reason)
{
    UnregisterActor(Actor);
}
```

可以同时监听 `OnDestroyed` 作为防御性补充，但注销必须幂等。

由于 `OnEndPlay`/`OnDestroyed` 是 Sparse Dynamic Multicast Delegate，只有真正注册的 Actor 才产生对应绑定数据。这里只对入索引 Actor 绑定，不是对世界全部 Actor 绑定。

### 13.4 Deinitialize

必须移除所有全局 Delegate：

```cpp
void UUniformSpatialHashSubsystem::Deinitialize()
{
    if (UWorld* World = GetWorld())
    {
        World->RemoveOnActorSpawnedHandler(ActorSpawnedHandle);
    }

    FWorldDelegates::LevelAddedToWorld.Remove(LevelAddedHandle);
    FWorldDelegates::PreLevelRemovedFromWorld.Remove(PreLevelRemovedHandle);
    FCoreDelegates::PostWorldOriginOffset.Remove(WorldOriginOffsetHandle);

    ClearIndex();
    Super::Deinitialize();
}
```

---

## 14. Actor Move 回调

### 14.1 运行时可靠方式

不要依赖 `GEngine->OnActorMoved()` 作为运行时移动通知。它主要用于编辑器移动广播，普通运行时 `SetActorLocation()` 并不保证走这个事件。

运行时应绑定 Actor 已有 RootComponent 的：

```cpp
USceneComponent::TransformUpdated
```

UE 5.8 声明：

```cpp
DECLARE_EVENT_ThreeParams(
    USceneComponent,
    FTransformUpdated,
    USceneComponent*,
    EUpdateTransformFlags,
    ETeleportType);
```

绑定：

```cpp
if (USceneComponent* Root = Actor->GetRootComponent())
{
    Root->TransformUpdated.AddUObject(
        this,
        &ThisClass::HandleComponentTransformUpdated);

    Slot.BoundRootComponent = Root;
}
```

这不会新增组件，只是在已经存在的 RootComponent Delegate 上注册监听。

### 14.2 回调只标脏

不要在每次 TransformUpdated 中立刻 Remove + Add。CharacterMovement、附着层级和物理移动可能在一帧内多次更新 Transform。

推荐：

```cpp
void HandleComponentTransformUpdated(
    USceneComponent* Component,
    EUpdateTransformFlags,
    ETeleportType)
{
    if (AActor* Actor = Component ? Component->GetOwner() : nullptr)
    {
        MarkActorDirty(Actor);
    }
}
```

DirtySet：

```cpp
TSet<FSpatialEntityId> DirtyEntities;
```

Tick：

```cpp
for (FSpatialEntityId Id : DirtyEntities)
{
    if (!IsValidEntity(Id))
    {
        continue;
    }

    const FBox NewBounds = CalculateEntityBounds(Id);
    UpdateEntityBoundsInternal(Id, NewBounds);
}
DirtyEntities.Reset();
```

这样同一 Entity 一帧移动 10 次，Hash 最多更新 1 次。

### 14.3 只在跨 Cell 时更新

重新计算新 Bounds 覆盖的 Cell 范围。如果新旧 Cell 集合完全相同，只更新 Slot.Bounds，不执行 Cell 删除和插入：

```cpp
if (OldCellRange == NewCellRange)
{
    Slot.Bounds = NewBounds;
    return;
}
```

对于小范围移动，这是最重要的优化之一。

### 14.4 RootComponent 更换

Actor 运行时更换 RootComponent 后，旧 Delegate 仍绑定旧 Root。更新阶段应检查：

```cpp
if (Slot.BoundRootComponent.Get() != Actor->GetRootComponent())
{
    RebindRootTransformDelegate(Actor, Slot);
}
```

如果业务代码频繁更换 Root，建议在更换后显式调用：

```cpp
SpatialIndex->NotifyActorRootChanged(Actor);
```

### 14.5 Component 内部变化

Root Transform 不变时，以下变化不一定触发 Root 的 TransformUpdated：

- 子组件修改 RelativeTransform。
- StaticMeshComponent 更换 Mesh。
- 组件运行时注册或注销。
- Bounds 受动画或其他内部状态影响。

这类 Actor 应由业务显式调用：

```cpp
SpatialIndex->MarkActorBoundsDirty(Actor);
```

不要为了覆盖少量特殊 Actor 而给所有 Actor 的所有组件绑定回调。

---

## 15. World Partition 支持

### 15.1 核心原则

索引只保存当前已加载 Entity：

- Runtime Cell 加载：Actor 注册。
- Runtime Cell 卸载：Actor 注销。
- 不使用强引用保持 Actor 常驻。
- 不主动解析未加载 Actor Descriptor 为 Actor。
- 不把 Hash Cell 与 World Partition Runtime Cell 当成同一种 Cell。

World Partition Cell 负责流送；Uniform Spatial Hash Cell 负责运行时查询。两者 CellSize、坐标和生命周期可以完全不同。

### 15.2 加载

建议同时保留两条路径：

1. `UWorld::AddOnActorSpawnedHandler` 捕获加载时实例化的 Actor。
2. `FWorldDelegates::LevelAddedToWorld` 扫描该 Level 中已有 Actor，作为初始化顺序和特殊加载路径的补充。

Level 回调必须过滤 World：

```cpp
void HandleLevelAdded(ULevel* Level, UWorld* World)
{
    if (World != GetWorld() || !Level)
    {
        return;
    }

    for (AActor* Actor : Level->Actors)
    {
        QueueActorForRegistration(Actor);
    }
}
```

注册必须幂等。ActorSpawned 和 LevelAdded 同时看到同一 Actor 时不能产生重复 StaticMesh Entity。

### 15.3 卸载

主要依赖 Actor `OnEndPlay` 注销。再使用 `PreLevelRemovedFromWorld` 提前清理该 Level 的 Actor：

```cpp
void HandlePreLevelRemoved(ULevel* Level, UWorld* World)
{
    if (World != GetWorld() || !Level)
    {
        return;
    }

    for (AActor* Actor : Level->Actors)
    {
        UnregisterActor(Actor);
    }
}
```

`UnregisterActor()` 必须幂等，因为之后仍可能收到 EndPlay/Destroyed。

### 15.4 World Origin Rebase

若项目启用 World Origin Rebasing，监听：

```cpp
FCoreDelegates::PostWorldOriginOffset
```

最简单可靠的处理是重建所有已加载 Entity 的 Cell Membership：

```text
PostWorldOriginOffset
    -> 清空 RegularGrid/LargeGrid Cell
    -> 保留 Entity Slot 和 Actor 映射
    -> 使用更新后的 Actor/SmartObject Bounds 重新插入
```

Origin Rebase 通常不是每帧发生，整体重建比尝试平移大量哈希 Cell Key 更容易保证正确性。

### 15.5 Data Layer

Data Layer 激活导致 Actor 加载时正常注册，停用并卸载时正常注销。若 Actor 仍加载但不应参与空间查询，应注销 Entity；也可以由业务维护专用 Category 位并通过查询过滤：

```cpp
SpatialIndex->UnregisterEntity(EntityId);
```

---

## 16. 查询 API

### 16.1 查询参数

```cpp
USTRUCT(BlueprintType)
struct FSpatialQueryFilter
{
    GENERATED_BODY()

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    uint64 RequiredAnyCategory = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    uint64 RequiredAllCategory = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    uint64 ExcludedCategory = 0;

    UPROPERTY(EditAnywhere, BlueprintReadWrite)
    int32 MaxResults = 0; // 0 表示不限制

};
```

内部查询描述：

```cpp
struct FSpatialQuery
{
    FBox Bounds;
    FSpatialQueryFilter Filter;

    bool bExactSphere = false;
    FVector SphereCenter = FVector::ZeroVector;
    double SphereRadiusSq = 0.0;
};
```

### 16.2 Box 查询

```cpp
void QueryBox(
    const FBox& Box,
    const FSpatialQueryFilter& Filter,
    TArray<FSpatialEntityId>& OutEntities) const;
```

流程：

1. Box 转换为 Cell 范围。
2. 遍历 RegularGrid 和 LargeGrid 对应 Cell。
3. QueryStamp 去重。
4. 校验 EntityId。
5. 位掩码过滤。
6. `Slot.Bounds.Intersect(QueryBox)` 精确 AABB 过滤。
7. 输出 EntityId。

### 16.3 Sphere 查询

```cpp
void QuerySphere(
    const FVector& Center,
    double Radius,
    const FSpatialQueryFilter& Filter,
    TArray<FSpatialEntityId>& OutEntities) const;
```

先用球体 AABB 查询 Cell：

```cpp
FBox QueryBounds(Center - FVector(Radius), Center + FVector(Radius));
```

再执行精确距离判断。按需求选择：

- Actor Pivot 到 Center 的距离。
- Bounds Center 到 Center 的距离。
- 点到 AABB 的最短距离。

推荐点到 AABB：

```cpp
const double DistSq = Slot.Bounds.ComputeSquaredDistanceToPoint(Center);
if (DistSq > Radius * Radius)
{
    continue;
}
```

### 16.4 回调式查询

为了避免先生成大数组再二次处理，提供回调式 API：

```cpp
enum class ESpatialQueryControl : uint8
{
    Continue,
    Stop
};

template<typename VisitorType>
void ForEachEntityInSphere(
    const FVector& Center,
    double Radius,
    const FSpatialQueryFilter& Filter,
    VisitorType&& Visitor);
```

用户可以找到第一个结果后立即停止：

```cpp
FSpatialEntityId BestId;

SpatialIndex->ForEachEntityInSphere(
    Origin,
    1500.0,
    Filter,
    [&BestId](const FSpatialEntityView& Entity)
    {
        BestId = Entity.Id;
        return ESpatialQueryControl::Stop;
    });
```

### 16.5 查询返回值

热路径优先返回 EntityId，而不是 Actor 数组：

```cpp
TArray<FSpatialEntityId> Results;
```

确实需要 Actor 时：

```cpp
for (FSpatialEntityId Id : Results)
{
    if (AActor* Actor = SpatialIndex->ResolveActor(Id))
    {
        // 使用 Actor
    }
}
```

SmartObject 查询返回 Handle：

```cpp
FSmartObjectHandle Handle = SpatialIndex->GetSmartObjectHandle(Id);
```

---

## 17. 用户使用示例

### 17.1 查询附近指定 StaticMesh Actor

先在 `UGameSpatialIndexSettings::IndexedStaticMeshes` 配置资产。Actor 加载或 Spawn 后自动进入索引。

```cpp
FSpatialQueryFilter Filter;
Filter.SourceMask = static_cast<uint32>(ESpatialSourceFlags::StaticMeshActor);
Filter.RequiredAnyCategory = SpatialCategory::StaticMesh;

TArray<FSpatialEntityId> Results;
SpatialIndex->QuerySphere(PlayerLocation, 2000.0, Filter, Results);

for (FSpatialEntityId Id : Results)
{
    AActor* Actor = SpatialIndex->ResolveActor(Id);
    if (!Actor)
    {
        continue;
    }

    // 业务处理
}
```

### 17.2 查询附近资源点

```cpp
FSpatialQueryFilter Filter;
Filter.RequiredAllCategory = SpatialCategory::Interactable
                           | SpatialCategory::Resource;

SpatialIndex->QuerySphere(Location, 3000.0, Filter, Results);
```

### 17.3 主动注册自定义 Actor

```cpp
FSpatialActorRegistrationParams Params;
Params.CategoryMask = SpatialCategory::Enemy;
Params.bTrackMovement = true;

FSpatialEntityId Id = SpatialIndex->RegisterActor(EnemyActor, Params);
```

Actor 移动和 EndPlay 由 Subsystem 自动处理。

### 17.4 找到最近对象

Hash 查询结果默认不保证距离排序。不要为了拿最近一个而对全部结果完整排序，使用单次扫描：

```cpp
FSpatialEntityId BestId;
double BestDistSq = TNumericLimits<double>::Max();

SpatialIndex->ForEachEntityInSphere(
    Origin,
    Radius,
    Filter,
    [&](const FSpatialEntityView& Entity)
    {
        const double DistSq = Entity.Bounds.ComputeSquaredDistanceToPoint(Origin);
        if (DistSq < BestDistSq)
        {
            BestDistSq = DistSq;
            BestId = Entity.Id;
        }
        return ESpatialQueryControl::Continue;
    });
```

### 17.5 SmartObject 查询

AI 仍使用引擎原 API：

```cpp
FSmartObjectRequest Request(QueryBox, SmartObjectFilter);
TArray<FSmartObjectRequestResult> Results;

SmartObjectSubsystem->FindSmartObjects(Request, Results, UserActor);
```

项目配置将 `SpacePartitionClassName` 指向适配器后，这个 API 内部自动使用统一 Spatial Hash，AI 侧不需要修改。

---

## 18. 更新顺序

推荐每帧顺序：

```text
1. 处理 PendingUnregister
2. 处理 PendingRegisterActors
3. 处理 DirtyEntities
4. 发布可选只读 Snapshot
5. 执行批量查询
```

先删除再注册，避免同一 Actor 在卸载/重新加载边界留下重复记录。

Subsystem TickGroup 建议位于主要 Actor Movement 更新之后、AI 查询之前。若无法保证所有调用顺序，提供显式：

```cpp
SpatialIndex->FlushPendingUpdates();
```

在关键批量查询前刷新脏数据。不要在每个单次查询中无条件 Flush，否则会破坏批处理优势。

---

## 19. 线程模型

### 19.1 基础版本

推荐第一版：

- 注册、删除、移动更新：游戏线程。
- 查询：游戏线程。
- Query 回调中禁止修改索引，修改操作排队到查询完成后。

这样无需锁，逻辑最简单，通常已经足够快。

### 19.2 工作线程查询

Actor/UObject 不能被普通工作线程随意访问。需要异步查询时，使用只读 POD Snapshot：

```cpp
struct FSpatialQueryRecord
{
    FSpatialEntityId Id;
    FBox Bounds;
    uint64 CategoryMask;
    FSmartObjectHandle SmartObjectHandle;
};
```

Snapshot 不保存需要在线程中解引用的 Actor 指针。工作线程返回 EntityId，游戏线程再校验 Generation 并解析 Actor。

可选实现：

- 双缓冲 Snapshot。
- Tick 末发布新 Snapshot。
- Worker 持有 `TSharedPtr<const FSpatialSnapshot>`。
- 旧 Snapshot 无查询引用后自动释放。

不要在高频查询路径给整个可变 Grid 加一把粗粒度互斥锁，这容易让移动更新和 AI 查询互相阻塞。

---

## 20. CellSize 选择

CellSize 没有全项目通用值。经验目标是：

- 常见查询半径覆盖少量 Cell。
- 单个 Cell 中候选数量不过多。
- 常见 Entity 覆盖 Cell 数接近 1 到 8。

粗略建议：

| 玩法 | 初始 CellSize 建议 |
|---|---:|
| 室内交互物、近距离 AI | 500 到 1000 cm |
| 开放世界资源、普通 AI 感知 | 1000 到 3000 cm |
| 大范围战略单位 | 5000 cm 以上 |

如果典型查询半径为 `R`，可以先从 `CellSize ≈ R` 或 `CellSize ≈ R / 2` 测试。

必须用项目真实分布做 Profile，关注：

- 每 Cell 平均和 P95 Entity 数量。
- 每 Entity 平均和 P95 Membership 数量。
- 每次查询访问 Cell 数。
- broad phase 候选数与最终命中数之比。
- Dirty Entity 每帧数量。
- LargeGrid Entity 数量。

---

## 21. 性能风险与优化顺序

### 21.1 不要每帧扫描全部 Actor

使用 Spawn、EndPlay、Level 和 Transform Delegate 驱动。Tick 只处理队列。

### 21.2 不要在 Move 回调立即更新 Hash

只加入 DirtySet，每帧合并更新。

### 21.3 不要在查询中做大量 UObject 判断

先 Cell，再位掩码，再 Bounds/距离，最后才是 Class、GameplayTag、接口和 Predicate。

### 21.4 不要让大对象污染普通 Grid

超过 `MaxCellsPerEntity` 后进入 LargeGrid。

### 21.5 不要默认排序结果

Hash 结果无序。最近对象用单次扫描，Top-K 用固定大小 Heap。

### 21.6 不要保存 Actor 强引用

使用 `TWeakObjectPtr<AActor>`，否则 World Partition 可能无法正常回收对象。

### 21.7 不要把 EntityId 当永久存档 ID

运行时 `Index + Generation` 只在当前 Subsystem 生命周期内有效。跨存档、网络长期标识应使用业务 GUID、Actor GUID 或其他稳定 ID。

### 21.8 避免查询期间直接增删

Visitor 内的注册/删除放入 Pending Mutation Queue，防止 Cell `TArray` 迭代失效。

---

## 22. 与 Chaos Overlap 的建议测试

不要只做空场景微基准。使用真实玩法数据比较：

### 22.1 测试场景

- 1 万、10 万、50 万个已加载业务对象。
- 不同对象密度和不同 CellSize。
- 查询半径 500、2000、10000 cm。
- 每帧 100、1000、10000 次查询。
- 0%、1%、10% Entity 每帧移动。
- StaticMesh、SmartObject、自定义 Actor 混合。

### 22.2 对比项目

Chaos：

```cpp
OverlapMultiByObjectType
    -> FOverlapResult
    -> Actor 去重
    -> Class/Asset/Tag 业务过滤
```

Spatial Hash：

```cpp
QuerySphere/QueryBox
    -> EntityId
    -> Mask + Bounds/距离过滤
```

### 22.3 记录指标

- GameThread 总耗时。
- 单查询平均、P95、P99 耗时。
- 候选数量和最终结果数量。
- 临时内存分配次数。
- Hash Update 时间。
- Actor Spawn/Destroy 峰值。
- World Partition Cell 加载/卸载峰值。
- Chaos Scene Query 时间。

Chaos 结果包含精确物理语义，Spatial Hash 结果是业务 broad phase，两者必须在同等业务目标下比较。若最终仍要求精确碰撞，应比较：

```text
纯 Chaos 全量 Overlap
vs
Spatial Hash 粗筛 + 少量 Chaos 精查
```

---

## 23. Debug 与验证工具

建议提供 Console Command：

```text
SpatialHash.Draw 0/1
SpatialHash.DrawCell x y z
SpatialHash.DrawEntity EntityId
SpatialHash.DumpStats
SpatialHash.Validate
SpatialHash.Rebuild
```

`Validate` 应检查：

- 所有 Cell Item 的 EntityId 有效。
- Cell Item 的 MembershipIndex 有效。
- Membership 的 Cell 和 IndexInCell 能反查同一 EntityId。
- ActorToEntities 中的 ID 有效且 Actor 一致。
- Entity Slot 中 Actor 存在时，ActorToEntities 包含该 ID。
- 未分配 Slot 没有 Membership。
- RegularGrid 中没有超过阈值的 Entity。

开发版本可抽样对照 Chaos 或世界 Actor 扫描，验证查询没有漏项。

---

## 24. 推荐实施阶段

### 阶段一：核心索引

- EntityId + Generation。
- RegularGrid + LargeGrid。
- StaticMesh 白名单。
- 自定义 Actor Register/Unregister。
- Spawn、EndPlay、Root TransformUpdated。
- 游戏线程 Box/Sphere 查询。
- Debug Draw 和 Validate。

### 阶段二：World Partition

- LevelAdded/PreLevelRemoved。
- Bootstrap Existing Actors。
- World Origin Rebase 重建。
- 大量 Cell 加载/卸载压力测试。

### 阶段三：SmartObject

- `USmartObjectSpacePartition` 适配器。
- SmartObject Handle Payload。
- 验证原生 `FindSmartObjects()` 的所有 Slot Filter 行为不变。
- 对比原生 `USmartObjectHashGrid` 的性能。

### 阶段四：并发和批量查询

- 查询 Scratch Buffer 池。
- 异步不可变 Snapshot。
- Mass/AI 批量查询接口。
- Profile 后再决定是否增加 Cell CombinedMask、SIMD 或 SoA。

---

## 25. UE 5.8 源码依据

以下路径基于本地 UE 5.8 源码：

### SmartObject 默认 HashGrid

```text
Engine/Plugins/Runtime/SmartObjects/Config/DefaultSmartObjects.ini:2
Engine/Plugins/Runtime/SmartObjects/Source/SmartObjectsModule/Public/SmartObjectHashGrid.h:16
Engine/Plugins/Runtime/SmartObjects/Source/SmartObjectsModule/Private/SmartObjectHashGrid.cpp:9
```

默认配置：

```ini
SpacePartitionClassName=/Script/SmartObjectsModule.SmartObjectHashGrid
```

### SmartObject 空间分区接口

```text
Engine/Plugins/Runtime/SmartObjects/Source/SmartObjectsModule/Public/SmartObjectTypes.h:321
```

接口：

```cpp
SetBounds
Add
Remove
Find
```

### SmartObject 查询调用链

```text
Engine/Plugins/Runtime/SmartObjects/Source/SmartObjectsModule/Private/SmartObjectSubsystem.cpp:2848
Engine/Plugins/Runtime/SmartObjects/Source/SmartObjectsModule/Private/EnvQueryGenerator_SmartObjects.cpp:45
```

### SmartObject Slot/Entrance Chaos Overlap

```text
Engine/Plugins/Runtime/SmartObjects/Source/SmartObjectsModule/Private/Annotations/SmartObjectSlotEntranceAnnotation.cpp:109
```

### Actor Spawn 回调

```text
Engine/Source/Runtime/Engine/Classes/Engine/World.h:3055
Engine/Source/Runtime/Engine/Private/World.cpp:6681
```

### Actor EndPlay/Destroyed

```text
Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h:207
Engine/Source/Runtime/Engine/Classes/GameFramework/Actor.h:2335
Engine/Source/Runtime/Engine/Private/Actor.cpp:3272
```

### RootComponent TransformUpdated

```text
Engine/Source/Runtime/Engine/Classes/Components/SceneComponent.h:80
Engine/Source/Runtime/Engine/Classes/Components/SceneComponent.h:331
Engine/Source/Runtime/Engine/Private/Components/SceneComponent.cpp:1006
```

### Level 加载/卸载

```text
Engine/Source/Runtime/Engine/Classes/Engine/World.h:4591
Engine/Source/Runtime/Engine/Private/World.cpp:4039
Engine/Source/Runtime/Engine/Private/World.cpp:4133
```

### World Origin Offset

```text
Engine/Source/Runtime/Engine/Private/World.cpp:8091
Engine/Source/Runtime/Engine/Private/World.cpp:8174
```

---

## 26. 最终推荐

该方案最适合以下查询：

- 附近指定 StaticMesh 资产对应的 Actor。
- 附近 SmartObject。
- 附近可交互物、资源点、掩体点、单位或业务 Marker。
- 高频 AI/Mass 局部候选检索。

最终建议保持两套系统职责清晰：

```text
Uniform Spatial Hash
    = 已加载业务 Entity 的高速语义空间索引

Chaos Scene Query
    = 真实物理世界的精确碰撞与几何查询
```

先用 Uniform Spatial Hash 将候选缩小到局部常数规模，再按需要执行少量 Chaos 检查，通常能得到比“所有业务查询都直接 Overlap”更稳定、更容易控制的性能。

---

## 27. FastGeo 接入

### 27.1 UE 5.8 源码结论

FastGeo World Partition Transformer 会把每个可转换 `UActorComponent` 复制成 `FFastGeoComponent`，然后从 Actor 移除原组件。完全转换的 Actor 会从 `ULevel::Actors` 删除。普通 Actor 的 FastGeo 组件会合并到 Level ComponentCluster；HLOD 使用独立 Cluster。因此运行时不能从 FastGeo Component 稳定恢复原 Actor，也不能继续依赖 Actor 生命周期或 `RootComponent::TransformUpdated`。

关键源码：

```text
Engine/Plugins/Experimental/FastGeoStreaming/Source/FastGeoStreaming/Public/FastGeoWorldPartitionRuntimeCellTransformer.h
Engine/Plugins/Experimental/FastGeoStreaming/Source/FastGeoStreaming/Internal/FastGeoContainer.h
Engine/Plugins/Experimental/FastGeoStreaming/Source/FastGeoStreaming/Internal/FastGeoComponentCluster.h
Engine/Plugins/Experimental/FastGeoStreaming/Source/FastGeoStreaming/Internal/FastGeoComponent.h
Engine/Plugins/Experimental/FastGeoStreaming/Source/FastGeoStreaming/Internal/FastGeoPrimitiveComponent.h
Engine/Plugins/Experimental/FastGeoStreaming/Source/FastGeoStreaming/Internal/FastGeoWorldSubsystem.h
```

项目当前只支持 UE 5.8，直接使用 FastGeoStreaming 模块的 `Internal` 头文件以及 `UFastGeoContainer::GetOnRegistered()`、`GetOnUnregistered()` 生命周期委托，不保留旧引擎版本的路径或条件编译回退。

### 27.2 索引粒度与身份

索引按 `FFastGeoPrimitiveComponent` 建 Entity，而不是把整个 Level Container 合并为一个大 Bounds。这样查询不会因为一个 World Partition Cell 内的任意 Primitive 命中而返回整个 Cell。

标准 FastGeo 数据不保留普通 Actor 到组件的分组。因此系统不以 Level 或 Container 为注册粒度，也不会扫描并自动注册 Container 内的全部 Primitive。业务持有或找到目标组件后，只把需要参与业务查询的 `FFastGeoPrimitiveComponent` 逐个注册，并可为每个组件指定独立的 CategoryMask。

注册接口接收基类 `FFastGeoPrimitiveComponent*`，所以可以直接传入：

```text
FFastGeoStaticMeshComponent*
FFastGeoInstancedStaticMeshComponent*
其他派生自 FFastGeoPrimitiveComponent 的 FastGeo 组件
```

接口有意限定为 `FFastGeoPrimitiveComponent`。默认点模式只要求有效 Transform；开启 `bCalculateBounds` 后还要求组件正确维护 Bounds。普通 `FFastGeoComponent` 不属于可空间查询的 Primitive，不能直接注册。

每个 FastGeo Entity 保存：

```text
TWeakObjectPtr<UObject> FastGeoContainer
ClusterTypeId + ClusterIndex
ComponentTypeId + ComponentIndex
FBox WorldBounds
```

四元 ID 用于诊断、反向标识和组件解析。UE 5.8 通过 `UFastGeoContainer::GetComponentCluster()` 和 `FFastGeoComponentCluster::GetComponent()` 从这些 ID 重新取得组件，再通过 `CastTo<FFastGeoPrimitiveComponent>()` 验证类型。Entity Slot 不再缓存 FastGeo 组件裸指针；Container 弱引用失效后解析自然返回空。

### 27.3 注册和删除

系统不扫描世界，也不自动注册 FastGeo，更不会因注册一个 Container 而把其中所有组件加入索引。业务在 Container 已注册且组件可用后，传入目标 Primitive：

```cpp
FFastGeoStaticMeshComponent* Component = /* 业务选中的单个组件 */;

FUniformSpatialFastGeoRegistrationParams Params;
Params.CategoryMask = UniformSpatialCategory::Interactable;
Params.bCalculateBounds = false;
Params.bTrackMovement = false;

const FUniformSpatialEntityId EntityId =
    SpatialHash->RegisterFastGeoComponent(Component, Params);
```

参数类型是 `FFastGeoPrimitiveComponent*`，`FFastGeoStaticMeshComponent*` 等派生类指针可直接传入。一次调用只创建一个 Entity，不会遍历同一个 Container 中的其他组件。重复注册同一个组件是幂等操作，返回已有 EntityId，不创建重复 Entity。

批量注册同样只处理显式传入的组件数组，并保持输入与输出下标对齐：

```cpp
TArray<FFastGeoPrimitiveComponent*> Components = /* 业务选中的组件 */;
TArray<FUniformSpatialEntityId> EntityIds;

const int32 NumRegistered = SpatialHash->BatchRegisterFastGeoComponents(
    Components,
    Params,
    EntityIds);
```

单个组件不再需要参与查询时调用：

```cpp
SpatialHash->UnregisterFastGeoComponent(Component);
```

Container 仍然是 FastGeo 组件的生命周期所有者，但不是注册单位。系统仅为已经显式注册的组件建立 `Container -> EntityIds` 反向表。Container 卸载时，`UnregisterFastGeoContainer()` 只批量清理该反向表里的 Entity，不会扫描 Container，也不会影响从未注册的组件。

在第一个组件注册时，系统为其 Container 绑定一次 `OnUnregistered`。Container 卸载后自动清理其中所有已注册 Entity；World Partition 生命周期清理同样只处理已经注册的组件。

### 27.4 点、Bounds 与更新通道

FastGeo 默认使用静态点快速通道：直接读取 `FFastGeoPrimitiveComponent::GetTransform().GetLocation()`，不计算 Bounds；点只加入一个 Regular Cell，也不会进入 Dirty 更新队列或读写查询去重戳。`bTrackMovement=false` 时，`RefreshFastGeoContainer()` 只验证组件生命周期并跳过空间更新。

需要真实组件 Bounds 时设置 `bCalculateBounds=true`。如果该组件可能移动，再设置 `bTrackMovement=true`，并在业务改变 FastGeo Transform 后调用：

```cpp
SpatialHash->RefreshFastGeoContainer(Container);
```

刷新通过 Container 的反向 Entity 列表，不扫描 Container 内全部组件。静态实体以 `O(1)` 条件判断跳过；可移动点读取 Transform 位置并调用 `UpdateEntityPosition()`，同 Cell 时为 `O(1)`，跨 Cell 时 swap-remove 后插入新 Cell，平均仍为 `O(1)`。可移动 AABB 才重新读取 Bounds；覆盖 Cell 不变时只替换 Bounds，覆盖范围变化时重新维护 Membership。

### 27.5 查询与反向索引

FastGeo Entity 没有 Actor。查询是否包含 FastGeo 由 `SourceMask` 唯一决定：

```cpp
FUniformSpatialQueryFilter Filter;
Filter.SourceMask = static_cast<int32>(EUniformSpatialSourceFlags::FastGeo);
Filter.RequiredAnyCategory = UniformSpatialCategory::FastGeo;
```

`ResolveFastGeoContainer(EntityId)` 完成 EntityId 到 Container 的平均 `O(1)` 解析；`GetEntitiesForFastGeoContainer()` 完成 Container 到已注册 Entity 列表的平均 `O(1 + r)` 解析，其中 `r` 是该 Container 中显式注册的组件数。

`GetEntityPosition(EntityId, OutPosition)` 可用于所有来源。点实体返回索引中的精确点位置，AABB 实体返回缓存 Bounds 中心；只解析 Entity Slot，不访问 Actor/FastGeo Component，也不重新计算 Bounds，平均复杂度为 `O(1)`。无效或 Generation 过期的 EntityId 返回 `false`。

### 27.6 复杂度

设一次显式注册的 Primitive 覆盖 `k` 个 Cell；某 Container 中共有 `r` 个已注册 Primitive，第 `i` 个已注册 Primitive 覆盖 `k_i` 个 Cell：

- 静态点单组件注册：反向 Hash 查找、读取 Transform 位置、加入唯一 Cell 都是平均 `O(1)`，合计平均 `O(1)`，并且不调用 `GetBounds()`。
- AABB 单组件注册：反向 Hash 查找平均 `O(1)`，读取 Bounds 为 `O(1)`，加入覆盖 Cell 为 `O(k)`，合计平均 `O(1 + k)`。
- 批量注册 `r` 个组件：`O(r + sum(k_i))`，只与传入数量有关，与 Container 的未注册组件数无关。
- 单组件删除：Entity 和反向 Hash 查找平均 `O(1)`，从每个 Cell swap-remove 为 `O(k)`，合计平均 `O(1 + k)`。
- Container 卸载清理：`O(r + sum(k_i))`，其中 `r` 仅统计该 Container 中显式注册的组件。
- Container 刷新：遍历 `r` 个已注册组件，因此至少 `O(r)`；静态实体只做条件判断，可移动点每个平均 `O(1)`，AABB 额外产生 `sum(k_changed)` 的 Membership 更新成本。
- Container 反向查询：Hash 查找平均 `O(1)`，返回或遍历结果为 `O(r)`，合计 `O(1 + r)`。
- 空间查询复杂度与普通 Entity 相同，不因 FastGeo 来源增加额外扫描。
