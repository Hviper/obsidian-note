
---

## 一、生命周期全景图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OpenSpec 完整生命周期                                  │
└─────────────────────────────────────────────────────────────────────────────┘

   用户想法         探索阶段          提案阶段          实施阶段          归档阶段
      │               │                │                │                │
      │               │                │                │                │
      ▼               ▼                ▼                ▼                ▼
 ┌────────┐      ┌────────┐       ┌────────┐       ┌────────┐       ┌────────┐
 │  Idea  │──────│Explore │───────│Propose │───────│ Apply  │───────│Archive │
 │        │      │        │       │        │       │        │       │        │
 │ "想添加│      │ 思考伙伴│       │ 生成工件│       │ 执行任务│       │ 完成归档│
 │  登录"  │      │ 澄清需求│       │ 规划方案│       │ 更新进度│       │ 同步规格│
 └────────┘      └────────┘       └────────┘       └────────┘       └────────┘
      │               │                │                │                │
      │               │                │                │                │
      │          ┌────┴────┐      ┌────┴────┐      ┌────┴────┐      ┌────┴────┐
      │          │ 触发方式 │      │ 触发方式 │      │ 触发方式 │      │ 触发方式 │
      │          │         │      │         │      │         │      │         │
      │          │ /opsx:  │      │ /opsx:  │      │ /opsx:  │      │ /opsx:  │
      │          │ explore │      │ propose │      │ apply   │      │ archive │
      │          └─────────┘      └─────────┘      └─────────┘      └─────────┘
      │               │                │                │                │
      │               │                │                │                │
      │               ▼                ▼                ▼                ▼
      │          无工件生成      proposal.md      代码实现          archive/
      │                          design.md        tasks.md 更新      YYYY-MM-DD-<name>/
      │                          tasks.md
      │
      └──────────────────────────────────────────────────────────────────────┐
                                                                               │
  生命周期流转（用户主导）────────────────────────────────────────────────────┘
```

---

## 二、四个核心 Skill 的职责边界

### 1. openspec-explore (探索阶段)

**角色定位**: 思考伙伴，不是执行者

**核心职责**:
- 深度对话，澄清用户真正想要什么
- 可视化想法（ASCII 图表）
- 探索代码库，发现现有模式和约束
- 比较不同方案
- 识别风险和未知

**输入**: 用户的模糊想法或具体问题
**输出**: 澄清后的理解，可能创建 proposal（如果用户要求）

**关键特点**:
```
┌─────────────────────────────────────────┐
│     Explore 阶段特点                     │
├─────────────────────────────────────────┤
│ ✓ 无固定流程 - 对话式探索                │
│ ✓ 不强制产出 - 可能只是讨论              │
│ ✓ 允许发散 - 有价值的题外话             │
│ ✓ 可视化优先 - 图表胜过文字              │
│ ✗ 不写代码 - 这是铁律                   │
│ ✗ 不强制结构 - 让模式自然涌现            │
└─────────────────────────────────────────┘
```

**触发条件**:
- 用户调用 `/opsx:explore`
- 或 skill 检测到用户在探索阶段
- 或用户说"我想了解..."、"我们在想..."、"能不能看看..."

**退出条件**:
- 用户说"可以开始了" → 转到 propose
- 用户说"去实现吧" → 转到 apply
- 用户离开 - 探索可以随时结束

---

### 2. openspec-propose (提案阶段)

**角色定位**: 工件生成器，结构化思考

**核心职责**:
- 创建 change 目录结构
- 生成所有必要的工件 (proposal, design, tasks)
- 遵循 schema 定义的工作流
- 生成可执行的任务清单

**输入**: 明确的变更名称或描述
**输出**: 完整的 change 工件集，准备好实施

**关键流程**:
```
┌─────────────────────────────────────────────────────────────────┐
│              Propose 阶段工件生成流程                            │
└─────────────────────────────────────────────────────────────────┘

  开始
    │
    ▼
┌──────────────────┐
│ 1. 确认变更名称   │  ← 如果用户没提供，询问用户
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 2. 创建 change   │  openspec new change "<name>"
│    目录结构      │  生成 .openspec.yaml
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 3. 获取工件顺序  │  openspec status --json
│    (依赖图)      │  解析 applyRequires 和依赖关系
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 4. 逐个生成工件  │  按依赖顺序
└────┬─────────────┘
     │
     ├─── proposal.md (what & why)
     │
     ├─── design.md (how)
     │
     └─── tasks.md (implementation steps)
     │
     ▼
┌──────────────────┐
│ 5. 验证完整性    │  所有 applyRequires 工件完成
└────┬─────────────┘
     │
     ▼
  完成 → 准备好执行 /opsx:apply
```

**触发条件**:
- 用户调用 `/opsx:propose`
- Explore 阶段用户说"创建提案吧"
- 用户明确说"我想添加/修改/实现..."

**退出条件**:
- 所有 `applyRequires` 工件生成完成
- 提示用户运行 `/opsx:apply`

---

### 3. openspec-apply-change (实施阶段)

**角色定位**: 任务执行器，代码实现者

**核心职责**:
- 读取 tasks.md 中的任务清单
- 按顺序实现每个任务
- 更新任务完成状态 `- [ ]` → `- [x]`
- 处理实施中的问题（阻塞、设计变更）

**输入**: change 名称（可选，从上下文推断）
**输出**: 实现的代码，更新的 tasks.md

**关键流程**:
```
┌─────────────────────────────────────────────────────────────────┐
│                 Apply 阶段实施流程                               │
└─────────────────────────────────────────────────────────────────┘

  开始
    │
    ▼
┌──────────────────┐
│ 1. 选择 change   │  ← 名称推断或让用户选择
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 2. 检查状态      │  openspec status --json
│    (schema/工件) │  确认工件完整
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 3. 获取指令      │  openspec instructions apply
│    (上下文文件)  │  读取 proposal/design/tasks
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 4. 显示进度      │  "N/M 任务完成"
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 5. 循环执行任务  │
└────┬─────────────┘
     │
     │   ┌─────────────────────┐
     │   │ 显示当前任务        │
     │   └──────┬──────────────┘
     │          │
     │          ▼
     │   ┌─────────────────────┐
     │   │ 实现代码变更        │
     │   └──────┬──────────────┘
     │          │
     │          ▼
     │   ┌─────────────────────┐
     │   │ 标记任务完成 ✓      │
     │   └──────┬──────────────┘
     │          │
     │          └─────► 下一个任务
     │
     ▼
┌──────────────────┐
│ 6. 完成或暂停    │
└────┬─────────────┘
     │
     ├─── 完成: 所有任务 ✓ → 建议归档
     │
     └─── 暂停: 遇到阻塞/设计问题 → 等待用户决策
```

**触发条件**:
- 用户调用 `/opsx:apply`
- Propose 完成后用户说"开始实现"
- 中途暂停后继续实施

**退出条件**:
- 所有任务完成 → 建议归档
- 遇到阻塞 → 暂停等待用户
- 用户中断 → 保存进度退出

**关键设计**:
```
┌─────────────────────────────────────────────────────────────┐
│            Apply 的"流体工作流"设计                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ✓ 可随时调用 - 不要求所有工件完美                           │
│  ✓ 允许工件更新 - 发现设计问题可更新 design.md               │
│  ✓ 支持暂停/恢复 - tasks.md 保存进度                         │
│  ✓ 非阶段锁定 - 可以在 apply 中回到 explore                  │
│                                                             │
│  示例:                                                      │
│  Task 3 发现设计问题 → 更新 design.md → 继续实施             │
│  Task 5 需求变更 → 更新 proposal.md → 调整 tasks → 继续       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### 4. openspec-archive-change (归档阶段)

**角色定位**: 变更终结者，规格同步者

**核心职责**:
- 检查工件和任务完整性
- 可选：同步 delta specs 到 main specs
- 移动 change 到 archive 目录
- 生成归档摘要

**输入**: change 名称（可选，从上下文推断）
**输出**: 归档目录，可选的规格更新

