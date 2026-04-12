# Harness工程系列：别信AI“搞定了”，可以这样让TA自证清白

Apr 11, 2026

> **关键词：** `Harness Engineering` · `Agent 验证机制` · `Sprint Contract` · `Generator-Evaluator 分离`

---

上篇聊了 Harness Engineering 的基本概念和 4 个文件的最小起步。

这篇往深走一步：**怎么验证 Agent 的产出质量。**

因为很多人加了 AGENTS.md，加了 feature_list.json，Agent 确实听话了。

但有个新问题冒出来了——

## Agent 的"自信陷阱"

场景还原：

> 你："帮我实现用户登录功能，要支持邮箱和手机号两种方式。"
>
> Agent："已完成！我实现了完整的登录功能，包括邮箱登录、手机号登录、输入校验和错误处理。所有功能已测试通过。"
>
> 你打开浏览器一试——邮箱登录 500 错误，手机号登录压根没有入口。

**它说的"已测试通过"，是它自己觉得通过了。**

这不是个例。LLM 的过度自信（overconfidence）是一个已被广泛观察到的现象——Anthropic 在实际的 Agent 工程实践中也发现了同样的规律：

**让 Agent 自我评估产出质量，它往往对自己的代码过于自信。**

不管实际情况如何，Agent 都倾向于报告"一切正常"。

原因不复杂：Generator 和 Evaluator 是同一个角色。让生成代码的人评价自己的代码，就像让学生自己给自己批卷子。

## 拆开它：Generator 和 Evaluator 必须分离

Anthropic 在其 Agent 工程实践中，提出了一个三角色架构：

```
Planner（规划者）→ Generator（生成者）→ Evaluator（评估者）
    ^                                        |
    └──────────── 迭代反馈 ────────────────────┘
```

