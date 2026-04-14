---
title: Harness工程系列：让 Agent 拥有持久记忆的多会话续接
date: 2026-04-14
tags:
  - Harness Engineering
  - AI Agent
  - 工作流优化
  - Claude Code
description: 通过 Initializer Agent + Coding Agent 两阶段模式，将项目状态外部化到磁盘，让每个新会话都能完整恢复上下文，彻底解决 Agent "失忆"难题。
---
## 摘要

用 AI Agent 做多会话项目时，最大的浪费不是 Token，是每次新开会话都在"教 Agent 认路"——项目骨架、已完成的工作、下一步该做什么，Agent 每次都像失忆了一样从头开始。

本文介绍基于 Anthropic 官方博客总结的**两阶段工作流**：用 Initializer Agent 画地图，用 Coding Agent 干活，配合交接笔记实现跨会话状态续接。四个模板拿来即用。

## 问题：Agent 的"失忆症"

一个典型场景：

**第一轮会话**：你让 Agent 搭建 Express + PostgreSQL 后端。它完成了数据库 schema、基础 CRUD 接口，提交了代码。

**第二轮会话**（新窗口）：你说"继续，把用户认证加上"。Agent 回应："好的！让我先看看项目结构……我来帮你创建 Express 项目吧！"——它不记得项目已经建好了。

**第三轮会话**：你说"给 Todo 加上分类标签功能"。Agent 把你上一轮刚加好的认证中间件删了，因为它觉得"这段代码好像没用"。

三轮下来，花在"教 Agent 认路"上的时间比写代码还多。问题的根源不在模型，在于**会话之间没有交接机制**。Agent 的"记忆"本质上是上下文窗口里的对话历史——窗口一关，记忆就归零。

## 解法：两阶段模式

Anthropic 团队提出了一个模式：**把 Agent 分成两个角色，分阶段工作**。

### Phase 1：Initializer Agent（首次会话）

只在项目开始时跑一次，职责是"画地图"：

- 理解项目需求，生成完整的 feature list
- 搭建项目骨架和目录结构
- 创建 `init.sh` 环境初始化脚本
- 创建 `claude-progress.md` 进度追踪文件
- 做第一次 git commit

**关键约束**：Initializer 不写业务代码。它的全部任务是把"地图"画好，给后面的 Coding Agent 铺路。

### Phase 2：Coding Agent（后续每一轮会话）

每次新开会话都用这个角色，职责是"干活"：

1. 先读 `claude-progress.md`，搞清楚当前进度
2. 从 `feature_list.json` 里拿下一个 pending 任务
3. **只做这一个任务**
4. 做完跑 `verify.sh`
5. 更新进度文件
6. 写交接笔记给下一轮会话

**关键约束**：Coding Agent 每次开工前的第一件事不是写代码，是读笔记。

## 四个模板——实现多会话续接

### 模板一：Initializer 提示词模板

```markdown
# Initializer Agent 指令

你是项目的 Initializer Agent。你的任务是搭建项目骨架，不写业务代码。

## 你要做的事

### 1. 理解需求
阅读以下需求描述，理解项目目标：
[在这里粘贴你的项目需求]

### 2. 生成 Feature List
创建 `feature_list.json`，把需求拆解为可独立实现的 feature。

每个 feature 包含：
- id：唯一标识（feat-001, feat-002...）
- name：功能名称
- description：一句话描述
- status：全部设为 "pending"
- priority：high / medium / low
- dependencies：依赖哪些其他 feature（id 列表）
- acceptance_criteria：验收标准（3-5 条具体行为）

要求：
- 按优先级和依赖关系排序
- 每个 feature 的粒度控制在 1-2 个会话能完成
- 不合并多个独立功能到一个 feature

### 3. 搭建项目骨架
创建基础目录结构和配置文件，包括：
- 项目配置（package.json / tsconfig.json 等）
- 目录结构（src/ tests/ docs/ contracts/）
- 基础的 lint 和 typecheck 配置

### 4. 创建 Harness 文件
- `AGENTS.md`：Agent 操作手册
- `init.sh`：环境初始化脚本
- `verify.sh`：验证管线
- `claude-progress.md`：初始进度记录

### 5. 首次提交
完成上述所有文件后，做一次 git commit。
消息格式：`feat: initialize project skeleton and harness files`

## 禁止事项
- 不写任何业务逻辑代码
- 不实现任何 feature
- 不修改 feature_list.json 中的状态（全部保持 pending）
```

### 模板二：Coding Agent 提示词模板