**关键流程**:
```
┌─────────────────────────────────────────────────────────────────┐
│                 Archive 阶段归档流程                             │
└─────────────────────────────────────────────────────────────────┘

  开始
    │
    ▼
┌──────────────────┐
│ 1. 选择 change   │  ← 列出活跃 changes，让用户选择
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 2. 检查工件完整性│  openspec status --json
│                  │  警告未完成的工件，确认是否继续
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 3. 检查任务完整性│  读取 tasks.md
│                  │  警告未完成任务，确认是否继续
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 4. 评估 spec 同步│  检查 delta specs
│                  │  对比 main specs
└────┬─────────────┘
     │
     ├─── 无 delta specs → 跳过同步
     │
     └─── 有 delta specs → 显示变更摘要，询问是否同步
                           │
                           ├─── 同步 → 执行同步
                           └─── 不同步 → 跳过
     │
     ▼
┌──────────────────┐
│ 5. 执行归档      │  mv openspec/changes/<name> \
│                  │     openspec/changes/archive/YYYY-MM-DD-<name>
└────┬─────────────┘
     │
     ▼
┌──────────────────┐
│ 6. 显示摘要      │  归档位置、规格同步状态、完成情况
└──────────────────┘

  完成
```

**触发条件**:
- 用户调用 `/opsx:archive`
- Apply 完成后所有任务 ✓
- 用户说"归档这个变更"

**退出条件**:
- 归档完成
- 用户取消

---

## 三、阶段之间的编排与触发

### 编排模型：用户主导，非自动流转

```
┌──────────────────────────────────────────────────────────────────┐
│          关键洞察：没有自动流转，用户是驾驶员                      │
└──────────────────────────────────────────────────────────────────┘

   Skill 之间不直接调用，用户决定何时转换阶段：

   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │ Explore │ ───────▶│ Propose │ ───────▶│  Apply  │
   └─────────┘         └─────────┘         └─────────┘
        │                    │                    │
        │                    │                    │
        │     用户说：        │     用户说：        │
        │     "创建提案吧"     │     "开始实现吧"    │
        │                    │                    │
        │                    │                    ▼
        │                    │              ┌─────────┐
        │                    │              │ Archive │
        │                    │              └─────────┘
        │                    │                    ▲
        │                    │                    │
        │                    │               用户说：
        │                    │               "归档吧"
        │                    │                    │
        │                    │                    │
        └────────────────────┴────────────────────┘
                    可以跳过、回退、重做

   灵活性：
   - 可以跳过 explore 直接 propose
   - 可以在 apply 中回到 explore（发现设计问题）
   - 可以不 archive（保持 change 活跃）
   - 可以多次 apply（分批实施）
```

### 触发矩阵

| 触发方式 | Explore | Propose | Apply | Archive |
|---------|---------|---------|-------|---------|
| **Skill 命令** | `/opsx:explore` | `/opsx:propose` | `/opsx:apply` | `/opsx:archive` |
| **自然语言** | "我在想..."<br>"我想了解..." | "我想添加..."<br>"帮我创建一个提案" | "开始实现吧"<br>"执行这些任务" | "归档这个变更"<br>"完成这个变更" |
| **阶段完成** | Explore 完成后建议 | Propose 完成后建议 | Apply 完成后建议 | - |
| **上下文推断** | 检测到探索性对话 | 检测到明确需求 | 检测到 tasks.md | 检测到所有任务完成 |

### 数据流：工件的生命周期

```
┌──────────────────────────────────────────────────────────────────┐
│              工件在生命周期中的流转                                │
└──────────────────────────────────────────────────────────────────┘

  Explore 阶段 (可能产生)
       │
       └──── 可能创建 → proposal.md (草稿)
                          │
                          ▼
  Propose 阶段 (必然产生)
       │
       ├─────→ proposal.md (what & why)
       ├─────→ design.md (how)
       └─────→ tasks.md (steps)
              │
              ▼
  Apply 阶段 (读取 + 更新)
       │
       ├─────→ 读取 proposal/design/tasks
       ├─────→ 实现代码
       └─────→ 更新 tasks.md (进度追踪)
              │
              ▼
  Archive 阶段 (归档 + 同步)
       │
       ├─────→ 移动到 archive/YYYY-MM-DD-<name>/
       └─────→ 可选：同步 delta specs → main specs
              │
              ▼
         归档目录/
           ├── .openspec.yaml
           ├── proposal.md
           ├── design.md
           ├── tasks.md
           └── specs/ (delta specs)
```

---

## 四、实战演示：添加用户登录认证功能

### 场景：用户说"我想为应用添加用户登录认证功能"

#### 阶段 1: Explore (探索)

**用户输入**:
```
"我想为应用添加用户登录认证功能"
```

**系统响应** (自动触发 openspec-explore 或用户调用 `/opsx:explore`):

```
## 进入探索模式

让我先了解一下你的应用上下文...

[读取代码库，查看现有架构]

┌─────────────────────────────────────────────────────────────────┐
│              认证选项光谱                                         │
└─────────────────────────────────────────────────────────────────┘

      简单                          复杂
       │                             │
       ▼                             ▼
  ┌─────────┐                 ┌──────────┐
  │ Email +  │                 │  OAuth   │
  │ Password │                 │ Providers │
  └─────────┘                 └──────────┘
       │                             │
       │                             │
       ▼                             ▼
  简单实现                    需要注册第三方
  安全性较低                  安全性较高

我看到你的应用是：
- Next.js 应用
- 已经有数据库连接（PostgreSQL）
- 目前没有用户系统

问题：
1. 你需要什么样的认证方式？
   - 传统邮箱密码？
   - OAuth (Google/GitHub)？
   - 两者都要？

2. 你需要什么级别的安全性？
   - 基础 session 认证？
   - JWT token？
   - 双因素认证？

3. 用户数据存储在哪里？
   - 继续用现有的 PostgreSQL？
   - 新的数据库？

4. 认证范围？
   - 只需要登录？
   - 还需要权限系统？

让我们先聊聊这些，然后再设计...
```

**对话继续**:
```
用户: 我需要邮箱密码登录，用现有的 PostgreSQL，先不用权限系统，
      但要为以后扩展留空间。

系统: 好的，让我画出架构方案...

┌─────────────────────────────────────────────────────────────────┐
│              认证架构方案                                         │
└─────────────────────────────────────────────────────────────────┘

  用户请求
     │
     ▼
  ┌──────────┐
  │ Next.js  │
  │  Route   │
  └────┬─────┘
       │
       ▼
  ┌──────────────────┐
  │ Auth Middleware  │ ← 新建
  │ (验证 session)   │
  └────┬─────────────┘
       │
       ├─── 公开路由 ──▶ 直接响应
       │
       └─── 受保护路由 ──▶ 检查 session
                              │
                              ▼
                         ┌────────────┐
                         │ PostgreSQL │
                         │  users表   │
                         └────────────┘

技术栈建议：
- 密码哈希：bcrypt
- Session 管理：iron-session 或 next-auth
- 数据库：Prisma (你已经在用)

扩展性：
- 预留 roles 字段
- 预留 permissions 表结构

这个方向对吗？还是你想看其他方案？
```

**用户确认**:
```
用户: 看起来不错，我们开始吧。

系统: 好的，想法已经清晰了。我可以为你创建一个变更提案吗？
      运行 /opsx:propose 我会生成 proposal、design 和 tasks。
```

---

#### 阶段 2: Propose (提案)

**用户输入**:
```
/opsx:propose add-user-auth
```

**系统响应**:

