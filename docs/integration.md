# 集成步骤

## 前置

- 框架源码：`Rabb1t79/Anomaly-main`（本补丁基于提交 `c64a2a39`）
- VS2022 Build Tools + Windows 11 SDK（当前开发机缺 SDK，无法本地编译）
- 部署环境：游戏目录下 `Anomaly/`（runtime 由该目录加载）

## 1. 应用 Core 补丁

```bash
cd Anomaly-main
git apply --check core/0001-damage-chain-integration.patch   # 先检查
git apply core/0001-damage-chain-integration.patch
```

补丁涉及六个文件：

| 文件 | 作用 |
| --- | --- |
| `include/anomaly/sdk/services/nte.h` | 服务 ID/版本与结构体 |
| `src/plugin/plugin_capability_policy.cpp` | 能力 `nte-damage-chain` 与服务映射 |
| `profiles/nte/nte-current.json` | 符号、feature、validator 声明 |
| `src/game/nte/nte_profile_runtime.cpp` | `ValidateDamageChainAbi` 与注册 |
| `src/game/ue5/ue5_nte_adapter.cpp` | 服务实现、可用性判断与发布 |
| `apps/abi_snapshot/main.cpp` | ABI 快照条目 |

## 2. 放入插件

```bash
mkdir -p plugins/AnomalyDamageChain
cp plugin/plugin.cpp plugin/manifest.json plugins/AnomalyDamageChain/
```

## 3. 构建

```bat
build.cmd
```

构建会同时生成 ABI 快照 `abi/anomaly-sdk-v1-windows-x64.json`。该文件是生成物，
本次改动没有手写它；必须在构建后随 Core 一起提交，否则 ABI 快照与头文件不一致。

## 4. 部署与验证顺序

1. 启动游戏，检查 runtime 日志中 `profile.ready`：
   - 出现 `ue5.process-event` 等既有特性状态；
   - 若 `nte.damage-chain` 不可用，日志会带 `damage-chain ... instruction contract changed` 或 `unavailable`。
2. 插件面板显示 `绑定=就绪`（两个钩子都建立）才说明服务地址或降级签名可用。
3. **先只验证不重放**：关闭「启用」，打怪一次，确认原伤害正常、游戏不崩。
4. 再开启「启用」，速率设为 2，观察 `伤害/重放/失败` 计数与目标血量。
5. 递增速率（5 → 15），每次确认稳定性。

## 5. 排查

| 现象 | 检查点 |
| --- | --- |
| `绑定=等待` | 服务未发布：看 `profile.ready` 里 `nte.damage-chain` 的 `validationDiagnostics` |
| 计数全为 0 | 钩子未建立，或判定函数始终返回假 |
| `失败` 增长 | 补发时对象失效，链已被保护性停止 |
| 进程崩溃 | 取 `%LOCALAPPDATA%/HT/Saved/Crashes` 最新 `UEMinidump.dmp`，用出错 RVA 对照 `docs/design.md` 的目标地址 |
