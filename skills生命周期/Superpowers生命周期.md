## 演示场景

**需求**: 为应用添加"用户登录认证"功能

---

## 完整生命周期时间线

### 阶段0: 用户启动
**用户动作**: 向Claude提出需求
```
用户: "我想为应用添加用户登录认证功能"
```

**系统响应**: 自动触发 `superpowers:brainstorming` skill

---

### 阶段1: 需求探索与设计 (brainstorming skill)

#### 自动执行流程:

1. **探索项目上下文** (自动)
   - Claude自动读取项目文件结构
   - 检查现有认证代码
   - 了解技术栈

2. **询问澄清问题** (交互式)
   ```
   Claude: "认证方式你倾向于哪种？"
   A. JWT Token (推荐)
   B. Session Cookie
   C. OAuth 2.0
   ```

3. **用户决策点 #1**
   ```
   用户: "选A，用JWT"
   ```

4. **提出设计方案** (自动)
   - Claude提出2-3种架构方案
   - 推荐最佳方案
   - 展示架构图

5. **用户决策点 #2**
   ```
   用户: "第一个方案看起来不错"
   ```

6. **编写设计文档** (自动)
   - Claude创建 `docs/superpowers/specs/2026-03-23-user-auth-design.md`
   - 包含完整的技术设计

7. **Spec审查循环** (自动)
   - Claude调度spec-document-reviewer子代理
   - 审查设计文档
   - 如果有问题，修复并重新审查

8. **用户审查Gate** (交互式)
   ```
   Claude: "设计文档已写好并提交到 docs/superpowers/specs/2026-03-23-user-auth-design.md
   请审查后告诉我是否需要修改，然后我们开始制定实施计划。"
   ```

9. **用户决策点 #3**
   ```
   用户: "设计看起来不错，继续吧"
   ```

**关键点**:
- ✅ 自动调用: 无需用户显式触发
- 🎯 内部自动流转: 不需要用户干预
- 🤝 交互节点: 需要用户决策

#### Skill内部调用链:

```
brainstorming (主控制器)
  ├── Read项目文件 (自动)
  ├── AskUserQuestion (交互式 - 决策点#1)
  ├── AskUserQuestion (交互式 - 决策点#2)
  ├── Write spec文档 (自动)
  ├── Agent: spec-document-reviewer (自动调度)
  │   └── 如果有问题 → 修复 → 重新调度
  ├── AskUserQuestion (交互式 - 决策点#3 - Gate)
  └── 【必需调用】Skill: writing-plans (自动触发)
```

---

### 阶段2: 制定实施计划 (writing-plans skill)

**触发方式**: `brainstorming` 内部自动调用（REQUIRED SUB-SKILL）

#### 自动执行流程:

1. **分解任务** (自动)
   - 将设计转化为bite-sized任务
   - 每个任务2-5分钟工作量
   - 指定精确的文件路径和代码

2. **编写计划文档** (自动)
   ```markdown
   # 用户认证实施计划

   ## Task 1: 创建用户模型
   - [ ] 编写测试: test_user_model.py
   - [ ] 实现: models/user.py
   - [ ] 验证: pytest tests/

   ## Task 2: JWT工具函数
   - [ ] 编写测试: test_jwt_utils.py
   - [ ] 实现: utils/jwt.py
   ...
   ```

3. **计划审查循环** (自动)
   - Claude调度plan-document-reviewer子代理
   - 审查计划质量
   - 如果有问题，修复并重新审查

4. **用户决策点 #4 - 选择执行方式**
   ```
   Claude: "计划已完成并保存到 docs/superpowers/plans/2026-03-23-user-auth-plan.md

   有两种执行方式:
   1. Subagent-Driven (推荐) - 每个任务调度独立子代理，快速迭代
   2. Inline Execution - 在当前会话批量执行

   选择哪种方式？"
   ```

**关键点**:
- ✅ 自动调用: 从brainstorming自动触发
- 🎯 文档生成: 自动创建详细计划
- 🤝 交互节点: 需要用户选择执行方式

#### Skill内部调用链:

```
writing-plans (主控制器)
  ├── 分析设计文档 (自动)
  ├── 分解任务 (自动)
  ├── Write实施计划 (自动)
  ├── Agent: plan-document-reviewer (自动调度)
  │   └── 如果有问题 → 修复 → 重新调度
  ├── AskUserQuestion (交互式 - 决策点#4)
  │   ├── 选择A → 【必需调用】Skill: subagent-driven-development
  │   └── 选择B → 【必需调用】Skill: executing-plans
  └── 用户选择后自动触发相应skill
```

