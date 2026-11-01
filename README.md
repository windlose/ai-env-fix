# Windows 上把 AI 开发环境装通

一个程序员在 Windows 上装 AI 工具时踩过的坑，以及对应的修法。每一条都是真机上实际遇到并修好的，不是转述。

**服务页**：https://windlose.github.io/ai-env-fix/

---

## 三篇排错记录

| # | 文章 | 讲什么 |
|---|---|---|
| 01 | [Windows 上装 AI 开发环境，我踩过的 8 个坑](文章/01-Windows-AI环境避坑清单.md) | 总览。八个静默失败，每个都有现象、根因、修法 |
| 02 | [装到一半的 node_modules](文章/02-node_modules装到一半.md) | `pnpm install` 显示成功但目录缺 `package.json`——Windows 文件锁导致的半截安装 |
| 03 | [插件装了却不显示](文章/03-插件装了不显示.md) | 依赖有、配置有、日志干净，界面没按钮——peer 版本上限的静默禁用 |

---

## 核心结论

**Windows 上的失败大多是静默的。**

- 退出码 0
- 日志没有 error
- 界面看起来正常
- 功能是坏的

所以排查的第一原则不是看错误信息，是**验证产出物**：

| 别信 | 要验 |
|---|---|
| `pnpm install` 没报错 | `node_modules/<包>/package.json` 在不在 |
| 插件在 `dependencies` 里 | boot 图里有没有它的名字 |
| `health = ok` | 写入一条，能不能读回来 |
| `commit = accepted` | 归档目录里有没有 `.done` 和结果文件 |

---

## 联系

需要远程协助，看服务页：https://windlose.github.io/ai-env-fix/

---

## 许可

文章内容可自由转载，注明出处即可。
