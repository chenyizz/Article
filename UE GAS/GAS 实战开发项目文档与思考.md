# GAS 实战开发项目文档与思考

## 一、项目目录结构

```
GAS/
│
├── Config/                 # 配置文件
│   ├── DefaultEngine.ini   # 引擎配置(包含资产管理等设置)
│   ├── DefaultGame.ini     # 游戏配置
│   └── DefaultInput.ini    # 输入配置
│
├── Content/                # 游戏内容资源
│   ├── Assets/         	# 资产
│   │   ├── Character/		#人物模型资产
│   │   ├── Animation/		#人物动画资产
│   │   │	├── PlayerAnimation		#...具体某个人物动作资产
│
├── Source/                 # 源代码目录
│   └── GASGame/           # 核心游戏逻辑模块
│       ├── AbilitySystem/  # 能力系统相关
│       ├── Animation/      # 动画系统
│       ├── Camera/         # 相机系统
│       ├── Character/      # 角色相关
│       ├── Equipment/      # 装备系统
│       ├── GameModes/      # 游戏模式
│       ├── Input/          # 输入系统
│       ├── Player/         # 玩家相关
│       ├── System/         # 系统级组件
│       └── UI/             # UI 系统
│   
│
└── GAS.uproject  # 项目文件
```

## 二、控制器增强输入

在老式的UE4中使用AXIS