```
## 创建变更: add-user-auth

[创建目录] openspec/changes/add-user-auth/
[生成工件]  按 schema 依赖顺序...

┌─────────────────────────────────────────────────────────────────┐
│              工件生成顺序                                         │
└─────────────────────────────────────────────────────────────────┘

  Schema: spec-driven (从 .openspec.yaml 读取)

  依赖图:
  proposal ──▶ design ──▶ tasks

  1. ✓ proposal.md (无依赖)
  2. ✓ design.md (依赖 proposal)
  3. ✓ tasks.md (依赖 design)

[读取指令] openspec instructions proposal --change "add-user-auth" --json
[生成]     openspec/changes/add-user-auth/proposal.md

  ┌─────────────────────────────────────┐
  │ proposal.md                         │
  ├─────────────────────────────────────┤
  │ # 添加用户认证系统                   │
  │                                     │
  │ ## 背景                             │
  │ 当前应用无用户系统，需要添加邮箱密码 │
  │ 认证功能，使用现有 PostgreSQL       │
  │                                     │
  │ ## 范围                             │
  │ - 用户注册                          │
  │ - 用户登录/登出                     │
  │ - Session 管理                     │
  │ - 密码哈希                          │
  │ - 预留权限扩展                      │
  │                                     │
  │ ## 成功标准                         │
  │ - 用户可以注册和登录                │
  │ - 受保护路由需要认证                │
  │ - 密码安全存储                      │
  └─────────────────────────────────────┘

[读取指令] openspec instructions design --change "add-user-auth" --json
[生成]     openspec/changes/add-user-auth/design.md

  ┌─────────────────────────────────────┐
  │ design.md                           │
  ├─────────────────────────────────────┤
  │ # 设计：用户认证系统                 │
  │                                     │
  │ ## 架构                             │
  │ - 使用 iron-session 管理 session   │
  │ - 使用 bcrypt 哈希密码              │
  │ - 使用 Prisma 操作数据库            │
  │                                     │
  │ ## 数据模型                         │
  │ User 表：                           │
  │ - id: string (UUID)                 │
  │ - email: string (unique)            │
  │ - passwordHash: string              │
  │ - createdAt: DateTime               │
  │ - role: string (预留)               │
  │                                     │
  │ ## API 设计                         │
  │ - POST /api/auth/register          │
  │ - POST /api/auth/login             │
  │ - POST /api/auth/logout            │
  │ - GET /api/auth/session            │
  │                                     │
  │ ## 安全考虑                         │
  │ - 密码最小长度 8 位                 │
  │ - Session 过期时间 7 天             │
  │ - HTTPS only (production)          │
  └─────────────────────────────────────┘

[读取指令] openspec instructions tasks --change "add-user-auth" --json
[生成]     openspec/changes/add-user-auth/tasks.md

  ┌─────────────────────────────────────┐
  │ tasks.md                            │
  ├─────────────────────────────────────┤
  │ # 实施任务                          │
  │                                     │
  │ ## 数据层                           │
  │ - [ ] 1. 更新 Prisma schema         │
  │ - [ ] 2. 创建数据库迁移             │
  │                                     │
  │ ## 依赖安装                         │
  │ - [ ] 3. 安装 iron-session         │
  │ - [ ] 4. 安装 bcrypt               │
  │                                     │
  │ ## 核心实现                         │
  │ - [ ] 5. 创建 session 工具函数     │
  │ - [ ] 6. 创建密码哈希工具           │
  │ - [ ] 7. 实现注册 API              │
  │ - [ ] 8. 实现登录 API              │
  │ - [ ] 9. 实现登出 API              │
  │ - [ ] 10. 创建认证中间件            │
  │                                     │
  │ ## 测试                             │
  │ - [ ] 11. 测试注册流程              │
  │ - [ ] 12. 测试登录流程              │
  │ - [ ] 13. 测试受保护路由            │
  └─────────────────────────────────────┘

[验证]     所有 applyRequires 工件完成 ✓

## 变更提案完成！

**变更名称**: add-user-auth
**位置**: openspec/changes/add-user-auth/
**工件**:
- proposal.md - 定义范围和成功标准
- design.md - 技术架构和数据模型
- tasks.md - 13 个实施任务

准备好实施了！运行 `/opsx:apply` 开始执行任务。
```

---

#### 阶段 3: Apply (实施)

**用户输入**:
```
/opsx:apply
```

**系统响应**:

```
## 实施变更: add-user-auth

[选择变更] 从上下文推断: add-user-auth
[检查状态] openspec status --change "add-user-auth" --json

Schema: spec-driven
工件状态: ✓ proposal.md, ✓ design.md, ✓ tasks.md

[读取指令] openspec instructions apply --change "add-user-auth" --json
[读取上下文] proposal.md, design.md, tasks.md

┌─────────────────────────────────────────────────────────────────┐
│              实施进度                                             │
└─────────────────────────────────────────────────────────────────┘

进度: 0/13 任务完成

开始实施...

---

### 任务 1/13: 更新 Prisma schema

[实施代码变更]
[编辑文件] prisma/schema.prisma

  + model User {
  +   id           String   @id @default(uuid())
  +   email        String   @unique
  +   passwordHash String
  +   createdAt    DateTime @default(now())
  +   role         String   @default("user")
  + }

[标记完成] tasks.md: - [x] 1. 更新 Prisma schema

---

### 任务 2/13: 创建数据库迁移

[执行命令] npx prisma migrate dev --name add-user-auth

[输出]
  Applying migration `20260323_add_user_auth`
  ✔ Applied migration 20260323_add_user_auth in 123ms

[标记完成] tasks.md: - [x] 2. 创建数据库迁移

---

### 任务 3/13: 安装 iron-session

[执行命令] npm install iron-session

[输出]
  + iron-session@8.0.1

[标记完成] tasks.md: - [x] 3. 安装 iron-session

---

### 任务 4/13: 安装 bcrypt

[执行命令] npm install bcrypt
[执行命令] npm install -D @types/bcrypt

[输出]
  + bcrypt@5.1.1
  + @types/bcrypt@5.0.2

[标记完成] tasks.md: - [x] 4. 安装 bcrypt

---

### 任务 5/13: 创建 session 工具函数

[创建文件] lib/session.ts

  + import { getIronSession } from 'iron-session';
  + import { cookies } from 'next/headers';
  +
  + interface SessionData {
  +   userId?: string;
  +   email?: string;
  + }
  +
  + export async function getSession() {
  +   return await getIronSession<SessionData>(cookies(), {
  +     password: process.env.SESSION_SECRET!,
  +     cookieName: 'auth-session',
  +   });
  + }

[标记完成] tasks.md: - [x] 5. 创建 session 工具函数

---

### 任务 6/13: 创建密码哈希工具

[创建文件] lib/password.ts

  + import bcrypt from 'bcrypt';
  +
  + export async function hashPassword(password: string) {
  +   return await bcrypt.hash(password, 10);
  + }
  +
  + export async function verifyPassword(
  +   password: string,
  +   hash: string
  + ) {
  +   return await bcrypt.compare(password, hash);
  + }

[标记完成] tasks.md: - [x] 6. 创建密码哈希工具

---

### 任务 7/13: 实现注册 API

[创建文件] app/api/auth/register/route.ts

  + import { hashPassword } from '@/lib/password';
  + import { prisma } from '@/lib/prisma';
  +
  + export async function POST(req: Request) {
  +   const { email, password } = await req.json();
  +
  +   const existingUser = await prisma.user.findUnique({
  +     where: { email }
  +   });
  +
  +   if (existingUser) {
  +     return Response.json(
  +       { error: 'User already exists' },
  +       { status: 400 }
  +     );
  +   }
  +
  +   const passwordHash = await hashPassword(password);
  +
  +   const user = await prisma.user.create({
  +     data: { email, passwordHash }
  +   });
  +
  +   return Response.json({ userId: user.id });
  + }

[标记完成] tasks.md: - [x] 7. 实现注册 API

---

### 任务 8/13: 实现登录 API

[创建文件] app/api/auth/login/route.ts

  + import { verifyPassword } from '@/lib/password';
  + import { getSession } from '@/lib/session';
  + import { prisma } from '@/lib/prisma';
  +
  + export async function POST(req: Request) {
  +   const { email, password } = await req.json();
  +
  +   const user = await prisma.user.findUnique({
  +     where: { email }
  +   });
  +
  +   if (!user || !(await verifyPassword(password, user.passwordHash))) {
  +     return Response.json(
  +       { error: 'Invalid credentials' },
  +       { status: 401 }
  +     );
  +   }
  +
  +   const session = await getSession();
  +   session.userId = user.id;
  +   session.email = user.email;
  +   await session.save();
  +
  +   return Response.json({ success: true });
  + }

[标记完成] tasks.md: - [x] 8. 实现登录 API

---

### 任务 9/13: 实现登出 API

[创建文件] app/api/auth/logout/route.ts

  + import { getSession } from '@/lib/session';
  +
  + export async function POST() {
  +   const session = await getSession();
  +   session.destroy();
  +   return Response.json({ success: true });
  + }

[标记完成] tasks.md: - [x] 9. 实现登出 API

---

### 任务 10/13: 创建认证中间件

[创建文件] lib/auth-middleware.ts

  + import { getSession } from './session';
  + import { NextResponse } from 'next/server';
  +
  + export async function requireAuth() {
  +   const session = await getSession();
  +
  +   if (!session.userId) {
  +     return NextResponse.redirect('/login');
  +   }
  +
  +   return session;
  + }

[标记完成] tasks.md: - [x] 10. 创建认证中间件

---

### 任务 11/13: 测试注册流程

[测试] 手动测试注册
  1. 访问注册页面
  2. 输入邮箱和密码
  3. 检查数据库是否创建用户
  4. 验证密码已哈希

[结果] ✓ 注册成功，数据库正确存储用户

[标记完成] tasks.md: - [x] 11. 测试注册流程

---

### 任务 12/13: 测试登录流程

[测试] 手动测试登录
  1. 访问登录页面
  2. 输入注册的邮箱和密码
  3. 检查 session 是否创建
  4. 访问受保护路由

[结果] ✓ 登录成功，session 正确设置

[标记完成] tasks.md: - [x] 12. 测试登录流程

---

### 任务 13/13: 测试受保护路由

[测试] 测试认证中间件
  1. 未登录访问受保护路由 → 重定向到登录
  2. 登录后访问受保护路由 → 正常访问
  3. 登出后访问 → 重定向到登录

[结果] ✓ 认证中间件工作正常

[标记完成] tasks.md: - [x] 13. 测试受保护路由

---

## 实施完成！

**变更**: add-user-auth
**Schema**: spec-driven
**进度**: 13/13 任务完成 ✓

### 本轮完成
- [x] 1. 更新 Prisma schema
- [x] 2. 创建数据库迁移
- [x] 3. 安装 iron-session
- [x] 4. 安装 bcrypt
- [x] 5. 创建 session 工具函数
- [x] 6. 创建密码哈希工具
- [x] 7. 实现注册 API
- [x] 8. 实现登录 API
- [x] 9. 实现登出 API
- [x] 10. 创建认证中间件
- [x] 11. 测试注册流程
- [x] 12. 测试登录流程
- [x] 13. 测试受保护路由

所有任务完成！准备好归档了吗？运行 `/opsx:archive` 完成这个变更。
```

