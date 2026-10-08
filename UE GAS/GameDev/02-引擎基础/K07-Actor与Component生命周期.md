---
id: K07
title: Actor 与 Component 生命周期
system: 引擎基础
prerequisite: K04
ue_version: UE 5.7.4
tags: [AActor, UActorComponent, USceneComponent, 生命周期, BeginPlay, Tick, EndPlay, CreateDefaultSubobject, ModularGameplay]
status: draft
created: 2026-10-08
updated: 2026-10-08
---

# K07 Actor 与 Component 生命周期

## 0. 一句话结论
**Actor 是关卡里"活着的东西"**（角色、传送门、掉落物），**Component 是挂在 Actor 身上的"功能模块"**（相机臂、技能组件、背包组件）。引擎给它们规定了一条固定的"生老病死流水线"：构造 → 注册 → 初始化 → `BeginPlay` → 每帧 `Tick` → `EndPlay` → 销毁。**你只能在流水线的正确工位做正确的事**——在构造工位去拿"世界"会拿到半成品，在 `BeginPlay` 之前去用别的 Actor 会拿到空指针，在 `EndPlay` 不解绑委托会在销毁后崩溃。

> 本单元配套总纲：`00-项目目录地图.md`。本单元**不新建文件**，以 K06 刚建的类当标本，把生命周期看透。

> 🏭 **工业目标 / 当前落地**：本单元**不创建任何组件**。本项目所有"功能模块"一律用官方 **ModularGameplay** 的组件体系（`UGameFrameworkComponent` 系列），并在**真正需要它的单元**（K09 相机、K19 PawnExtension）落地——**不建"占位组件"**，不留技术债。本单元无新增技术债。

## 1. 为什么需要它
- **场景**：在俯视角 ARPG 里，下列每件事都要"卡时间点"：
  1. 生成角色时要**提前挂好相机臂 / 技能组件** → 只能在**构造函数**里做。
  2. 开局要**绑定输入、初始化 GAS、读配置** → 要等**`BeginPlay`**（此时所有对象都已就位）。
  3. 每帧要**移动相机、更新瞄准** → 放**`Tick`**。
  4. 玩家死亡/退出时要**解绑委托、关计时器** → 放**`EndPlay`**。
- **不学它会导致什么问题（具体反例）**：
  1. **在构造函数里 `GetWorld()`**：构造函数执行时对象还没"进入世界"，`GetWorld()` 可能返回 `nullptr`，一解引用就崩。
  2. **在构造函数里 `SpawnActor`**：此时世界/物理还没准备好，生成物状态异常，甚至崩溃。
  3. **在 `BeginPlay` 之前访问别的 Actor**：引擎还没把场景里的 Actor 都初始化完，你拿到的是"半成品"。
  4. **`Tick` 里不判 `IsValid`**：绑定的 Actor 早被销毁，你还在用它的指针。
  5. **`EndPlay` 不解绑 `OnDestroyed` 委托**：自己已被销毁但委托还挂着，回调触发时访问已释放内存 → 崩溃。
  6. **组件创建顺序/时机不对**：`CreateDefaultSubobject` 只能在构造函数；运行时加组件要用 `NewObject` + `RegisterComponent`，用错方法组件不生效。

## 2. 前置知识
- 前置单元：[K04 C++ 类与 UObject 体系](K04-C++类与UObject体系.md)（Actor/Component 是 `UObject` 后代）、[K06 Gameplay 框架核心类](K06-Gameplay框架核心类.md)（`AGameDevPawn` 等已经是 Actor）。
- 需要先搞懂的 3 个概念（生活化的话）：
  1. **回调（Callback）/ 虚函数重写**：你把某个函数"接"到引擎的固定时点上，引擎到点就喊你。像给闹钟设了"早上 7 点叫我"。
  2. **构造 vs 初始化**：构造是"造零件"（进不了世界、拿不到世界）；初始化/BeginPlay 是"零件上工"（已进入世界、可互相打交道）。像"车在流水线上组装"与"车开上路"。
  3. **注册（Register）**：让组件正式"挂靠"到 Actor 并接入世界/复制系统。像员工办理入职登记，办完才算正式上岗。

