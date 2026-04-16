# 亚马逊 SP 广告智能运营助手 MVP 技术设计

> 基于 openclaw 平台的 Amazon SP 广告智能运营助手 MVP 技术实现设计。

版本日期：2026-04-16

## 1. 项目概述

### 1.1 项目定位

围绕单个 Listing / 广告对象持续运行的 Amazon SP 广告智能运营助手 MVP，基于 openclaw 平台能力构建。

### 1.2 核心架构

采用 **纯 Agent + MCP 工具** 方案：所有业务逻辑在 openclaw Agent 的 System Prompt + Skill 中实现，通过 MCP 工具调用 Amazon Ads API 和业务数据存储。

```
飞书消息 → openclaw Agent (含 Skill 策略层)
              ├── Amazon Ads MCP Server (fork 开源)
              ├── Storage MCP Server (SQLite CRUD)
              ├── Seller Data MCP Server (可选，降级时跳过)
              └── Cron 定时任务 (openclaw 内置)
         → 结果返回飞书
```

### 1.3 设计原则

- **零侵入 openclaw 源码** — 不修改 openclaw 任何核心代码
- **独立仓库 + 安装脚本** — 业务代码独立 Git 管理，通过脚本注册到 openclaw
- **配置分离** — openclaw 升级不影响业务代码，业务代码更新不影响 openclaw
- **降级友好** — 任意 MCP 能力缺失时，系统可降级运行并通过飞书请求用户补充

## 2. 整体架构与组件关系

```
┌─────────────────────────────────────────────────────────┐
│                    飞书 (Feishu)                         │
│  用户对话 ──→ Feishu Plugin ──→ openclaw Agent           │
│            ←── 消息/卡片推送  ←── Cron 定时任务           │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              openclaw Agent (核心调度)                    │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ System Prompt │  │ Skill 策略层  │  │  Cron 调度    │  │
│  │ (角色+流程)   │  │ (业务规则)    │  │ (定时任务)    │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────┘
           │              │              │
           ▼              ▼              ▼
┌─────────────────────────────────────────────────────────┐
│                  MCP 工具层                              │
│  ┌─────────────┐ ┌──────────────┐ ┌─────────────────┐  │
│  │ Amazon Ads  │ │  Storage     │ │  Seller Data    │  │
│  │ MCP Server  │ │  MCP Server  │ │  MCP (可选)     │  │
│  │ (fork 开源)  │ │  (SQLite)    │ │                 │  │
│  └─────────────┘ └──────────────┘ └─────────────────┘  │
└─────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│              Amazon Ads API / Seller API                 │
└─────────────────────────────────────────────────────────┘
```

### 2.1 组件来源映射

| 组件            | 来源                                 | 说明                              |
| --------------- | ------------------------------------ | --------------------------------- |
| Feishu Plugin   | openclaw 已有 (`extensions/feishu/`) | 飞书消息通道，直接复用            |
| openclaw Agent  | openclaw 已有                        | Agent 运行时 + System Prompt      |
| Skill 策略层    | **新建** (Prompt 文件)               | 投前/巡检/复盘等业务规则          |
| Cron 调度       | openclaw 已有 (`src/cron/`)          | 日巡检、周复盘定时触发            |
| Amazon Ads MCP  | **新建** (fork 开源)                 | 广告 CRUD、关键词管理、竞价调整   |
| Storage MCP     | **新建**                             | SQLite 操作（Run 存储、审计日志） |
| Seller Data MCP | **新建**（可选）                     | Listing、库存、价格数据           |

### 2.2 决策分层机制

| 维度         | 在 openclaw                | 在飞书                 |
| ------------ | -------------------------- | ---------------------- |
| 风险等级判断 | Skill 规则决定哪些需要审批 | —                      |
| 方案生成     | Agent + MCP 数据分析生成   | —                      |
| 审批决策     | —                          | 用户点击确认/修改/取消 |
| API 执行     | Agent 调用 MCP 执行        | —                      |
| 结果通知     | Agent 推送到飞书           | 用户看到结果           |