---

### 阶段3A: 执行计划 - Subagent模式 (subagent-driven-development skill)

**触发方式**: 用户选择"Subagent-Driven"方式

#### 自动执行流程:

**准备阶段**:
1. **创建隔离工作树** (自动调用 using-git-worktrees)
   ```
   Claude: 创建工作树 .claude/worktrees/user-auth-20260323
   ```

2. **提取所有任务** (自动)
   - 从计划中提取所有任务
   - 创建TodoWrite任务列表

**任务循环 (每个任务)**:

3. **调度实施子代理** (自动)
   ```
   Claude调度子代理: implementer-agent

   implementer-agent: "开始实施Task 1..."
   implementer-agent:
     - 编写测试 test_user_model.py
     - 运行测试 (红色)
     - 实现最小代码
     - 运行测试 (绿色)
     - 提交: "feat: add user model"
   ```

4. **Spec合规审查** (自动)
   ```
   Claude调度子代理: spec-reviewer-agent

   spec-reviewer-agent: ✅ 符合规范
   或
   spec-reviewer-agent: ❌ 缺失: 密码强度验证
   ```

5. **如果审查失败** (自动循环)
   ```
   implementer-agent: 修复问题，添加密码强度验证
   spec-reviewer-agent: ✅ 现在符合规范
   ```

6. **代码质量审查** (自动)
   ```
   Claude调度子代理: code-quality-reviewer-agent

   code-quality-reviewer-agent:
     优点: 测试覆盖良好
     问题: 魔法数字 8 (密码最小长度)
   ```

7. **如果质量问题** (自动循环)
   ```
   implementer-agent: 提取常量 MIN_PASSWORD_LENGTH = 8
   code-quality-reviewer-agent: ✅ 通过
   ```

8. **标记任务完成** (自动)
   - 更新TodoWrite状态
   - 继续下一个任务

**完成阶段**:

9. **所有任务完成后** (自动)
   ```
   所有任务已完成 (5/5)
   ```

10. **最终代码审查** (自动)
    ```
    Claude调度子代理: final-code-reviewer

    final-code-reviewer:
      ✅ 整体实现质量良好
      ✅ 所有测试通过
      ✅ 准备合并
    ```

11. **【必需调用】finishing-a-development-branch** (自动触发)

**关键点**:
- 🔄 自动循环: 每个任务都是独立的实施-审查循环
- 🤖 子代理隔离: 每个子代理有独立上下文
- ✅ 双重审查: Spec合规 + 代码质量
- 🎯 自动流转: 无需用户干预

#### Skill内部调用链:

```
subagent-driven-development (主控制器)
  ├── 【建议调用】Skill: using-git-worktrees (自动)
  ├── TodoWrite: 创建所有任务 (自动)
  │
  └── FOR EACH 任务:
      ├── Agent: implementer (自动调度)
      │   ├── 编写测试 (自动)
      │   ├── 运行测试 (自动)
      │   ├── 实现代码 (自动)
      │   ├── 运行测试 (自动)
      │   └── Git commit (自动)
      │
      ├── Agent: spec-reviewer (自动调度)
      │   └── 如果失败 → implementer修复 → 重新审查 (循环)
      │
      ├── Agent: code-quality-reviewer (自动调度)
      │   └── 如果失败 → implementer修复 → 重新审查 (循环)
      │
      └── TaskUpdate: 标记完成 (自动)

  ├── Agent: final-code-reviewer (自动调度)
  └── 【必需调用】Skill: finishing-a-development-branch (自动)
```

---

### 阶段3B: 执行计划 - Inline模式 (executing-plans skill)

**触发方式**: 用户选择"Inline Execution"方式

#### 自动执行流程:

1. **创建隔离工作树** (自动调用 using-git-worktrees)

2. **读取计划** (自动)

3. **批量执行** (自动)
   ```
   Task 1: 创建用户模型
   - 编写测试 → 运行 → 实现 → 运行 → 提交

   Task 2: JWT工具
   - 编写测试 → 运行 → 实现 → 运行 → 提交

   ...
   ```

4. **【必需调用】finishing-a-development-branch** (自动触发)