## 3. UE 里的相关概念（术语表）

| 术语 | 生活化类比 | 在本项目里具体指什么 | 官方一句话定义 |
|---|---|---|---|
| `AActor` | 关卡里活着的东西 | 玩家、NPC、传送门、掉落物 | 可放置/可生成对象基类 |
| `UActorComponent` | 挂在身上的功能模块 | 技能组件、装备组件、相机臂管理 | Actor 组件基类 |
| `USceneComponent` | 有位置的模块 | 相机臂、网格、碰撞体、挂点 | 带 Transform 的组件 |
| `RootComponent` | 身体主干 | Actor 的变换基准（其它组件挂它下面） | 根场景组件 |
| `CDO` | 物种标准像 | 每个类的默认实例（构造函数的产物存在这） | 类默认对象 |
| 构造函数 | 流水线组装 | 创建组件、设默认值；**拿不到世界** | 对象构造 |
| `PostInitializeComponents` | 零件组装完、上线前 | 组件都创建/初始化后调用一次 | 组件初始化后回调 |
| `BeginPlay` | 正式开工 | 进入世界、可安全访问其他 Actor；绑输入/初始化 | 游戏开始回调 |
| `Tick` | 每帧例行动作 | 每帧更新（相机、瞄准、AI 感知） | 每帧回调 |
| `EndPlay` | 下班/离职 | 解绑委托、清计时器、释放资源 | 结束播放回调 |
| `Destroy` / `OnDestroyed` | 销毁 + 讣告 | 请求销毁 Actor / 销毁时的广播 | 销毁与委托 |
| `CreateDefaultSubobject` | 出厂自带部件 | 在构造函数里创建组件（默认子对象） | 创建默认子对象 |
| `AddComponent` / `RegisterComponent` | 后期加装 + 登记 | 运行时给 Actor 加组件并注册生效 | 组件注册 |
| `IGameFrameworkInitStateInterface` | 模块化"上岗顺序表" | ModularGameplay 组件的初始化状态机接口 | 框架初始化状态接口 |
| `UGameFrameworkComponent` | 官方模块化组件基类 | 本项目所有功能组件应继承它 | 框架组件基类 |

> **一句话记住**：**构造函数只组装、不碰世界；`BeginPlay` 才开工；`Tick` 干活；`EndPlay` 收拾**。组件同理，且有一层"注册/初始化"预约。

## 4. 物理落位（物理模块 ↔ 逻辑层 ↔ 具体文件）
> 三维同时标注：**物理模块（DLL）** / **逻辑层** / **具体文件**。物理模块规划见 `00-项目目录地图.md` 第 1.5 节。
> **本单元不新建/不修改任何文件**，只读 K06 已建的类，把它们的生命周期钩子看清楚。

**本单元涉及的类（均已存在，只读）：**

| 类 / 文件 | 物理模块（DLL） | 逻辑层（子目录） | 具体文件 | 新增/修改 |
|---|---|---|---|---|
| `AGameDevPawn` | `GameDev` | 角色层 | `Source/GameDev/Character/GameDevPawn.h` + `.cpp` | 只读（K06 建） |
| `AGameDevPlayerController` | `GameDev` | 基础设施层 | `Source/GameDev/Framework/GameDevPlayerController.h` + `.cpp` | 只读（K06 建） |
| `AGameDevGameMode` | `GameDev` | 基础设施层 | `Source/GameDev/Framework/GameDevGameMode.h` + `.cpp` | 只读（K06 建） |
| `AGameDevGameState` | `GameDev` | 基础设施层 | `Source/GameDev/Framework/GameDevGameState.h` + `.cpp` | 只读（K06 建） |
| `AGameDevPlayerState` | `GameDev` | 角色层 | `Source/GameDev/Character/GameDevPlayerState.h` + `.cpp` | 只读（K06 建） |

