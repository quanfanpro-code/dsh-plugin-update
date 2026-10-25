# dsh-plugin-update

给 AI 代理用的技能：更新、回滚和排查 DeepSeek Harness（DSH）第三方插件。

## 内容来源

操作方法依据 DeepSeek Harness [官方文档与源码](https://github.com/deepseek-ai/deepseek-harness)，本机实测经验单独标明。每次使用前核实实际安装版本和 profile，同时核对最新说明与对应版本资料，避免套用已经变化的接口。

## 安装

把整个 `dsh-plugin-update` 文件夹复制到你的技能目录（AI 工具读取 `SKILL.md` 的目录）。常见位置：

- `%USERPROFILE%\.workbuddy\skills\`
- `%USERPROFILE%\.dsh\skills\`

具体以你的 AI 工具配置为准，放入后重启工具即可在技能列表看到。

## 适用场景

- 更新桌面版 DSH 的第三方插件
- 插件市场里点更新报错
- 查询哪些插件有新版本、回滚插件
- 不清楚 `dsh plugin` 命令怎么用
- 安装插件依赖后工具调用报 `prepare` 错误、钩子异常

## 故障复盘

[2026-10-25 钩子与核心库冲突复盘](references/2026-10-25-钩子与核心库冲突.md)记录本次故障的起因、证据、走错的步骤、修复方法和六项实测结果。重点是先读同版本官方资料、区分重复核心库与第三方共享库问题，以及恢复后继续验证钩子，不能停在临时禁用。

## 许可

AGPL-3.0，见 [LICENSE](LICENSE)。

## 声明

非官方项目，与 DeepSeek 无隶属或合作关系。