```markdown
# Coding Agent 指令

你是项目的 Coding Agent。你的任务是逐个实现 feature。

## 开工前必读

1. 先读 `claude-progress.md`，了解项目当前进度
2. 再读 `feature_list.json`，找到下一个 pending 且无阻塞依赖的 feature
3. 读该 feature 对应的 `contracts/feat-xxx.contract.json`（如果有）

## 工作规则

### 作用域约束（最重要）
- 整个会话只处理**一个** feature
- 只修改与当前 feature 直接相关的文件
- 不触碰已完成 feature 的代码和测试
- 不顺手重构、不顺手优化、不顺手清理

### 一次只做一个 feature
- 从 feature_list.json 中选择优先级最高的 pending feature
- 如果该 feature 有 dependencies 且依赖项未完成，跳到下一个

### 编码标准
- TypeScript strict mode
- 每个功能函数必须有对应的单元测试
- 新增文件不超过 Sprint Contract 中的 max_new_files 限制

### 完成验证
- 编码完成后运行 `./verify.sh`
- 全部通过后，将 feature_list.json 中该 feature 的 status 改为 "done"
- 更新 `claude-progress.md`
- git commit，消息格式：`feat(feat-xxx): <简短描述>`

### 会话结束前必做
在 `claude-progress.md` 末尾追加交接笔记（格式见下方模板）

## 禁止事项
- 不同时处理多个 feature
- 不修改其他 feature 的代码（除非是当前 feature 的直接依赖）
- 不跳过 verify.sh 直接提交
- 不删除或修改已完成 feature 的代码
```

> **设计意图**：把"作用域约束"直接写进提示词，是为了从**提示词层面堵住 Agent "顺手改一改"** 的冲动。作用域是贯穿所有模板的底层规则。

### 模板三：增强版 init.sh

```bash
#!/bin/bash
# init.sh — 会话初始化脚本（增强版）
# 每个新会话开始时运行

set -e

echo "========== 会话初始化 =========="

# 1. 环境检查
echo "[1/5] 检查运行环境..."
node_version=$(node -v 2>/dev/null || echo "未安装")
echo "  Node.js: $node_version"

if ! command -v node &> /dev/null; then
    echo "  ✗ Node.js 未安装，请先安装"
    exit 1
fi

# 2. 依赖安装
echo "[2/5] 安装依赖..."
if [ -f "package-lock.json" ]; then
    npm ci --silent
else
    npm install --silent
fi
echo "  ✓ 依赖安装完成"

# 3. 环境变量检查
echo "[3/5] 检查环境变量..."
required_vars=("DATABASE_URL")
for var in "${required_vars[@]}"; do
    if [ -z "${!var}" ]; then
        echo "  ⚠ 缺少环境变量: $var（检查 .env 文件）"
    fi
done
echo "  ✓ 环境变量检查完成"

# 4. 现有测试验证
# 设计意图：确保上一轮留下的代码是能跑的，避免"坏代码"污染新会话
echo "[4/5] 运行现有测试..."
if npm run test --silent 2>/dev/null; then
    echo "  ✓ 现有测试全部通过"
else
    echo "  ✗ 现有测试有失败，请先修复再继续"
    exit 1
fi

# 5. 输出当前进度
# 设计意图：新会话一开场就能看到上一轮做了什么，减少"认路"时间
echo "[5/5] 当前项目进度："
echo "---"
if [ -f "claude-progress.md" ]; then
    tail -20 claude-progress.md
else
    echo "  （无进度记录）"
fi
echo "---"

echo "========== 初始化完成，可以开工 =========="
```

### 模板四：交接笔记格式

```markdown
---

## 会话记录 — 2026-04-11 14:30

### 本次完成
- **Feature**: feat-003 用户注册接口
- **改动文件**:
  - `src/routes/auth.ts` — 新增 POST /api/auth/register
  - `src/validators/user.ts` — 新增邮箱和密码校验
  - `tests/auth.register.test.ts` — 新增 8 个测试用例
- **验证结果**: verify.sh 全部通过

### 遇到的问题
- PostgreSQL 的 unique constraint 报错信息不够友好，临时用了 try-catch 包装。后续可优化为自定义错误类型。
- （如有异常中断：记录异常类型 / 触发条件 / 是否已修复，避免下一轮踩同一个坑）

### 下一步建议
- 下一个 pending feature: **feat-004 用户登录接口**
- 注意：feat-004 依赖 feat-003 的用户表结构，不需要重新建表
- `src/routes/auth.ts` 已有基础路由结构，在此基础上新增登录路由即可

### 注意事项
- 不要改动 `src/middleware/errorHandler.ts`，这个文件上一轮刚稳定下来
- 测试数据库用的是 test_db，连接串在 .env.test 里
```