**物理模块 ↔ 逻辑层 ↔ 物理目录 对照树（本单元涉及的，已存在）：**

```text
Source/
└── GameDev/
    ├── Framework/
    │   ├── GameDevGameMode.h / .cpp            ← AGameDevGameMode（有 InitGame）
    │   ├── GameDevGameState.h / .cpp           ← AGameDevGameState
    │   └── GameDevPlayerController.h / .cpp    ← AGameDevPlayerController（有 BeginPlay / SetupInputComponent）
    └── Character/
        ├── GameDevPawn.h / .cpp                ← AGameDevPawn（有 SetupPlayerInputComponent / PossessedBy）
        └── GameDevPlayerState.h / .cpp         ← AGameDevPlayerState
```

- **规则**：生命周期钩子是"往已有类里重写虚函数"，**不产生新文件**。只有当你需要一个新的组件或 Actor 时，才按 `00-项目目录地图.md` 第 3 节决策规则新建文件。
- **未来组件落位预告（非本单元）**：
  - 俯视角相机臂 → K09，放 `Character/`（属于角色表现）。
  - GAS/输入初始化组件（PawnExtension）→ K19，放 `Character/`。
  - 装备、技能等组件 → 各自系统模块（`Equipment/`、`Combat/`）。
  - 全部继承官方 `UGameFrameworkComponent`，**禁止自研组件基类**。
- 本单元不新增目录/文件，**无需更新 `00-项目目录地图.md`**。

## 5. 涉及的 UE 类与函数
> 这里指"引擎自带"的类与函数，不是我们自写的。每个都给头文件与生活化作用。
> 本表**不填物理模块列**（引擎类不属于我们的 DLL）。

| 类 / 函数 | 头文件 | 用大白话讲它是干嘛的 |
|---|---|---|
| `AActor` | `GameFramework/Actor.h` | 关卡里实体基类 |
| `AActor::PostInitializeComponents()` | `GameFramework/Actor.h` | 组件初始化完、上线前的一次回调 |
| `AActor::BeginPlay()` | `GameFramework/Actor.h` | 正式开工回调（可安全访问世界） |
| `AActor::Tick(float)` | `GameFramework/Actor.h` | 每帧回调 |
| `AActor::EndPlay(EEndPlayReason)` | `GameFramework/Actor.h` | 结束回调（收拾现场） |
| `AActor::Destroy()` | `GameFramework/Actor.h` | 请求销毁本 Actor |
| `AActor::OnDestroyed` | `GameFramework/Actor.h` | 销毁广播（委托） |
| `AActor::CreateDefaultSubobject<T>()` | `GameFramework/Actor.h` | 在构造函数里创建组件（默认子对象） |
| `AActor::AddComponentByClass(...)` | `GameFramework/Actor.h` | 运行时按类添加组件 |
| `UActorComponent` | `Components/ActorComponent.h` | 组件基类 |
| `UActorComponent::RegisterComponent()` | `Components/ActorComponent.h` | 让组件正式挂靠生效 |
| `UActorComponent::InitializeComponent()` | `Components/ActorComponent.h` | 组件初始化回调 |
| `UActorComponent::BeginPlay()` | `Components/ActorComponent.h` | 组件开工回调 |
| `UActorComponent::TickComponent(...)` | `Components/ActorComponent.h` | 组件每帧回调 |
| `UActorComponent::EndPlay(EEndPlayReason)` | `Components/ActorComponent.h` | 组件结束回调 |
| `USceneComponent` | `Components/SceneComponent.h` | 带 Transform 的组件基类 |
| `UWorld` | `Engine/World.h` | 世界；`GetWorld()` 的来源 |
| `UGameFrameworkComponent` | `Components/GameFrameworkComponent.h` | 官方模块化组件基类（本项目组件应继承） |
| `IGameFrameworkInitStateInterface` | `Components/GameFrameworkInitStateInterface.h` | 模块化组件的初始化状态机接口 |