---

#### 阶段 4: Archive (归档)

**用户输入**:
```
/opsx:archive
```

**系统响应**:

```
## 归档变更

[选择变更] 从上下文推断: add-user-auth
[检查工件] openspec status --change "add-user-auth" --json

工件状态:
✓ proposal.md - done
✓ design.md - done
✓ tasks.md - done

[检查任务] tasks.md

任务状态:
✓ 13/13 任务完成

[检查 delta specs] openspec/changes/add-user-auth/specs/

发现 delta specs:
- auth.spec.md (新增认证能力规格)

[对比 main specs] openspec/specs/

auth.spec.md 变更:
  + 新增能力：用户认证
  + 新增 API：POST /api/auth/register
  + 新增 API：POST /api/auth/login
  + 新增 API：POST /api/auth/logout
  + 新增数据模型：User

是否同步到 main specs?

[选项]
1. 立即同步 (推荐)
2. 跳过同步，只归档

用户选择: 1

[执行同步] 使用 openspec-sync-specs skill

同步完成:
✓ openspec/specs/auth/spec.md 已创建

[执行归档]

[创建目录] openspec/changes/archive/
[移动]     openspec/changes/add-user-auth → openspec/changes/archive/2026-03-23-add-user-auth

---

## 归档完成！

**变更**: add-user-auth
**Schema**: spec-driven
**归档位置**: openspec/changes/archive/2026-03-23-add-user-auth/
**规格同步**: ✓ 已同步到 main specs

所有工件完成。所有任务完成。变更已归档。

---

## 变更历史

你现在可以查看：
- 归档工件: openspec/changes/archive/2026-03-23-add-user-auth/
- 同步的规格: openspec/specs/auth/spec.md
```

---

## 五、编排的核心设计原则

### 1. 用户是驾驶员

```
┌─────────────────────────────────────────────────────────────────┐
│          关键设计：Skills 不自动触发彼此                           │
└─────────────────────────────────────────────────────────────────┘

错误设计（自动流转）：
  Explore ──自动──▶ Propose ──自动──▶ Apply ──自动──▶ Archive

正确设计（用户驱动）：
  Explore ──用户说"创建提案"──▶ Propose ──用户说"开始实现"──▶ Apply
                                                                    │
                                                                    │
                                     用户说"归档" ──────────────────┘

原因：
1. 用户可能在 Explore 后不想继续
2. 用户可能在 Propose 后想修改
3. 用户可能想在 Apply 中回到 Explore
4. 用户可能想分批 Apply
5. 用户可能不想 Archive（保持 change 活跃）
```

### 2. 流体工作流（非阶段锁定）

```
┌─────────────────────────────────────────────────────────────────┐
│          允许回退、跳跃、重做                                      │
└─────────────────────────────────────────────────────────────────┘

场景 1: Apply 中发现设计问题
  Apply (Task 5) ──发现设计缺陷──▶ 回到 Explore
                                    │
                                    └──更新──▶ design.md
                                                │
                                                └──继续──▶ Apply (Task 5)

场景 2: Propose 后改变主意
  Propose ──看完 proposal 不喜欢──▶ 回到 Explore
                                      │
                                      └──重新设计──▶ Propose (新版本)

场景 3: 跳过阶段
  明确需求 ──直接──▶ Propose (跳过 Explore)

场景 4: 多次 Apply
  Apply (Task 1-5) ──暂停──▶ 用户离开
                                 │
                                 └──回来──▶ Apply (Task 6-10)
```

### 3. 工件驱动的状态管理

```
┌─────────────────────────────────────────────────────────────────┐
│          所有状态存储在文件中，不在内存中                          │
└─────────────────────────────────────────────────────────────────┘

文件系统作为状态存储：

openspec/changes/add-user-auth/
├── .openspec.yaml      ← Schema 定义，工件依赖图
├── proposal.md         ← 范围、背景
├── design.md           ← 技术决策
└── tasks.md            ← 实施进度 (- [x] vs - [ ])

优势：
1. 可以随时中断和恢复
2. 版本控制友好（git 可以追踪变更）
3. 跨会话持久化
4. 无需数据库
5. 人类可读
```

### 4. Schema 驱动的工作流

```
┌─────────────────────────────────────────────────────────────────┐
│          Schema 定义了工件类型和依赖关系                           │
└─────────────────────────────────────────────────────────────────┘

示例 Schema: spec-driven

.openspec.yaml:
  schema: spec-driven

Schema 定义（在 openspec 系统中）：
  artifacts:
    - id: proposal
      dependencies: []
    - id: design
      dependencies: [proposal]
    - id: tasks
      dependencies: [design]

  apply:
    requires: [tasks]  ← Apply 之前必须完成 tasks

Propose 阶段如何生成工件：
1. 读取 schema
2. 解析依赖图
3. 按依赖顺序生成：proposal → design → tasks
4. 验证 apply.requires 都完成

不同 Schema 可以定义不同的工作流：
- spec-driven: proposal → design → tasks
- test-driven: spec → tests → implementation → docs
- minimal: tasks (只有任务列表)
```

