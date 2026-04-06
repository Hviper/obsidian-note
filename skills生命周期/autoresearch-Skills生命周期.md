# Autoresearch 深度分析：生命周期、功能与组合使用

## 一、整体架构

Autoresearch 是一个**插件式 Claude Code Skill**，包含 1 个核心技能 + 8 个子命令 + 1 个规划向导。

```
autoresearch (插件)
├── /autoresearch          ← 核心：自主迭代循环
├── /autoresearch:plan     ← 规划向导
├── /autoresearch:debug    ← Bug 狩猎
├── /autoresearch:fix      ← 错误修复
├── /autoresearch:security ← 安全审计
├── /autoresearch:ship     ← 发布工作流
├── /autoresearch:scenario ← 场景探索
├── /autoresearch:predict  ← 多角色预测
└── /autoresearch:learn   ← 文档生成
```

---

## 二、核心迭代循环的生命周期（8 阶段）

### Phase 0: 前置检查（Precondition Checks）

**在进入循环前必须完成**：

```bash
# 1. 验证 Git 仓库存在
git rev-parse --git-dir

# 2. 检查工作区是否干净（是否有未提交的更改）
git status --porcelain

# 3. 检查是否有 stale lock 文件
ls .git/index.lock

# 4. 检查是否在 detached HEAD 状态
git symbolic-ref HEAD

# 5. 检查是否有干扰的 git hooks
ls .git/hooks/pre-commit
```

**如果任何检查失败**：停止并告知用户，不进入循环。
**如果任何警告**：记录警告，谨慎继续，告知用户。

---

### Phase 1: 回顾（Review）- 每次迭代必做

**构建情境感知**，完成全部 6 个步骤：

```
1. 读取范围内文件的当前状态（完整上下文）
2. 读取结果日志最近 10-20 条
3. 必须运行：git log --oneline -20 查看最近变更
4. 必须运行：git diff HEAD~1 审查上次的成功变更
5. 识别：什么成功了，什么失败了，什么还没尝试
6. 如果有界：检查 current_iteration vs max_iterations
```

**关键：Git 就是记忆**。每次迭代必须读取 git 历史，学习之前的实验。

**为什么每次都读 git 历史？**
- Git 是记忆。回滚后，状态可能与预期不同。
- Git log 显示哪些实验被保留，哪些被回滚。
- 保留变更的 git diff 揭示了**具体什么改进了指标**——用于指导下一次迭代。
- 永远不要假设——总是验证。

---

### Phase 2: 构思（Ideate）- 战略性选择

**选择下一个变更**，优先级顺序：

```
1. 修复上次迭代的崩溃/失败
2. 利用成功案例 → 运行 git diff 查看保留了什么，尝试变体
3. 探索新方法 → 交叉参考结果日志和 git 历史
4. 组合接近成功的方案
5. 简化 → 在保持指标的情况下删除代码
6. 激进实验 → 当渐进式变化停滞时，尝试完全不同的方法
```

**反模式**：
- 不要重复已被丢弃的相同变更——先检查 git log
- 不要一次做多个不相关的变更（无法归因改进）
- 不要追求边际收益而引入丑陋的复杂性
- 不要忽略 git 历史——这是迭代之间的主要学习机制

---

### Phase 3: 修改（Modify）- 原子性变更

**一次只做一个变更**：

| ✅ 正确（一个变更） | ❌ 错误（多个变更） |
|---------|---------|
| 改一个参数 | 改多个参数 |
| 一个文件 | 跨多个不相关文件 |
| 可以用一句话描述 | 需要用"和"连接多个动作 |

**一句话测试**：如果需要用"和"来描述，那就是两个变更。拆分它们。

**自检**：
```bash
# 变更后验证原子性
FILES_CHANGED=$(git diff --name-only | wc -l)

# 启发式：超过 5 个文件可能意味着多个变更——审查
if [ "$FILES_CHANGED" -gt 5 ]; then
    echo "⚠ ${FILES_CHANGED} 个文件变更 — 验证单一意图"
fi
```

---

### Phase 4: 提交（Commit）- 验证前提交

**必须在运行验证之前提交**。这使实验失败时能够干净地回滚。

```bash
# 只提交范围内的文件（更安全，不用 git add -A）
# 列出你修改的具体文件并单独添加：
git add <file1> <file2> ...
# 避免 git add -A — 它会暂存所有文件包括 .env、node_modules 和用户无关的工作

# 检查是否确实有内容要提交
git diff --cached --quiet
# → 如果退出码 0（无暂存变更）：跳过提交，记录为"no-op"，进入下一次迭代
# → 如果退出码 1（存在变更）：继续提交

# 使用描述性实验消息提交
git commit -m "experiment(<scope>): <一句话描述你改变了什么和为什么>"
```