> 说明：ModularGameplay 相关头文件名以编辑器里插件 `Public/` 下真实文件为准（见 `_state/PLUGINS.md`）；**禁止**因此自研替代。

## 6. 架构设计

### 6.1 Actor 的生命周期流水线
```mermaid
sequenceDiagram
    participant Engine
    participant Actor
    participant Comps as Components
    Engine->>Actor: 构造函数（CDO 先执行一次）
    Note over Actor: 只创建组件/设默认值，禁止 GetWorld/Spawn
    Engine->>Actor: PostActorCreated
    Engine->>Comps: 组件 RegisterComponent（逐个注册）
    Actor->>Comps: PostInitializeComponents（组件都就绪）
    Note over Actor: 可在此做“组件间”校验
    Engine->>Actor: BeginPlay（进入世界，可访问其它 Actor）
    Actor->>Comps: 各组件 BeginPlay
    loop 每帧
        Engine->>Actor: Tick(DeltaSeconds)
        Actor->>Comps: 各组件 TickComponent
    end
    Engine->>Actor: EndPlay(Reason)（关卡切换/销毁）
    Actor->>Comps: 各组件 EndPlay
    Engine->>Actor: Destroyed / OnDestroyed 广播
    Note over Actor: 之后被 GC 回收
```

- **构造函数**：可能为 CDO 执行一次，也可能为实例执行。**只做"组装"**（创建组件、设默认值）。
- **PostInitializeComponents**：本 Actor 的组件都已创建并初始化，适合做"组件之间"的检查。
- **BeginPlay**：世界就绪，**才能安全 `GetWorld()`、访问其他 Actor、绑委托、初始化 GAS**。
- **Tick**：每帧；**能不开就不开**（用 `PrimaryActorTick.bCanEverTick` 控制，省性能）。
- **EndPlay**：解绑委托、清计时器；漏了就是崩溃隐患。

### 6.2 Component 的生命周期流水线
```mermaid
flowchart LR
    A[构造函数创建<br/>CreateDefaultSubobject] --> B[RegisterComponent<br/>接入世界/复制]
    B --> C[InitializeComponent<br/>内部初始化]
    C --> D[BeginPlay<br/>开工]
    D --> E[TickComponent<br/>每帧(可选)]
    E --> F[EndPlay<br/>收拾]
    F --> G[UnregisterComponent / GC]
```

- **创建（构造）**：在所属 Actor 的构造函数里用 `CreateDefaultSubobject`，或用 `FObjectInitializer` 变体。
- **注册**：组件必须 `RegisterComponent()` 才真正生效（构造函数创建+`SetupAttachment` 的组件由引擎自动注册；运行时手动建的必须自己调）。
- **初始化 / BeginPlay / Tick / EndPlay**：与 Actor 同构，但**独立触发、独立开关**。
- **ModularGameplay 升级**：本项目组件继承 `UGameFrameworkComponent` 并实现 `IGameFrameworkInitStateInterface`，把"谁先初始化"做成**状态机**，避免依赖顺序硬编码（K19 详用）。

### 6.3 组件创建三种方式（别用错）
| 方式 | 时机 | 典型写法 | 适用 |
|---|---|---|---|
| `CreateDefaultSubobject` | **仅构造函数** | `Root = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));` | 固定部件（相机臂、网格、常驻组件） |
| `NewObject` + `RegisterComponent` | 运行时 | `UComp* C = NewObject<UComp>(this); C->RegisterComponent();` | 动态增减组件 |
| `AddComponentByClass` / `AddComponent` | 运行时 | `AddComponentByClass(Cls, false, FTransform(), false);` | 运行时按类/模板加组件 |