### 5. 渐进式强化

```
┌─────────────────────────────────────────────────────────────────┐
│          从模糊到清晰，逐步结构化                                  │
└─────────────────────────────────────────────────────────────────┘

模糊想法
    │
    ▼
Explore 阶段（无结构）
    - 对话式探索
    - 可视化想法
    - 可能创建 proposal 草稿
    │
    ▼
Propose 阶段（结构化工件）
    - proposal.md（明确范围）
    - design.md（技术方案）
    - tasks.md（可执行步骤）
    │
    ▼
Apply 阶段（代码实现）
    - 具体的代码变更
    - 进度追踪
    │
    ▼
Archive 阶段（持久化知识）
    - 归档变更历史
    - 同步规格到 main specs
    - 为未来提供参考

每个阶段都增加了结构，但允许回退到更模糊的状态。
```

---

## 六、架构洞察

### 1. 单向数据流（大部分情况）

```
Explore → Propose → Apply → Archive

工件生成方向：
  (无工件) → proposal/design/tasks → 代码变更 → 归档目录

例外：
- Apply 中可以回到 Explore（发现设计问题）
- Propose 后可以回到 Explore（改变主意）
- 但 Archive 是终态（不可逆）
```

### 2. 状态机视角

```
┌─────────────────────────────────────────────────────────────────┐
│          Change 的生命周期状态机                                   │
└─────────────────────────────────────────────────────────────────┘

  ┌─────────┐
  │  Idea   │ (用户的想法，还未创建 change)
  └────┬────┘
       │ /opsx:explore 或 自然语言
       ▼
  ┌─────────┐
  │Exploring│ (探索中，可能无工件或只有草稿)
  └────┬────┘
       │ /opsx:propose
       ▼
  ┌─────────┐
  │Proposing│ (生成工件中)
  └────┬────┘
       │ 工件完成
       ▼
  ┌─────────┐
  │  Ready  │ (工件完成，等待实施)
  └────┬────┘
       │ /opsx:apply
       ▼
  ┌─────────┐
  │Implement│ (实施中，0 < tasks < N)
  └────┬────┘
       │ 所有任务完成
       ▼
  ┌─────────┐
  │Complete │ (所有任务完成)
  └────┬────┘
       │ /opsx:archive
       ▼
  ┌─────────┐
  │Archived │ (已归档，终态)
  └─────────┘

状态查询：
  openspec status --change "<name>" --json
  → 返回当前状态和工件进度
```

### 3. CLI 作为状态引擎

```
┌─────────────────────────────────────────────────────────────────┐
│          openspec CLI 的角色                                      │
└─────────────────────────────────────────────────────────────────┘

Skills 不直接管理状态，而是调用 CLI：

  Skill                      CLI
  ─────────────────────────────────────────
  openspec-explore      →    (无 CLI 调用，自由探索)
  openspec-propose      →    openspec new change
                             openspec status --json
                             openspec instructions
  openspec-apply        →    openspec status --json
                             openspec instructions apply
  openspec-archive      →    openspec status --json
                             openspec list --json

CLI 职责：
1. 创建 change 目录结构
2. 读取 schema 定义
3. 解析工件依赖图
4. 验证工件完整性
5. 提供指令模板

Skill 职责：
1. 与用户对话
2. 调用 CLI 获取状态
3. 生成工件内容
4. 实现代码变更
5. 更新任务进度

这种分离使得：
- CLI 可以独立使用
- Skills 专注 AI 逻辑
- 状态管理由 CLI 保证一致性
```

---

## 七、总结：OpenSpec 生命周期的核心设计

### 核心价值观

1. **用户主导，AI 辅助**
   - Skills 不自动流转
   - 用户决定何时转换阶段
   - AI 提供建议但不强制

2. **渐进式结构化**
   - 从模糊到清晰
   - 从对话到工件
   - 从计划到代码

3. **流体工作流**
   - 允许回退
   - 允许跳跃
   - 不锁定阶段

4. **状态外置化**
   - 所有状态存储在文件中
   - 版本控制友好
   - 跨会话持久化

5. **Schema 可扩展**
   - 不同工作流定义不同 schema
   - 工件类型和依赖可配置
   - 支持多种开发模式

### 生命周期一图总结

```
用户想法
    │
    │  "我想添加登录功能"
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Explore (探索)                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  AI: "让我了解你的应用...                               │   │
│  │      你需要什么认证方式？                                │   │
│  │      需要权限系统吗？                                    │   │
│  │      现有架构是什么？"                                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│  输出: 澄清的需求，可能的技术方案                                │
└─────────────────────────────────────────────────────────────────┘
    │
    │  用户: "看起来不错，创建提案吧"
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Propose (提案)                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  AI: 调用 openspec new change                           │   │
│  │      按依赖顺序生成工件:                                 │   │
│  │        proposal.md ← 范围和成功标准                      │   │
│  │        design.md ← 技术架构                              │   │
│  │        tasks.md ← 实施步骤                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│  输出: 完整的变更工件集                                          │
└─────────────────────────────────────────────────────────────────┘
    │
    │  用户: "开始实施"
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Apply (实施)                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  AI: 读取 tasks.md                                       │   │
│  │      按顺序实施每个任务:                                  │   │
│  │        Task 1: 更新 schema... ✓                         │   │
│  │        Task 2: 创建迁移... ✓                            │   │
│  │        ...                                               │   │
│  │        Task 13: 测试... ✓                               │   │
│  │      更新任务进度 [ ] → [x]                              │   │
│  └─────────────────────────────────────────────────────────┘   │
│  输出: 实现的代码，完成的任务                                    │
└─────────────────────────────────────────────────────────────────┘
    │
    │  用户: "归档吧"
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Archive (归档)                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  AI: 检查工件完整性                                      │   │
│  │      检查任务完整性                                      │   │
│  │      可选: 同步 delta specs → main specs                │   │
│  │      移动到 archive/YYYY-MM-DD-<name>/                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│  输出: 归档目录，同步的规格                                      │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
  完成！
```

### 关键文件流转

```
Explore 阶段 (可能产生):
  openspec/changes/<name>/proposal.md (草稿)

Propose 阶段 (必然产生):
  openspec/changes/<name>/
  ├── .openspec.yaml      ← Schema 定义
  ├── proposal.md         ← 范围
  ├── design.md           ← 方案
  └── tasks.md            ← 任务清单

Apply 阶段 (更新 + 生成):
  openspec/changes/<name>/tasks.md  ← 更新进度
  src/                               ← 生成代码
  lib/                               ← 生成代码

Archive 阶段 (移动 + 同步):
  openspec/changes/archive/YYYY-MM-DD-<name>/  ← 归档
  ├── .openspec.yaml
  ├── proposal.md
  ├── design.md
  ├── tasks.md
  └── specs/                          ← delta specs

  openspec/specs/auth/spec.md         ← 同步到 main specs
```

### 编排的本质

**OpenSpec Skills 的编排不是代码级的编排，而是用户意图的流转。**

每个 Skill 是一个独立的能力模块：
- **Explore**: 思考伙伴（对话式）
- **Propose**: 结构化工具（生成工件）
- **Apply**: 执行引擎（实现任务）
- **Archive**: 终结者（归档知识）

用户通过自然语言或命令显式调用这些 Skills，而 Skills 之间通过文件系统（工件）传递状态。这种设计确保了：

1. **可控性**: 用户始终掌控流程
2. **可恢复性**: 任何阶段都可以暂停和恢复
3. **可追溯性**: 所有决策都记录在工件中
4. **可扩展性**: 新 Schema 可以定义新的工作流

---

## 八、Schema 系统深度剖析

### 8.1 Schema 的定义位置与结构

**Schema 物理位置**：
```
/opt/homebrew/lib/node_modules/@fission-ai/openspec/schemas/spec-driven/
├── schema.yaml          # 核心 schema 定义
└── templates/           # 工件模板
    ├── proposal.md
    ├── spec.md
    ├── design.md
    └── tasks.md
```