> **原文链接** ：[https://www.anthropic.com/engineering/harness-design-long-running-apps](https://www.anthropic.com/engineering/harness-design-long-running-apps)

三个角色各干各的：

| 角色          | 职责           | 关键原则                     | 在我们场景中是谁 |
| ------------- | -------------- | ---------------------------- | ---------------- |
| **Planner**   | 拆任务、定标准 | 先定义"做完"长什么样，再动手 | **你（人）**     |
| **Generator** | 写代码、出产物 | 只管实现，不自己判断质量     | **Agent**        |
| **Evaluator** | 跑测试、做检查 | 持怀疑态度，用证据说话       | **自动化脚本**   |

**说清楚一个关键点** ：在本篇的落地方案中， **Planner 就是你自己** 。你来写 Sprint Contract，你来定义 done_criteria。这确实需要你投入前期规划的时间——但这个时间花在"事前定标准"上，远比花在"事后反复沟通修 bug"上值得。

**核心发现** ：把 Evaluator 单独拆出来，并刻意调教它持怀疑态度，比让 Generator 自我批评 **可控得多** 。

怎么落地？往下看。

## Thoughtworks 的思路：Guides + Sensors

Thoughtworks 在 Martin Fowler 博客上发了一篇系统文章，把 Harness 的验证能力拆成两大类：

> **原文链接** ：[https://martinfowler.com/articles/harness-engineering.html](https://martinfowler.com/articles/harness-engineering.html)

| 维度         | Guides（前馈控制）       | Sensors（反馈控制）    |
| ------------ | ------------------------ | ---------------------- |
| **作用时机** | 行动前，预防问题         | 行动后，检测修正       |
| **计算型**   | 文档、规则文件、代码模板 | 测试、Lint、Type Check |
| **推理型**   | AI 生成的指导建议        | AI Review、语义分析    |

翻译成大白话：

- **Guides** = 出发前给地图。Agent 动手前就知道边界在哪。
- **Sensors** = 到了之后量结果。Agent 做完后用客观工具检验。

**两手都要有，缺一个就会出问题。**

只有 Guides 没有 Sensors：Agent 知道规矩但可能阳奉阴违，没人验收。

只有 Sensors 没有 Guides：Agent 乱跑一通再修，浪费大量时间和 Token。

在我们的模板中：**Sprint Contract 是 Guides，verify.sh 和 acceptance_criteria.json 是 Sensors。**

## 动手：三个模板搭建验证体系

理论够了，上模板。

下面三个文件加进项目， **Agent 的产出就有了客观验收标准** 。

### 模板一：Sprint Contract（成功标准定义）

**这个文件的作用** ：Agent 动手前，先把"做完"的标准写死。不是 Agent 自己定义，是你来定义。

**谁来写？你。** 这是 Planner 的职责。你需要清楚地知道这个 feature 做完应该是什么样子——如果你自己都说不清楚，Agent 更说不清楚。

```json
{
  "sprint_contract": {
    "feature_id": "feat-001",
    "feature_name": "用户邮箱登录",
    "created_at": "2026-04-11",

    "done_criteria": [
      "登录页面有邮箱输入框和密码输入框",
      "输入合法邮箱+正确密码，跳转到首页",
      "输入错误密码，显示'密码错误'提示",
      "邮箱格式不合法，提交按钮置灰",
      "连续 5 次错误，锁定账号 15 分钟"
    ],

    "test_requirements": {
      "unit_tests": [
        "邮箱格式校验函数",
        "密码强度校验函数",
        "登录错误次数计数器"
      ],
      "integration_tests": [
        "POST /api/auth/login 返回 200 + token",
        "POST /api/auth/login 错误密码返回 401",
        "锁定状态下登录返回 429"
      ],
      "e2e_tests": ["完整登录流程（输入 → 提交 → 跳转）", "错误提示展示流程"]
    },

    "constraints": {
      "no_touch": ["src/components/Layout.tsx", "src/styles/global.css"],
      "max_new_files": 5,
      "must_pass": ["npm run lint", "npm run typecheck", "npm run test"]
    }
  }
}
```

**几个关键设计** ：

**done_criteria 要具体到行为** 。不写"实现登录功能"，写"输入合法邮箱+正确密码，跳转到首页"。模糊标准等于没标准。

**constraints 划红线** 。`no_touch` 列出不允许改的文件，`max_new_files` 限制新增文件数量。Agent 特别喜欢"顺手"改东改西，这里直接堵死。

**test_requirements 先定义后实现** 。先说清楚要测什么，再让 Agent 写代码。这就是 Anthropic 说的"Sprint Contract"——合同先签，活儿后干。

### 模板二：verify.sh（自动验证脚本）

**这个文件的作用** ：Agent 说"做完了"之后，自动跑一遍验证。通过了才算完成，没通过就打回去。

**前置条件** ：verify.sh 依赖项目中已配置好的 npm scripts。在使用前，请确保你的 `package.json` 中已有以下配置：

```json
{
  "scripts": {
    "lint": "eslint . --ext .ts,.tsx",
    "typecheck": "tsc --noEmit",
    "test": "jest"
  }
}
```

如果你用的是其他工具链（比如 Biome 替代 ESLint、Vitest 替代 Jest），替换成对应的命令即可。**关键是这些 npm scripts 要能跑通，verify.sh 才有意义。**

```bash
#!/bin/bash
# verify.sh — Agent 产出验证管线
# 用法：Agent 每完成一个 feature，运行此脚本

set -e  # 任何一步失败就停

MAX_RETRY=3  # Agent 自动修复最多重试 3 次
RETRY_COUNT=0

echo "========== 验证开始 =========="

# 第一关：代码风格
echo "[1/5] Lint 检查..."
npm run lint
echo "  ✓ Lint 通过"

# 第二关：类型安全
echo "[2/5] TypeScript 类型检查..."
npm run typecheck
echo "  ✓ 类型检查通过"

# 第三关：单元测试 + 集成测试
echo "[3/5] 测试套件..."
npm run test -- --coverage
echo "  ✓ 测试通过"

# 第四关：E2E（可选，大功能开）
if [ "$RUN_E2E" = "true" ]; then
    echo "[4/5] E2E 测试..."
    npx playwright test
    echo "  ✓ E2E 通过"
else
    echo "[4/5] E2E 跳过（设置 RUN_E2E=true 开启）"
fi

# 第五关：验收证据校验（检查 acceptance_criteria 中引用的文件是否存在）
echo "[5/5] Evidence 文件校验..."
if [ -f "contracts/feat-*.acceptance.json" ] 2>/dev/null; then
    # 提取 evidence 字段中引用的文件路径，检查是否存在
    EVIDENCE_FILES=$(grep -oP '"evidence":\s*"test:\s*\K[^"]+' contracts/*.acceptance.json 2>/dev/null || true)
    for f in $EVIDENCE_FILES; do
        if [ ! -f "src/**/$f" ] && [ ! -f "tests/**/$f" ] && ! find . -name "$f" -type f | grep -q .; then
            echo "  ✗ Evidence 引用的文件不存在: $f"
            exit 1
        fi
    done
    echo "  ✓ Evidence 校验通过"
else
    echo "  ✓ 无 acceptance 文件，跳过 evidence 校验"
fi

echo "========== 全部验证通过 =========="
echo "可以提交了。"
```

**这个脚本就是 Evaluator。** 它不听 Agent 怎么说，只看客观结果。

需要注意的是：**verify.sh 通过，意味着代码在已定义的检查维度上是合格的——但它不能替代你对业务逻辑的最终判断。** 比如 Lint 和类型检查通过不代表功能设计合理，测试通过也依赖于测试用例本身的质量。verify.sh 帮你拦住大部分低级问题，但最终的功能验收还需要你来把关。

**关于失败后的处理** ：在 AGENTS.md 里加上如下规则，让 Agent 知道该怎么做：

```markdown
## 验证规则（强制）

每完成一个 feature 后：

1. 运行 `./verify.sh`
2. 如果失败：修复问题后重新运行，**最多重试 3 次**
3. 重试 3 次仍失败：停止修复，输出失败日志，等待人工介入
4. 全部通过后，更新 feature_list.json 状态为 "done"
5. **禁止**跳过验证直接标记完成
```

**为什么要限制重试次数？** 不限制的话，Agent 可能陷入"改了又挂、挂了再改"的死循环，白烧 Token 却越改越乱。3 次修不好，大概率是理解出了问题，需要人介入。

### 模板三：acceptance_criteria.json（验收清单）

**这个文件的作用** ：把 Sprint Contract 中的 done_criteria 做成可追踪的状态。每条标准是否通过，一目了然。

**注意** ：acceptance_criteria.json 中的 checklist 必须与 Sprint Contract 的 done_criteria **一一对应** 。如果开发过程中 done_criteria 发生变更，两边需要同步更新。 **以 Sprint Contract 为准** ——它是合同，acceptance_criteria 是验收报告。

```json
{
  "feature_id": "feat-001",
  "feature_name": "用户邮箱登录",
  "status": "in_review",
  "verified_at": null,

  "checklist": [
    {
      "id": "ac-001",
      "criterion": "登录页面有邮箱输入框和密码输入框",
      "type": "ui",
      "passed": false,
      "evidence": null
    },
    {
      "id": "ac-002",
      "criterion": "输入合法邮箱+正确密码，跳转到首页",
      "type": "e2e",
      "passed": false,
      "evidence": null
    },
    {
      "id": "ac-003",
      "criterion": "输入错误密码，显示'密码错误'提示",
      "type": "e2e",
      "passed": false,
      "evidence": null
    },
    {
      "id": "ac-004",
      "criterion": "邮箱格式不合法，提交按钮置灰",
      "type": "ui",
      "passed": false,
      "evidence": null
    },
    {
      "id": "ac-005",
      "criterion": "连续5次错误，锁定账号15分钟",
      "type": "integration",
      "passed": false,
      "evidence": "test: auth.lockout.test.ts"
    }
  ],

  "verification_result": {
    "lint": null,
    "typecheck": null,
    "unit_tests": null,
    "e2e_tests": null,
    "evidence_check": null
  }
}
```

**`evidence` 字段是关键** 。不是打个勾就行，要写清楚"凭什么说通过了"。

比如 `ac-005` 的 evidence 写的是 `test: auth.lockout.test.ts`——意思是有一个测试文件能证明这条标准通过了。

**而且 verify.sh 的第五关会校验这个文件是否真实存在。** Agent 不能随便填一个文件名糊弄过去——evidence 引用的文件必须在项目中找得到。

Agent 不能自己说"我检查过了"，它得拿出证据，而且证据要经得起验证。

## 三个模板怎么配合

放在一起看整个验证流程：

```
开始新 feature
    │
    ▼
[1] 你写好 Sprint Contract
    定义 done_criteria + test_requirements + constraints
    │
    ▼
[2] Agent 读取 Sprint Contract，开始编码
    只做合同范围内的事
    │
    ▼
[3] Agent 说"做完了"
    │
    ▼
[4] 运行 verify.sh
    ┌─ Lint ──── 通过？
    ├─ TypeCheck ── 通过？
    ├─ Tests ──── 通过？
    ├─ E2E ───── 通过？
    └─ Evidence ── 引用文件存在？
    │
    ├─ 失败 → Agent 自动修复（最多 3 次）→ 回到 [4]
    ├─ 3 次仍失败 → 停下来，等人工介入
    │
    ▼
[5] 更新 acceptance_criteria.json
    逐条标记 passed + evidence
    │
    ▼
[6] 全部通过 → 更新 feature_list.json 为 "done" → 提交
```

**Sprint Contract 是合同，verify.sh 是验收员，acceptance_criteria.json 是验收报告。**

三个文件各司其职，Agent 的产出质量就不再靠运气了。

## 加上这三个文件后，你的项目结构

```
项目根目录/
├── AGENTS.md                    ← 阶段一已有
├── init.sh                      ← 阶段一已有
├── feature_list.json            ← 阶段一已有
├── claude-progress.md           ← 阶段一已有
├── verify.sh                    ← 新增：自动验证脚本
└── contracts/
    ├── feat-001.contract.json   ← 新增：每个 feature 的合同
    └── feat-001.acceptance.json ← 新增：每个 feature 的验收清单
```

从阶段一的 4 个文件，到现在的 4+3 文件体系。

**阶段一解决了"Agent 乱跑"的问题，这一步解决了"Agent 自说自话"的问题。**

## 一个真实的对比

没有验证体系时：

> Agent："已完成用户登录功能。"
>
> 你测试 → 发现 3 个 bug → 手动反馈 → Agent 修复 → 你再测 → 还有 1 个边界情况 → 再反馈 → 来回 4 轮才搞定。

有验证体系后：

> Agent 完成编码 → 跑 verify.sh → Lint 挂了 → 自动修复 → 再跑 → 单元测试挂了 → 发现遗漏的边界情况 → 补上 → 全部通过 → 更新验收清单 → 提交。

**区别在于：前者靠你的时间做质检，后者靠自动化管线先过一遍。**

当然，自动化管线本身也有成本——你需要提前配好 lint、typecheck、test 这些基础设施，也需要花时间写 Sprint Contract。**但这些都是一次性或可复用的投入，而"反复沟通修 bug"的时间是每次都在烧的。**

你省下来的不只是时间，还有反复沟通的心智消耗。

## 写在最后

这篇的核心就一句话：

**不要让 Agent 自己评价自己。用 Sprint Contract 定义标准，用 verify.sh 执行检查，用 acceptance_criteria.json 留下证据。**

三个模板直接复制到项目里就能用（前提是你的项目已有基础的 lint/typecheck/test 配置）。

下一篇聊另一个高频痛点：**每次新开会话，Agent 都像失忆了一样，之前做的全忘了。** 怎么用 Anthropic 的两阶段模式（Initializer Agent + Coding Agent）实现多会话连续作战？模板照样给你准备好。

你在用 Agent 开发时，有没有被它的"虚假自信"坑过？用了什么方法解决的？欢迎在**评论区**留言交流。

## 延伸阅读

- [**Anthropic Agent 工程实践**](https://www.anthropic.com/engineering/harness-design-long-running-apps)： Planner + Generator + Evaluator 的设计思路来源
- [**Thoughtworks Guides + Sensors**](https://martinfowler.com/articles/harness-engineering.html)： 前馈控制 + 反馈控制的系统框架
- **实战课程** ：[learn-harness-engineering](https://github.com/walkinglabs/learn-harness-engineering)
