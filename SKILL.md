---
name: dsh-plugin-update
description: 更新、回滚和排查 DeepSeek Harness（DSH）第三方插件。使用前核对官方文档与实际安装版本；官方机制和本机实测经验分别标明。用于插件升级失败、重启后未生效、工具调用报 prepare 错误、钩子异常，以及 dsh plugin 用法。
metadata:
  agent_created: "true"
---

# DSH 第三方插件更新指南

## 铁律：用之前先读官方文档

操作方法依据 DeepSeek Harness 官方文档；本机故障和修复结果另标为实测经验。每次使用前，先核实桌面版/CLI 的实际版本与 profile，再联网核对以下原文；与本文不一致时以对应版本的官方资料为准：

- 官方仓库：https://github.com/deepseek-ai/deepseek-harness
- CLI 插件命令说明：https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/apps/cli/README.md
- 桌面版说明（Bundled command runtime 一节）：https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/apps/desktop/README.md
- 插件管理器（兼容性与豁免）：https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/packages/boot/plugin-manager/README.md

链接失效时，到官方仓库对应路径找最新 README。

最新分支的接口可能已不同于已安装版本。排障还要核对对应发布标签的文档和源码；例如本次 `0.2.1-rc.3` 对应 `dsh-v0.2.1-rc.3`，其他版本先从官方发布页核实标签，不猜名称。不熟悉的配置、服务接口、钩子输出协议，查清后再改。

安装成功、独立导入成功、界面能打开，都不能证明工具和钩子可用。记录当前证据能证明到哪一步；用户已授权恢复功能时，继续完成恢复与功能验证，不把临时停用当成最终交付。

## 先分清是哪种环境

插件管理分两种环境，命令不能混用：

- 桌面版（Electron 应用）：自带的 profile 名为 `desktop`。官方原文：npm 安装的 dsh 会拒绝它的插件管理请求，必须用桌面版自带的 dsh 命令来管理。
- 普通 npm CLI：自己创建并管理其他 profile。

桌面版 profile 目录（Windows）：`%USERPROFILE%\.dsh\profiles\desktop\`，核心文件 `package.json` 记录各插件依赖版本。

## 桌面版更新流程（官方原文）

官方文档原文：

"To manage Desktop's plugins, launch Desktop once to initialize its profile, fully quit the application, then run `dsh plugin --profile desktop add <package>`, `list`, or `remove <package>`. Reopen Desktop to use the changes."

步骤：

1. 启动一次桌面版，初始化 profile（已经用过则可跳过）。
2. 完全退出桌面版（不是只关窗口，从任务栏/托盘退出）。
3. 用桌面版自带命令执行，Windows 下路径一般为：
   `%LOCALAPPDATA%\Programs\DeepSeek Harness\resources\runtime\cli\bin\dsh.cmd`
   官方说明：该脚本用桌面版自带的 Electron 与内置 pnpm 运行，桌面版关闭时也能用。
   常用命令：
   - 查待更新：`dsh.cmd plugin --profile desktop outdated`
   - 更新：`dsh.cmd plugin --profile desktop add 包名@版本`
   - 已装列表：`dsh.cmd plugin --profile desktop list`
   - 卸载：`dsh.cmd plugin --profile desktop remove 包名`
   `dsh plugin` 的其余 pnpm 动作会原样转发给 pnpm 在 profile 目录执行。
4. 重新打开桌面版，改动生效。

## 版本没变怎么办

`dsh plugin` 把参数原样转发给 pnpm 在 profile 目录执行，排查就按 pnpm 这条线走：

1. 先读命令输出和错误信息，里面通常给出诊断日志位置，日志里是真正原因。
2. 若被 pnpm 的新版本冷却策略挡住（pnpm 11 起 `minimumReleaseAge` 默认 1440 分钟，发布不满 1 天的版本不会被采用，`outdated` 里根本不显示它）：要用官方排除机制放行——在 `%USERPROFILE%\.dsh\profiles\desktop\pnpm-workspace.yaml` 的 `minimumReleaseAgeExclude` 里点名 `包名@版本`，再执行 `add 包名@版本`。点名精确版本本身不能绕过这个策略。放行等于跳过新版本冷却保护，必须先让用户知情同意。pnpm 官方文档：https://pnpm.io/settings/dependency-resolution#minimumreleaseage 和 https://pnpm.io/supply-chain-security 。
3. 若被版本兼容检查拒绝：见下一节。
4. （实测）更新会连带升级关联包：pnpm 会一起更新它认定的"首选版本"依赖，例如更新 dshmarket@1.66.42 时顺带把 billion-context 从 0.1.201 升到 0.1.202，并自动往 `minimumReleaseAgeExclude` 补一条。装完要核对完整依赖清单和 node_modules 实际版本，不能只看目标包。

## 兼容性检查与豁免（官方）

官方说明：安装和 profile 启动都会校验插件的 `engines.dsh` 范围（缺省时用 DSH peer 范围），对照 `dsh --version` 显示的运行时版本；不兼容插件必须显式豁免精确版本才能装上或启动。

豁免手段：`dsh plugin allow-version` / `revoke-version`（持久化与风险见官方 plugin-manager README 的 version-compatibility-and-exemptions 一节）。

不要为了省事强行豁免：不兼容插件可能崩溃或丢数据，必须让用户知情同意。

## 回滚

改动前按本机规则备份并校验拟改动的 `package.json`、锁文件、`cordis.patch.yml`、`pnpm-workspace.yaml` 及相关脚本；将要移动或替换的插件目录也要备份。保留原编码与 BOM。出问题时，恢复相互匹配的配置和依赖，或用 `add 包名@旧版本` 装回旧版；不能只恢复一个文件就声称已回滚。

## 更新后检查

1. 检查 `dsh.cmd plugin --profile desktop outdated` 的退出码和输出；成功且无待更新项，只说明在当前源与发布冷却策略下没有可见更新。
2. 打开 profile 的 `package.json` 核对版本号。
3. 完全退出再重开桌面版，确认插件功能正常。
4. 打开插件市场逐个看状态：显示「已生效」才算真正跑起来；仍显示「已安装，重启后生效」的，按下一节排查。
5. 实际调用受影响的工具。钩子要同时验证正常操作放行、应拦截操作被拦截、理由正确、目标文件未被误改，以及拦截后普通工具仍可使用。文件读写验证应在真实 Agent 会话中完成，不需要为此额外发送模型请求。

## 重启后市场仍显示「已安装，重启后生效」

这句不是误报。市场的判定是「包装好了，但当前运行的进程里没加载它」；插件在启动时加载失败就会被跳过，所以反复重启也不会变好，要查加载报错。

诊断（本机实测经验，非官方文档，仅作排查参考）：

1. 看市场接口给的实时状态。桌面版本地端口上有一个不需要登录的接口，把端口换成当前地址里的端口：
   `http://127.0.0.1:<端口>/dsh-market/installed`
   在返回的 `activation` 里找该插件，`state` 的含义：`live` 正在运行；`restart` 已装但没加载（就是这句提示的来源）；`inert` 只是普通依赖、不是插件。`reasons` 字段写了原因。