### 6.4 客户端 / 服务端
- **Actor 的 `BeginPlay`/`Tick`/`EndPlay` 在每个拥有它的端都会跑**（服务端跑服务端那份，客户端跑客户端那份）。
- **权威判定**：改世界状态的逻辑要 `HasAuthority()` 才做；表现逻辑（相机、特效）各端自己做。
- **构造函数在 CDO 与实例上都会运行**，且**与服务端/客户端无关**——所以构造函数里**绝对不能**写网络逻辑或访问世界。
- **组件复制**：组件是否复制取决于其配置（Replicated），M04/K20+ 详述。

## 7. 函数逐个设计

### 7.1 `AActor::PostInitializeComponents()`
- **签名**：`virtual void AActor::PostInitializeComponents();`
- **所属文件**：重写在你的 Actor 实现文件（如 `Source/GameDev/Character/GameDevPawn.cpp`）。
- **参数表**：无。
- **返回值**：无。
- **调用时机**：本 Actor 的所有组件都创建并初始化之后、`BeginPlay` 之前（每端各一次）。
- **内部步骤（伪代码）**：
  ```
  PostInitializeComponents()
  {
      Super::PostInitializeComponents();   // 必须调用父类
      // 适合做“组件之间”的一致性检查/引用绑定（此时所有组件都在）
  }
  ```
- **常见坑**：忘记 `Super::` 会导致父类（含 ModularGameplay 的组件初始化）逻辑缺失。

### 7.2 `AActor::BeginPlay()`
- **签名**：`virtual void AActor::BeginPlay();`
- **所属文件**：重写在你的 Actor 实现文件。
- **参数表**：无。
- **返回值**：无。
- **调用时机**：Actor 进入世界并正式开始时（每端各一次，可能早于/晚于同关卡其它 Actor，**不要假设顺序**）。
- **内部步骤（伪代码）**：
  ```
  BeginPlay()
  {
      Super::BeginPlay();          // 先跑父类
      // 1) 绑委托 / 注册消息监听
      // 2) 初始化运行时数据（读配置、生成子对象）
      // 3) 需要 GAS 的地方在此绑定 Owner/Avatar（K19）
  }
  ```
- **常见坑**：
  1. 假设"其它 Actor 一定已 BeginPlay" —— 不保证，跨 Actor 依赖要用事件/延迟。
  2. 忘记 `Super::BeginPlay()`。

### 7.3 `AActor::Tick(float DeltaSeconds)`
- **签名**：`virtual void AActor::Tick(float DeltaSeconds);`
- **所属文件**：重写在你的 Actor 实现文件；需在构造函数里开 `PrimaryActorTick.bCanEverTick = true;`。
- **参数表**：
  | 参数 | 类型 | 含义 | 取值范围 |
  |---|---|---|---|
  | DeltaSeconds | `float` | 距上一帧的秒数 | 通常 0.008~0.033 |
- **返回值**：无。
- **调用时机**：每帧（若开启了 Tick）。
- **内部步骤（伪代码）**：
  ```
  Tick(float DeltaSeconds)
  {
      Super::Tick(DeltaSeconds);
      // 只放“每帧必须做”的事：相机跟随、瞄准、感知轮询
      // 禁止放：昂贵的全场景查找、字符串比较
  }
  ```
- **常见坑**：默认不 Tick（`bCanEverTick=false`）；开了就要承担性能成本，能不用就不用。

### 7.4 `AActor::EndPlay(const EEndPlayReason::Type EndPlayReason)`
- **签名**：`virtual void AActor::EndPlay(const EEndPlayReason::Type EndPlayReason);`
- **所属文件**：重写在你的 Actor 实现文件。
- **参数表**：
  | 参数 | 类型 | 含义 | 取值范围 |
  |---|---|---|---|
  | EndPlayReason | `EEndPlayReason::Type` | 结束原因 | Destroyed/LevelTransition/EndPlayInEditor/RemovedFromWorld/Quit |
- **返回值**：无。
- **调用时机**：Actor 被销毁或离开世界时（每端各一次）。
- **内部步骤（伪代码）**：
  ```
  EndPlay(EEndPlayReason::Type EndPlayReason)
  {
      // 1) 解绑所有委托、取消计时器
      // 2) 反注册消息监听
      Super::EndPlay(EndPlayReason);   // 父类放最后调（保持与 BeginPlay 对称的反向清理）
  }
  ```
