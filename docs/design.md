# 设计说明

## 目标

一次命中之后，按速率重复施加同一份伤害判定；目标失效（或链停止）时结束。

## 两个原生目标

由运行期抓取调用者并回查 188MB 代码段确定：

| 角色 | RVA | 说明 |
| --- | --- | --- |
| Apply（外层） | `0x7682FF0` | 每次伤害施加都会被调用；由派发器在 `0x789AB86` 经虚表槽 `0x318` 调用 |
| Gate（内层） | `0x767F0D0` | 返回 bool，决定本次是否真正施加伤害 |

签名来源：Tokky 的 AOB 表 `pair2`，两条在当前版本各唯一命中。
Apply 前 16 字节 `48 85 D2 0F 84 41 01 00 00 57 48 83 EC 60 48 8B`，
Gate 前 16 字节 `48 8B C4 48 89 58 08 48 89 70 18 48 89 78 20 55`。

两者都带栈参数（Win64：`[rsp+0x28]` 起），因此 detour 必须是手写机器码，
不能用普通 C 函数转发。

## 契约遵循

| 规则 | 实现 |
| --- | --- |
| detour 必须在 `begin_callback` / `end_callback` 内 | 外层与判定两处 detour 都领回调契约；`release` 依赖其计数等待在途调用 |
| 非易失寄存器 | 所有 stub 保存/恢复 `RSI`、`RDI`；外层另存 `XMM0-XMM5` |
| 栈参数位置 | 抓取自 `[rsp+0x28]`，重建时放到 `[rsp+0x20]` |
| 展开信息 | 三个 stub 统一序言（`mov r11,rsp` / `push rdi` / `push rsi` / `sub rsp,0xA8`，第 12 字节结束），生成 `UNWIND_INFO` 并用 `RtlAddFunctionTable` 注册；注册失败则不安装钩子 |
| 线程归属 | `Start` 记录游戏线程，回调比对；非本线程只转发不改状态 |
| 签名归属 | Profile 声明符号，Core validator 校验字节契约，插件通过服务取地址 |

## 校验链

`profiles/nte/nte-current.json` 新增：

```json
"nte.DamageChain.Apply": { "pattern": "...", "requiredBy": ["anomaly.nte.damage-chain"] },
"nte.DamageChain.Gate":  { "pattern": "...", "requiredBy": ["anomaly.nte.damage-chain"] },
"features": { "nte.damage-chain": ["nte.DamageChain.Apply", "nte.DamageChain.Gate"] },
"featureLayoutValidators": { "nte.damage-chain": ["nte-damage-chain-abi-v1"] }
```

Core 注册 `ValidateDamageChainAbi`：符号可用、有 unwind 记录、入口字节契约匹配。
任一不满足，feature 判为不可用，服务不发布，`profile.ready` 中出现对应诊断。

插件侧两级降级：服务为空 → 退回自带签名；仍取不到 → 只告警、不挂钩。

## 已知限制

1. **单链**：同一时刻只维护一条链，切换目标会覆盖参数；未实现多目标并行。
2. **参数语义未经 Dump 证实**：`rdx`/`r8` 与 `[rdx+0xA8]` 的含义来自反汇编推断，未对照原生 Dump。
3. **RWX 内存**：stub 页面为可写可执行，未在生成后收窄权限。
4. **重放的异常安全**：stub 已注册展开信息，但游戏函数内部抛出的异常仍可能终止进程。
