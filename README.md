# Pi 源码学习笔记

这组笔记研究 [`earendil-works/pi`](https://github.com/earendil-works/pi) 的工程实现。内容按功能和调用链组织，不按目录逐个翻译。

当前基线：`5d9fedf73cc2ff5c39cedf9bb91827e28e3facaf`（`v0.80.7-13-g5d9fedf73`）。每篇笔记会单独注明所依据的 commit；源码更新后，旧笔记不会自动视为适用于新版。

## 课程

| 讲次 | 主题 | 状态 |
| --- | --- | --- |
| 00 | [项目身份、源码基线与工程地图](docs/00-project-map.md) | 已完成 |
| 01 | [一条 prompt 的端到端主链](docs/01-prompt-lifecycle.md) | 已完成 |
| 02 | [消息、上下文与状态不是同一个对象](docs/02-message-context-state.md) | 已完成 |

后续课程会进入低层 Agent 循环、工具执行、模型接入、会话树、上下文压缩、安全和多实例协调。