- **常见坑**：忘记解绑 `OnDestroyed` 等委托 → 销毁后回调访问野指针崩溃。

### 7.5 `UActorComponent::RegisterComponent()` / `InitializeComponent()` / `BeginPlay()`
- **签名**：
  - `void UActorComponent::RegisterComponent();`
  - `virtual void UActorComponent::InitializeComponent();`
  - `virtual void UActorComponent::BeginPlay();`
- **所属文件**：引擎 `Components/ActorComponent.h`；重写在自写组件 `.cpp`。
- **参数表**：`RegisterComponent` 无参。
- **返回值**：无。
- **调用时机**：
  - `RegisterComponent`：运行时手动创建组件后立即调用；构造函数里 `CreateDefaultSubobject` 的组件由引擎自动注册，**不要重复手动注册**。
  - `InitializeComponent`：注册后、`BeginPlay` 前，引擎调用。
  - `BeginPlay`：其所属 Actor `BeginPlay` 时（组件自己也会被调用）。
- **内部步骤（伪代码）**：
  ```
  InitializeComponent()
  {
      Super::InitializeComponent();
      // 依赖“组件已注册但世界未必开始”的初始化
  }
  BeginPlay()
  {
      Super::BeginPlay();
      // 开工逻辑（绑委托、缓存引用）
  }
  ```
- **常见坑**：
  1. 对构造函数创建的组件再手动 `RegisterComponent` → 重复注册报错。
  2. 在 `InitializeComponent` 里做需要"世界已开始"的事 → 太早；放 `BeginPlay`。

### 7.6 `UActorComponent::TickComponent(...)`
- **签名**：`virtual void UActorComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction);`
- **所属文件**：引擎 `Components/ActorComponent.h`；重写在自写组件 `.cpp`。
- **参数表**：
  | 参数 | 类型 | 含义 | 取值范围 |
  |---|---|---|---|
  | DeltaTime | `float` | 帧间隔秒数 | 0.008~0.033 |
  | TickType | `ELevelTick` | Tick 类型 | 常规为 `LEVELTICK_All` |
  | ThisTickFunction | `FActorComponentTickFunction*` | 本次 Tick 的调度信息 | 引擎传入 |
- **返回值**：无。
- **调用时机**：每帧（组件需 `PrimaryComponentTick.bCanEverTick = true;`）。
- **内部步骤（伪代码）**：
  ```
  TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
  {
      Super::TickComponent(DeltaTime, TickType, ThisTickFunction);
      // 组件自己的每帧逻辑
  }
  ```
- **常见坑**：默认不 Tick；忘记 `Super::TickComponent` 会破坏组件 Tick 内部机制。

### 7.7 `AActor::CreateDefaultSubobject<T>()`
- **签名**：`template<class TReturnType> TReturnType* AActor::CreateDefaultSubobject(FName SubobjectName, bool bTransient = false);`
- **所属文件**：引擎 `GameFramework/Actor.h`；调用点在自写 Actor/组件的**构造函数**里。
- **参数表**：
  | 参数 | 类型 | 含义 | 取值范围 |
  |---|---|---|---|
  | SubobjectName | `FName` | 子对象唯一名（用 `TEXT("Xxx")`） | 同一 Actor 内唯一 |
  | bTransient | `bool` | 是否为临时对象（不序列化） | 一般 false |
- **返回值**：`TReturnType*` —— 新建组件指针。
- **调用时机**：**仅构造函数**。
- **内部步骤（伪代码）**：
  ```
  AMyActor::AMyActor()
  {
      // 1) 创建根组件并设为 RootComponent
      RootComponent = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));
      // 2) 创建子组件并挂到根上
      USceneComponent* Arm = CreateDefaultSubobject<USceneComponent>(TEXT("CameraArm"));
      Arm->SetupAttachment(RootComponent);
  }
  ```
