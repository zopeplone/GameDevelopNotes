
## 简介
所有可以放入关卡的对象都是 **Actor**，比如摄像机、静态网格体、玩家起始位置...
Actor不直接保存变换（位置、旋转和缩放）数据，Transform通过场景组件 `USceneComponent` 来表示，Actor可以将单个场景组件指定为"根"，意味着这个Actor的场景位置、旋转和缩放都根据此组件来绘制
AActor是所有Actor的基类，所有Actor都必须继承自AActor

## 为何Actor不像GameObject一样自带Transform？
我们知道，如果一个对象需要在3D世界中表示，那么它必然要携带一个Transform matrix来表示其位置。关键在于，在UE看来，Actor并不只是3D中的“表示”，一些不在世界里展示的“不可见对象”也可以是Actor，如AInfo(派生类AWorldSetting,AGameMode,AGameSession,APlayerState,AGameState等)，AHUD,APlayerCameraManager等，代表了这个世界的某种信息、状态、规则。你可以把这些看作都是一个个默默工作的灵体Actor。所以，Actor的概念在UE里其实不是某种具象化的3D世界里的对象，而是世界里的种种元素，用更泛化抽象的概念来看，小到一个个地上的石头，大到整个世界的运行规则，都是Actor.  
当然，你也可以说即使带着Transform，把坐标设置为原点，然后不可见不就行了？这样其实当然也是可以，不过可能因为UE跟贴近C++一些的缘故，所以设计哲学上就更偏向于C++的哲学“不为你不需要的东西付代价”。一个Transform再加上附带的逆矩阵之类的表示，内存占用上其实也是挺可观的。要知道UE可是会抠门到连bool变量都要写成uint bPending:1;位域来节省一个字节的内存的。  
换一个角度讲，如果把带Transform也当成一个Actor的额外能力可以自由装卸的话，那其实也可以自圆其说。经过了UE的权衡和考虑，把Transform封装进了SceneComponent,当作RootComponent。但在权衡到使用的便利性的时候，大部分Actor其实是有Transform的，我们会经常获取设置它的坐标，如果总是得先获取一下SceneComponent，然后再调用相应接口的话，那也太繁琐了。所以UE也为了我们直接提供了一些便利性的Actor方法，如(Get/Set)ActorLocation等，其实内部都是转发到RootComponent

## Component
`UActorComponent` 是所有组件的基类。组件是渲染网格体和图像、实现碰撞和播放音频的唯一方法，因此玩家游戏期间在场景中看到或进行交互的一切其实都是某一类组件的成果；
Actor本身不携带功能，所有的功能都是由Component实现

- **场景组件**（类 `USceneComponent`、`UActorComponent` 的子项）支持基于位置的行为，这类行为不需要几何表示（不需要渲染）。这包括弹簧臂、摄像机、物理力和约束（但不包括物理对象），甚至音频。
- **Primitive组件**（类 `UPrimitiveComponent`、`USceneComponent` 的子项）是拥有几何表示的场景组件，通常用于渲染视觉元素或与物理对象发生碰撞或重叠。这包括静态或骨架网格体、Sprite或公告板、粒子系统以及盒体、胶囊体和球体碰撞体积。

## Actor和Component的关系
```cpp
/*
所有由该 Actor 拥有的 ActorComponent。  
使用 Set 结构存储，因为一个 Actor 可能会拥有大量组件。
*/
TSet<TObjectPtr<UActorComponent>> OwnedComponents;
/*
按实例（per-instance）由用户手动添加的 ActorComponent 数组
*/
TArray<UActorComponent*> InstanceComponents;
/*
用于存储需要进行网络复制（replicated）的组件集合。  
这里使用数组存储是为了节省空间，因为这类组件通常数量不会很多。
*/
TArray<UActorComponent*> ReplicatedComponents;
/*
定义该 Actor 在世界中变换（位置、旋转、缩放）的组件。  
所有其它组件都必须以某种方式附加到这个组件上。
*/
TObjectPtr<USceneComponent> RootComponent;
```
OwnedComponents 保存着这个Actor所拥有的所有Component,一般其中会有一个SceneComponent作为RootComponent

InstanceComponents