**提交消息格式**：使用约定式提交格式，类型为 `experiment`：`experiment(<scope>): <description>`。这与 commit-lint 保持兼容，同时清晰标记 autoresearch 迭代。

**Hook 失败处理**：
1. 读取 hook 的错误输出，理解为什么被阻止
2. 如果可修复（lint 错误、格式化）：修复问题，重新暂存，重试提交——不要使用 `--no-verify`
3. 如果 2 次尝试内无法修复：记录为 `status=hook-blocked`，回滚范围内文件变更，进入下一次迭代
4. 永远不要用 `--no-verify` 绕过 hooks——hooks 用于保护代码质量

---

### Phase 5: 验证（Verify）- 机械验证

运行约定的验证命令。捕获输出。

**超时规则**：如果验证超过正常时间的 2 倍，终止并视为崩溃。

**提取指标**：解析验证输出的具体指标数值。

#### 验证命令模板（按语言）

| 语言 | 验证命令 | 指标 | 方向 |
|----------|---------------|--------|-----------|
| **Node.js** | `npx jest --coverage 2>&1 \| grep 'All files' \| awk '{print $4}'` | 覆盖率 % | 更高 |
| **Python** | `pytest --cov=src --cov-report=term 2>&1 \| grep TOTAL \| awk '{print $4}'` | 覆盖率 % | 更高 |
| **Rust** | `cargo test 2>&1 \| grep -oP '\d+ passed' \| grep -oP '\d+'` | 通过的测试 | 更高 |
| **Go** | `go test -count=1 ./... 2>&1 \| grep -c '^ok'` | 通过的包 | 更高 |
| **Java** | `mvn test 2>&1 \| grep 'Tests run:' \| tail -1 \| grep -oP 'Failures: \d+' \| grep -oP '\d+'` | 失败数 | 更低 |
| **Bundle** | `npx esbuild src/index.ts --bundle --minify \| wc -c` | 字节 | 更低 |
| **Lighthouse** | `npx lighthouse http://localhost:3000 --output=json \| jq '.categories.performance.score * 100'` | 分数 0-100 | 更高 |
| **延迟** | `wrk -t2 -c10 -d10s http://localhost:3000/api 2>&1 \| grep 'Avg Lat' \| awk '{print $2}'` | ms | 更低 |

---

### Phase 5.1: 噪声处理（针对波动指标）

有些指标天生有噪声——基准测试时间、ML 准确率、Lighthouse 分数。单次测量可能误导。使用这些策略防止错误的 keep/discard 决策。

#### 策略 1：多轮验证

运行验证 N 次并使用中位数过滤异常值：

```bash
# 单次运行（对噪声指标不可靠）：
npm run benchmark  # 可能随机报告 142ms 或 158ms

# 多轮取中位数（可靠）：
for i in 1 2 3; do
  npm run benchmark 2>&1 | grep 'avg' | awk '{print $2}'
done | sort -n | sed -n '2p'  # 3 次的中位数
```

配置：
```
Noise: high           # 自动触发 3 次中位数
Noise-Runs: 5         # 自定义：5 次而非默认 3 次
```

#### 策略 2：最小改进阈值

忽略小于噪声底线的改进：

```
Min-Delta: 2.0   # 只在改进 > 2% 时保留
```

#### 策略 3：确认运行

在做出最终保留决策前重新验证

---

### Phase 5.5: 守卫（Guard）- 回归检查

如果定义了**守卫**命令，在验证之后运行。

守卫是必须**始终通过**的命令——它在优化主指标时保护现有功能。常见守卫：`npm test`、`npm run typecheck`、`pytest`、`cargo test`。

**关键区别**：
- **验证**回答："指标改进了吗？"（目标）
- **守卫**回答："其他东西坏了吗？"（安全网）

**守卫规则**：
- 只有在定义了守卫时才运行（可选）
- 在验证**之后**运行——如果指标没改进，检查守卫没有意义
- 守卫只有通过/失败（退出码 0 = 通过）。无需提取指标
- 如果守卫失败，回滚优化并尝试重新优化（最多 2 次尝试）
- 永远不要修改守卫/测试文件——总是调整实现
- 明确记录守卫失败，以便代理学习哪些变更会导致回归