- **常见坑**：
  1. 名字必须**同一 Actor 内唯一**，重名会报错或覆盖。
  2. 只能在构造函数调用；运行时用 `NewObject` + `RegisterComponent`。
  3. 创建了但忘了 `SetupAttachment`，组件不在正确层级（K09 相机就靠这个）。

## 8. 完整代码示例

> ⚠️ 本单元为**纯文档**。8.1 是"读你已有的类"（重点）；8.2 是**通用模板示意**，真实组件在 K09/K19 才按此落地。

### 8.1 读 K06 已建类，标出生命周期钩子
- `Source/GameDev/Character/GameDevPawn.cpp`：
  - `AGameDevPawn::AGameDevPawn()` —— **构造函数**：K06 里为空，K09 会在此 `CreateDefaultSubobject` 相机臂。
  - `SetupPlayerInputComponent(...)` —— Pawn 专用初始化钩子：被 possess 后绑输入（K08/K10）。
  - `PossessedBy(...)` —— **仅服务端**、被占有时回调（K19 做 GAS 初始化）。
- `Source/GameDev/Framework/GameDevPlayerController.cpp`：
  - `BeginPlay()` —— 玩家控制器开工；目前只打日志。
  - `SetupInputComponent()` —— 引擎初始化输入时调用。
- `Source/GameDev/Framework/GameDevGameMode.cpp`：
  - 构造函数 —— 只做"接线"（设各类默认类），**不 `GetWorld`**，正确符合生命周期。
  - `InitGame(...)` —— GameMode 开局初始化（仅服务端）。

### 8.2 通用模板示意（逐行注释，标示各工位该做的事）
```cpp
// ============ 示意：一个带组件的 Actor 的生命周期骨架（不写入工程） ============

// 【构造函数工位】只组装，不碰世界
AMyActor::AMyActor()
{
    // 开启 Tick（默认关闭）；只有确实每帧要做才开
    PrimaryActorTick.bCanEverTick = true;

    // 创建根组件
    RootComponent = CreateDefaultSubobject<USceneComponent>(TEXT("Root"));

    // 创建"相机臂"这类常驻组件，并挂到根上
    CameraArm = CreateDefaultSubobject<USceneComponent>(TEXT("CameraArm"));
    CameraArm->SetupAttachment(RootComponent);
    // 注意：此处不要 GetWorld()、不要 SpawnActor、不要绑网络
}

// 【上线前工位】组件都就绪，做组件间校验
void AMyActor::PostInitializeComponents()
{
    Super::PostInitializeComponents();
    // 例如：确认必须存在的组件不为空
}

// 【开工工位】世界已就绪，可安全访问他物
void AMyActor::BeginPlay()
{
    Super::BeginPlay();
    // 绑委托（记得在 EndPlay 解绑）
    // 读配置、初始化运行时数据
    // GAS 相关在此绑定 Owner/Avatar（K19）
}

// 【每帧工位】只做必须每帧做的事
void AMyActor::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);
    // 相机跟随、瞄准等
}

// 【收尾工位】对称地清理 BeginPlay 里建立的东西
void AMyActor::EndPlay(const EEndPlayReason::Type EndPlayReason)
{
    // 先解绑委托、清计时器、反注册监听
    Super::EndPlay(EndPlayReason); // 父类清理放最后
}
```

## 9. 验证方法
- **怎么运行**：本单元无代码改动，用断点 + 日志验证"顺序"。
- **预期观察（验证顺序）**：
  1. 在 `AGameDevPawn::AGameDevPawn`、`AGameDevPawn::PossessedBy`、`AGameDevGameMode::InitGame` 各打断点，Play PIE，按命中顺序观察：**GameMode 构造 → InitGame → Pawn 构造 → ... → PossessedBy**。
  2. 在 `AGameDevPlayerController::BeginPlay` 打断点，Watch `GetWorld()`（此时应**非空**）；对比在构造函数里断点时的 `GetWorld()`（可能为空）。