**项目级配置**：
```
/Users/aidis/Desktop/test/openspec/config.yaml
  └── schema: spec-driven  # 指定使用哪个 schema
```

### 8.2 Schema.yaml 核心结构

```yaml
name: spec-driven
version: 1
description: Default OpenSpec workflow - proposal → specs → design → tasks

artifacts:
  - id: proposal
    generates: proposal.md
    description: Initial proposal document outlining the change
    template: proposal.md
    instruction: |  # AI 生成指导
      Create the proposal document that establishes WHY this change is needed.
      ...
    requires: []     # 无依赖

  - id: specs
    generates: "specs/**/*.md"
    description: Detailed specifications for the change
    template: spec.md
    instruction: |
      Create specification files that define WHAT the system should do.
      ...
    requires:        # 依赖 proposal
      - proposal

  - id: design
    generates: design.md
    description: Technical design document with implementation details
    template: design.md
    instruction: |
      Create the design document that explains HOW to implement the change.
      ...
    requires:        # 依赖 proposal
      - proposal

  - id: tasks
    generates: tasks.md
    description: Implementation checklist with trackable tasks
    template: tasks.md
    instruction: |
      Create the task list that breaks down the implementation work.
      ...
    requires:        # 依赖 specs + design
      - specs
      - design

apply:
  requires: [tasks]  # Apply 前必须完成 tasks
  tracks: tasks.md   # 追踪进度的文件
  instruction: |
    Read context files, work through pending tasks, mark complete as you go.
```

### 8.3 依赖图解析与工件生成顺序

```
┌─────────────────────────────────────────────────────────────────┐
│          Schema 依赖图 (DAG)                                     │
└─────────────────────────────────────────────────────────────────┘

  ┌──────────┐
  │ proposal │ (无依赖，根节点)
  └────┬─────┘
       │
       ├───────────────────┐
       │                   │
       ▼                   ▼
  ┌──────────┐       ┌──────────┐
  │  specs   │       │  design  │
  └────┬─────┘       └────┬─────┘
       │                   │
       └─────────┬─────────┘
                 │
                 ▼
           ┌──────────┐
           │  tasks   │
           └────┬─────┘
                │
                │ apply.requires: [tasks]
                ▼
           ┌──────────┐
           │  APPLY   │ (实施阶段)
           └──────────┘

生成顺序（拓扑排序）：
  1. proposal (无依赖)
  2. specs + design (并行，都只依赖 proposal)
  3. tasks (依赖 specs + design)
```

**依赖解析算法（伪代码）**：
```python
def resolve_artifact_order(schema):
    artifacts = schema.artifacts
    completed = set()
    order = []

    while len(completed) < len(artifacts):
        # 找到所有依赖已满足的工件
        ready = [
            a for a in artifacts
            if a.id not in completed
            and all(dep in completed for dep in a.requires)
        ]

        if not ready:
            raise CycleDetectedError()

        # 按顺序生成（或并行生成）
        for artifact in ready:
            order.append(artifact)
            completed.add(artifact.id)

    return order
```

### 8.4 openspec instructions 命令的工作机制

**命令格式**：
```bash
openspec instructions <artifact-id> --change "<name>" --json
```

**返回的 JSON 结构**：
```json
{
  "context": "Project background from config.yaml...",
  "rules": ["Keep proposals under 500 words", ...],
  "template": "# Full template content...",
  "instruction": "Create the proposal document...",
  "outputPath": "openspec/changes/add-user-auth/proposal.md",
  "dependencies": ["proposal"]
}
```

**Skill 如何使用这些信息**：
```
1. 读取 dependencies 文件
   ├─ proposal.md (如果生成 design 或 specs)
   └─ design.md + specs/*.md (如果生成 tasks)

2. 读取 template
   └─ 用作输出文件的基础结构

3. 应用 context + rules 作为约束
   ├─ context: 项目技术栈、约定等
   └─ rules: 工件特定规则（如字数限制）

4. 遵循 instruction 生成内容
   └─ instruction 提供详细的内容指导

5. 写入 outputPath
   └─ 生成工件文件
```

### 8.5 openspec status 的状态验证逻辑

**命令格式**：
```bash
openspec status --change "<name>" --json
```

**返回结构**：
```json
{
  "schemaName": "spec-driven",
  "artifacts": [
    {
      "id": "proposal",
      "status": "done",
      "path": "proposal.md"
    },
    {
      "id": "specs",
      "status": "done",
      "path": "specs/**/*.md"
    },
    {
      "id": "design",
      "status": "done",
      "path": "design.md"
    },
    {
      "id": "tasks",
      "status": "done",
      "path": "tasks.md"
    }
  ],
  "applyRequires": ["tasks"],
  "state": "ready"
}
```

**状态判断逻辑（伪代码）**：
```python
def check_change_status(change_name):
    schema = load_schema(config.schema)
    artifacts_status = []

    for artifact in schema.artifacts:
        path = f"openspec/changes/{change_name}/{artifact.generates}"
        if file_exists(path):
            status = "done"
        else:
            status = "missing"
        artifacts_status.append({
            "id": artifact.id,
            "status": status,
            "path": path
        })

    # 判断是否可以 apply
    apply_ready = all(
        get_artifact_status(a_id) == "done"
        for a_id in schema.apply.requires
    )

    state = "ready" if apply_ready else "blocked"

    return {
        "schemaName": schema.name,
        "artifacts": artifacts_status,
        "applyRequires": schema.apply.requires,
        "state": state
    }
```

### 8.6 工件模板的设计哲学

**模板的作用**：
- 提供结构骨架（不提供内容）
- 标注占位符（`<!-- -->` 注释）
- 确保格式一致性

**proposal.md 模板**：
```markdown
## Why
<!-- Explain the motivation for this change. What problem does this solve? Why now? -->

## What Changes
<!-- Describe what will change. Be specific about new capabilities, modifications, or removals. -->

## Capabilities

### New Capabilities
<!-- Capabilities being introduced. Replace <name> with kebab-case identifier -->
- `<name>`: <brief description of what this capability covers>

### Modified Capabilities
<!-- Existing capabilities whose REQUIREMENTS are changing -->
- `<existing-name>`: <what requirement is changing>

## Impact
<!-- Affected code, APIs, dependencies, systems -->
```

**tasks.md 模板**（关键：进度追踪格式）：
```markdown
## 1. <!-- Task Group Name -->

- [ ] 1.1 <!-- Task description -->
- [ ] 1.2 <!-- Task description -->

## 2. <!-- Task Group Name -->

- [ ] 2.1 <!-- Task description -->
- [ ] 2.2 <!-- Task description -->
```

**关键设计**：
- `- [ ]` 格式可以被 CLI 解析追踪进度
- `- [x]` 表示已完成
- Apply 阶段会自动更新这些标记

### 8.7 Schema 可扩展性机制

**查看可用 schemas**：
```bash
openspec schemas --json
```

**自定义 schema 的方式**：

1. **Fork 现有 schema**：
```bash
openspec schema fork spec-driven my-custom-schema
```
这会复制 schema 到项目本地：
```
openspec/schemas/my-custom-schema/
├── schema.yaml
└── templates/
```

2. **创建新 schema**：
```bash
openspec schema init my-workflow
```

3. **修改项目配置**：
```yaml
# config.yaml
schema: my-custom-schema
```

**Schema 扩展能力**：
- 定义不同的工件类型
- 自定义依赖关系
- 定制指令模板
- 添加项目特定的 context 和 rules

### 8.8 关键洞察：Schema 系统的核心设计

```
┌─────────────────────────────────────────────────────────────────┐
│          Schema 系统的三个层次                                    │
└─────────────────────────────────────────────────────────────────┘

1. **声明层** (schema.yaml)
   - 定义工件类型和依赖
   - 提供生成指令
   - 配置模板路径

2. **模板层** (templates/)
   - 提供结构骨架
   - 占位符标注
   - 格式规范

3. **执行层** (openspec CLI)
   - 解析依赖图
   - 验证状态
   - 提供指令上下文

这种分层设计使得：
- 工作流可配置（声明层）
- 输出格式标准化（模板层）
- 状态管理自动化（执行层）
```