**守卫失败恢复（最多 2 次重新优化尝试）**：

当守卫失败但指标改进时，优化想法可能仍然可行——只需要不同的实现，不会破坏行为：

1. 回滚变更（使用 `safe_revert()`）
2. 阅读守卫输出，理解什么坏了（哪些测试、哪些断言）
3. 重新优化以避免回归
4. 提交重新优化的版本，重新运行验证 + 守卫
5. 如果都通过 → 保留。如果守卫再次失败 → 再试一次，然后放弃

---

### Phase 6: 决策（Decide）- 无歧义

```bash
# 回滚函数 — 用于所有 discard/crash 决策
safe_revert() {
  echo "回滚: $(git log --oneline -1)"

  # 尝试 1：git revert（保留历史 — 首选）
  if git revert HEAD --no-edit 2>/dev/null; then
    echo "✓ 通过 git revert 回滚（实验保留在历史中以供学习）"
    return 0
  fi

  # 尝试 2：revert 冲突 — 回退到 reset
  git revert --abort 2>/dev/null
  echo "⚠ Revert 冲突 — 使用 git reset --hard HEAD~1"
  git reset --hard HEAD~1
  echo "✓ 通过 reset 回滚（实验从历史中移除）"
  return 0
}
```

```
如果指标提升 AND (无守卫 OR 守卫通过):
    STATUS = "keep"
    # 什么都不做 — 提交保留。Git 历史保留这个成功。
如果指标提升 AND 守卫失败:
    safe_revert()
    # 重新优化（最多 2 次尝试）
    对于 1..2 中的每次尝试:
        分析守卫输出 → 重新优化实现（不是测试）
        git add <修改的文件> && git commit -m "experiment(<scope>): rework — <描述>"
        重新运行验证
        如果指标提升:
            重新运行守卫
            如果守卫通过:
                STATUS = "keep (reworked)"
                跳出
        safe_revert()
    如果 2 次尝试后仍然失败:
        STATUS = "discard"
        原因 = "守卫失败，无法重新优化"
如果指标相同或下降:
    STATUS = "discard"
    safe_revert()
如果崩溃:
    # 尝试修复（最多 3 次）
    如果可修复:
        修复 → 重新提交 → 重新验证 → 重新守卫
    否则:
        STATUS = "crash"
        safe_revert()
```

**为什么 `git revert` 而不是 `git reset --hard`？**
- `git revert` 在历史中保留失败的实验——这就是"记忆"。未来的迭代可以读取 `git log` 并看到尝试了什么和失败。
- `git reset --hard` 完全销毁提交——代理失去尝试的记忆。
- `git revert` 在 Claude Code 中也更安全——它是非破坏性操作，不会触发安全警告。
- 回退：如果 `git revert` 产生合并冲突，使用 `git revert --abort` 然后 `git reset --hard HEAD~1`。

---

### Phase 7: 记录（Log）- 结果日志

追加到结果日志（TSV 格式）：

```
iteration  commit   metric   delta    guard  status        description
0          a1b2c3d  99.08    0.0      pass   baseline      initial state
1          b2c3d4e  99.15    +0.07    pass   keep          reduce conv channels
2          -        99.12    -0.03    -      discard       add more layers
3          -        0.0      0.0      -      crash         double batch size (OOM)
4          -        -        -        -      no-op         attempted to modify read-only config
5          -        -        -        -      hook-blocked  pre-commit lint rejected formatting
```

**有效状态**：`keep`、`keep (reworked)`、`discard`、`crash`、`no-op`、`hook-blocked`

---

### Phase 8: 重复（Repeat）

#### 无界模式（默认）

进入 Phase 1。**永不停止。永不询问是否应该继续。**

#### 有界模式（使用 Iterations: N）

```
如果 current_iteration < max_iterations:
    进入 Phase 1
如果达到目标:
    打印："目标在第 {N} 次迭代达到！最终指标：{value}"
    打印最终总结
    停止
否则:
    打印最终总结
    停止
```

**最终总结格式**：
```
=== Autoresearch 完成 (N/N 次迭代) ===
基线：{baseline} → 最终：{current} ({delta})
保留：X | 丢弃：Y | 崩溃：Z | 跳过：W (no-ops + hook-blocked)
最佳迭代：#{n} — {描述}
```

#### 卡住时（超过 5 次连续丢弃）

适用于两种模式：
1. 从头重新读取所有范围内文件
2. 重新阅读原始目标/方向
3. 审查整个结果日志寻找模式
4. 尝试组合 2-3 个以前成功的变更
5. 尝试与无效做法相反的方法
6. 尝试激进的结构性变更

