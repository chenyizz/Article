---
title: GameDev 插件与模块清单
tags: [UE, 插件, 模块, 依赖, 参考]
summary: 本项目所需 UE 插件/模块的唯一权威清单；确认插件/模块名只读本文件，禁止扫描引擎目录
created: 2026-10-07
updated: 2026-10-07
status: active
source: 用户编辑器 Plugins 面板核对 + UE 官方文档
related: [00-项目目录地图.md, _state/TODO.md, _state/PROGRESS.md, ../../../AGENTS.md]
---

# 插件与模块清单（PLUGINS）

> 规则：本文件是确认"插件/模块叫什么、提供哪个模块、是否要启用"的**唯一出处**。
> **禁止**进入 UE 引擎安装目录查找（见 `Article\AGENTS.md` →「禁止访问引擎目录」）。
> 需要新增/改名时，由用户在编辑器 Plugins 面板确认后回填本文件。
> **agent 不修改 `.uproject` 的 `Plugins` 段**，只输出待追加的 JSON 片段给用户手动粘贴。

## 0. 三个概念（先分清）

| 词 | 生活化类比 | 在本项目里指什么 | 官方定义 |
|---|---|---|---|
| 插件（Plugin） | 外购的专业设备 | 可插拔功能包，如 GAS、CommonUI；在 `.uproject` 的 `Plugins` 段启用 | 可插拔模块集合 |
| 模块（Module） | 公司里的一个部门 | 编译单元（DLL/静态库），在 `Build.cs` 里登记依赖 | 可独立编译的代码集合 |
| 依赖（Dependency） | 部门间的协作协议 | `Build.cs` 数组里列出的模块名，缺了就编译失败 | 模块构建依赖 |

**一句话**：插件是"设备"，模块是"部门"，依赖是"协作协议"。启用插件 ≠ 能调用，还要在 `Build.cs` 登记依赖。

## 1. 已启用插件（本阶段）

| 插件名 | 提供模块 | 用途 | 本项目状态 |
|---|---|---|---|
| `GameplayAbilities` | `GameplayAbilities`、`GameplayAbilitiesEditor` | GAS 主体（技能/属性/效果） | 启用（.uproject） |
| `CommonUI` | `CommonUI` | UI 框架 | 启用（.uproject） |
| `ModularGameplay` | `ModularGameplay` | 模块化框架 | 启用（.uproject） |
| `EnhancedInput` | `EnhancedInput` | 增强输入 | 启用（.uproject） |
| `GameplayMessageRouter` | `GameplayMessageRuntime` | `UGameplayMessageSubsystem`，跨系统消息 | 启用（.uproject） |
| `ModularGameplayActors` | `ModularGameplayActors` | `AModularPawn` 等模块化 Actor | 启用（.uproject） |
| `CommonGame` | `CommonGame` | CommonUI 的 Game 层封装 | 启用（.uproject） |
| `CommonUser` | `CommonUser` | CommonUI 的 User/Setting 层 | 启用（.uproject） |

**随 `GameplayAbilities` 自动带起的插件（无需手填）**：`Niagara`、`GameplayTagsEditor`、`DataRegistry`。

## 2. 引擎模块（只进 Build.cs，无需插件）

| 模块名 | 用途 |
|---|---|
| `GameplayTags` | GameplayTag 本体 |
| `GameplayTasks` | AbilityTask 依赖 |
| `Slate` / `SlateCore` | CommonUI 的底层 UI |
| `UMG` | Widget |

## 3. Build.cs 依赖对照表（当前实际）

`Source/GameDev/GameDev.Build.cs` 的 `PublicDependencyModuleNames`：

| 模块 | 来源 |
|---|---|
| `Core` / `CoreUObject` / `Engine` / `InputCore` | 引擎核心 |
| `EnhancedInput` | EnhancedInput 插件 |
| `GameplayAbilities` | GameplayAbilities 插件 |
| `GameplayTags` / `GameplayTasks` | 引擎模块 |
| `CommonUI` | CommonUI 插件 |
| `ModularGameplay` | ModularGameplay 插件 |
| `GameplayMessageRuntime` | GameplayMessageRouter 插件 |
| `ModularGameplayActors` | ModularGameplayActors 插件 |
| `CommonGame` | CommonGame 插件 |
| `CommonUser` | CommonUser 插件 |
| `UMG` / `Slate` / `SlateCore` | 引擎模块 |

## 4. 待接入插件

当前**无**。所有所需插件均已安装并在 `GameDev.uproject` 启用（见 §1）。

> 历史备注：`GameplayMessageRouter` / `ModularGameplayActors` / `CommonGame` / `CommonUser` 曾列为 Lyra 专属待接入项（预期 K25/K07/K59 前），现已全部接入。

## 5. 行为约束（禁止自造轮子）

- 插件未接入时，**禁止**用自研实现替代。
- 缺失依赖写 TODO 占位，**不写**替代实现。
- **严禁**自写 `UGameplayMessageSubsystem` / `AModularPawn` 等官方插件已有之物。

## 6. 维护流程

1. 新增依赖前先读本文件。
2. 需要新插件时，由用户在编辑器 Plugins 面板确认名称/模块。
3. 用户回填本文件对应行后，agent 才允许改 `Build.cs`。
4. **agent 不修改 `.uproject` 的 `Plugins` 段**，只输出 JSON 片段给用户手动粘贴。
5. 新增/移除插件同理，保持本文件与真实工程一致。