**数据流**：
```
用户请求 → Skill 调用 CLI → CLI 读取 schema.yaml
                                   ↓
                            解析依赖图
                                   ↓
                     返回工件生成顺序
                                   ↓
                    Skill 读取模板 + 指令
                                   ↓
                          生成工件内容
                                   ↓
                       写入文件系统
                                   ↓
                  CLI 验证状态 → 更新 state
```

---

## 九、总结与关键洞察

### OpenSpec 的核心价值

1. **用户主导的工作流**
   - Skills 不自动流转，用户明确控制每个阶段转换
   - 允许回退、跳跃、重做，完全灵活

2. **文件系统作为状态引擎**
   - 所有状态存储在工件文件中
   - 版本控制友好（git 可追踪）
   - 跨会话持久化

3. **声明式 Schema 系统**
   - 工件类型和依赖关系可配置
   - 模板保证格式一致性
   - CLI 负责依赖解析和状态验证

4. **渐进式结构化**
   - 从模糊想法到结构化工件
   - 从结构化计划到具体代码
   - 每个阶段都增加价值，但允许回退

### 技术架构亮点

**编排模型**：用户意图驱动，而非代码自动流转
- Explore: 思考伙伴（对话式）
- Propose: 结构化工具（生成工件）
- Apply: 执行引擎（实现任务）
- Archive: 终结者（归档知识）

**状态管理**：文件系统 + CLI
- `.openspec.yaml`: Schema 定义
- `proposal/design/tasks.md`: 工件状态
- `- [x] vs - [ ]`: 进度追踪

**依赖解析**：DAG 拓扑排序
- 自动识别可生成的工件
- 并行生成无依赖冲突的工件
- 验证所有必需工件完成

### 实战价值

这个系统的真正价值在于：
1. **可控性** - 用户始终知道自己在哪个阶段
2. **可恢复性** - 任何时候都可以暂停和恢复
3. **可追溯性** - 所有决策都有记录
4. **可扩展性** - 可以自定义 Schema 适配不同工作流

---

## 十、OpenSpec vs Superpowers：两种生命周期哲学的对比

### 10.1 核心哲学差异

```
┌───────────────────────────────────────────────────────────────────────┐
│          两种截然不同的生命周期设计哲学                                   │
└───────────────────────────────────────────────────────────────────────┘

OpenSpec: 用户主导的"工具箱"模型
├── 哲学: 用户是驾驶员，AI 是工具
├── 编排: 显式调用，手动流转
└── 控制: 用户决定一切，AI 辅助决策

Superpowers: AI 托管的"流水线"模型
├── 哲学: AI 是工程助手，用户是决策者
├── 编排: 自动流转，REQUIRED SUB-SKILL
└── 控制: AI 执行细节，用户控制关键节点
```

### 10.2 编排模式对比

**OpenSpec 的编排模型**：
```
用户想法
    │
    ▼
[用户决策] ──是否探索？──▶ /opsx:explore
    │                        │
    │                        ▼
    │                   探索完成
    │                        │
    └────────────[用户决策] ──是否提案？──▶ /opsx:propose
                                  │
                                  ▼
                             生成工件
                                  │
                             [用户决策] ──是否实施？──▶ /opsx:apply
                                  │
                                  ▼
                             执行任务
                                  │
                             [用户决策] ──是否归档？──▶ /opsx:archive
```

**Superpowers 的编排模型**：
```
用户提出需求
    │
    ▼
【自动触发】brainstorming
    │
    ├─── 自动探索
    ├─── [决策点#1] 技术方案选择
    ├─── 自动设计
    ├─── [决策点#2] 方案确认
    ├─── 自动生成 spec
    ├─── 自动审查循环 (循环直到通过)
    ├─── [决策点#3] 文档审查
    └─── 【必需调用】writing-plans  ← 自动流转
              │
              ▼
        writing-plans
              │
              ├─── 自动分解任务
              ├─── 自动生成计划
              ├─── 自动审查循环
              ├─── [决策点#4] 执行方式选择
              └─── 【必需调用】相应执行 skill  ← 自动流转
                          │
                          ▼
                   subagent-driven-development
                          │
                          ├─── 自动创建工作树
                          └─── FOR EACH 任务:
                                  ├─── Agent: implementer (自动)
                                  ├─── Agent: spec-reviewer (自动)
                                  │       └── 失败 → 循环修复
                                  └─── Agent: code-quality-reviewer (自动)
                                          └── 失败 → 循环修复
                              │
                              ▼
                    【必需调用】finishing-a-development-branch
                              │
                              ├─── 自动验证测试
                              ├─── [决策点#5] 完成方式选择
                              └─── 自动执行 git 操作
```

### 10.3 决策权分布对比

| 维度 | OpenSpec | Superpowers |
|------|----------|-------------|
| **阶段转换** | 用户显式决定 | AI 自动流转（REQUIRED SUB-SKILL） |
| **决策节点** | 每个阶段都是决策点 | 5个关键决策点 |
| **细节执行** | 用户主导，AI 辅助 | AI 自动执行，用户只在关键点介入 |
| **流程控制** | 完全灵活，可跳过/回退 | 结构化流程，自动推进 |

**OpenSpec 决策分布**：
```
用户控制: ████████████████████ 100%
AI 辅助:  ░░░░░░░░░░░░░░░░░░░░ 0%

用户在每个阶段都明确控制：
- 是否探索？
- 是否创建提案？
- 是否开始实施？
- 是否归档？
- 是否回退？
```

**Superpowers 决策分布**：
```
用户控制: ████░░░░░░░░░░░░░░░░ 20% (5个关键决策点)
AI 执行:  ████████████████░░░░ 80% (自动化流程)

用户只在关键节点决策：
#1: 选择技术方案
#2: 批准设计方案
#3: 审查设计文档
#4: 选择执行方式
#5: 选择完成方式
```

### 10.4 状态管理对比

**OpenSpec**：
```
文件系统驱动：
├── openspec/changes/<name>/proposal.md  (范围)
├── openspec/changes/<name>/design.md    (方案)
├── openspec/changes/<name>/tasks.md     (进度)
└── openspec/changes/<name>/.openspec.yaml (schema)

状态查询: openspec status --json
进度追踪: tasks.md 中的 - [x] vs - [ ]

特点：
✅ 版本控制友好
✅ 跨会话持久化
✅ 人类可读
✅ 无需额外工具
```

**Superpowers**：
```
混合驱动：
├── docs/superpowers/specs/<date>-<name>-design.md (设计)
├── docs/superpowers/plans/<date>-<name>-plan.md   (计划)
├── TodoWrite 任务列表 (内存)
├── Git worktrees (隔离环境)
└── 子代理上下文 (临时)

状态查询: TaskList / TodoWrite
进度追踪: TaskUpdate 更新状态

特点：
✅ 实时进度可视化
✅ 子代理隔离
✅ 自动清理机制
⚠️ 部分状态在内存中
```

### 10.5 质量保证机制对比

**OpenSpec**：
```
质量保证: 依赖用户决策和显式验证

工件生成:
├── Schema 定义格式规范
├── 模板确保结构一致
└── 依赖图确保逻辑顺序

质量检查:
├── 用户显式审查每个工件
├── apply 前验证工件完整性
└── archive 前检查任务完成度

特点：
✅ 灵活，用户控制质量标准
✅ 适合不同质量要求的项目
⚠️ 依赖用户经验判断
```

**Superpowers**：
```
质量保证: 强制最佳实践 + 多重审查循环

强制机制:
├── test-driven-development (TDD 流程)
├── verification-before-completion (完成前验证)
├── systematic-debugging (调试流程)
└── 循环审查机制

审查层次:
├── spec-document-reviewer (设计文档审查)
├── plan-document-reviewer (计划文档审查)
├── spec-reviewer (实现符合规格审查)
├── code-quality-reviewer (代码质量审查)
└── final-code-reviewer (最终审查)

循环机制:
实现 → 审查 → [失败] → 修复 → 重新审查 → [通过]
(自动循环直到通过)

特点：
✅ 强制高质量标准
✅ 自动化质量检查
✅ 多层审查确保质量
⚠️ 流程较重，速度较慢
```