2. 验证插件能不能被导入。到 profile 目录执行：
   `cd %USERPROFILE%\.dsh\profiles\desktop`
   `node -e "import('插件包名').then(m => console.log('OK')).catch(e => console.error(e.message))"`
   常见报错 `does not provide an export named ...` 表示插件和它依赖的公共库版本不匹配——新插件版本用了新库才有的函数，而 profile 里被依赖的库还是旧版。
3. 验证插件的服务端部分有没有挂上：请求插件自己的路由。返回 404 通常说明没挂上，403 或 200 说明在运行（具体路由看插件 package.json 的 dsh 字段或 cordis.patch.yml）。

修复：

- 第三方共享库落后：核对插件声明和缺失导出后，再单独升级对应库，例如 `dsh.cmd plugin --profile desktop update 依赖名`，必要时点名精确版本。不要将此办法套到桌面版已提供的核心库上：遇到缺依赖先核对官方运行时的加载规则，不能为让独立导入通过就批量补装 `@deepseek-ai/dsh-*`。
- 插件本身有 bug：回滚到上一个版本，见「回滚」。
- 动手前先备份 profile 的 package.json 与锁文件，出问题按「回滚」还原。

实测案例：dsh-md-render 0.5.2 导入时报 `The requested module 'dsh-shared' does not provide an export named 'createTrustFence'`，原因是 profile 里 dsh-shared 还是 0.1.6，而 0.5.2 需要 0.1.7 才有的 createTrustFence；单独把 dsh-shared 升到 0.1.7 后导入正常，重启后插件正常加载。

## 工具调用报 prepare 错误或钩子异常

遇到 `Cannot read properties of undefined (reading 'prepare')`，或安装钩子依赖后所有工具失效，先读[2026-10-25 钩子与核心库冲突复盘](references/2026-10-25-钩子与核心库冲突.md)。其中区分官方机制、本机证据、验证脚本错误和最终验收结果。

- 先从会话日志和同版本源码定位报错层次，不凭“刚装了钩子”就判定钩子不兼容或官方尚未修复。
- 核对 profile 与桌面安装包里的核心库实际加载来源；相同版本的两份实体包也可能冲突。只移除依赖声明不一定消除磁盘上的副本，单独停用桥接也不一定恢复调度。
- 只有确认额外副本抢先加载时，才在停止目标 DSH、备份并校验后，将相关副本移到集中备份目录，保持依赖清单和锁文件一致。复盘中的 8 个包是本次证据范围，不是通用删除清单。
- 钩子结构化 JSON 输出的退出码、输出流和字符编码，要按对应版本协议核对。正常工具能用后，在已授权的恢复范围内重新启用并验证钩子；临时停用只是排障步骤。
- 只有证据确认存在当前无法绕过的上游阻碍时，才报告未完成项和依据；不能把尚未验证写成不能用，也不能把核心库冲突直接归为钩子插件 bug。