---

## 三、八大子命令详解

### 1. `/autoresearch` - 核心迭代循环

**用途**：通用目标驱动优化

**参数**：
```
Goal: <目标描述>
Scope: <文件 glob>
Metric: <指标名称和方向>
Verify: <提取指标的 shell 命令>
Guard: <回归测试命令（可选）>
Iterations: <N>（可选，有界模式）
```

**使用示例**：
```
/autoresearch
Goal: Increase test coverage from 72% to 90%
Scope: src/**/*.test.ts, src/**/*.ts
Metric: coverage % (higher is better)
Verify: npm test -- --coverage | grep "All files"
Guard: npm test
```

---

### 2. `/autoresearch:plan` - 规划向导

**用途**：将自然语言目标转换为可执行的 autoresearch 配置

**工作流程**：
```
1. 捕获目标（用户输入或内联）
2. 分析上下文（扫描代码库工具）
3. 定义范围（建议文件 glob，验证存在）
4. 定义指标（建议机械指标，验证输出数字）
5. 定义方向（越高越好/越低越好）
6. 定义验证（构建 shell 命令，干运行确认）
7. 确认并启动
```

**关键门禁**：
- 指标必须机械（输出可解析的数字，而非主观）
- 验证命令必须在当前代码库上通过干运行才能接受
- 范围必须解析为 ≥1 个文件

---

### 3. `/autoresearch:debug` - Bug 狩猎

**用途**：使用科学方法迭代定位所有 bug

**工作流程**：
```
1. 收集症状（用户提供或分析错误信息）
2. 形成假设
3. 设计实验验证
4. 分析结果
5. 迭代直到代码库干净
```

**适合场景**：
- "找到所有 bug"
- "调试这个"
- "为什么失败"
- "调查"

---

### 4. `/autoresearch:fix` - 错误修复

**用途**：迭代修复错误（测试失败、类型错误、lint 警告、构建失败）

**工作流程**：
```
1. 收集错误信息
2. 迭代修复每个错误
3. 验证修复
4. 记录结果
```

**适合场景**：
- "修复所有错误"
- "让测试通过"
- "修复构建"
- "清理错误"

---

### 5. `/autoresearch:security` - 安全审计

**用途**：STRIDE 威胁模型 + OWASP Top 10 + 红队

**工作流程**：
```
1. 代码库侦察（扫描技术栈、依赖、配置、API 路由）
2. 资产识别（数据存储、认证系统、外部服务、用户输入）
3. 信任边界映射
4. STRIDE 威胁模型
5. 攻击面映射
6. 迭代测试每个向量
7. 最终报告（严重性排名 + 缓解建议）
```

**关键行为**：
- 遵循红队对抗思维
- 每个发现都需要**代码证据**（file:line + 攻击场景）
- 跟踪 OWASP Top 10 + STRIDE 覆盖率

**标志**：
- `--diff`：只审计变更的文件
- `--fix`：自动修复严重问题
- `--fail-on {severity}`：CI/CD 门禁

**输出目录**：`security/{YYMMDD}-{HHMM}-{audit-slug}/`

---

### 6. `/autoresearch:ship` - 发布工作流

**用途**：通用发布流程（代码、内容、营销、销售、研究、设计）

**工作流程**（8 阶段）：
```
1. Identify - 自动检测要发布的内容
2. Inventory - 评估当前状态和准备差距
3. Checklist - 生成特定领域的发布检查清单
4. Prepare - 迭代修复失败的检查项
5. Dry-run - 模拟发布（无副作用）
6. Ship - 执行实际交付
7. Verify - 验证发布成功
8. Log - 记录到 ship-log.tsv
```

**支持的发布类型**：
- `code-pr`：GitHub PR
- `code-release`：Git tag + GitHub release
- `deployment`：CI/CD 触发
- `content`：通过 CMS 发布
- `marketing-email`：通过 ESP 发送
- `marketing-campaign`：激活广告
- `sales`：发送提案
- `research`：上传到仓库
- `design`：导出资产

**标志**：
- `--dry-run`：只验证不发布
- `--auto`：如果无错误自动批准
- `--force`：跳过非关键检查项
- `--rollback`：撤销上次发布
- `--monitor N`：发布后监控 N 分钟

**输出目录**：`ship/{YYMMDD}-{HHMM}-{ship-slug}/`

---

### 7. `/autoresearch:scenario` - 场景生成器