MVP 阶段所有涉及广告账户写入的操作默认需要飞书确认。

## 3. MCP Server 设计

### 3.1 Amazon Ads MCP Server（核心）

基于 fork 开源项目封装为独立 MCP Server，使用 `@modelcontextprotocol/sdk` 实现。

**工具清单**：

| 工具名称                      | 对应用例    | 说明                           |
| ----------------------------- | ----------- | ------------------------------ |
| `list_campaigns`              | 查询        | 列出 SP Campaign 列表及状态    |
| `get_campaign_details`        | UC-05/06/09 | 获取单个 Campaign 详细指标     |
| `get_campaign_report`         | UC-05/06    | 获取指定时间范围的广告报告数据 |
| `create_campaign`             | UC-03/04    | 创建 SP Campaign               |
| `update_campaign`             | UC-10       | 修改 Campaign 预算/状态/竞价   |
| `list_ad_groups`              | 查询        | 列出 Ad Group                  |
| `create_ad_group`             | UC-03/04    | 创建 Ad Group                  |
| `list_keywords`               | UC-02/05    | 列出关键词及表现数据           |
| `create_keywords`             | UC-03/04    | 批量创建关键词                 |
| `update_keywords`             | UC-10       | 修改关键词竞价/状态            |
| `add_negative_keywords`       | UC-03/10    | 添加否定词                     |
| `get_keyword_recommendations` | UC-02       | 获取亚马逊推荐关键词           |
| `get_account_budget`          | UC-05       | 获取账户预算使用情况           |

**凭证管理**：通过环境变量注入 `AMZ_ADS_CLIENT_ID`、`AMZ_ADS_CLIENT_SECRET`、`AMZ_ADS_REFRESH_TOKEN`、`AMZ_ADS_PROFILE_ID`。stdio 传输模式，由 openclaw Agent 进程启动。

### 3.2 Storage MCP Server（业务数据）

独立的 MCP Server，负责所有业务数据的 CRUD 操作，底层使用 SQLite。

**工具清单**：

| 工具名称                 | 说明                                             |
| ------------------------ | ------------------------------------------------ |
| `save_run`               | 保存 Run 结果（类型、状态、摘要、详细数据）      |
| `query_runs`             | 按 Listing/类型/时间范围查询 Run 历史            |
| `get_run_detail`         | 获取单个 Run 完整详情                            |
| `save_keyword_library`   | 保存词库（核心词、长尾词、否定词）               |
| `load_keyword_library`   | 加载词库                                         |
| `log_action`             | 记录操作审计日志（谁、什么时候、做了什么、结果） |
| `query_audit_log`        | 查询审计日志                                     |
| `save_report`            | 保存日报/周报                                    |
| `query_reports`          | 查询历史报告                                     |
| `save_manual_input`      | 保存用户手动提供的数据（降级场景）               |
| `load_manual_input`      | 加载用户手动提供的数据缓存                       |
| `save_skill_candidate`   | 保存候选 Skill 规则                              |
| `query_skill_candidates` | 查询候选 Skill 规则                              |

**SQLite 表结构**：

