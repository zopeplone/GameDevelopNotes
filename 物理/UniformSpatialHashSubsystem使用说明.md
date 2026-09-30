# UniformSpatialHashSubsystem 使用说明

`UUniformSpatialHashSubsystem` 是按 World 创建的空间索引 Subsystem。它只索引业务显式注册的对象，不会自动扫描世界。获取当前 World 的 Subsystem：

```cpp
UUniformSpatialHashSubsystem* SpatialHash =
    World->GetSubsystem<UUniformSpatialHashSubsystem>();
// 或：UUniformSpatialHashSubsystem::Get(WorldContextObject)
```

## 注册 Actor

```cpp
FUniformSpatialActorRegistrationParams Params;
Params.Source = EUniformSpatialEntitySource::CustomActor;
Params.CategoryMask = UniformSpatialCategory::Interactable;
Params.bTrackMovement = true;
Params.bCalculateActorBounds = false; // false + CustomExtent 为零：无体积点

FUniformSpatialEntityId EntityId =
    SpatialHash->RegisterActor(Actor, Params);
```

开启 `bCalculateActorBounds` 会使用 Actor 的 AABB；关闭时可以通过 `CustomExtent` 指定体积。`bTrackMovement=true` 时，Subsystem 会监听 RootComponent 的 `TransformUpdated`，并在 Tick 中合并处理移动。

批量注册：

```cpp
TArray<FUniformSpatialEntityId> EntityIds;
SpatialHash->BatchRegisterActors(Actors, Params, EntityIds);
```

## 注册 FastGeo 组件

FastGeo 的注册粒度是单个 `FFastGeoPrimitiveComponent`，不是 Container：

```cpp
FUniformSpatialFastGeoRegistrationParams Params;
Params.CategoryMask = UniformSpatialCategory::FastGeo;
Params.bCalculateBounds = false; // 默认：Transform 位置的无体积点
Params.bTrackMovement = false;   // 默认：静态，不刷新空间位置

FUniformSpatialEntityId EntityId =
    SpatialHash->RegisterFastGeoComponent(FastGeoComponent, Params);
```

`FFastGeoStaticMeshComponent*`、`FFastGeoInstancedStaticMeshComponent*` 等派生类可以直接传入。需要真实组件 Bounds 时设置 `bCalculateBounds=true`；组件可能移动时设置 `bTrackMovement=true`，并调用 `RefreshFastGeoContainer(Container)` 更新。

批量注册使用 `BatchRegisterFastGeoComponents()`，只处理传入数组中的组件。Container 卸载或触发 `OnUnregistered` 时，Subsystem 会自动清理该 Container 下已经注册的 Entity。

## 空间查询

按来源和 Category 过滤后查询 Sphere：

```cpp
FUniformSpatialQueryFilter Filter;
Filter.SourceMask = static_cast<int32>(EUniformSpatialSourceFlags::All);
Filter.RequiredAnyCategory = UniformSpatialCategory::Interactable;
Filter.MaxResults = 32; // 0 表示不限制

TArray<FUniformSpatialEntityId> Results;
SpatialHash->QuerySphere(Center, Radius, Filter, Results);
```

查询 AABB：

```cpp
Filter.SourceMask = static_cast<int32>(
    EUniformSpatialSourceFlags::FastGeo |
    EUniformSpatialSourceFlags::CustomActor);

SpatialHash->QueryBox(QueryBounds, Filter, Results);
```

过滤字段含义：

- `SourceMask`：限制 Actor、FastGeo 等来源。
- `RequiredAnyCategory`：至少命中一个 Category 位。
- `RequiredAllCategory`：必须命中全部 Category 位。
- `ExcludedCategory`：排除包含这些位的 Entity。
- `MaxResults`：限制返回数量，零表示不限制。

## EntityId 与更新

```cpp
FVector Position;
if (SpatialHash->GetEntityPosition(EntityId, Position))
{
    // 点实体返回点位置；AABB 实体返回 Bounds 中心。
}

SpatialHash->UnregisterEntity(EntityId);
```

Actor 结束生命周期时会通过 `OnEndPlay` 自动注销；World Partition 卸载 Level 时会清理相关 Actor 和 FastGeo Container。

Actor 移动更新会自动进入 Dirty 队列；如果查询必须立即看到本帧移动结果，可在查询前调用：

```cpp
SpatialHash->FlushPendingUpdates();
```

FastGeo 没有统一的移动回调，只有标记 `bTrackMovement=true` 的组件会在 `RefreshFastGeoContainer()` 中更新。静态点实体只占用一个网格 Cell，注册、删除和位置读取均为平均 `O(1)`。