**用途**：从种子场景生成、扩展、压力测试用例

**12 个探索维度**：
```
1. happy path（正常流程）
2. error（错误处理）
3. edge case（边缘情况）
4. abuse（滥用）
5. scale（规模）
6. concurrent（并发）
7. temporal（时间相关）
8. data variation（数据变化）
9. permission（权限）
10. integration（集成）
11. recovery（恢复）
12. state transition（状态转换）
```

**工作流程**：
```
1. 种子分析（解析场景，识别角色、目标、前置条件、组件）
2. 分解（12 个探索维度）
3. 情景生成（每个迭代创建一个具体情景）
4. 分类（去重：new/variant/duplicate/out-of-scope/low-value）
5. 扩展（派生边缘情况、失败模式）
6. 记录
7. 重复
```

**标志**：
- `--domain <type>`：设置领域（software、product、business、security、marketing）
- `--depth <level>`：探索深度（shallow、standard、deep）
- `--scope <glob>`：限制特定文件/功能
- `--format <type>`：输出格式
- `--focus <area>`：优先维度

**输出目录**：`scenario/{YYMMDD}-{HHMM}-{slug}/`

---

### 8. `/autoresearch:predict` - 多角色预测

**用途**：使用群体智能原理的多视角代码分析

**默认角色**：
```
- Architect（架构师）
- Security Analyst（安全分析师）
- Performance Engineer（性能工程师）
- Reliability Engineer（可靠性工程师）
- Devil's Advocate（反对者，强制）
```

**工作流程**：
```
1. 代码库侦察
2. 角色生成（3-5 个专家角色）
3. 独立分析（每个角色从独特视角分析）
4. 结构化辩论（1-2 轮交叉审查）
5. 共识（综合器聚合发现 + 反群体思维检查）
6. 知识输出
7. 报告
8. 交接（写入 handoff.json）
```

**关键行为**：
- 基于文件的知识表示：.md 文件就是知识图谱，零外部依赖
- Git-hash 标记：每个输出都嵌入 commit SHA 用于陈旧检测
- 增量更新：只重新分析自上次运行以来更改的文件
- 反群体思维机制：强制 Devil's Advocate，通过翻转率 + 熵检测

**标志**：
- `--chain <targets>`：链式调用其他工具。单个：`--chain debug`。多个：`--chain scenario,debug,fix`（顺序）
- `--personas N`：角色数量（默认：5，范围：3-8）
- `--rounds N`：辩论轮次（默认：2，范围：1-3）
- `--depth <level>`：深度预设（shallow、standard、deep）
- `--adversarial`：使用对抗性角色集
- `--budget <N>`：所有角色的最大发现总数
- `--fail-on <severity>`：如果在或超过严重程度则非零退出
- `--scope <glob>`：限制分析特定文件

**输出目录**：`predict/{YYMMDD}-{HHMM}-{slug}/`

---

### 9. `/autoresearch:learn` - 文档引擎

**用途**：自动生成/更新代码库文档

**4 种模式**：
| 模式 | 用途 | 有迭代？ |
|------|------|----------|
| `init` | 从头学习，生成所有文档 | 是（验证-修复循环） |
| `update` | 学习变更，刷新现有文档 | 是（验证-修复循环） |
| `check` | 只读健康/新鲜度评估 | 否（仅诊断） |
| `summarize` | 快速代码库摘要 | 最少（仅大小检查） |

**工作流程**：
```
1. Scout - 并行代码库侦察（规模感知 + monorepo 检测）
2. Analyze - 项目类型分类、技术栈检测、陈旧度测量
3. Map - 动态文档发现（docs/*.md）、差距分析、条件文档选择
4. Generate - 使用结构化提示模板和完整上下文生成文档
5. Validate - 机械验证（代码引用、链接、完整性、大小合规）
6. Fix - 验证-修复循环：带反馈重新生成失败的文档（最多 3 次重试）
7. Finalize - 清单检查、git diff 摘要、大小合规
8. Log - 记录结果到 learn-results.tsv
```

**关键行为**：
- 完全动态文档发现 — 扫描 `docs/*.md`，无硬编码文件列表
- 状态感知模式检测 — 根据 docs/ 状态自动选择 init/update
- 项目类型自适应 — 只在存在部署配置时创建 deployment-guide.md
- 验证-修复循环最多 3 次重试 — 未解决时升级给用户
- 规模感知侦察 — 为 5k+ 文件代码库调整并行性