```sql
-- Run 结果
CREATE TABLE runs (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    type TEXT NOT NULL,           -- inspection/review/prelaunch/plan/publish/query
    status TEXT NOT NULL,         -- success/warning/error
    summary TEXT,
    detail_json TEXT,
    created_at TEXT NOT NULL
);

-- 关键词词库
CREATE TABLE keyword_libs (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    keywords_json TEXT NOT NULL,
    neg_keywords_json TEXT,
    version INTEGER DEFAULT 1,
    created_at TEXT NOT NULL
);

-- 审计日志
CREATE TABLE audit_log (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    action TEXT NOT NULL,         -- create/update/pause/resume/delete
    actor TEXT NOT NULL,          -- agent/user/system
    target_type TEXT,             -- campaign/ad_group/keyword
    target_id TEXT,
    result TEXT,                  -- success/failure
    detail_json TEXT,
    created_at TEXT NOT NULL
);

-- 日报/周报
CREATE TABLE reports (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    type TEXT NOT NULL,           -- daily/weekly
    content_json TEXT NOT NULL,
    created_at TEXT NOT NULL
);

-- 用户手动输入（降级场景）
CREATE TABLE manual_inputs (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    data_type TEXT NOT NULL,      -- listing/inventory/review/price
    content_json TEXT NOT NULL,
    source TEXT DEFAULT 'manual', -- manual/api
    expires_at TEXT,              -- 缓存过期时间（默认 24 小时）
    created_at TEXT NOT NULL
);

-- 候选 Skill 规则
CREATE TABLE skill_candidates (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,         -- user/agent/dev
    rule_text TEXT NOT NULL,
    status TEXT DEFAULT 'candidate', -- candidate/approved/rejected
    approved_by TEXT,
    created_at TEXT NOT NULL,
    applied_at TEXT
);
```

### 3.3 Seller Data MCP Server（降级可选）

Listing、库存、价格、评论等数据读取。如 MVP 阶段数据获取有困难，系统降级为通过飞书请求用户手动提供。

**工具清单**（如有条件接入）：

| 工具名称               | 说明                                |
| ---------------------- | ----------------------------------- |
| `get_listing_info`     | 获取 Listing 标题、图片、价格、描述 |
| `get_inventory_status` | 获取库存状态                        |
| `get_reviews_summary`  | 获取评论摘要和评分                  |
| `get_buyer_box_price`  | 获取购物车价格                      |

### 3.4 降级策略

| 能力           | 正常模式                  | 降级模式                                                        |
| -------------- | ------------------------- | --------------------------------------------------------------- |
| Listing 数据   | Seller MCP 自动获取       | 飞书请求用户填写，数据缓存到 `manual_inputs` 表                 |
| 库存状态       | Seller MCP 自动获取       | 用户主动告知或上传截图                                          |
| 关键词搜索量   | Keyword MCP               | 仅基于产品信息 + 亚马逊推荐词生成基础词库                       |
| 自然排名       | Rank MCP                  | 跳过排名数据，仅输出广告指标                                    |
| 广告执行       | Amazon Ads MCP 写操作可用 | 仅生成建议方案，用户确认后手动执行                              |
| Storage MCP    | SQLite 正常读写           | 结果直接输出到飞书 Thread（依赖消息历史回溯），恢复后补录       |
| 飞书通道       | 飞书插件正常推送          | 结果写入日志文件 + 控制台输出，通道恢复后补推                   |
| Amazon Ads API | API 正常响应              | 指数退避重试（最多 3 次），持续失败时通过飞书告警，跳过本次 Run |
| 通知能力       | 飞书主动推送              | 仅在系统内查看（Storage MCP 保存结果供后续查询）                |

Agent 每次需要数据时先查 `manual_inputs`，有缓存且未过期（默认 24 小时 TTL）则直接使用，否则向用户请求。正常 MCP 恢复后自动切回，不再使用过期缓存。

## 4. Skill 策略层与 Agent 配置

### 4.1 Agent 配置

通过 openclaw 原生配置机制（`~/.openclaw/openclaw.json`）注册专用 Agent。MCP Server 通过全局 `mcp.servers` 配置键注册，不在 Agent 级别配置。

**MCP Server 注册**（setup.sh 通过 `openclaw mcp set` 执行，参数为 JSON 对象）：

```bash
openclaw mcp set amazon-ads '{"command":"node","args":["/opt/aws-sp-assistant/mcp-servers/amazon-ads-mcp/dist/index.js"]}'
openclaw mcp set biz-storage '{"command":"node","args":["/opt/aws-sp-assistant/mcp-servers/storage-mcp/dist/index.js"]}'
# seller-data-mcp 为可选，按需注册
```

