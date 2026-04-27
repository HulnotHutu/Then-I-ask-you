# Then-I-ask-you

> 借助 AI 深度理解技术文档的学习工作台 —— 让 AI 替你读 RFC、man 手册、论文，你只需要提问和修正认知。

## 动机

人阅读文档的速度远不如 AI。面对动辄上百页的 RFC、晦涩的 man 手册、冗长的学术论文，正常人的反应往往是"下次一定"。

这个项目的思路是：**把"读文档"这件事交给 AI**。你只需要向 AI 提问、质疑 AI 的理解、补充新的参考资料，AI 负责完成预览 → 速读 → 精读 → 总结的全过程，并以结构化的"三段式认知法"输出，帮助你建立对概念的深层理解。

## 工作流程

整个工作台围绕"认知三段式"展开，分为三个阶段：

### 阶段一：初始化

```
documents/   →   AI 阅读 & 学习   →   knowledge.md   →   answer.md
```

1. AI 扫描 `documents/` 目录，按**预览 → 速读 → 精读 → 三句话复述**的流程内化文档
2. AI 将学习成果总结写入 `documents/knowledge.md`（后续回答只读 knowledge.md，不再重复扫描原始文档）
3. AI 基于 knowledge.md 生成 `answer.md`（三段式：类比 → 认知修正 → 精确定义）

### 阶段二：认知修正

```
用户提问/描述   →   AI 检索 knowledge.md   →   更新 answer.md 的"但它不完全是……"
```

- 用户阅读 `answer.md` 后产生新的疑问或提出自己的理解
- AI 根据 knowledge.md 判断用户理解中的认知偏差
- AI 在 `answer.md` 的纠正表中追加新条目，并对用户的描述打分（0-10）

### 阶段三：认知补全

```
引用新文档   →   AI 学习新文档   →   扩充 knowledge.md   →   更新 answer.md 的"它到底是什么？"
```

- 用户通过 `reference` 引入新的参考文档
- AI 学习新文档，将新知识章节追加到 `knowledge.md`
- AI 更新 `answer.md` 的精确定义部分

## 支持的文件类型

| 类型 | 说明 | 状态 |
|:---|:---|:---|
| RFC 文档 | 来自 rfc-editor.org 的 txt 文件 | 已支持 |
| man 手册 (tool) | 可执行程序 / shell 命令 (section 1) | 已支持 |
| man 手册 (syscall) | 系统调用 (section 2) | 已支持 |
| man 手册 (library) | 库调用 (section 3) | 计划中 |
| PDF | 学术论文等 | 计划中 |
| Web 文章 | 在线技术文章 | 计划中 |

### man 手册章节参考

| Section | 内容 |
|:---|:---|
| 1 | 可执行程序或 shell 命令 |
| 2 | 系统调用（内核提供的函数） |
| 3 | 库调用（程序库中的函数） |
| 4 | 特殊文件（通常位于 /dev） |
| 5 | 文件格式和规范（如 /etc/passwd） |
| 6 | 游戏 |
| 7 | 杂项（包括宏包和规范） |
| 8 | 系统管理命令（通常仅 root） |
| 9 | 内核例程（非标准） |

核心聚焦在 **tool（section 1）** 和 **function（section 2/3）** 两类内容。

## 目录结构

```
workbench/
├── .excalidraw/          # 用户板书文件（executor 不可读）
├── prompt/
│   ├── system-prompt.txt # executor 的系统提示词
│   └── attack.txt        # 特殊交互模式定义
├── documents/            # executor 可读取的参考文档
│   ├── tcp.txt           # RFC 793 原始文档
│   ├── TCP-Selective-Acknowledgment-Options.txt
│   ├── D-SACK-for-Extension.txt
│   └── knowledge.md      # AI 总结提炼的知识库
├── man/                  # man 手册 txt 文件（tool / syscall）
│   ├── grep.txt
│   └── mmap.txt
├── answer.md             # executor 输出的三段式回答
├── summary.md            # man 手册的学习摘要（追加写入）
├── questions.md          # 用户提出的问题记录
└── README.md
```

### 权限边界

- `executor` 可读范围：`documents/*`、`man/*`
- `executor` 可写范围：`answer.md`、`summary.md`、`questions.md`、`documents/knowledge.md`
- `executor` **不可读**：`.excalidraw/`、`user.md`、`extrend/`
- `user.md`（用户自己的思路总结）和 `extrend/`（知识扩展）对 executor 完全隔离

## 交互模式

除标准学习模式外，三种特殊交互模式已实现为独立 skill（`/reverse-role`、`/brainstorm`、`/socratic`）：

| 模式 | Skill | 调用方式 | 说明 |
|:---|:---|:---|:---|
| 反客为主 | `reverse-role` | `/reverse-role [议题]` | AI 扮演文档作者，为设计决策辩护 |
| 头脑风暴 | `brainstorm` | `/brainstorm [论点]` | AI 坚守论点，用户不断反驳，AI 必须说服用户 |
| 不友好 | `socratic` | `/socratic [话题]` | 苏格拉底式诘问，只提问不给答案，迫使用户自己推导 |

设计选型分析见 `prompt/attack.txt`（原始设计草图）。选择 skill 而非 system-prompt 模式切换的原因：
- Skill 天然隔离三种差异极大的对话风格，互不干扰
- 用户通过 slash command 精确切换，不会误触发
- 每个 skill 独立维护，修改一个不影响其他

## 进度

- [x] `knowledge-summary` — 通用 txt 文档的认知三段式学习
- [x] `reverse-role` — 反客为主：AI 扮演作者为设计辩护
- [x] `brainstorm` — 头脑风暴：AI 坚守论点接受反驳
- [x] `socratic` — 苏格拉底式诘问：只提问不直接给答案
- [x] `command-summary` — Tool 手册的学习摘要（3 层渐进：速查 → 理解 → 避坑）
- [x] `syscall-summary` — Syscall 手册的学习摘要（3 层渐进 + 内核视角）

## 致谢

本项目受 [deepseek-v4](https://deepseek.com) 驱动，核心方法论来自"三段式认知法"（类比建立直觉 → 修正认知偏差 → 精确定义）。