**标志**：
- `--mode <mode>`：操作模式（默认：自动检测）
- `--scope <glob>`：限制代码库学习到特定目录
- `--depth <level>`：文档全面性（quick、standard、deep）
- `--scan`：在 summarize 模式强制重新侦察
- `--topics <list>`：将 summarize 聚焦特定主题
- `--file <name>`：选择性更新 — 目标单个文档
- `--no-fix`：跳过验证-修复循环
- `--format <fmt>`：输出格式（默认：markdown）

**输出目录**：`learn/{YYMMDD}-{HHMM}-{slug}/`

---

## 四、Skill 与 Command 的组合使用

### 1. 独立使用

每个子命令可以**独立调用**：

```bash
# 安全审计
/autoresearch:security

# 发布工作流
/autoresearch:ship --type deployment

# 预测 + 链式调试
/autoresearch:predict --chain debug

# 场景探索
/autoresearch:scenario --domain software --depth standard

# 文档生成
/autoresearch:learn --mode update

# Bug 狩猎
/autoresearch:debug

# 错误修复
/autoresearch:fix

# 规划向导
/autoresearch:plan
```

---

### 2. 链式组合（Chaining）

`/autoresearch:predict` 支持 `--chain` 标志，可以**顺序调用其他工具**：

```bash
# 预测 → 调试（单链）
/autoresearch:predict --chain debug
Scope: src/api/**
Goal: Investigate intermittent 500 errors

# 预测 → 场景 → 调试 → 修复（多链顺序流水线）
/autoresearch:predict --chain scenario,debug,fix
Scope: src/**
Goal: Full quality pipeline for new feature

# 预测 → 安全（安全审查链）
/autoresearch:predict --chain security
Scope: src/auth/**

# 预测 → 调试 → 修复 → 发布（完整流水线）
/autoresearch:predict --chain debug,fix,ship
Scope: src/new-feature/**
```

---

### 3. 迭代组合

可以在 `/autoresearch` 主循环中**组合使用其他工具作为验证**：

```bash
# 使用 security 作为验证
/autoresearch
Goal: Find and fix all security vulnerabilities
Scope: src/api/**, src/auth/**
Metric: Security score (higher is better)
Verify: /autoresearch:security --scope src/api,src/auth | grep "Score" | awk '{print $2}'
Iterations: 10

# 使用 debug 作为验证
/autoresearch
Goal: Achieve zero bugs in authentication module
Scope: src/auth/**/*.ts
Metric: Bugs found (lower is better)
Verify: /autoresearch:debug | grep "Bugs found:" | awk '{print $3}'
```

---

### 4. 典型组合场景

| 场景 | 组合 |
|------|------|
| 发布前安全审计 | `/autoresearch:security --fail-on critical --fix` |
| 多角度质量审查 | `/autoresearch:predict --chain security,debug` |
| 完整质量流水线 | `/autoresearch:predict --chain scenario,debug,fix → ship` |
| 文档驱动开发 | `/autoresearch:learn --mode update` → 编写代码 → `/autoresearch` |
| CI/CD 集成 | `/autoresearch:security --fail-on critical` + `/autoresearch:test` |
| 快速迭代优化 | `/autoresearch` with `Iterations: 10` |
| 过夜批处理 | `/autoresearch` (无界模式) |

---

### 5. 工作流组合示例

#### 发布前安全审查工作流

```bash
# 1. 预测潜在问题
/autoresearch:predict --chain security
Scope: src/**
Goal: Pre-deployment security review

# 2. 自动修复严重问题
/autoresearch:security --fix --fail-on critical
Iterations: 15

# 3. 发布
/autoresearch:ship --auto
```

#### 新功能质量保证工作流

```bash
# 1. 场景探索
/autoresearch:scenario --depth deep
Scenario: User uploads file with multiple formats
Domain: software

# 2. 多角度预测
/autoresearch:predict --chain debug
Scope: src/new-feature/**

# 3. 修复发现的问题
/autoresearch:fix

# 4. 更新文档
/autoresearch:learn --mode update

# 5. 发布
/autoresearch:ship --monitor 10
```

#### 代码库健康检查工作流

```bash
# 1. 文档健康检查
/autoresearch:learn --mode check

# 2. Bug 狩猎
/autoresearch:debug

# 3. 安全审计
/autoresearch:security --diff

# 4. 修复所有问题
/autoresearch:fix

# 5. 更新文档
/autoresearch:learn --mode update
```

---

## 五、关键配置参数

### 循环模式