**Agent Profile**（setup.sh 注入到 `~/.openclaw/openclaw.json` 的 `agents.list[]`）：

```jsonc
// ~/.openclaw/openclaw.json 片段
{
  "agents": {
    "list": [
      {
        "id": "aws-sp-assistant",
        // systemPromptOverride 通过读取 Skill 文件拼接，见 setup.sh
        "systemPromptOverride": "/* 见下方 System Prompt */",
      },
    ],
  },
}
```

**System Prompt 核心内容**：

```
你是一位专业的亚马逊 SP 广告运营助手，围绕单个 Listing/广告对象持续提供运营支持。

## 核心职责
1. 投前检查 → 评估 Listing 是否具备投放条件
2. 词库建设 → 生成并维护关键词词库
3. 方案搭建 → 生成 Campaign/Ad Group/Keyword 结构方案
4. 日常巡检 → 识别异常，输出优化建议
5. 周度复盘 → 归因分析，输出下周建议
6. 动作执行 → 经用户确认后执行广告操作

## 决策规则
- 所有广告账户写入操作（创建、修改、暂停）必须经用户在飞书确认
- 低风险读取操作可直接执行
- 预算调整幅度超过 ±20% 需要明确确认
- 暂停操作需要附带原因说明

## 工作流程
- 每次执行形成独立 Run，结果通过 save_run 持久化
- 所有执行动作通过 log_action 记录审计日志
- 定时任务（日巡检、周复盘）由 Cron 触发
```

### 4.2 Skill 文件结构

每个核心用例对应一个 Skill Prompt 文件：

```
skills/
├── pre-launch-check.md      # UC-01 投前检查策略
├── keyword-library.md        # UC-02 词库初始化策略
├── sp-campaign-plan.md       # UC-03 SP 搭建方案策略
├── sp-campaign-publish.md    # UC-04 SP 发布执行流程
├── daily-inspection.md       # UC-05 日巡检策略
├── weekly-review.md          # UC-06 周复盘策略
├── notification-template.md  # UC-07 推送通知话术模板
├── task-collaboration.md     # UC-08 任务式协作流程
├── custom-query.md           # UC-09 自定义查询规范
└── risk-rules.md             # 风险等级判定规则
```

### 4.3 Skill 示例（日巡检）

```markdown
# skills/daily-inspection.md

## 触发

每天上午 9:00 由 Cron 自动触发，或用户主动发起。

## 数据获取

1. 调用 get_campaign_report 获取昨日广告数据
2. 调用 get_campaign_details 获取各 Campaign 状态
3. 调用 query_runs 获取上次巡检结果作为基线

## 异常识别规则

- ACOS 较 7 日均值上升超过 30% → 标记异常
- CTR 低于 0.2% → 标记低效
- 日消耗达到预算 90% 且转化不足 → 标记预算风险
- 广告活动状态非预期暂停 → 标记异常
- 单个关键词花费占比超过总花费 40% → 标记集中风险

## 输出结构

- 整体健康评分（绿/黄/红）
- 异常项列表（每项含：对象、指标、变化幅度、归因、建议动作）
- 日报摘要（适合飞书卡片展示）
- 建议动作列表（标注风险等级和是否需要确认）

## 持久化

- 调用 save_run 保存巡检结果
- 调用 save_report 保存日报
```

### 4.4 Cron 定时任务

| 任务     | Cron 表达式   | 说明                  |
| -------- | ------------- | --------------------- |
| 日巡检   | `0 9 * * *`   | 每天上午 9 点触发     |
| 周复盘   | `0 10 * * 1`  | 每周一上午 10 点触发  |
| 异常监控 | `0 */4 * * *` | 每 4 小时检查关键指标 |

Cron 触发后由 openclaw 的 isolated-agent 机制执行，结果通过飞书主动推送。

### 4.5 Skill 自进化机制