> **设计意图**："下一步建议"是最值钱的部分。它告诉下一个会话——该干什么、有什么需要注意的、哪些文件可以复用。有了它，新会话不用"猜"，只要"读"。

## 完整的多会话工作流

把四个模板串起来，完整流程：

```
会话 1（Initializer Agent）
    │
    ├─ 理解需求
    ├─ 生成 feature_list.json（示例：20 个 feature）
    ├─ 搭建项目骨架
    ├─ 创建 harness 文件
    ├─ git commit: "feat: initialize project"
    └─ 写交接笔记 → claude-progress.md

会话 2（Coding Agent）
    │
    ├─ 运行 init.sh → 环境好、测试通过
    ├─ 读 claude-progress.md → 知道从哪接
    ├─ 拿 feat-001 → 编码 → verify.sh → 通过
    ├─ git commit: "feat(feat-001): 用户表和基础模型"
    └─ 写交接笔记 → claude-progress.md

会话 3（Coding Agent）
    │
    ├─ 运行 init.sh → 环境好、测试通过
    ├─ 读 claude-progress.md → feat-001 已完成
    ├─ 拿 feat-002 → 编码 → verify.sh → 通过
    ├─ git commit: "feat(feat-002): 用户注册接口"
    └─ 写交接笔记 → claude-progress.md

    ...以此类推，每个会话消化一个 feature
```

**每个会话的生命周期**：初始化 → 读笔记 → 干活 → 验证 → 提交 → 写笔记。

## 为什么这个模式有效

### 维度一：外部化记忆

> —— 把状态从上下文搬到磁盘

Agent 的上下文窗口是易失的，会话一关就清零。但 `feature_list.json`、`claude-progress.md` 和 git 历史都写在磁盘上，新会话读文件就能完整恢复状态。

关键不只是"有个文件存着"，而是**记忆被拆成了三层**：

| 层级   | 文件                          | 作用                                             |
| ------ | ----------------------------- | ------------------------------------------------ |
| 任务层 | feature_list.json             | 还有多少事要做、彼此依赖关系                     |
| 过程层 | claude-progress.md + 交接笔记 | 上一个人做完时的现场、踩过的坑、叮嘱下一个人的话 |
| 代码层 | git 历史                      | 真实的产出和可回滚的快照                         |

三层加起来，一个新会话不用"猜"，只要"读"。

### 维度二：标准化流程

> —— 用规则代替模糊指令

Agent 最怕的不是任务难，是任务模糊。"帮我继续做项目"这种话，Agent 会自己脑补，脑补就会偏。

两阶段模式把每个会话都压成同一个流程：**读笔记 → 拿一个具体 feature → 做完验证 → 提交 → 写笔记**。起点、终点、产出物都是确定的。配合"一次只做一个 feature"的作用域约束，Agent 不会越界，也不会漂移。

## Harness Engineering 五大子系统

本文方法覆盖了 Harness 工程中的五大子系统：

| 子系统                                      | 职责                       | 对应产物                                 |
| ------------------------------------------- | -------------------------- | ---------------------------------------- |
| **指令（Instructions）**              | Agent 该做什么、不该做什么 | AGENTS.md、提示词模板                    |
| **作用域（Scope）**                   | 边界在哪，不能越界         | Sprint Contract、"一次一个 feature" 规则 |
| **状态（State）**                     | 项目当前进度、任务清单     | feature_list.json、claude-progress.md    |
| **验证（Verification）**              | 产出是否合格               | verify.sh、验收清单、contract            |
| **会话生命周期（Session Lifecycle）** | 每个会话怎么开始、怎么结束 | init.sh、交接笔记                        |

## 最后

四个模板可以直接复制到你的项目里使用。下次做多会话项目时，试一下这个两阶段工作流，你会发现 Agent 终于有"记忆"了。

如果在实践中遇到问题，欢迎在 GitHub Issue 中提出，一起探讨更好的 Harness 实践。

> 下一篇预告：Agent 烧 Token 烧到心痛？用渐进式披露 + 提示词缓存把成本降下来。

## 延伸阅读

- [*Effective harnesses for long-running agents*](https://www.anthropic.com/blog/effective-harnesses-for-long-running-agents) — Anthropic 官方博客，两阶段模式的完整设计与实战经验（2025-11-26）
- [AgentPatterns.ai](http://agentpatterns.ai/) — 关于 `init.sh`、`claude-progress.md`、feature list 等 harness 组件的模式整理
- [Harness Engineering from CC to AI Coding](https://zhanghandong.github.io/harness-engineering-from-cc-to-ai-coding/) — 《马书》在线阅读