| 参数 | 说明 |
|------|------|
| 无 `Iterations` | 无界模式，永远运行直到手动中断（Ctrl+C） |
| `Iterations: N` | 有界模式，运行 N 次后停止并打印总结 |

### 验证相关

| 参数 | 说明 |
|------|------|
| `Verify` | **必需**，提取指标的 shell 命令 |
| `Guard` | 可选，回归测试命令 |
| `Noise: high` | 高噪声模式，自动多轮验证取中位数 |
| `Noise-Runs: N` | 自定义验证轮次（默认 3） |
| `Min-Delta: X` | 最小改进阈值，忽略小于 X 的改进 |

### 原子性配置

| 参数 | 说明 |
|------|------|
| `Atomicity: strict` | 严格模式（默认），单文件/单意图 |
| `Atomicity: relaxed` | 宽松模式，允许多文件协调变更 |
| `Max-Files-Per-Change: N` | 超过 N 个文件时警告（默认 5） |

### Git 记忆配置

| 参数 | 说明 |
|------|------|
| `Memory-Depth: N` | 读取多少过去的提交（默认 20） |
| `Diff-Review: HEAD~N` | 差异审查多远（默认 HEAD~1） |

---

## 六、核心原则（7 条）

### 1. 约束 = 赋能（Constraint = Enabler）

自治通过有意的约束成功，而非尽管有约束。

**为什么**：约束让代理有信心（完全理解上下文）、验证简单（无歧义）、迭代速度（低成本 = 快速反馈循环）。

**应用**：开始前定义：哪些文件在范围内？唯一的指标是什么？每次迭代的时间预算是多少？

### 2. 分离策略与战术（Separate Strategy from Tactics）

人类设定方向。代理执行迭代。

| 战略（人类） | 战术（代理） |
|-------------------|------------------|
| "提高页面加载速度" | "延迟加载图片、代码分割路由" |
| "增加测试覆盖率" | "为未覆盖的边缘情况添加测试" |
| "重构认证模块" | "提取中间件、简化处理器" |

**为什么**：人类理解为什么。代理处理如何。混合这些角色浪费人类的创造力和代理的迭代速度。

### 3. 指标必须机械（Metrics Must Be Mechanical）

如果你不能用命令验证，你就不能自主迭代。

- 测试通过/失败（退出码 0）
- 基准测试时间（毫秒）
- 覆盖率百分比
- Lighthouse 分数
- 文件大小（字节）
- 代码行数

**反模式**："看起来更好"、"可能改进了"、"似乎更干净" → 这些会杀死自主循环，因为没有决策函数。

### 4. 验证必须快（Verification Must Be Fast）

如果验证时间比工作本身更长，激励就会错位。

| 快（支持迭代） | 慢（杀死迭代） |
|-------------------------|----------------------|
| 单元测试（秒） | 完整 E2E 套件（分钟） |
| 类型检查（秒） | 手动 QA（小时） |
| Lint 检查（即时） | 代码审查（异步） |

**应用**：使用仍然能捕获真实问题的**最快**验证。慢验证留给循环之后。

### 5. 迭代成本塑造行为（Iteration Cost Shapes Behavior）

- 便宜的迭代 → 大胆探索、许多实验
- 昂贵的迭代 → 保守、很少实验

**应用**：最小化迭代成本。使用快速测试、增量构建、针对性验证。节省的每一分钟 = 更多运行的实验。

### 6. Git 是记忆和审计跟踪（Git as Memory and Audit Trail）

每次成功的变更都会提交。这启用：
- **因果跟踪** — 哪个变更推动了改进？
- **堆叠胜利** — 每次提交都建立在先前的成功之上
- **模式学习** — 代理看到在这个代码库中什么有效
- **人类审查** — 研究人员检查代理的决策序列

**应用**：验证前提交。失败时回滚。代理读取自己的 git 历史来指导下一次实验。

### 7. 诚实限制（Honest Limitations）

说明系统能做什么和不能做什么。不要过度推销。

**应用**：在设置时，明确说明约束。如果代理遇到无法解决的障碍（缺少权限、外部依赖、需要人类判断），清楚地说明，而不是猜测。

---

## 七、输出产物

每次运行会创建**带时间戳的输出目录**：

```
autoresearch-results.tsv  → 核心迭代结果日志
security/                 → 安全审计
ship/                     → 发布工作流
scenario/                 → 场景生成
predict/                  → 预测分析
learn/                    → 文档生成
```

### 目录结构示例