| 路径           | 实现方式                                                    | MVP 优先级 |
| -------------- | ----------------------------------------------------------- | ---------- |
| 用户主动声明   | 用户在飞书描述规则，Agent 解析后调用 `save_skill_candidate` | 高         |
| Agent 后台复盘 | 周复盘时分析有效判断，生成候选 Skill 建议                   | 中         |
| 开发人员提炼   | 从 run 数据和 audit_log 人工分析，手动更新 Skill 文件       | 中         |

## 5. 飞书交互设计

### 5.1 消息类型

**入站（用户 → Agent）**：

| 场景       | 用户操作         | Agent 行为        |
| ---------- | ---------------- | ----------------- |
| 发起任务   | 自然语言描述任务 | 匹配 Skill 执行   |
| 自定义查询 | 提问式查询       | 调用 MCP 查询数据 |
| 确认执行   | 点击卡片按钮     | 调用 MCP 执行写入 |
| 补充信息   | 回复数据请求     | 注入变量继续流程  |
| 规则声明   | 描述业务规则     | 沉淀为候选 Skill  |

**出站（Agent → 用户）**：

| 场景       | 消息形式              | 触发时机           |
| ---------- | --------------------- | ------------------ |
| 日巡检摘要 | 结构化卡片            | Cron 每天 9:00     |
| 周复盘报告 | 长文本 + 飞书文档链接 | Cron 每周一 10:00  |
| 异常告警   | 紧急卡片              | 异常监控检测到异常 |
| 执行回执   | 简短文本              | 广告操作完成后     |
| 待确认事项 | 交互式卡片（带按钮）  | 方案生成后         |
| 数据请求   | 带字段说明的文本      | 降级需人工补充     |

### 5.2 飞书卡片模板

**巡检结果卡片**：

- 整体健康度标记（绿/黄/红）
- 昨日核心指标（花费、ACOS、转化、CTR）
- 异常项列表（含归因和建议）
- 操作按钮：查看详情 / 执行建议 / 忽略

**待确认执行卡片**：

- 操作类型和完整方案详情
- 预计影响范围
- 操作按钮：确认执行 / 修改方案 / 取消

**异常告警卡片**：

- 异常描述和数据指标
- 建议动作列表
- 操作按钮：立即处理 / 加入明日巡检

### 5.3 Thread 绑定

| 维度        | 设计                                                  |
| ----------- | ----------------------------------------------------- |
| Thread 粒度 | 每个 Listing 绑定一个长期 Thread                      |
| 上下文保持  | 同一 Listing 的所有对话、Run 结果在同一 Thread 内     |
| Run 回流    | 巡检/复盘/执行完成后摘要自动推送到对应 Thread         |
| Cron 投递   | Cron 触发的 Run 结果通过 ThreadBinding 推送到飞书会话 |
| 多 Listing  | 不同 Listing 的 Thread 相互隔离                       |

## 6. 部署结构

### 6.1 部署方案：独立仓库 + 安装脚本

业务代码放独立 Git 仓库，通过安装脚本自动注册到 openclaw 配置。

```
/opt/aws-sp-assistant/                ← 业务代码（独立 Git 仓库）
├── README.md
├── package.json
├── setup.sh                         ← 一键安装脚本
├── uninstall.sh                     ← 一键卸载脚本
├── config/
│   ├── system-prompt.md             # Agent System Prompt 模板
│   └── .env.example                 # 环境变量模板
├── mcp-servers/
│   ├── amazon-ads-mcp/              # Fork 开源 → 自建
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── tools/
│   │   │   └── auth.ts
│   │   └── tsconfig.json
│   └── storage-mcp/                 # 业务数据存储 MCP
│       ├── package.json
│       ├── src/
│       │   ├── index.ts
│       │   ├── tools/
│       │   ├── db/
│       │   │   ├── schema.ts
│       │   │   └── migrations.ts
│       │   └── types.ts
│       └── tsconfig.json
├── skills/                          # Skill 策略 Prompt 文件
│   ├── pre-launch-check.md
│   ├── keyword-library.md
│   ├── sp-campaign-plan.md
│   ├── sp-campaign-publish.md
│   ├── daily-inspection.md
│   ├── weekly-review.md
│   ├── notification-template.md
│   ├── task-collaboration.md
│   ├── custom-query.md
│   └── risk-rules.md
├── data/                            # 运行时数据（.gitignore）
│   ├── sqlite.db
│   └── logs/
└── docs/
    ├── deployment.md
    └── mcp-tool-reference.md
```

