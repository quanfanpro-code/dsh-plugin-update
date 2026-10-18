---
name: dsh-plugin-update
description: 更新 DeepSeek Harness（DSH）第三方插件的操作指南。铁律：使用前必须先联网核对官方最新文档；内容仅来自官方文档（仅排查章节含实测经验，已标注）。当用户要求更新或升级 DSH 插件、DSH 插件市场更新报错、重启后仍显示重启后生效、查询哪些插件有新版本、回滚插件版本，或询问 dsh plugin 用法时使用。
agent_created: true
---

# DSH 第三方插件更新指南

## 铁律：用之前先读官方文档

本技能只依据 DeepSeek Harness 官方文档编写。官方文档会变化，每次使用前必须联网核对以下原文；与本文不一致时以官方原文为准，并顺手更新本技能：

- 官方仓库：https://github.com/deepseek-ai/deepseek-harness
- CLI 插件命令说明：https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/apps/cli/README.md
- 桌面版说明（Bundled command runtime 一节）：https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/apps/desktop/README.md
- 插件管理器（兼容性与豁免）：https://raw.githubusercontent.com/deepseek-ai/deepseek-harness/master/packages/boot/plugin-manager/README.md

链接失效时，到官方仓库对应路径找最新 README。

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
2. 若被 pnpm 供应链策略拦截（例如 minimumReleaseAge 不采用刚发布不久的版本）：直接点名精确版本 `add 包名@x.y.z`。这是 pnpm 官方行为，见 pnpm 官方文档：https://pnpm.io/supply-chain-security 和 https://pnpm.io/settings 。
3. 若被版本兼容检查拒绝：见下一节。

## 兼容性检查与豁免（官方）

官方说明：安装和 profile 启动都会校验插件的 `engines.dsh` 范围（缺省时用 DSH peer 范围），对照 `dsh --version` 显示的运行时版本；不兼容插件必须显式豁免精确版本才能装上或启动。

豁免手段：`dsh plugin allow-version` / `revoke-version`（持久化与风险见官方 plugin-manager README 的 version-compatibility-and-exemptions 一节）。

不要为了省事强行豁免：不兼容插件可能崩溃或丢数据，必须让用户知情同意。

## 回滚

改动前先备份 profile 的 `package.json`。出问题：恢复备份重新安装，或 `add 包名@旧版本` 装回旧版。

## 更新后检查

1. `dsh.cmd plugin --profile desktop outdated` 无输出即全部最新。
2. 打开 profile 的 `package.json` 核对版本号。
3. 完全退出再重开桌面版，确认插件功能正常。
4. 打开插件市场逐个看状态：显示「已生效」才算真正跑起来；仍显示「已安装，重启后生效」的，按下一节排查。

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

- 依赖库落后：把那个公共库单独升级，例如 `dsh.cmd plugin --profile desktop update 依赖名`（不行就点名精确版本 `add 依赖名@版本号`），升级后再跑一次导入测试，通过后重启桌面版。
- 插件本身有 bug：回滚到上一个版本，见「回滚」。
- 动手前先备份 profile 的 package.json 与锁文件，出问题按「回滚」还原。

实测案例：dsh-md-render 0.5.2 导入时报 `The requested module 'dsh-shared' does not provide an export named 'createTrustFence'`，原因是 profile 里 dsh-shared 还是 0.1.6，而 0.5.2 需要 0.1.7 才有的 createTrustFence；单独把 dsh-shared 升到 0.1.7 后导入正常，重启后插件正常加载。