### 10.6 上下文管理对比

**OpenSpec**：
```
单一上下文模型：
├── 主会话处理所有阶段
├── 工件文件传递上下文
└── 无子代理隔离

上下文流转:
Explore (对话) → 文件 → Propose (读取+生成) → 文件 → Apply (读取+实施)

特点:
✅ 简单，无调度开销
✅ 全局视野，容易跨任务协调
⚠️ 上下文可能膨胀
⚠️ 长对话可能丢失早期信息
```

**Superpowers**：
```
多层级上下文模型：
├── 主会话 (主控制器)
├── Git worktrees (文件隔离)
├── 子代理 (任务隔离)
│   ├── implementer-agent (实施上下文)
│   ├── spec-reviewer-agent (审查上下文)
│   └── code-quality-reviewer-agent (质量上下文)
└── TodoWrite (进度状态)

上下文流转:
主会话调度 → 子代理 (独立上下文) → 返回结果 → 主会话继续

特点:
✅ 防止上下文污染
✅ 每个任务有清晰边界
✅ 子代理专注单一职责
⚠️ 调度开销
⚠️ 跨任务协调需要主会话
```

### 10.7 灵活性 vs 结构化对比

**OpenSpec: 高灵活性，低自动化**
```
灵活场景:
1. 跳过 explore，直接 propose
2. 在 apply 中回到 explore
3. 多次 apply (分批实施)
4. 修改已生成的工件
5. 不 archive (保持活跃)

适合:
✅ 不确定需求的项目
✅ 需要频繁调整方向
✅ 有经验的开发者
✅ 非标准化工作流
```

**Superpowers: 高结构化，高自动化**
```
结构化流程:
1. 固定阶段顺序: brainstorming → writing-plans → execution → finishing
2. REQUIRED SUB-SKILL 强制流转
3. 每个阶段有固定输入输出
4. 内置审查循环

适合:
✅ 需求明确的项目
✅ 标准化开发流程
✅ 团队协作项目
✅ 高质量要求的场景
```

### 10.8 典型使用场景对比

**OpenSpec 适合**:
```
场景 1: 探索性项目
"我想试试添加一个新功能，还不确定具体要做什么"
→ Explore 阶段深度探索
→ 可能发现不需要实施
→ 或者完全改变方向

场景 2: 经验丰富的开发者
"我知道要做什么，给我生成任务清单就行"
→ 直接 /opsx:propose
→ 快速实施

场景 3: 复杂决策过程
"这个功能涉及多个系统，我需要反复思考和调整"
→ Explore ↔ Propose ↔ Apply 多次循环
→ 灵活回退和修改
```

**Superpowers 适合**:
```
场景 1: 标准功能开发
"添加用户登录认证"
→ 自动完成整个流程
→ 只在5个关键点决策
→ 保证高质量

场景 2: 团队协作项目
"我们需要确保代码质量和一致性"
→ 强制 TDD 流程
→ 多重审查机制
→ 标准化输出

场景 3: 学习最佳实践
"我想学习如何正确地开发功能"
→ 观察 AI 的自动化流程
→ 学习 TDD、审查、验证等实践
```

### 10.9 技术架构对比

| 维度 | OpenSpec | Superpowers |
|------|----------|-------------|
| **核心机制** | CLI + 文件系统 | Skills 调用链 + 子代理 |
| **状态存储** | 文件 (Markdown + YAML) | 文件 + 内存 (TodoWrite) |
| **依赖解析** | Schema DAG (openspec CLI) | REQUIRED SUB-SKILL (Skills) |
| **进度追踪** | tasks.md checkbox | TodoWrite + TaskUpdate |
| **上下文隔离** | 无 | Git worktrees + 子代理 |
| **质量保证** | 用户审查 | 多重审查循环 |

### 10.10 设计哲学的根源

**OpenSpec 的哲学根源**：
```
Unix 哲学 + 文件系统优先

核心信念:
1. 文件是最好的接口
2. 用户知道自己要什么
3. 工具应该做一件事并做好
4. 组合而非集成

体现:
├── CLI 命令独立使用
├── 文件作为状态存储
├── Skills 不相互调用
└── 用户显式控制流程
```

**Superpowers 的哲学根源**：
```
软件工程最佳实践 + 自动化优先

核心信念:
1. 最佳实践应该强制执行
2. AI 可以托管细节执行
3. 质量源于流程，而非个人
4. 集成而非组合

体现:
├── REQUIRED SUB-SKILL 自动流转
├── 多重审查循环强制质量
├── Skills 之间紧密集成
└── AI 自动化大部分工作
```

### 10.11 选择建议

**选择 OpenSpec 如果**:
- ✅ 你是经验丰富的开发者
- ✅ 项目需求不确定，需要探索
- ✅ 你想要完全控制流程
- ✅ 你喜欢 Unix 风格的工具
- ✅ 项目需要灵活调整方向
- ✅ 你重视版本控制和可追溯性

**选择 Superpowers 如果**:
- ✅ 你想要学习最佳实践
- ✅ 项目需求明确，标准化开发
- ✅ 你想要 AI 托管细节
- ✅ 团队需要强制质量标准
- ✅ 你喜欢自动化流程
- ✅ 项目重视代码质量和审查

### 10.12 融合可能性

**可以借鉴 Superpowers 的地方**：
```
为 OpenSpec 添加可选的自动化:

1. 循环审查机制:
   propose 阶段:
     生成 proposal → 自动审查 → [有问题] → 修复 → 重新审查

2. 验证机制:
   apply 阶段:
     每个任务完成后 → 自动测试 → [失败] → 暂停 → 建议调试

3. 子代理模式:
   复杂任务:
     主会话 → 调度子代理 → 隔离上下文 → 返回结果
```

**可以借鉴 OpenSpec 的地方**：
```
为 Superpowers 添加灵活性:

1. 跳过阶段:
   允许用户显式跳过 brainstorming，直接进入 writing-plans

2. 回退机制:
   在 apply 中发现设计问题 → 回到 brainstorming → 更新设计 → 继续

3. 文件驱动状态:
   将 TodoWrite 状态持久化到文件 → 版本控制友好
```

---

## 十一、最终总结：理解两种范式的价值

### OpenSpec 和 Superpowers 代表了两种互补的范式

**OpenSpec = 工具箱范式**：
- 用户是工匠，选择合适的工具
- 工具之间独立，组合使用
- 灵活性优先，用户对结果负责

**Superpowers = 流水线范式**：
- AI 是工程师，托管执行
- 流程之间集成，自动流转
- 质量优先，流程对结果负责

### 没有绝对的好坏，只有适合的场景

**OpenSpec 更适合**:
- 探索性、不确定性高的项目
- 经验丰富的开发者
- 需要灵活调整的项目

**Superpowers 更适合**:
- 标准化、确定性高的项目
- 学习最佳实践的新手
- 团队协作和代码质量要求高的项目

### 两个系统的共同价值

**都解决的核心问题**:
1. 如何组织 AI 辅助开发的工作流
2. 如何追踪进度和状态
3. 如何保证质量
4. 如何降低认知负担

**都提供的能力**:
1. 结构化的思考过程
2. 可追溯的决策记录
3. 渐进式的实施方法
4. 清晰的生命周期管理

### 对开发者的启示

**理解这两种范式的价值在于**：
1. 认识到 AI 辅助开发有多种模式
2. 根据项目特点选择合适的模式
3. 可以在不同阶段使用不同的方法
4. 可以融合两种模式的优点

**最重要的是**：
选择适合你和你的项目的方式，而不是盲目追求自动化或灵活性。好的工具应该服务于你的工作方式，而不是强迫你适应工具。