- **控制台辅助**：
  1. `show Collision` 之类无关；用 `stat unit` 观察帧时间（Tick 开销，可选）。
  2. World Outliner 选中 Actor，查看其 **Components** 面板，理解组件层级（Root 与子组件）。
- **断点看变量**：在 `PostInitializeComponents` 断点，`this->GetComponents()` 应已包含构造函数里创建的所有组件；在构造函数断点，`GetWorld()` 可能为 `nullptr`。

## 10. 常见错误与排查
| 现象 | 原因 | 解决 |
|---|---|---|
| 构造函数崩溃/`GetWorld()` 为空 | 在构造里访问世界 | 把世界相关逻辑移到 `BeginPlay`/`PostInitializeComponents` |
| 构造里 `SpawnActor` 状态异常 | 世界未就绪 | 改到 `BeginPlay` 之后 |
| `BeginPlay` 里拿到的其它 Actor 为空 | Actor 间 BeginPlay 顺序不保证 | 用事件/委托或延迟一帧；不要假设顺序 |
| 组件不生效 | 运行时建的组件没 `RegisterComponent` | 手动建的组件要 `RegisterComponent()` |
| 重复注册报错 | 对 `CreateDefaultSubobject` 的组件又手动注册 | 构造里创建的组件由引擎自动注册，勿重复 |
| 销毁后崩溃 | `EndPlay` 没解绑委托/清计时器 | 在 `EndPlay` 对称清理 |
| 每帧卡顿 | 无谓地开了大量 Tick | 关闭不必要的 `bCanEverTick`；把轮询改事件 |
| 客户端也执行了改世界逻辑 | 没做 `HasAuthority()` 判定 | 改世界只在 `HasAuthority()` 为真时做 |
| `Super::` 漏调导致行为异常 | 重写时忘了 `Super::` | 每个生命周期重写都先/后调 `Super::`（EndPlay 通常最后调） |

## 11. 动手练习
1. **必做（顺序追踪）**：按第 9 节在 `AGameDevGameMode`、`AGameDevPawn`、`AGameDevPlayerController` 打断点，画一张"实际命中顺序"时序图，对照第 6.1 节。
2. **必做（构造 vs 开工）**：在 `AGameDevPawn::AGameDevPawn()` 里临时加 `UE_LOG(..., TEXT("GetWorld=%s"), *GetNameSafe(GetWorld()));`，再在 `BeginPlay` 里加同样一条（`BeginPlay` 需临时重写，用完还原），对比两次输出，理解"构造时拿不到世界"。**验证后还原。**
3. **可选（永久有用）**：给 `AGameDevGameState` 重写 `EndPlay`，在其中打一条日志并解开未来会加的委托（现在只打日志）。这是长期有效的清理位，不是练习物。
4. **思考题（写进笔记）**：为什么"在构造函数里创建组件"和"在运行时 `NewObject` 创建组件"必须用不同写法？（提示：CDO/序列化/注册时机）

## 12. 关联单元
- 上游：K04 C++ 类与 UObject 体系、K06 Gameplay 框架核心类。
- 下游：K08 Enhanced Input 输入系统（`SetupInputComponent` 落地）、K09 俯视角相机与角色（构造函数创建相机臂）、K19 GAS 初始化与 Owner/Avatar 绑定（`PossessedBy`/`BeginPlay` 落地）、M04 网络同步（EndPlay/复制时机）。
- 相关：K01 整体技术架构（ModularGameplay 体系）。

## 13. 参考
- 本项目 `00-项目目录地图.md`、`02-引擎基础/K04-C++类与UObject体系.md`、`02-引擎基础/K06-Gameplay框架核心类.md`
- Epic Games, *Actor Lifecycle*（官方 Actor 生命周期文档）
- Epic Games, *Actor Component Lifecycle*、*Components*（官方组件生命周期文档）
- Epic Games, *Modular Gameplay / GameFrameworkComponent*（官方模块化组件体系）