**关键点**:
- 📦 批量执行: 在同一会话顺序执行
- ⚡ 更快速度: 无子代理调度开销
- 🎯 适合场景: 任务耦合度高

---

### 阶段4: 完成开发分支 (finishing-a-development-branch skill)

**触发方式**: 从执行阶段自动触发（REQUIRED SUB-SKILL）

#### 自动执行流程:

1. **验证测试** (自动)
   ```bash
   pytest tests/
   结果: 34/34 通过 ✅
   ```

2. **用户决策点 #5 - 选择完成方式**
   ```
   Claude: 实施完成。你想要:

   1. 合并到main分支 (本地)
   2. 推送并创建Pull Request
   3. 保持分支不变 (稍后处理)
   4. 放弃这次工作

   选择哪个选项？
   ```

3. **执行用户选择**:

   **选项1: 本地合并** (自动)
   ```bash
   git checkout main
   git pull
   git merge user-auth-20260323
   pytest tests/ # 验证合并后测试通过
   git branch -d user-auth-20260323
   # 清理工作树
   ```

   **选项2: 创建PR** (自动)
   ```bash
   git push -u origin user-auth-20260323
   gh pr create --title "feat: 用户登录认证" --body "..."
   # 保留工作树供PR审查使用
   ```

   **选项3: 保持现状** (自动)
   ```
   Claude: 分支 user-auth-20260323 已保留。
   工作树位于: .claude/worktrees/user-auth-20260323
   ```

   **选项4: 放弃** (需要确认)
   ```
   Claude: 这将永久删除:
   - 分支: user-auth-20260323
   - 所有提交: <commit-list>
   - 工作树: .claude/worktrees/user-auth-20260323

   输入 'discard' 确认:
   ```

**关键点**:
- ✅ 测试验证: 强制验证，测试失败则停止
- 🤝 交互节点: 用户选择完成方式
- 🧹 自动清理: 根据选择自动处理工作树

#### Skill内部调用链:

```
finishing-a-development-branch (主控制器)
  ├── Bash: 运行测试 (自动)
  │   └── 如果失败 → 停止，报告错误
  │
  ├── AskUserQuestion (交互式 - 决策点#5)
  │   ├── 选项1 → Git merge + 清理工作树 (自动)
  │   ├── 选项2 → Git push + 创建PR (自动)
  │   ├── 选项3 → 保留现状 (自动)
  │   └── 选项4 → 确认后删除 (自动)
  │
  └── Bash: 执行相应git操作 (自动)
```

---

## 异常流程处理

### 场景1: 测试失败

**发生时机**: 任何运行测试的阶段

**触发skill**: `superpowers:systematic-debugging`

**流程**:
```
测试失败
  ↓
【必需调用】Skill: systematic-debugging
  ├── Phase 1: 根因调查 (自动)
  │   - 读取错误信息
  │   - 重现问题
  │   - 收集证据
  ├── Phase 2: 模式分析 (自动)
  ├── Phase 3: 假设与测试 (自动)
  └── Phase 4: 实施 (自动)
      └── 【相关调用】Skill: test-driven-development
```

### 场景2: 完成前验证

**发生时机**: 声称工作完成前

**触发skill**: `superpowers:verification-before-completion`

**流程**:
```
即将声称"完成"
  ↓
【强制规则】verification-before-completion
  ├── 识别: 什么命令能证明？ (自动)
  ├── 运行: 执行完整命令 (自动)
  ├── 读取: 完整输出 (自动)
  ├── 验证: 输出是否确认声明？ (自动)
  └── 只有验证通过才能声称完成
```

---

## 用户显式调用节点总结

| 决策点 | 阶段 | 触发时机 | 用户动作 | 后续自动流转 |
|--------|------|----------|----------|-------------|
| #1 | brainstorming | 选择技术方案 | 选择A/B/C | 继续设计流程 |
| #2 | brainstorming | 批准设计方案 | 确认方案 | 编写设计文档 |
| #3 | brainstorming | 审查设计文档 | 批准文档 | 自动触发writing-plans |
| #4 | writing-plans | 选择执行方式 | 选择Subagent或Inline | 自动触发相应执行skill |
| #5 | finishing-a-development-branch | 选择完成方式 | 选择合并/PR/保留/放弃 | 执行相应git操作 |