### 6.2 与 openclaw 的关系

```
openclaw (npm 全局安装，独立升级)
    │
    │  通过配置引用，不修改源码
    │
    ▼
~/.openclaw/                         ← openclaw 用户配置目录
├── openclaw.json                    # Agent 注册 + MCP Server 注册（指向业务代码路径）
└── ...

    │  路径指向
    ▼
/opt/aws-sp-assistant/               ← 业务代码（独立仓库）
├── mcp-servers/*/dist/index.js      # MCP Server 编译产物
├── skills/*.md                      # Skill Prompt 文件
└── data/sqlite.db                   # 业务数据
```

### 6.3 升级安全保证

| 维度            | 策略                                                                  |
| --------------- | --------------------------------------------------------------------- |
| openclaw 升级   | `npm update -g openclaw`，只替换安装目录，不影响配置和业务代码        |
| 业务代码更新    | `/opt/aws-sp-assistant/` 内 `git pull && npm run build`，配置路径不变 |
| MCP Server 兼容 | 使用标准 `@modelcontextprotocol/sdk`，协议向前兼容                    |
| Skill 文件      | 纯 Prompt 文本，与 openclaw 版本无关                                  |

**openclaw 升级后验证步骤**：

1. 检查 MCP Server 连接：`openclaw mcp list`
2. 检查 Cron 任务触发：`openclaw cron list`
3. 检查飞书消息收发：`openclaw channels status --probe`

### 6.4 部署流程

```bash
# 1. 安装 openclaw
npm install -g openclaw

# 2. 克隆项目
git clone <aws-sp-assistant-repo> /opt/aws-sp-assistant
cd /opt/aws-sp-assistant

# 3. 一键安装
./setup.sh
# setup.sh 自动完成:
#   - npm install && npm run build（编译 MCP Server）
#   - 注册 Agent、MCP Server、Cron 到 openclaw 配置
#   - 初始化 SQLite 数据库

# 4. 配置环境变量
cp .env.example .env
# 编辑 .env 填入 Amazon Ads API 凭证和飞书应用凭证

# 5. 验证
openclaw mcp list
openclaw cron list
openclaw channels status --probe

# 6. 启动
openclaw gateway run
```

## 7. MVP 交付物清单

| 类别       | 交付物                             | 对应用例            |
| ---------- | ---------------------------------- | ------------------- |
| MCP Server | amazon-ads-mcp（13+ 工具）         | UC-01~11            |
| MCP Server | storage-mcp（13+ 工具）            | UC-05/06/07/11/12   |
| MCP Server | seller-data-mcp（可选）            | UC-01/02            |
| Agent      | aws-sp-assistant Agent Profile     | 全局                |
| Skill      | 10 个策略文件                      | UC-01~09 + 风险规则 |
| Cron       | 3 个定时任务                       | UC-05/06/07         |
| 配置       | Agent + MCP + Cron 配置文件        | 全局                |
| 脚本       | setup.sh / uninstall.sh            | 部署                |
| 文档       | 部署说明 + MCP 工具参考 + 使用说明 | 7.4                 |

## 8. 风险与约束

- Amazon Ads MCP Server 基于 fork 开源项目，需验证 API 覆盖度和安全性
- Seller Data MCP 如无法直接接入 Amazon Selling Partner API，将降级为飞书手动输入
- 业务策略效果依赖 Prompt 质量，需要持续迭代调优
- MVP 阶段所有写入操作需飞书确认，不自动执行高风险操作
- Skill 自进化以候选沉淀 + 人工审核为主，不做完全自动升级