#### 安全审计输出
```
security/{YYMMDD}-{HHMM}-{audit-slug}/
├── overview.md              # 审计概述
├── threat-model.md         # STRIDE 威胁模型
├── attack-surface-map.md   # 攻击面映射
├── findings.md             # 发现详情
├── owasp-coverage.md       # OWASP 覆盖率
├── dependency-audit.md     # 依赖审计
├── recommendations.md      # 缓解建议
└── security-audit-results.tsv  # 结构化结果
```

#### 发布工作流输出
```
ship/{YYMMDD}-{HHMM}-{ship-slug}/
├── checklist.md        # 发布检查清单
├── ship-log.tsv        # 发布日志
└── summary.md          # 发布总结
```

#### 场景生成输出
```
scenario/{YYMMDD}-{HHMM}-{slug}/
├── scenarios.md        # 生成的场景
├── use-cases.md        # 用例
├── edge-cases.md       # 边缘情况
├── scenario-results.tsv # 结构化结果
└── summary.md          # 总结
```

#### 预测分析输出
```
predict/{YYMMDD}-{HHMM}-{slug}/
├── overview.md              # 分析概述
├── codebase-analysis.md     # 代码库分析
├── dependency-map.md        # 依赖映射
├── component-clusters.md    # 组件聚类
├── persona-debates.md       # 角色辩论
├── hypothesis-queue.md      # 假设队列
├── findings.md              # 发现
├── predict-results.tsv      # 结构化结果
└── handoff.json             # 交接配置
```

#### 文档生成输出
```
learn/{YYMMDD}-{HHMM}-{slug}/
├── learn-results.tsv        # 学习结果
├── summary.md               # 总结
├── validation-report.md     # 验证报告
└── scout-context.md         # 侦察上下文
```

---

## 八、领域适配

| 领域 | 指标 | 范围 | 验证命令 | 守卫 |
|--------|--------|-------|----------------|-------|
| 后端代码 | 测试通过 + 覆盖率 % | `src/**/*.ts` | `npm test` | — |
| 前端 UI | Lighthouse 分数 | `src/components/**` | `npx lighthouse` | `npm test` |
| ML 训练 | val_bpb / loss | `train.py` | `uv run train.py` | — |
| 博客/内容 | 字数 + 可读性 | `content/*.md` | 自定义脚本 | — |
| 性能 | 基准测试时间（ms） | 目标文件 | `npm run bench` | `npm test` |
| 重构 | 测试通过 + LOC 减少 | 目标模块 | `npm test && wc -l` | `npm run typecheck` |
| 安全 | OWASP + STRIDE 覆盖率 + 发现 | API/auth/middleware | `/autoresearch:security` | — |
| 发布 | 检查清单通过率（%） | 任何制品 | `/autoresearch:ship` | 领域特定 |
| 调试 | Bug 发现 + 覆盖率 | 目标文件 | `/autoresearch:debug` | — |
| 修复 | 错误数（更少） | 目标文件 | `/autoresearch:fix` | `npm test` |
| 场景分析 | 场景覆盖分数（更高） | 功能/领域文件 | `/autoresearch:scenario` | — |
| 预测 | 发现 + 假设（更高） | 目标文件 | `/autoresearch:predict` | — |
| 文档 | 验证通过率（更高） | `docs/*.md` | `/autoresearch:learn` | `npm test` |

适配循环到你的领域。原则是通用的；指标是领域特定的。

---

## 九、总结

**Autoresearch 是一个高度模块化、可组合的自动化工具系统**，核心是**基于 Git 记忆的自主迭代循环**。

**关键特点**：
- 🎯 **目标驱动**：设定清晰的量化目标
- 🔄 **自动迭代**：Claude 持续运行改进循环
- 🛡️ **安全机制**：自动回滚失败改动
- 📊 **可验证**：每次改动都有数值验证
- 🧠 **Git 记忆**：通过 git 历史学习成功和失败
- 🧩 **模块化**：8 个子命令覆盖不同场景
- 🔗 **可组合**：支持链式调用和迭代组合

**应用场景**：
- 代码优化（性能、覆盖率、质量）
- 安全审计（STRIDE + OWASP）
- Bug 狩猎（科学方法）
- 错误修复（自动迭代）
- 场景探索（边缘情况）
- 多角度预测（群体智能）
- 文档生成（自动更新）
- 发布工作流（通用交付）

**核心理念**：
> 自治在你约束范围、明确成功、机械化验证，并让代理优化战术而人类优化策略时扩展。

这不是"移除人类"。这是重新分配人类努力从执行到方向。人类通过专注于不可简化的创造性/战略性工作变得更有价值。