**关键洞察**:
- 用户只在**关键决策点**介入
- 其他所有步骤**自动流转**
- Skills之间通过**REQUIRED SUB-SKILL**形成调用链
- 每个skill有**明确的责任边界**

---

## 完整调用图谱

```
用户提出需求
    ↓
【自动触发】Skill: brainstorming
    ├── 自动: 探索项目
    ├── 交互: 技术方案选择 (决策点#1)
    ├── 交互: 设计方案确认 (决策点#2)
    ├── 自动: 编写spec文档
    ├── 自动: Agent: spec-document-reviewer (循环直到通过)
    ├── 交互: 用户审查文档 (决策点#3)
    └── 【必需调用】Skill: writing-plans
                ↓
        Skill: writing-plans
            ├── 自动: 分解任务
            ├── 自动: 编写实施计划
            ├── 自动: Agent: plan-document-reviewer (循环直到通过)
            ├── 交互: 选择执行方式 (决策点#4)
            │   ├── 选择Subagent → 【必需调用】Skill: subagent-driven-development
            │   └── 选择Inline → 【必需调用】Skill: executing-plans
            └── 自动触发相应skill
                        ↓
            ┌───────────────────────────────────────┐
            │ Skill: subagent-driven-development    │
            │   ├── 【建议调用】Skill: using-git-worktrees
            │   ├── 自动: FOR EACH 任务:           │
            │   │   ├── Agent: implementer         │
            │   │   │   └── 遵循 TDD流程           │
            │   │   ├── Agent: spec-reviewer       │
            │   │   │   └── 如果失败 → 循环修复    │
            │   │   └── Agent: code-quality-reviewer
            │   │       └── 如果失败 → 循环修复    │
            │   ├── 自动: Agent: final-code-reviewer
            │   └── 【必需调用】Skill: finishing-a-development-branch
            └───────────────────────────────────────┘
                        ↓
            Skill: finishing-a-development-branch
                ├── 自动: 验证测试通过
                ├── 交互: 选择完成方式 (决策点#5)
                │   ├── 1. 本地合并 → 自动执行git操作 + 清理工作树
                │   ├── 2. 创建PR → 自动执行git操作
                │   ├── 3. 保持现状 → 保留分支和工作树
                │   └── 4. 放弃 → 确认后删除
                └── 完成
```

---

## 支持性Skills调用时机

| Skill | 调用时机 | 调用方式 | 作用 |
|-------|----------|----------|------|
| `using-git-worktrees` | 执行计划前 | 自动或建议 | 创建隔离工作环境 |
| `test-driven-development` | 实施任务时 | 子代理自动遵循 | 确保TDD流程 |
| `systematic-debugging` | 遇到bug/测试失败 | 必需调用 | 系统化调试 |
| `verification-before-completion` | 声称完成前 | 强制规则 | 验证完成状态 |
| `requesting-code-review` | 代码审查时 | 自动调度 | 提供审查模板 |
| `receiving-code-review` | 接收审查反馈 | 按需调用 | 处理审查意见 |

---

## 核心设计理念

### 1. 托管式工作流
- **用户只需在关键节点决策**
- **Skills自动编排实施细节**
- **减少认知负担，提高质量**

### 2. 强制最佳实践
- **TDD**: 通过test-driven-development强制执行
- **验证**: 通过verification-before-completion强制执行
- **调试**: 通过systematic-debugging强制流程
- **审查**: 双重审查机制（spec + quality）

### 3. 隔离与上下文管理
- **工作树隔离**: 避免污染主分支
- **子代理隔离**: 每个任务独立上下文
- **防止上下文污染**: 提高决策质量

### 4. 循环质量保证
- **Spec审查循环**: 确保符合设计
- **代码质量审查循环**: 确保实现质量
- **修复-审查循环**: 确保问题真正解决

---

## 总结

Superpowers插件通过以下方式托管整个开发生命周期：

1. **自动流转**: 从需求到设计，从计划到实施，从验证到完成，形成完整自动化链路
2. **决策节点**: 用户只在5个关键节点介入，其余自动处理
3. **强制质量**: 通过REQUIRED SUB-SKILL和强制规则确保最佳实践
4. **隔离上下文**: 工作树和子代理隔离，防止污染和混淆
5. **循环验证**: 多层审查循环，确保质量而非速度

这是一个**真正的软件工程助手**，而不只是代码生成工具。