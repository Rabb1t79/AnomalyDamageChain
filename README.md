# AnomalyDamageChain

伤害链式反应（倍攻连锁反应）：玩家攻击命中后，按设定速率持续重复同一次伤害判定，直到目标失效。

本仓库**独立于** `Rabb1t79/Anomaly-main`，不参与其分支合并。它包含两部分：

| 目录 | 内容 |
| --- | --- |
| `plugin/` | 插件本体（`plugin.cpp` / `manifest.json`），只依赖公开纯 C ABI |
| `core/` | 对框架 Core 的集成补丁（Profile / validator / 服务） |
| `docs/` | 设计说明与集成步骤 |

## 为什么需要 Core 补丁

插件不能再自己硬编码原生签名。按框架约定，新增偏移或签名必须同时落到 Profile 与 validator，
让 runtime 构建出来的 Core 能识别并校验它；插件通过服务拿到**已被校验过的地址**。

```
Profile 符号 ──► Core validator ──► 服务 / 发布 ──► 插件取地址
     ▲                                                  │
     └────────── 校验失败：feature 不可用 ◄──────────────┘
```

## 快速开始

```bash
git clone https://github.com/Rabb1t79/Anomaly-main.git
cd Anomaly-main
git apply /path/to/AnomalyDamageChain/core/0001-damage-chain-integration.patch

mkdir -p plugins/AnomalyDamageChain
cp /path/to/AnomalyDamageChain/plugin/* plugins/AnomalyDamageChain/

build.cmd            # 需要 VS2022 + Windows SDK
```

构建后再由 `build.cmd` / CI 重新生成 `abi/anomaly-sdk-v1-windows-x64.json`；该文件是生成物，不要手改。

## 状态

- 代码未在本仓库编译：仓库只存放源码与补丁。
- 未在游戏内验证。
- 已知限制见 `docs/design.md`。
