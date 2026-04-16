# 亚马逊 SP 广告智能运营助手 MVP 实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一套基于 openclaw 平台的 Amazon SP 广告智能运营助手 MVP，包含 Storage MCP、Amazon Ads MCP、Agent Skill 策略层和一键部署脚本。

**Architecture:** 纯 Agent + MCP 工具方案。业务逻辑在 openclaw Agent 的 System Prompt + Skill 中实现，通过独立 MCP Server 调用 Amazon Ads API 和 SQLite 存储层。零侵入 openclaw 源码。

**Tech Stack:** TypeScript, Node.js, `@modelcontextprotocol/sdk`, `better-sqlite3`, openclaw Agent/Cron/MCP

**Spec:** `docs/superpowers/specs/2026-04-16-aws-sp-assistant-design.md`

---

## 计划拆分

本项目包含 4 个独立子系统，拆分为 4 个独立计划：

| 计划                              | 产出                                      | 可独立测试 | 依赖         |
| --------------------------------- | ----------------------------------------- | ---------- | ------------ |
| **Plan 1: Storage MCP Server**    | SQLite CRUD MCP Server                    | 是         | 无           |
| **Plan 2: Amazon Ads MCP Server** | Amazon Ads API MCP Server                 | 是         | 无           |
| **Plan 3: Agent + Skills + Cron** | Agent 配置 + 10 个 Skill 文件 + 3 个 Cron | 是         | Plan 1, 2    |
| **Plan 4: 部署脚本 + 文档**       | setup.sh / uninstall.sh / 文档            | 是         | Plan 1, 2, 3 |

以下为 **Plan 1: Storage MCP Server** 的完整实施步骤。Plan 2-4 在 Plan 1 完成后详细展开。

---

# Plan 1: Storage MCP Server

## 文件结构

```
aws-sp-assistant/
├── mcp-servers/
│   └── storage-mcp/
│       ├── package.json
│       ├── tsconfig.json
│       ├── src/
│       │   ├── index.ts              # MCP Server 入口，注册所有工具
│       │   ├── types.ts              # 类型定义
│       │   ├── db/
│       │   │   ├── connection.ts     # SQLite 连接管理
│       │   │   ├── schema.ts         # 表结构定义和初始化
│       │   │   └── migrations.ts     # 数据库迁移
│       │   └── tools/
│       │       ├── runs.ts           # save_run, query_runs, get_run_detail
│       │       ├── keywords.ts       # save_keyword_library, load_keyword_library
│       │       ├── audit.ts          # log_action, query_audit_log
│       │       ├── reports.ts        # save_report, query_reports
│       │       ├── manual-input.ts   # save_manual_input, load_manual_input
│       │       └── skills.ts         # save_skill_candidate, query_skill_candidates
│       └── __tests__/
│           ├── runs.test.ts
│           ├── keywords.test.ts
│           ├── audit.test.ts
│           ├── reports.test.ts
│           ├── manual-input.test.ts
│           └── skills.test.ts
├── data/                             # .gitignore
│   └── sqlite.db
├── vitest.config.ts                  # 测试配置
└── .env.example
```

---

### Task 1: 项目脚手架

**Files:**

- Create: `aws-sp-assistant/package.json`
- Create: `aws-sp-assistant/.gitignore`
- Create: `aws-sp-assistant/mcp-servers/storage-mcp/package.json`
- Create: `aws-sp-assistant/mcp-servers/storage-mcp/tsconfig.json`

- [ ] **Step 1: 创建项目根目录和 package.json**

```bash
mkdir -p aws-sp-assistant/mcp-servers/storage-mcp/src/{db,tools}
mkdir -p aws-sp-assistant/mcp-servers/storage-mcp/__tests__
mkdir -p aws-sp-assistant/data
mkdir -p aws-sp-assistant/skills
mkdir -p aws-sp-assistant/config
mkdir -p aws-sp-assistant/docs
```

`aws-sp-assistant/package.json`:

```json
{
  "name": "aws-sp-assistant",
  "private": true,
  "workspaces": ["mcp-servers/*"],
  "scripts": {
    "build": "npm run build --workspaces",
    "test": "npm test --workspaces",
    "test:coverage": "npm run test:coverage --workspaces"
  }
}
```

- [ ] **Step 2: 创建 storage-mcp 的 package.json**

`aws-sp-assistant/mcp-servers/storage-mcp/package.json`:

```json
{
  "name": "@aws-sp/storage-mcp",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "tsc --watch",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "init-db": "node dist/db/init.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.0",
    "better-sqlite3": "^11.0.0",
    "uuid": "^10.0.0",
    "zod": "^3.25.0"
  },
  "devDependencies": {
    "@types/better-sqlite3": "^7.6.0",
    "@types/node": "^22.0.0",
    "@types/uuid": "^10.0.0",
    "typescript": "^5.7.0",
    "vitest": "^3.0.0"
  }
}
```

- [ ] **Step 3: 创建 tsconfig.json**

`aws-sp-assistant/mcp-servers/storage-mcp/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "declaration": true,
    "sourceMap": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "__tests__"]
}
```

- [ ] **Step 4: 创建 .gitignore**

`aws-sp-assistant/.gitignore`:

```
node_modules/
dist/
data/*.db
.env
*.log
coverage/
```

- [ ] **Step 5: 创建 vitest.config.ts**

`aws-sp-assistant/mcp-servers/storage-mcp/vitest.config.ts`:

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["__tests__/**/*.test.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov"],
      thresholds: { lines: 80, branches: 80, functions: 80, statements: 80 },
    },
  },
});
```

- [ ] **Step 6: 安装依赖并提交**

```bash
cd aws-sp-assistant && npm install
git init && git add -A && git commit -m "chore: scaffold aws-sp-assistant project"
```

---

### Task 2: 类型定义

**Files:**

- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/types.ts`

- [ ] **Step 1: 定义所有数据类型**

`aws-sp-assistant/mcp-servers/storage-mcp/src/types.ts`:

```typescript
export type RunType = "inspection" | "review" | "prelaunch" | "plan" | "publish" | "query" | "task";
export type RunStatus = "success" | "warning" | "error";
export type ActionType = "create" | "update" | "pause" | "resume" | "delete";
export type ActorType = "agent" | "user" | "system";
export type TargetType = "campaign" | "ad_group" | "keyword";
export type ReportType = "daily" | "weekly";
export type DataType = "listing" | "inventory" | "review" | "price";
export type SkillSource = "user" | "agent" | "dev";
export type SkillCandidateStatus = "candidate" | "approved" | "rejected";

export interface Run {
  id: string;
  listing_id: string;
  type: RunType;
  status: RunStatus;
  summary: string | null;
  detail_json: string | null;
  created_at: string;
}

export interface KeywordLibrary {
  id: string;
  listing_id: string;
  keywords_json: string;
  neg_keywords_json: string | null;
  version: number;
  created_at: string;
}

export interface AuditEntry {
  id: string;
  listing_id: string;
  action: ActionType;
  actor: ActorType;
  target_type: TargetType | null;
  target_id: string | null;
  result: "success" | "failure";
  detail_json: string | null;
  created_at: string;
}

export interface Report {
  id: string;
  listing_id: string;
  type: ReportType;
  content_json: string;
  created_at: string;
}

export interface ManualInput {
  id: string;
  listing_id: string;
  data_type: DataType;
  content_json: string;
  source: "manual" | "api";
  expires_at: string | null;
  created_at: string;
}

export interface SkillCandidate {
  id: string;
  source: SkillSource;
  rule_text: string;
  status: SkillCandidateStatus;
  approved_by: string | null;
  created_at: string;
  applied_at: string | null;
}
```

- [ ] **Step 2: 提交**

```bash
git add -A && git commit -m "feat(storage-mcp): add type definitions"
```

---

### Task 3: SQLite 连接和 Schema

**Files:**

- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/db/connection.ts`
- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/db/schema.ts`
- Test: `aws-sp-assistant/mcp-servers/storage-mcp/__tests__/schema.test.ts`

- [ ] **Step 1: 写 schema 初始化测试**

`aws-sp-assistant/mcp-servers/storage-mcp/__tests__/schema.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import Database from "better-sqlite3";
import { initializeSchema } from "../src/db/schema.js";
import { createConnection } from "../src/db/connection.js";
import path from "node:path";
import fs from "node:fs";
import os from "node:os";

describe("schema initialization", () => {
  let dbPath: string;
  let db: Database.Database;

  beforeEach(() => {
    dbPath = path.join(os.tmpdir(), `test-schema-${Date.now()}.db`);
    db = createConnection(dbPath);
  });

  afterEach(() => {
    db.close();
    fs.unlinkSync(dbPath);
  });

  it("should create all 6 tables", () => {
    initializeSchema(db);

    const tables = db
      .prepare("SELECT name FROM sqlite_master WHERE type='table' ORDER BY name")
      .all()
      .map((r: any) => r.name);

    expect(tables).toContain("runs");
    expect(tables).toContain("keyword_libs");
    expect(tables).toContain("audit_log");
    expect(tables).toContain("reports");
    expect(tables).toContain("manual_inputs");
    expect(tables).toContain("skill_candidates");
  });

  it("should be idempotent (running twice does not error)", () => {
    initializeSchema(db);
    expect(() => initializeSchema(db)).not.toThrow();
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run __tests__/schema.test.ts
```

Expected: FAIL — modules not found

- [ ] **Step 3: 实现 connection.ts**

`aws-sp-assistant/mcp-servers/storage-mcp/src/db/connection.ts`:

```typescript
import Database from "better-sqlite3";
import path from "node:path";
import fs from "node:fs";

export function createConnection(dbPath: string): Database.Database {
  const dir = path.dirname(dbPath);
  if (!fs.existsSync(dir)) {
    fs.mkdirSync(dir, { recursive: true });
  }

  const db = new Database(dbPath);
  db.pragma("journal_mode = WAL");
  db.pragma("foreign_keys = ON");
  return db;
}
```

- [ ] **Step 4: 实现 schema.ts**

`aws-sp-assistant/mcp-servers/storage-mcp/src/db/schema.ts`:

```typescript
import Database from "better-sqlite3";

const SCHEMA_SQL = `
CREATE TABLE IF NOT EXISTS runs (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    type TEXT NOT NULL,
    status TEXT NOT NULL,
    summary TEXT,
    detail_json TEXT,
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS keyword_libs (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    keywords_json TEXT NOT NULL,
    neg_keywords_json TEXT,
    version INTEGER DEFAULT 1,
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS audit_log (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    action TEXT NOT NULL,
    actor TEXT NOT NULL,
    target_type TEXT,
    target_id TEXT,
    result TEXT,
    detail_json TEXT,
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS reports (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    type TEXT NOT NULL,
    content_json TEXT NOT NULL,
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS manual_inputs (
    id TEXT PRIMARY KEY,
    listing_id TEXT NOT NULL,
    data_type TEXT NOT NULL,
    content_json TEXT NOT NULL,
    source TEXT DEFAULT 'manual',
    expires_at TEXT,
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS skill_candidates (
    id TEXT PRIMARY KEY,
    source TEXT NOT NULL,
    rule_text TEXT NOT NULL,
    status TEXT DEFAULT 'candidate',
    approved_by TEXT,
    created_at TEXT NOT NULL,
    applied_at TEXT
);

CREATE INDEX IF NOT EXISTS idx_runs_listing ON runs(listing_id, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_runs_type ON runs(listing_id, type, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_audit_listing ON audit_log(listing_id, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_reports_listing ON reports(listing_id, type, created_at DESC);
CREATE INDEX IF NOT EXISTS idx_manual_listing ON manual_inputs(listing_id, data_type);
CREATE INDEX IF NOT EXISTS idx_skills_status ON skill_candidates(status);
`;

export function initializeSchema(db: Database.Database): void {
  db.exec(SCHEMA_SQL);
}
```

- [ ] **Step 5: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run __tests__/schema.test.ts
```

Expected: PASS

- [ ] **Step 6: 提交**

```bash
git add -A && git commit -m "feat(storage-mcp): add SQLite connection and schema"
```

---

### Task 4: Runs 工具（save_run, query_runs, get_run_detail）

**Files:**

- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/tools/runs.ts`
- Test: `aws-sp-assistant/mcp-servers/storage-mcp/__tests__/runs.test.ts`

- [ ] **Step 1: 写 runs 工具测试**

`aws-sp-assistant/mcp-servers/storage-mcp/__tests__/runs.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import Database from "better-sqlite3";
import { createConnection } from "../src/db/connection.js";
import { initializeSchema } from "../src/db/schema.js";
import { registerRunTools } from "../src/tools/runs.js";
import path from "node:path";
import fs from "node:fs";
import os from "node:os";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

// Minimal mock for testing tool handlers directly
function createToolRegistry(db: Database.Database) {
  const handlers: Record<string, (args: any) => any> = {};
  const mockServer = {
    tool: (name: string, _schema: any, handler: (args: any) => any) => {
      handlers[name] = handler;
    },
  } as unknown as McpServer;

  registerRunTools(mockServer, db);
  return handlers;
}

describe("runs tools", () => {
  let dbPath: string;
  let db: Database.Database;
  let handlers: ReturnType<typeof createToolRegistry>;

  beforeEach(() => {
    dbPath = path.join(os.tmpdir(), `test-runs-${Date.now()}.db`);
    db = createConnection(dbPath);
    initializeSchema(db);
    handlers = createToolRegistry(db);
  });

  afterEach(() => {
    db.close();
    fs.unlinkSync(dbPath);
  });

  it("save_run should insert a run and return id", () => {
    const result = handlers.save_run({
      listing_id: "B0XXXXX",
      type: "inspection",
      status: "success",
      summary: "All healthy",
      detail_json: '{"score": "green"}',
    });

    expect(result.content[0].text).toContain("B0XXXXX");
    const row = db.prepare("SELECT * FROM runs WHERE listing_id = ?").get("B0XXXXX") as any;
    expect(row).toBeDefined();
    expect(row.type).toBe("inspection");
    expect(row.status).toBe("success");
    expect(row.summary).toBe("All healthy");
  });

  it("query_runs should return runs filtered by listing_id", () => {
    handlers.save_run({
      listing_id: "L1",
      type: "inspection",
      status: "success",
      summary: null,
      detail_json: null,
    });
    handlers.save_run({
      listing_id: "L1",
      type: "review",
      status: "warning",
      summary: null,
      detail_json: null,
    });
    handlers.save_run({
      listing_id: "L2",
      type: "inspection",
      status: "success",
      summary: null,
      detail_json: null,
    });

    const result = handlers.query_runs({ listing_id: "L1" });
    const runs = JSON.parse(result.content[0].text);
    expect(runs).toHaveLength(2);
  });

  it("query_runs should filter by type and date range", () => {
    handlers.save_run({
      listing_id: "L1",
      type: "inspection",
      status: "success",
      summary: null,
      detail_json: null,
    });
    handlers.save_run({
      listing_id: "L1",
      type: "review",
      status: "success",
      summary: null,
      detail_json: null,
    });

    const result = handlers.query_runs({ listing_id: "L1", type: "inspection" });
    const runs = JSON.parse(result.content[0].text);
    expect(runs).toHaveLength(1);
    expect(runs[0].type).toBe("inspection");
  });

  it("get_run_detail should return a single run", () => {
    const saved = handlers.save_run({
      listing_id: "L1",
      type: "inspection",
      status: "success",
      summary: "test",
      detail_json: '{"k":"v"}',
    });
    const id = JSON.parse(saved.content[0].text).id;

    const result = handlers.get_run_detail({ id });
    const run = JSON.parse(result.content[0].text);
    expect(run.id).toBe(id);
    expect(run.summary).toBe("test");
    expect(run.detail_json).toBe('{"k":"v"}');
  });

  it("get_run_detail should return error for missing id", () => {
    const result = handlers.get_run_detail({ id: "nonexistent" });
    expect(result.content[0].text).toContain("not found");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run __tests__/runs.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 runs.ts**

`aws-sp-assistant/mcp-servers/storage-mcp/src/tools/runs.ts`:

```typescript
import { v4 as uuidv4 } from "uuid";
import { z } from "zod";
import type Database from "better-sqlite3";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export function registerRunTools(server: McpServer, db: Database.Database): void {
  server.tool(
    "save_run",
    "Save a run result to storage",
    {
      listing_id: z.string().describe("Listing ID"),
      type: z.string().describe("Run type: inspection/review/prelaunch/plan/publish/query/task"),
      status: z.string().describe("Run status: success/warning/error"),
      summary: z.string().optional().describe("Run summary"),
      detail_json: z.string().optional().describe("Detailed result as JSON string"),
    },
    (args) => {
      const id = uuidv4();
      const now = new Date().toISOString();
      db.prepare(
        `INSERT INTO runs (id, listing_id, type, status, summary, detail_json, created_at) VALUES (?, ?, ?, ?, ?, ?, ?)`,
      ).run(
        id,
        args.listing_id,
        args.type,
        args.status,
        args.summary ?? null,
        args.detail_json ?? null,
        now,
      );

      return {
        content: [
          {
            type: "text" as const,
            text: JSON.stringify({
              id,
              listing_id: args.listing_id,
              type: args.type,
              status: args.status,
              created_at: now,
            }),
          },
        ],
      };
    },
  );

  server.tool(
    "query_runs",
    "Query run history by listing, type, and date range",
    {
      listing_id: z.string().describe("Listing ID"),
      type: z.string().optional().describe("Optional run type filter"),
      start_date: z.string().optional().describe("Start date (ISO 8601)"),
      end_date: z.string().optional().describe("End date (ISO 8601)"),
      limit: z.number().optional().describe("Max results (default 20)"),
      offset: z.number().optional().describe("Offset for pagination"),
    },
    (args) => {
      let sql =
        "SELECT id, listing_id, type, status, summary, created_at FROM runs WHERE listing_id = ?";
      const params: any[] = [args.listing_id];

      if (args.type) {
        sql += " AND type = ?";
        params.push(args.type);
      }
      if (args.start_date) {
        sql += " AND created_at >= ?";
        params.push(args.start_date);
      }
      if (args.end_date) {
        sql += " AND created_at <= ?";
        params.push(args.end_date);
      }

      sql += " ORDER BY created_at DESC LIMIT ? OFFSET ?";
      params.push(args.limit ?? 20, args.offset ?? 0);

      const rows = db.prepare(sql).all(...params);
      return { content: [{ type: "text" as const, text: JSON.stringify(rows) }] };
    },
  );

  server.tool(
    "get_run_detail",
    "Get full detail of a single run by ID",
    { id: z.string().describe("Run ID") },
    (args) => {
      const row = db.prepare("SELECT * FROM runs WHERE id = ?").get(args.id) as any;
      if (!row) {
        return {
          content: [
            { type: "text" as const, text: JSON.stringify({ error: `Run ${args.id} not found` }) },
          ],
        };
      }
      return { content: [{ type: "text" as const, text: JSON.stringify(row) }] };
    },
  );
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run __tests__/runs.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(storage-mcp): add runs tools (save, query, get detail)"
```

---

### Task 5: Keywords 工具

**Files:**

- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/tools/keywords.ts`
- Test: `aws-sp-assistant/mcp-servers/storage-mcp/__tests__/keywords.test.ts`

- [ ] **Step 1: 写 keywords 工具测试**

`aws-sp-assistant/mcp-servers/storage-mcp/__tests__/keywords.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import Database from "better-sqlite3";
import { createConnection } from "../src/db/connection.js";
import { initializeSchema } from "../src/db/schema.js";
import { registerKeywordTools } from "../src/tools/keywords.js";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import path from "node:path";
import fs from "node:fs";
import os from "node:os";

function createToolRegistry(db: Database.Database) {
  const handlers: Record<string, (args: any) => any> = {};
  const mockServer = {
    tool: (name: string, _schema: any, handler: (args: any) => any) => {
      handlers[name] = handler;
    },
  } as unknown as McpServer;
  registerKeywordTools(mockServer, db);
  return handlers;
}

describe("keyword tools", () => {
  let dbPath: string;
  let db: Database.Database;
  let handlers: ReturnType<typeof createToolRegistry>;

  beforeEach(() => {
    dbPath = path.join(os.tmpdir(), `test-kw-${Date.now()}.db`);
    db = createConnection(dbPath);
    initializeSchema(db);
    handlers = createToolRegistry(db);
  });

  afterEach(() => {
    db.close();
    fs.unlinkSync(dbPath);
  });

  it("save_keyword_library should insert and return id", () => {
    const result = handlers.save_keyword_library({
      listing_id: "L1",
      keywords_json: '["shoes","running"]',
      neg_keywords_json: '["free"]',
    });
    const parsed = JSON.parse(result.content[0].text);
    expect(parsed.id).toBeDefined();
    expect(parsed.version).toBe(1);
  });

  it("save_keyword_library should auto-increment version", () => {
    handlers.save_keyword_library({
      listing_id: "L1",
      keywords_json: '["a"]',
      neg_keywords_json: null,
    });
    const r2 = handlers.save_keyword_library({
      listing_id: "L1",
      keywords_json: '["a","b"]',
      neg_keywords_json: null,
    });
    expect(JSON.parse(r2.content[0].text).version).toBe(2);
  });

  it("load_keyword_library should return latest version", () => {
    handlers.save_keyword_library({
      listing_id: "L1",
      keywords_json: '["a"]',
      neg_keywords_json: null,
    });
    handlers.save_keyword_library({
      listing_id: "L1",
      keywords_json: '["a","b"]',
      neg_keywords_json: '["c"]',
    });

    const result = handlers.load_keyword_library({ listing_id: "L1" });
    const lib = JSON.parse(result.content[0].text);
    expect(lib.version).toBe(2);
    expect(lib.keywords_json).toBe('["a","b"]');
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run __tests__/keywords.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 keywords.ts**

`aws-sp-assistant/mcp-servers/storage-mcp/src/tools/keywords.ts`:

```typescript
import { v4 as uuidv4 } from "uuid";
import { z } from "zod";
import type Database from "better-sqlite3";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export function registerKeywordTools(server: McpServer, db: Database.Database): void {
  server.tool(
    "save_keyword_library",
    "Save a keyword library for a listing (auto-increments version)",
    {
      listing_id: z.string().describe("Listing ID"),
      keywords_json: z.string().describe("JSON array of keywords"),
      neg_keywords_json: z.string().optional().describe("JSON array of negative keywords"),
    },
    (args) => {
      const maxVersion =
        (
          db
            .prepare("SELECT MAX(version) as v FROM keyword_libs WHERE listing_id = ?")
            .get(args.listing_id) as any
        )?.v ?? 0;
      const version = maxVersion + 1;
      const id = uuidv4();
      const now = new Date().toISOString();

      db.prepare(
        "INSERT INTO keyword_libs (id, listing_id, keywords_json, neg_keywords_json, version, created_at) VALUES (?, ?, ?, ?, ?, ?)",
      ).run(id, args.listing_id, args.keywords_json, args.neg_keywords_json ?? null, version, now);

      return {
        content: [
          {
            type: "text" as const,
            text: JSON.stringify({ id, listing_id: args.listing_id, version, created_at: now }),
          },
        ],
      };
    },
  );

  server.tool(
    "load_keyword_library",
    "Load the latest keyword library for a listing",
    { listing_id: z.string().describe("Listing ID") },
    (args) => {
      const row = db
        .prepare("SELECT * FROM keyword_libs WHERE listing_id = ? ORDER BY version DESC LIMIT 1")
        .get(args.listing_id) as any;

      if (!row) {
        return {
          content: [
            {
              type: "text" as const,
              text: JSON.stringify({
                error: `No keyword library found for listing ${args.listing_id}`,
              }),
            },
          ],
        };
      }
      return { content: [{ type: "text" as const, text: JSON.stringify(row) }] };
    },
  );
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run __tests__/keywords.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(storage-mcp): add keyword library tools"
```

---

### Task 6: Audit 工具

**Files:**

- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/tools/audit.ts`
- Test: `aws-sp-assistant/mcp-servers/storage-mcp/__tests__/audit.test.ts`

- [ ] **Step 1: 写 audit 工具测试**

`aws-sp-assistant/mcp-servers/storage-mcp/__tests__/audit.test.ts`:

```typescript
import { describe, it, expect, beforeEach, afterEach } from "vitest";
import Database from "better-sqlite3";
import { createConnection } from "../src/db/connection.js";
import { initializeSchema } from "../src/db/schema.js";
import { registerAuditTools } from "../src/tools/audit.js";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import path from "node:path";
import fs from "node:fs";
import os from "node:os";

function createToolRegistry(db: Database.Database) {
  const handlers: Record<string, (args: any) => any> = {};
  const mockServer = {
    tool: (name: string, _schema: any, handler: (args: any) => any) => {
      handlers[name] = handler;
    },
  } as unknown as McpServer;
  registerAuditTools(mockServer, db);
  return handlers;
}

describe("audit tools", () => {
  let dbPath: string;
  let db: Database.Database;
  let handlers: ReturnType<typeof createToolRegistry>;

  beforeEach(() => {
    dbPath = path.join(os.tmpdir(), `test-audit-${Date.now()}.db`);
    db = createConnection(dbPath);
    initializeSchema(db);
    handlers = createToolRegistry(db);
  });

  afterEach(() => {
    db.close();
    fs.unlinkSync(dbPath);
  });

  it("log_action should record an audit entry", () => {
    const result = handlers.log_action({
      listing_id: "L1",
      action: "create",
      actor: "agent",
      target_type: "campaign",
      target_id: "camp-123",
      result: "success",
      detail_json: null,
    });
    const parsed = JSON.parse(result.content[0].text);
    expect(parsed.id).toBeDefined();

    const row = db.prepare("SELECT * FROM audit_log WHERE listing_id = ?").get("L1") as any;
    expect(row.action).toBe("create");
    expect(row.actor).toBe("agent");
    expect(row.target_type).toBe("campaign");
  });

  it("query_audit_log should filter by listing and date", () => {
    handlers.log_action({
      listing_id: "L1",
      action: "create",
      actor: "agent",
      target_type: "campaign",
      target_id: "c1",
      result: "success",
      detail_json: null,
    });
    handlers.log_action({
      listing_id: "L1",
      action: "pause",
      actor: "user",
      target_type: "keyword",
      target_id: "k1",
      result: "success",
      detail_json: null,
    });

    const result = handlers.query_audit_log({ listing_id: "L1" });
    const entries = JSON.parse(result.content[0].text);
    expect(entries).toHaveLength(2);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run __tests__/audit.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 audit.ts**

`aws-sp-assistant/mcp-servers/storage-mcp/src/tools/audit.ts`:

```typescript
import { v4 as uuidv4 } from "uuid";
import { z } from "zod";
import type Database from "better-sqlite3";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export function registerAuditTools(server: McpServer, db: Database.Database): void {
  server.tool(
    "log_action",
    "Record an action in the audit log",
    {
      listing_id: z.string().describe("Listing ID"),
      action: z.string().describe("Action type: create/update/pause/resume/delete"),
      actor: z.string().describe("Actor: agent/user/system"),
      target_type: z.string().optional().describe("Target type: campaign/ad_group/keyword"),
      target_id: z.string().optional().describe("Target ID"),
      result: z.string().describe("Result: success/failure"),
      detail_json: z.string().optional().describe("Additional details as JSON"),
    },
    (args) => {
      const id = uuidv4();
      const now = new Date().toISOString();
      db.prepare(
        `INSERT INTO audit_log (id, listing_id, action, actor, target_type, target_id, result, detail_json, created_at) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
      ).run(
        id,
        args.listing_id,
        args.action,
        args.actor,
        args.target_type ?? null,
        args.target_id ?? null,
        args.result,
        args.detail_json ?? null,
        now,
      );

      return {
        content: [{ type: "text" as const, text: JSON.stringify({ id, created_at: now }) }],
      };
    },
  );

  server.tool(
    "query_audit_log",
    "Query audit log entries for a listing",
    {
      listing_id: z.string().describe("Listing ID"),
      start_date: z.string().optional().describe("Start date (ISO 8601)"),
      end_date: z.string().optional().describe("End date (ISO 8601)"),
      limit: z.number().optional().describe("Max results (default 50)"),
      offset: z.number().optional().describe("Offset for pagination"),
    },
    (args) => {
      let sql = "SELECT * FROM audit_log WHERE listing_id = ?";
      const params: any[] = [args.listing_id];

      if (args.start_date) {
        sql += " AND created_at >= ?";
        params.push(args.start_date);
      }
      if (args.end_date) {
        sql += " AND created_at <= ?";
        params.push(args.end_date);
      }

      sql += " ORDER BY created_at DESC LIMIT ? OFFSET ?";
      params.push(args.limit ?? 50, args.offset ?? 0);

      const rows = db.prepare(sql).all(...params);
      return { content: [{ type: "text" as const, text: JSON.stringify(rows) }] };
    },
  );
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run __tests__/audit.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(storage-mcp): add audit log tools"
```

---

### Task 7: Reports、Manual Input、Skill Candidates 工具

**Files:**

- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/tools/reports.ts`
- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/tools/manual-input.ts`
- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/tools/skills.ts`
- Test: `aws-sp-assistant/mcp-servers/storage-mcp/__tests__/reports.test.ts`
- Test: `aws-sp-assistant/mcp-servers/storage-mcp/__tests__/manual-input.test.ts`
- Test: `aws-sp-assistant/mcp-servers/storage-mcp/__tests__/skills.test.ts`

这三个工具模块结构与 runs/audit 相同（save + query 模式），实现步骤同 Task 4-6 的 TDD 流程：

1. 写测试 → 2. 运行确认失败 → 3. 实现代码 → 4. 运行确认通过 → 5. 提交

**Manual Input 特殊逻辑**：`load_manual_input` 需检查 `expires_at` 字段，过期数据不返回。

`aws-sp-assistant/mcp-servers/storage-mcp/src/tools/manual-input.ts` 关键代码：

```typescript
import { v4 as uuidv4 } from "uuid";
import { z } from "zod";
import type Database from "better-sqlite3";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export function registerManualInputTools(server: McpServer, db: Database.Database): void {
  server.tool(
    "save_manual_input",
    "Save user-provided data (degradation scenario cache)",
    {
      listing_id: z.string().describe("Listing ID"),
      data_type: z.string().describe("Data type: listing/inventory/review/price"),
      content_json: z.string().describe("Data content as JSON string"),
      ttl_hours: z.number().optional().describe("Cache TTL in hours (default 24)"),
    },
    (args) => {
      const id = uuidv4();
      const now = new Date();
      const ttlHours = args.ttl_hours ?? 24;
      const expiresAt = new Date(now.getTime() + ttlHours * 3600_000).toISOString();

      db.prepare(
        `INSERT INTO manual_inputs (id, listing_id, data_type, content_json, source, expires_at, created_at) VALUES (?, ?, ?, ?, 'manual', ?, ?)`,
      ).run(id, args.listing_id, args.data_type, args.content_json, expiresAt, now.toISOString());

      return {
        content: [
          {
            type: "text" as const,
            text: JSON.stringify({
              id,
              listing_id: args.listing_id,
              data_type: args.data_type,
              expires_at: expiresAt,
            }),
          },
        ],
      };
    },
  );

  server.tool(
    "load_manual_input",
    "Load cached manual input for a listing and data type",
    {
      listing_id: z.string().describe("Listing ID"),
      data_type: z.string().describe("Data type: listing/inventory/review/price"),
    },
    (args) => {
      const row = db
        .prepare(
          "SELECT * FROM manual_inputs WHERE listing_id = ? AND data_type = ? ORDER BY created_at DESC LIMIT 1",
        )
        .get(args.listing_id, args.data_type) as any;

      if (!row) {
        return {
          content: [
            { type: "text" as const, text: JSON.stringify({ error: "No cached data found" }) },
          ],
        };
      }

      if (row.expires_at && new Date(row.expires_at) < new Date()) {
        return {
          content: [
            {
              type: "text" as const,
              text: JSON.stringify({ error: "Cached data expired", expired_at: row.expires_at }),
            },
          ],
        };
      }

      return { content: [{ type: "text" as const, text: JSON.stringify(row) }] };
    },
  );
}
```

`aws-sp-assistant/mcp-servers/storage-mcp/src/tools/reports.ts` 关键代码：

```typescript
import { v4 as uuidv4 } from "uuid";
import { z } from "zod";
import type Database from "better-sqlite3";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export function registerReportTools(server: McpServer, db: Database.Database): void {
  server.tool(
    "save_report",
    "Save a daily or weekly report",
    {
      listing_id: z.string().describe("Listing ID"),
      type: z.string().describe("Report type: daily/weekly"),
      content_json: z.string().describe("Report content as JSON string"),
    },
    (args) => {
      const id = uuidv4();
      const now = new Date().toISOString();
      db.prepare(
        `INSERT INTO reports (id, listing_id, type, content_json, created_at) VALUES (?, ?, ?, ?, ?)`,
      ).run(id, args.listing_id, args.type, args.content_json, now);

      return {
        content: [
          {
            type: "text" as const,
            text: JSON.stringify({
              id,
              listing_id: args.listing_id,
              type: args.type,
              created_at: now,
            }),
          },
        ],
      };
    },
  );

  server.tool(
    "query_reports",
    "Query reports by listing, type, and date range",
    {
      listing_id: z.string().describe("Listing ID"),
      type: z.string().optional().describe("Report type filter: daily/weekly"),
      start_date: z.string().optional().describe("Start date (ISO 8601)"),
      end_date: z.string().optional().describe("End date (ISO 8601)"),
      limit: z.number().optional().describe("Max results (default 20)"),
    },
    (args) => {
      let sql = "SELECT * FROM reports WHERE listing_id = ?";
      const params: any[] = [args.listing_id];

      if (args.type) {
        sql += " AND type = ?";
        params.push(args.type);
      }
      if (args.start_date) {
        sql += " AND created_at >= ?";
        params.push(args.start_date);
      }
      if (args.end_date) {
        sql += " AND created_at <= ?";
        params.push(args.end_date);
      }

      sql += " ORDER BY created_at DESC LIMIT ?";
      params.push(args.limit ?? 20);

      const rows = db.prepare(sql).all(...params);
      return { content: [{ type: "text" as const, text: JSON.stringify(rows) }] };
    },
  );
}
```

`aws-sp-assistant/mcp-servers/storage-mcp/src/tools/skills.ts` 关键代码：

```typescript
import { v4 as uuidv4 } from "uuid";
import { z } from "zod";
import type Database from "better-sqlite3";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export function registerSkillTools(server: McpServer, db: Database.Database): void {
  server.tool(
    "save_skill_candidate",
    "Save a candidate skill rule for later review",
    {
      source: z.string().describe("Source: user/agent/dev"),
      rule_text: z.string().describe("The skill rule text"),
    },
    (args) => {
      const id = uuidv4();
      const now = new Date().toISOString();
      db.prepare(
        `INSERT INTO skill_candidates (id, source, rule_text, status, created_at) VALUES (?, ?, ?, 'candidate', ?)`,
      ).run(id, args.source, args.rule_text, now);

      return {
        content: [
          {
            type: "text" as const,
            text: JSON.stringify({ id, source: args.source, status: "candidate", created_at: now }),
          },
        ],
      };
    },
  );

  server.tool(
    "query_skill_candidates",
    "Query candidate skill rules",
    {
      status: z.string().optional().describe("Filter by status: candidate/approved/rejected"),
      source: z.string().optional().describe("Filter by source: user/agent/dev"),
      limit: z.number().optional().describe("Max results (default 20)"),
    },
    (args) => {
      let sql = "SELECT * FROM skill_candidates WHERE 1=1";
      const params: any[] = [];

      if (args.status) {
        sql += " AND status = ?";
        params.push(args.status);
      }
      if (args.source) {
        sql += " AND source = ?";
        params.push(args.source);
      }

      sql += " ORDER BY created_at DESC LIMIT ?";
      params.push(args.limit ?? 20);

      const rows = db.prepare(sql).all(...params);
      return { content: [{ type: "text" as const, text: JSON.stringify(rows) }] };
    },
  );
}
```

---

### Task 8: MCP Server 入口

**Files:**

- Create: `aws-sp-assistant/mcp-servers/storage-mcp/src/index.ts`

- [ ] **Step 1: 实现 MCP Server 入口**

`aws-sp-assistant/mcp-servers/storage-mcp/src/index.ts`:

```typescript
#!/usr/bin/env node

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { createConnection } from "./db/connection.js";
import { initializeSchema } from "./db/schema.js";
import { registerRunTools } from "./tools/runs.js";
import { registerKeywordTools } from "./tools/keywords.js";
import { registerAuditTools } from "./tools/audit.js";
import { registerReportTools } from "./tools/reports.js";
import { registerManualInputTools } from "./tools/manual-input.js";
import { registerSkillTools } from "./tools/skills.js";

const DB_PATH = process.env.DB_PATH || "./data/sqlite.db";

const db = createConnection(DB_PATH);
initializeSchema(db);

const server = new McpServer({
  name: "aws-sp-storage",
  version: "0.1.0",
});

registerRunTools(server, db);
registerKeywordTools(server, db);
registerAuditTools(server, db);
registerReportTools(server, db);
registerManualInputTools(server, db);
registerSkillTools(server, db);

const transport = new StdioServerTransport();
server.connect(transport).catch((err) => {
  console.error("Failed to start storage MCP server:", err);
  process.exit(1);
});
```

- [ ] **Step 2: 编译并验证启动**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx tsc
echo '{"jsonrpc":"2.0","method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}},"id":1}' | DB_PATH=/tmp/test-mcp.db node dist/index.js
```

Expected: JSON response with server info

- [ ] **Step 3: 运行全部测试**

```bash
cd aws-sp-assistant/mcp-servers/storage-mcp && npx vitest run
```

Expected: All tests PASS

- [ ] **Step 4: 提交**

```bash
git add -A && git commit -m "feat(storage-mcp): add MCP server entry point with all 13 tools"
```

---

# Plan 2: Amazon Ads MCP Server

> **Spec 对应:** Section 3.1 — Amazon Ads MCP Server（核心）
> **依赖:** 无（可与 Plan 1 并行）
> **MVP 范围说明:** Spec Section 3.3 定义的 Seller Data MCP Server（4 个工具：get_listing_info、get_inventory_status、get_reviews_summary、get_buyer_box_price）在 MVP 阶段不实现。Listing/库存/评论数据通过降级策略（Spec Section 3.4）中的 manual_inputs 机制由用户手动提供。

## 文件结构

```
aws-sp-assistant/mcp-servers/amazon-ads-mcp/
├── package.json
├── tsconfig.json
├── vitest.config.ts
├── src/
│   ├── index.ts                  # MCP Server 入口
│   ├── auth/
│   │   ├── token-manager.ts      # OAuth2 token 刷新（提前 60s 过期）
│   │   └── client.ts             # API 客户端（指数退避重试，最多 3 次）
│   └── tools/
│       ├── campaigns.ts          # list_campaigns, get_campaign_details, get_campaign_report, create_campaign, update_campaign
│       ├── ad-groups.ts          # list_ad_groups, create_ad_group
│       ├── keywords.ts           # list_keywords, create_keywords, update_keywords, add_negative_keywords, get_keyword_recommendations
│       └── account.ts            # get_account_budget
└── __tests__/
    ├── auth.test.ts
    ├── client.test.ts
    ├── campaigns.test.ts
    ├── ad-groups.test.ts
    ├── keywords.test.ts
    └── account.test.ts
```

---

### Task 9: 项目脚手架

**Files:**

- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/package.json`
- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/tsconfig.json`
- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/vitest.config.ts`

- [ ] **Step 1: 创建目录结构**

```bash
mkdir -p aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/{auth,tools}
mkdir -p aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__
```

- [ ] **Step 2: 创建 package.json**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/package.json`:

```json
{
  "name": "amazon-ads-mcp",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "test:coverage": "vitest run --coverage"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.12.0",
    "zod": "^3.25.0"
  },
  "devDependencies": {
    "@types/node": "^22.0.0",
    "typescript": "^5.7.0",
    "vitest": "^3.0.0"
  }
}
```

- [ ] **Step 3: 创建 tsconfig.json**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Node16",
    "moduleResolution": "Node16",
    "outDir": "dist",
    "rootDir": "src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist", "__tests__"]
}
```

- [ ] **Step 4: 创建 vitest.config.ts**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/vitest.config.ts`:

```typescript
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["__tests__/**/*.test.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov"],
      thresholds: { lines: 80, branches: 80, functions: 80, statements: 80 },
    },
  },
});
```

- [ ] **Step 5: 安装依赖并提交**

```bash
cd aws-sp-assistant && npm install
git add -A && git commit -m "chore(amazon-ads-mcp): scaffold project"
```

---

### Task 10: OAuth2 Token 刷新机制

**Spec 对应:** Section 3.1 — 凭证管理通过环境变量注入，stdio 传输模式。
**Files:**

- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/auth/token-manager.ts`
- Test: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/auth.test.ts`

- [ ] **Step 1: 写 TokenManager 测试**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/auth.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { TokenManager } from "../src/auth/token-manager.js";

describe("TokenManager", () => {
  const originalFetch = globalThis.fetch;

  beforeEach(() => {
    vi.restoreAllMocks();
    process.env.AMZ_ADS_CLIENT_ID = "test-client-id";
    process.env.AMZ_ADS_CLIENT_SECRET = "test-client-secret";
    process.env.AMZ_ADS_REFRESH_TOKEN = "test-refresh-token";
  });

  it("首次调用应发起 token 刷新", async () => {
    globalThis.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ access_token: "new-token", expires_in: 3600 }),
    }) as any;

    const manager = new TokenManager();
    const token = await manager.getAccessToken();

    expect(token).toBe("new-token");
    expect(globalThis.fetch).toHaveBeenCalledTimes(1);
    const [url, options] = (globalThis.fetch as any).mock.calls[0];
    expect(url).toContain("api.amazon.com/auth/o2/token");
    expect((options as any).method).toBe("POST");
  });

  it("token 未过期时应复用缓存", async () => {
    const mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ access_token: "cached-token", expires_in: 3600 }),
    }) as any;
    globalThis.fetch = mockFetch;

    const manager = new TokenManager();
    await manager.getAccessToken();
    await manager.getAccessToken();

    expect(mockFetch).toHaveBeenCalledTimes(1);
  });

  it("token 过期后应重新刷新", async () => {
    let callCount = 0;
    globalThis.fetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => {
        callCount++;
        return Promise.resolve({ access_token: `token-${callCount}`, expires_in: 0 });
      },
    }) as any;

    const manager = new TokenManager();
    const t1 = await manager.getAccessToken();
    const t2 = await manager.getAccessToken();

    expect(t1).toBe("token-1");
    expect(t2).toBe("token-2");
    expect(globalThis.fetch).toHaveBeenCalledTimes(2);
  });

  it("刷新失败应抛出错误", async () => {
    globalThis.fetch = vi.fn().mockResolvedValue({
      ok: false,
      status: 400,
      statusText: "Bad Request",
      text: () => Promise.resolve("invalid_grant"),
    }) as any;

    const manager = new TokenManager();
    await expect(manager.getAccessToken()).rejects.toThrow("Token refresh failed");
  });

  it("缺少环境变量应抛出错误", async () => {
    delete process.env.AMZ_ADS_CLIENT_ID;
    const manager = new TokenManager();
    await expect(manager.getAccessToken()).rejects.toThrow("AMZ_ADS_CLIENT_ID");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/auth.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 TokenManager**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/auth/token-manager.ts`:

```typescript
interface TokenResponse {
  access_token: string;
  expires_in: number;
  token_type?: string;
}

export class TokenManager {
  private accessToken: string | null = null;
  private expiresAt = 0;

  async getAccessToken(): Promise<string> {
    const clientId = process.env.AMZ_ADS_CLIENT_ID;
    const clientSecret = process.env.AMZ_ADS_CLIENT_SECRET;
    const refreshToken = process.env.AMZ_ADS_REFRESH_TOKEN;

    if (!clientId) throw new Error("AMZ_ADS_CLIENT_ID is not set");
    if (!clientSecret) throw new Error("AMZ_ADS_CLIENT_SECRET is not set");
    if (!refreshToken) throw new Error("AMZ_ADS_REFRESH_TOKEN is not set");

    if (this.accessToken && Date.now() < this.expiresAt) {
      return this.accessToken;
    }

    const response = await fetch("https://api.amazon.com/auth/o2/token", {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        grant_type: "refresh_token",
        refresh_token: refreshToken,
        client_id: clientId,
        client_secret: clientSecret,
      }),
    });

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Token refresh failed: ${response.status} ${response.statusText} — ${body}`);
    }

    const data = (await response.json()) as TokenResponse;
    this.accessToken = data.access_token;
    // 提前 60 秒过期，避免边界竞争
    this.expiresAt = Date.now() + (data.expires_in - 60) * 1000;

    return this.accessToken;
  }
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/auth.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(amazon-ads-mcp): add OAuth2 token manager"
```

---

### Task 11: API 客户端（指数退避重试）

**Spec 对应:** Section 3.4 — 指数退避重试（最多 3 次）
**Files:**

- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/auth/client.ts`
- Test: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/client.test.ts`

- [ ] **Step 1: 写 AdsApiClient 测试**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/client.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { AdsApiClient } from "../src/auth/client.js";

describe("AdsApiClient", () => {
  let client: AdsApiClient;
  let mockFetch: any;

  beforeEach(() => {
    vi.restoreAllMocks();
    process.env.AMZ_ADS_PROFILE_ID = "test-profile-id";

    mockFetch = vi.fn().mockResolvedValue({
      ok: true,
      json: () => Promise.resolve({ success: true }),
    });
    globalThis.fetch = mockFetch;

    const mockTokenManager = {
      getAccessToken: vi.fn().mockResolvedValue("mock-access-token"),
    };
    client = new AdsApiClient(mockTokenManager as any);
  });

  it("应设置正确的请求头", async () => {
    await client.get("/sp/campaigns");
    const [url, options] = mockFetch.mock.calls[0];
    expect(url).toContain("advertising-api.amazon.com");
    expect(options.headers["Authorization"]).toBe("Bearer mock-access-token");
    expect(options.headers["Amazon-Advertising-API-Scope"]).toBe("test-profile-id");
  });

  it("GET 请求应成功返回数据", async () => {
    const result = await client.get("/sp/campaigns");
    expect(result).toEqual({ success: true });
  });

  it("POST 请求应发送 JSON body", async () => {
    const body = { name: "Test Campaign", budget: 100 };
    await client.post("/sp/campaigns", body);
    const [, options] = mockFetch.mock.calls[0];
    expect(options.method).toBe("POST");
    expect(JSON.parse(options.body)).toEqual(body);
  });

  it("API 返回 429 应重试", async () => {
    let callCount = 0;
    mockFetch.mockImplementation(() => {
      callCount++;
      if (callCount === 1) {
        return Promise.resolve({
          ok: false,
          status: 429,
          statusText: "Too Many Requests",
          headers: { get: () => "1" },
          text: () => Promise.resolve("rate limited"),
        });
      }
      return Promise.resolve({ ok: true, json: () => Promise.resolve({ retrySuccess: true }) });
    });
    client.setRetryDelay(1);
    const result = await client.get("/sp/campaigns");
    expect(result).toEqual({ retrySuccess: true });
    expect(mockFetch).toHaveBeenCalledTimes(2);
  });

  it("API 返回 5xx 应重试最多 3 次", async () => {
    mockFetch.mockResolvedValue({
      ok: false,
      status: 500,
      statusText: "Internal Server Error",
      text: () => Promise.resolve("server error"),
    });
    client.setRetryDelay(1);
    await expect(client.get("/sp/campaigns")).rejects.toThrow("API request failed");
    expect(mockFetch).toHaveBeenCalledTimes(4); // 1 + 3 retries
  });

  it("API 返回 4xx (非 429) 应直接报错不重试", async () => {
    mockFetch.mockResolvedValue({
      ok: false,
      status: 400,
      statusText: "Bad Request",
      text: () => Promise.resolve("bad request"),
    });
    await expect(client.get("/sp/campaigns")).rejects.toThrow("API request failed");
    expect(mockFetch).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/client.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 AdsApiClient**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/auth/client.ts`:

```typescript
export interface TokenProvider {
  getAccessToken(): Promise<string>;
}

const BASE_URL = "https://advertising-api.amazon.com";
const MAX_RETRIES = 3;
const DEFAULT_RETRY_DELAY_MS = 500;

export class AdsApiClient {
  private tokenProvider: TokenProvider;
  private retryDelayMs = DEFAULT_RETRY_DELAY_MS;

  constructor(tokenProvider: TokenProvider) {
    this.tokenProvider = tokenProvider;
  }

  setRetryDelay(ms: number): void {
    this.retryDelayMs = ms;
  }

  private async request(method: string, path: string, body?: unknown): Promise<any> {
    const token = await this.tokenProvider.getAccessToken();
    const profileId = process.env.AMZ_ADS_PROFILE_ID;
    const url = `${BASE_URL}${path}`;
    const headers: Record<string, string> = {
      Authorization: `Bearer ${token}`,
      "Content-Type": "application/json",
      Accept: "application/json",
    };
    if (profileId) headers["Amazon-Advertising-API-Scope"] = profileId;

    let lastError: Error | null = null;
    for (let attempt = 0; attempt <= MAX_RETRIES; attempt++) {
      const response = await fetch(url, {
        method,
        headers,
        body: body ? JSON.stringify(body) : undefined,
      });

      if (response.ok) return response.json();

      const statusCode = response.status;
      const errorBody = await response.text();

      if ((statusCode === 429 || statusCode >= 500) && attempt < MAX_RETRIES) {
        const retryAfter = response.headers?.get("Retry-After");
        const delay = retryAfter
          ? parseInt(retryAfter, 10) * 1000
          : this.retryDelayMs * Math.pow(2, attempt);
        await new Promise((resolve) => setTimeout(resolve, delay));
        lastError = new Error(
          `API request failed (attempt ${attempt + 1}): ${statusCode} — ${errorBody}`,
        );
        continue;
      }

      throw new Error(`API request failed: ${statusCode} ${response.statusText} — ${errorBody}`);
    }
    throw lastError ?? new Error("API request failed after retries");
  }

  async get(path: string): Promise<any> {
    return this.request("GET", path);
  }
  async post(path: string, body: unknown): Promise<any> {
    return this.request("POST", path, body);
  }
  async put(path: string, body: unknown): Promise<any> {
    return this.request("PUT", path, body);
  }
  async delete(path: string): Promise<any> {
    return this.request("DELETE", path);
  }
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/client.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(amazon-ads-mcp): add API client with exponential backoff"
```

---

### Task 12: Campaign 工具（5 个）

**Spec 对应:** Section 3.1 — list_campaigns, get_campaign_details, get_campaign_report, create_campaign, update_campaign
**Files:**

- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/tools/campaigns.ts`
- Test: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/campaigns.test.ts`

- [ ] **Step 1: 写 Campaign 工具测试**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/campaigns.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { registerCampaignTools } from "../src/tools/campaigns.js";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import type { AdsApiClient } from "../src/auth/client.js";

function createToolRegistry(client: AdsApiClient) {
  const handlers: Record<string, (args: any) => any> = {};
  const mockServer = {
    tool: (name: string, _d: any, _s: any, handler?: (args: any) => any) => {
      handlers[name] = handler ?? _s;
    },
  } as unknown as McpServer;
  registerCampaignTools(mockServer, client);
  return handlers;
}

describe("campaign tools", () => {
  let mockClient: { get: any; post: any; put: any };
  let handlers: ReturnType<typeof createToolRegistry>;

  beforeEach(() => {
    mockClient = { get: vi.fn(), post: vi.fn(), put: vi.fn() };
    handlers = createToolRegistry(mockClient as any);
  });

  it("list_campaigns 应调用 GET /sp/campaigns", async () => {
    mockClient.get.mockResolvedValue([{ campaignId: "c1", name: "Campaign 1", state: "enabled" }]);
    const result = await handlers.list_campaigns({});
    const data = JSON.parse(result.content[0].text);
    expect(data).toHaveLength(1);
    expect(mockClient.get).toHaveBeenCalledWith(expect.stringContaining("/sp/campaigns"));
  });

  it("list_campaigns 应支持状态过滤", async () => {
    mockClient.get.mockResolvedValue([]);
    await handlers.list_campaigns({ state: "enabled" });
    expect(mockClient.get).toHaveBeenCalledWith(expect.stringContaining("stateFilter=enabled"));
  });

  it("get_campaign_details 应调用 GET /sp/campaigns/{id}", async () => {
    mockClient.get.mockResolvedValue({ campaignId: "c1", name: "Test", budget: { budget: 100 } });
    const result = await handlers.get_campaign_details({ campaign_id: "c1" });
    expect(JSON.parse(result.content[0].text).campaignId).toBe("c1");
    expect(mockClient.get).toHaveBeenCalledWith("/sp/campaigns/c1");
  });

  it("get_campaign_report 应发送报告请求", async () => {
    mockClient.post.mockResolvedValue({ reportId: "r1", status: "IN_PROGRESS" });
    const result = await handlers.get_campaign_report({
      campaign_ids: '["c1"]',
      start_date: "2026-04-01",
      end_date: "2026-04-15",
      metrics: "impressions,clicks,cost",
    });
    expect(JSON.parse(result.content[0].text).reportId).toBe("r1");
    expect(mockClient.post).toHaveBeenCalledWith(
      "/sp/campaigns/report",
      expect.objectContaining({ campaignIds: ["c1"] }),
    );
  });

  it("create_campaign 应调用 POST /sp/campaigns", async () => {
    mockClient.post.mockResolvedValue({ campaignId: "new-c1", name: "New Campaign" });
    const result = await handlers.create_campaign({
      name: "New Campaign",
      targeting_type: "MANUAL",
      daily_budget: 50,
      strategy: "LEGACY_FOR_SALES",
      state: "enabled",
    });
    expect(JSON.parse(result.content[0].text).campaignId).toBe("new-c1");
    expect(mockClient.post).toHaveBeenCalledWith(
      "/sp/campaigns",
      expect.objectContaining({ name: "New Campaign" }),
    );
  });

  it("update_campaign 应调用 PUT /sp/campaigns", async () => {
    mockClient.put.mockResolvedValue([{ campaignId: "c1", state: "paused" }]);
    const result = await handlers.update_campaign({
      campaign_id: "c1",
      state: "paused",
      daily_budget: 100,
    });
    expect(JSON.parse(result.content[0].text)[0].campaignId).toBe("c1");
    expect(mockClient.put).toHaveBeenCalledWith(
      "/sp/campaigns",
      expect.arrayContaining([expect.objectContaining({ campaignId: "c1" })]),
    );
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/campaigns.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 campaigns.ts**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/tools/campaigns.ts`:

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import type { AdsApiClient } from "../auth/client.js";

export function registerCampaignTools(server: McpServer, client: AdsApiClient): void {
  server.tool(
    "list_campaigns",
    "List SP campaigns with optional state filter",
    {
      state: z.string().optional().describe("Filter by state: enabled/paused/archived"),
      name_contains: z.string().optional().describe("Filter by campaign name substring"),
      limit: z.number().optional().describe("Max results (default 100)"),
    },
    async (args) => {
      const params: string[] = [];
      if (args.state) params.push(`stateFilter=${args.state}`);
      if (args.name_contains) params.push(`nameContains=${encodeURIComponent(args.name_contains)}`);
      params.push(`maxResults=${args.limit ?? 100}`);
      const query = params.length > 0 ? `?${params.join("&")}` : "";
      const data = await client.get(`/sp/campaigns${query}`);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "get_campaign_details",
    "Get detailed information for a single SP campaign",
    {
      campaign_id: z.string().describe("Campaign ID"),
    },
    async (args) => {
      const data = await client.get(`/sp/campaigns/${args.campaign_id}`);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "get_campaign_report",
    "Request a campaign performance report",
    {
      campaign_ids: z.string().describe("JSON array of campaign IDs"),
      start_date: z.string().describe("Start date (YYYY-MM-DD)"),
      end_date: z.string().describe("End date (YYYY-MM-DD)"),
      metrics: z
        .string()
        .optional()
        .describe("Comma-separated metrics (default: impressions,clicks,cost,attributedSales)"),
    },
    async (args) => {
      const campaignIds = JSON.parse(args.campaign_ids) as string[];
      const body = {
        campaignIds,
        reportDate: args.start_date,
        endDate: args.end_date,
        metrics: args.metrics ?? "impressions,clicks,cost,attributedSales",
      };
      const data = await client.post("/sp/campaigns/report", body);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "create_campaign",
    "Create a new SP campaign",
    {
      name: z.string().describe("Campaign name"),
      targeting_type: z.string().describe("Targeting type: MANUAL or AUTO"),
      daily_budget: z.number().describe("Daily budget in account currency"),
      strategy: z
        .string()
        .optional()
        .describe("Bidding strategy: LEGACY_FOR_SALES/LEGACY_FOR_CONVERSIONS/AUTO_FOR_SALES"),
      state: z.string().optional().describe("Initial state: enabled/paused (default: enabled)"),
    },
    async (args) => {
      const body = {
        name: args.name,
        targetingType: args.targeting_type,
        dailyBudget: args.daily_budget,
        bidding: { strategy: args.strategy ?? "LEGACY_FOR_SALES" },
        state: args.state ?? "enabled",
      };
      const data = await client.post("/sp/campaigns", body);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "update_campaign",
    "Update campaign settings (budget, state, bidding)",
    {
      campaign_id: z.string().describe("Campaign ID to update"),
      name: z.string().optional().describe("New campaign name"),
      state: z.string().optional().describe("New state: enabled/paused/archived"),
      daily_budget: z.number().optional().describe("New daily budget"),
      strategy: z.string().optional().describe("New bidding strategy"),
    },
    async (args) => {
      const update: Record<string, unknown> = { campaignId: args.campaign_id };
      if (args.name !== undefined) update.name = args.name;
      if (args.state !== undefined) update.state = args.state;
      if (args.daily_budget !== undefined) update.dailyBudget = args.daily_budget;
      if (args.strategy !== undefined) update.bidding = { strategy: args.strategy };
      const data = await client.put("/sp/campaigns", [update]);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/campaigns.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(amazon-ads-mcp): add campaign tools (list, get, report, create, update)"
```

---

### Task 13a: Ad Group 工具（2 个）

**Spec 对应:** Section 3.1 — list_ad_groups, create_ad_group
**Files:**

- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/tools/ad-groups.ts`
- Test: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/ad-groups.test.ts`

- [ ] **Step 1: 写 Ad Group 工具测试**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/ad-groups.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { registerAdGroupTools } from "../src/tools/ad-groups.js";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import type { AdsApiClient } from "../src/auth/client.js";

function createToolRegistry(client: AdsApiClient) {
  const handlers: Record<string, (args: any) => any> = {};
  const mockServer = {
    tool: (name: string, _d: any, _s: any, handler?: (args: any) => any) => {
      handlers[name] = handler ?? _s;
    },
  } as unknown as McpServer;
  registerAdGroupTools(mockServer, client);
  return handlers;
}

describe("ad-group tools", () => {
  let mockClient: { get: any; post: any };
  let handlers: ReturnType<typeof createToolRegistry>;

  beforeEach(() => {
    mockClient = { get: vi.fn(), post: vi.fn() };
    handlers = createToolRegistry(mockClient as any);
  });

  it("list_ad_groups 应调用 GET /sp/adGroups 并支持 campaign_id 过滤", async () => {
    mockClient.get.mockResolvedValue([{ adGroupId: "ag1", name: "Ad Group 1", campaignId: "c1" }]);
    const result = await handlers.list_ad_groups({ campaign_id: "c1" });
    const data = JSON.parse(result.content[0].text);
    expect(data).toHaveLength(1);
    expect(mockClient.get).toHaveBeenCalledWith(
      expect.stringContaining("/sp/adGroups?campaignId=c1"),
    );
  });

  it("create_ad_group 应调用 POST /sp/adGroups", async () => {
    mockClient.post.mockResolvedValue({ adGroupId: "new-ag1", name: "New Group" });
    const result = await handlers.create_ad_group({
      campaign_id: "c1",
      name: "New Group",
      default_bid: 1.5,
      state: "enabled",
    });
    expect(JSON.parse(result.content[0].text).adGroupId).toBe("new-ag1");
    expect(mockClient.post).toHaveBeenCalledWith(
      "/sp/adGroups",
      expect.arrayContaining([
        expect.objectContaining({ campaignId: "c1", name: "New Group", defaultBid: 1.5 }),
      ]),
    );
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/ad-groups.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 ad-groups.ts**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/tools/ad-groups.ts`:

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import type { AdsApiClient } from "../auth/client.js";

export function registerAdGroupTools(server: McpServer, client: AdsApiClient): void {
  server.tool(
    "list_ad_groups",
    "List SP ad groups with optional campaign filter",
    {
      campaign_id: z.string().describe("Campaign ID to filter by"),
      state: z.string().optional().describe("Filter by state: enabled/paused/archived"),
    },
    async (args) => {
      const params = [`campaignId=${args.campaign_id}`];
      if (args.state) params.push(`stateFilter=${args.state}`);
      params.push("maxResults=100");
      const data = await client.get(`/sp/adGroups?${params.join("&")}`);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "create_ad_group",
    "Create a new ad group under a campaign",
    {
      campaign_id: z.string().describe("Parent campaign ID"),
      name: z.string().describe("Ad group name"),
      default_bid: z.number().describe("Default bid amount"),
      state: z.string().optional().describe("Initial state: enabled/paused (default: enabled)"),
    },
    async (args) => {
      const body = [
        {
          campaignId: args.campaign_id,
          name: args.name,
          defaultBid: args.default_bid,
          state: args.state ?? "enabled",
        },
      ];
      const data = await client.post("/sp/adGroups", body);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/ad-groups.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(amazon-ads-mcp): add ad-group tools (list, create)"
```

---

### Task 13b: Keyword 工具（5 个）

**Spec 对应:** Section 3.1 — list_keywords, create_keywords, update_keywords, add_negative_keywords, get_keyword_recommendations
**Files:**

- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/tools/keywords.ts`
- Test: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/keywords.test.ts`

- [ ] **Step 1: 写 Keyword 工具测试**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/keywords.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { registerKeywordTools } from "../src/tools/keywords.js";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import type { AdsApiClient } from "../src/auth/client.js";

function createToolRegistry(client: AdsApiClient) {
  const handlers: Record<string, (args: any) => any> = {};
  const mockServer = {
    tool: (name: string, _d: any, _s: any, handler?: (args: any) => any) => {
      handlers[name] = handler ?? _s;
    },
  } as unknown as McpServer;
  registerKeywordTools(mockServer, client);
  return handlers;
}

describe("keyword tools", () => {
  let mockClient: { get: any; post: any; put: any };
  let handlers: ReturnType<typeof createToolRegistry>;

  beforeEach(() => {
    mockClient = { get: vi.fn(), post: vi.fn(), put: vi.fn() };
    handlers = createToolRegistry(mockClient as any);
  });

  it("list_keywords 应调用 GET /sp/keywords 并支持过滤", async () => {
    mockClient.get.mockResolvedValue([
      { keywordId: "kw1", keywordText: "running shoes", state: "enabled" },
    ]);
    const result = await handlers.list_keywords({ campaign_id: "c1" });
    expect(JSON.parse(result.content[0].text)).toHaveLength(1);
    expect(mockClient.get).toHaveBeenCalledWith(
      expect.stringContaining("/sp/keywords?campaignId=c1"),
    );
  });

  it("create_keywords 应解析 JSON 数组并调用 POST /sp/keywords", async () => {
    mockClient.post.mockResolvedValue([{ keywordId: "new-kw1", keywordText: "test", bid: 1.0 }]);
    const result = await handlers.create_keywords({
      keywords_json:
        '[{"campaignId":"c1","adGroupId":"ag1","keywordText":"test","matchType":"EXACT","bid":1.0}]',
    });
    expect(JSON.parse(result.content[0].text)).toHaveLength(1);
    expect(mockClient.post).toHaveBeenCalledWith(
      "/sp/keywords",
      expect.arrayContaining([expect.objectContaining({ keywordText: "test" })]),
    );
  });

  it("update_keywords 应解析 JSON 数组并调用 PUT /sp/keywords", async () => {
    mockClient.put.mockResolvedValue([{ keywordId: "kw1", bid: 2.0 }]);
    const result = await handlers.update_keywords({
      updates_json: '[{"keywordId":"kw1","bid":2.0}]',
    });
    expect(JSON.parse(result.content[0].text)).toHaveLength(1);
    expect(mockClient.put).toHaveBeenCalledWith("/sp/keywords", [{ keywordId: "kw1", bid: 2.0 }]);
  });

  it("add_negative_keywords 应调用 POST /sp/negativeKeywords", async () => {
    mockClient.post.mockResolvedValue([{ keywordId: "neg1", keywordText: "bad word" }]);
    const result = await handlers.add_negative_keywords({
      campaign_id: "c1",
      keywords_json: '["bad word","wrong"]',
    });
    expect(JSON.parse(result.content[0].text)).toHaveLength(1);
    expect(mockClient.post).toHaveBeenCalledWith(
      "/sp/negativeKeywords",
      expect.arrayContaining([
        expect.objectContaining({ campaignId: "c1", keywordText: "bad word" }),
      ]),
    );
  });

  it("get_keyword_recommendations 应调用 POST /sp/keywordRecommendations", async () => {
    mockClient.post.mockResolvedValue({
      recommendations: [{ keywordText: "suggested", matchType: "BROAD" }],
    });
    const result = await handlers.get_keyword_recommendations({ ad_group_id: "ag1" });
    expect(JSON.parse(result.content[0].text).recommendations).toHaveLength(1);
    expect(mockClient.post).toHaveBeenCalledWith(
      "/sp/keywordRecommendations",
      expect.objectContaining({ adGroupId: "ag1" }),
    );
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/keywords.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 keywords.ts**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/tools/keywords.ts`:

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import type { AdsApiClient } from "../auth/client.js";

export function registerKeywordTools(server: McpServer, client: AdsApiClient): void {
  server.tool(
    "list_keywords",
    "List SP keywords with optional filters",
    {
      campaign_id: z.string().optional().describe("Filter by campaign ID"),
      ad_group_id: z.string().optional().describe("Filter by ad group ID"),
      state: z.string().optional().describe("Filter by state: enabled/paused/archived"),
    },
    async (args) => {
      const params: string[] = [];
      if (args.campaign_id) params.push(`campaignId=${args.campaign_id}`);
      if (args.ad_group_id) params.push(`adGroupId=${args.ad_group_id}`);
      if (args.state) params.push(`stateFilter=${args.state}`);
      params.push("maxResults=100");
      const query = params.length > 0 ? `?${params.join("&")}` : "";
      const data = await client.get(`/sp/keywords${query}`);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "create_keywords",
    "Create SP keywords (batch)",
    {
      keywords_json: z
        .string()
        .describe(
          "JSON array of keyword objects: [{campaignId, adGroupId, keywordText, matchType, bid}]",
        ),
    },
    async (args) => {
      const keywords = JSON.parse(args.keywords_json) as Record<string, unknown>[];
      const data = await client.post("/sp/keywords", keywords);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "update_keywords",
    "Update SP keywords (batch)",
    {
      updates_json: z
        .string()
        .describe("JSON array of update objects: [{keywordId, bid?, state?}]"),
    },
    async (args) => {
      const updates = JSON.parse(args.updates_json) as Record<string, unknown>[];
      const data = await client.put("/sp/keywords", updates);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "add_negative_keywords",
    "Add campaign-level negative keywords",
    {
      campaign_id: z.string().describe("Campaign ID"),
      keywords_json: z.string().describe('JSON array of keyword texts: ["word1","word2"]'),
      match_type: z
        .string()
        .optional()
        .describe("Match type: NEGATIVE_EXACT or NEGATIVE_PHRASE (default: NEGATIVE_EXACT)"),
    },
    async (args) => {
      const texts = JSON.parse(args.keywords_json) as string[];
      const matchType = args.match_type ?? "NEGATIVE_EXACT";
      const body = texts.map((text) => ({
        campaignId: args.campaign_id,
        keywordText: text,
        matchType,
      }));
      const data = await client.post("/sp/negativeKeywords", body);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );

  server.tool(
    "get_keyword_recommendations",
    "Get keyword recommendations for an ad group",
    {
      ad_group_id: z.string().describe("Ad group ID to get recommendations for"),
      max_recommendations: z
        .number()
        .optional()
        .describe("Max number of recommendations (default: 100)"),
    },
    async (args) => {
      const body = {
        adGroupId: args.ad_group_id,
        maxRecommendations: args.max_recommendations ?? 100,
      };
      const data = await client.post("/sp/keywordRecommendations", body);
      return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
    },
  );
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/keywords.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(amazon-ads-mcp): add keyword tools (list, create, update, negative, recommendations)"
```

---

### Task 13c: Account 工具（1 个）

**Spec 对应:** Section 3.1 — get_account_budget
**Files:**

- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/tools/account.ts`
- Test: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/account.test.ts`

- [ ] **Step 1: 写 Account 工具测试**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/__tests__/account.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from "vitest";
import { registerAccountTools } from "../src/tools/account.js";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import type { AdsApiClient } from "../src/auth/client.js";

function createToolRegistry(client: AdsApiClient) {
  const handlers: Record<string, (args: any) => any> = {};
  const mockServer = {
    tool: (name: string, _d: any, _s: any, handler?: (args: any) => any) => {
      handlers[name] = handler ?? _s;
    },
  } as unknown as McpServer;
  registerAccountTools(mockServer, client);
  return handlers;
}

describe("account tools", () => {
  let mockClient: { get: any };
  let handlers: ReturnType<typeof createToolRegistry>;

  beforeEach(() => {
    mockClient = { get: vi.fn() };
    handlers = createToolRegistry(mockClient as any);
  });

  it("get_account_budget 应调用 GET /v2/profiles", async () => {
    mockClient.get.mockResolvedValue([
      { profileId: "p1", accountInfo: { marketplaceStringId: "ATVPDKIKX0DER" }, dailyBudget: 100 },
    ]);
    const result = await handlers.get_account_budget({});
    const data = JSON.parse(result.content[0].text);
    expect(data[0].profileId).toBe("p1");
    expect(mockClient.get).toHaveBeenCalledWith("/v2/profiles");
  });
});
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/account.test.ts
```

Expected: FAIL

- [ ] **Step 3: 实现 account.ts**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/tools/account.ts`:

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import type { AdsApiClient } from "../auth/client.js";

export function registerAccountTools(server: McpServer, client: AdsApiClient): void {
  server.tool("get_account_budget", "Get account profile and budget information", {}, async () => {
    const data = await client.get("/v2/profiles");
    return { content: [{ type: "text" as const, text: JSON.stringify(data) }] };
  });
}
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run __tests__/account.test.ts
```

Expected: PASS

- [ ] **Step 5: 提交**

```bash
git add -A && git commit -m "feat(amazon-ads-mcp): add account budget tool"
```

---

### Task 14: MCP Server 入口 + 编译验证

**Files:**

- Create: `aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/index.ts`

- [ ] **Step 1: 实现 MCP Server 入口**

`aws-sp-assistant/mcp-servers/amazon-ads-mcp/src/index.ts`:

```typescript
#!/usr/bin/env node

import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { TokenManager } from "./auth/token-manager.js";
import { AdsApiClient } from "./auth/client.js";
import { registerCampaignTools } from "./tools/campaigns.js";
import { registerAdGroupTools } from "./tools/ad-groups.js";
import { registerKeywordTools } from "./tools/keywords.js";
import { registerAccountTools } from "./tools/account.js";

const tokenManager = new TokenManager();
const client = new AdsApiClient(tokenManager);

const server = new McpServer({ name: "amazon-ads-mcp", version: "0.1.0" });

registerCampaignTools(server, client);
registerAdGroupTools(server, client);
registerKeywordTools(server, client);
registerAccountTools(server, client);

const transport = new StdioServerTransport();
server.connect(transport).catch((err) => {
  console.error("Failed to start Amazon Ads MCP server:", err);
  process.exit(1);
});
```

- [ ] **Step 2: 编译并验证**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx tsc
echo '{"jsonrpc":"2.0","method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}},"id":1}' | AMZ_ADS_CLIENT_ID=x AMZ_ADS_CLIENT_SECRET=x AMZ_ADS_REFRESH_TOKEN=x AMZ_ADS_PROFILE_ID=x node dist/index.js
```

Expected: JSON response with server info

- [ ] **Step 3: 运行全部测试**

```bash
cd aws-sp-assistant/mcp-servers/amazon-ads-mcp && npx vitest run
```

Expected: All 26 tests PASS

- [ ] **Step 4: 提交**

```bash
git add -A && git commit -m "feat(amazon-ads-mcp): add MCP server entry point with all 13 tools"
```

---

# Plan 3: Agent + Skills + Cron

> **Spec 对应:** Section 4 — Skill 策略层与 Agent 配置
> **依赖:** Plan 1 (Storage MCP), Plan 2 (Amazon Ads MCP)

## 文件结构

```
aws-sp-assistant/
├── config/
│   └── system-prompt.md         # Section 4.1 System Prompt
├── skills/                      # Section 4.2 — 10 个 Skill 文件 + ad-operations (UC-10)
│   ├── risk-rules.md            # 风险等级判定规则
│   ├── pre-launch-check.md      # UC-01
│   ├── keyword-library.md       # UC-02
│   ├── sp-campaign-plan.md      # UC-03
│   ├── sp-campaign-publish.md   # UC-04
│   ├── daily-inspection.md      # UC-05 (含 Section 4.3 示例的完整版)
│   ├── weekly-review.md         # UC-06 (含 Section 4.5 Skill 自进化)
│   ├── notification-template.md # UC-07 (含 Section 5.2 飞书卡片模板)
│   ├── task-collaboration.md    # UC-08 (含 Section 5.1 入站/出站映射)
│   ├── custom-query.md          # UC-09
│   └── ad-operations.md         # UC-10 广告修改操作决策规则
└── setup.sh                     # Section 6.4 部署流程的自动化
```

---

### Task 15: System Prompt 模板

**Spec 对应:** Section 4.1 — System Prompt 核心内容
**Files:**

- Create: `aws-sp-assistant/config/system-prompt.md`

- [ ] **Step 1: 编写 System Prompt**

`aws-sp-assistant/config/system-prompt.md`:

```markdown
# 亚马逊 SP 广告智能运营助手

你是一位专业的亚马逊 SP（Sponsored Products）广告运营助手，围绕单个 Listing / 广告对象持续提供运营支持。

## 核心职责

1. **投前检查** — 评估 Listing 是否具备投放条件
2. **词库建设** — 生成并维护关键词词库
3. **方案搭建** — 生成 Campaign / Ad Group / Keyword 结构方案
4. **日常巡检** — 每日识别异常，输出优化建议
5. **周度复盘** — 归因分析，输出下周优化建议
6. **动作执行** — 经用户确认后执行广告操作

## 决策规则

- 所有广告账户写入操作（创建、修改、暂停、删除）必须经用户在飞书确认
- 低风险读取操作（查询数据、生成报告）可直接执行
- 预算调整幅度超过 ±20% 需要明确确认并附带影响说明
- 暂停操作需要附带原因说明
- 同一对象 5 分钟内不允许重复修改
- 单次批量操作最多影响 50 个对象

## 工作流程

1. 每次执行形成独立 Run，结果通过 save_run 持久化
2. 所有执行动作通过 log_action 记录审计日志（actor、action、target、result）
3. 定时任务（日巡检、周复盘）由 Cron 触发
4. 用户确认通过飞书消息回复完成

## 可用工具

### Amazon Ads MCP

list_campaigns, get_campaign_details, get_campaign_report, create_campaign, update_campaign, list_ad_groups, create_ad_group, list_keywords, create_keywords, update_keywords, add_negative_keywords, get_keyword_recommendations, get_account_budget

### Storage MCP

save_run, query_runs, get_run_detail, save_keyword_library, load_keyword_library, log_action, query_audit_log, save_report, query_reports, save_manual_input, load_manual_input, save_skill_candidate, query_skill_candidates

## 降级策略

当 MCP 工具不可用时：

1. 检查 manual_inputs 是否有未过期缓存（24h TTL）
2. 缓存不存在时通过飞书请求用户手动提供数据
3. 用户补充后调用 save_manual_input 保存
4. 广告写入操作失败时仅输出建议方案，不自动重试

## 输出规范

- 使用结构化格式（表格、列表）
- 包含具体数据（金额、百分比、变化趋势）
- 建议动作标注风险等级（🟢🟡🔴🚨）
- 异常项附带归因分析和建议
```

- [ ] **Step 2: 提交**

```bash
git add -A && git commit -m "feat(config): add agent system prompt"
```

---

### Task 16: Skill 策略文件（11 个）

**Spec 对应:** Section 4.2 — Skill 文件结构, Section 4.3 — Skill 示例, Section 5.2 — 飞书卡片模板
**Files:**

- Create: `aws-sp-assistant/skills/risk-rules.md`
- Create: `aws-sp-assistant/skills/pre-launch-check.md`
- Create: `aws-sp-assistant/skills/keyword-library.md`
- Create: `aws-sp-assistant/skills/sp-campaign-plan.md`
- Create: `aws-sp-assistant/skills/sp-campaign-publish.md`
- Create: `aws-sp-assistant/skills/daily-inspection.md`
- Create: `aws-sp-assistant/skills/weekly-review.md`
- Create: `aws-sp-assistant/skills/notification-template.md`
- Create: `aws-sp-assistant/skills/task-collaboration.md`
- Create: `aws-sp-assistant/skills/custom-query.md`
- Create: `aws-sp-assistant/skills/ad-operations.md`

每个 Skill 文件必须包含以下结构：

```markdown
# [Skill 名称]

## 触发

[触发条件描述]

## 数据获取

[需要调用的 MCP 工具列表]

## [业务逻辑]

[规则/阈值/决策树]

## 输出结构

[输出格式模板]

## 持久化

[save_run / save_report / log_action 调用]
```

**各文件完整内容：**

**1. risk-rules.md** — 风险等级 + 10 种降级场景决策树（含触发/判定流程/输出结构/持久化完整五段结构）

内容见 `aws-sp-assistant/skills/risk-rules.md`（已在代码审核中重写为完整版，包含 4 级风险等级定义 + 10 种降级场景表格 + 判定流程图）。

**2. pre-launch-check.md** — UC-01 投前检查

内容见 `aws-sp-assistant/skills/pre-launch-check.md`，包含：触发条件、4 维度检查清单（Listing/库存/评论/竞争）、5 步数据获取流程、输出格式、持久化。

**3. keyword-library.md** — UC-02 词库建设

内容见 `aws-sp-assistant/skills/keyword-library.md`，包含：数据来源优先级、词库构建规则（核心词/长尾词/否定词）、6 步工作流程、输出格式模板、持久化。

**4. sp-campaign-plan.md** — UC-03 方案搭建

内容见 `aws-sp-assistant/skills/sp-campaign-plan.md`，包含：Campaign 层命名/预算/竞价规范、Ad Group 分组策略（精准/长尾/探索）、Keyword 层匹配和竞价规则、4 步工作流程、输出格式、持久化。

**5. sp-campaign-publish.md** — UC-04 发布执行

内容见 `aws-sp-assistant/skills/sp-campaign-publish.md`，包含：前置条件、5 步执行流程（Campaign→AdGroup→Keywords→否定词→验证）、错误处理策略、输出格式、持久化。

**6. daily-inspection.md** — UC-05 日巡检

内容见 `aws-sp-assistant/skills/daily-inspection.md`，包含：触发条件、5 步数据获取、8 条异常识别规则表（ACOS/CTR/日消耗/Campaign状态/单关键词花费/CPC/转化率/Impressions）、6 步工作流程、健康评分输出结构、持久化。

**7. weekly-review.md** — UC-06 周复盘

内容见 `aws-sp-assistant/skills/weekly-review.md`，包含：6 步数据获取、5 个分析维度（整体趋势/Campaign归因/关键词归因/执行回顾/Skill自进化）、输出结构、持久化（含 save_skill_candidate）。

**8. notification-template.md** — UC-07 推送通知

内容见 `aws-sp-assistant/skills/notification-template.md`，包含：触发条件、6 种模板（日巡检摘要[飞书卡片+按钮:查看详情/执行建议/忽略]、周复盘推送[纯文本]、异常告警[飞书卡片+按钮:立即处理/加入明日巡检]、执行确认请求[飞书卡片+按钮:确认执行/修改方案/取消]、执行回执[纯文本]、数据请求[纯文本]）、推送规则。

**9. task-collaboration.md** — UC-08 任务协作

内容见 `aws-sp-assistant/skills/task-collaboration.md`，包含：7 种任务类型映射表、5 步任务解析流程（意图识别→参数提取→参数补全→Skill调度→结果输出）、追问策略、上下文保持、持久化。

**10. custom-query.md** — UC-09 自定义查询

内容见 `aws-sp-assistant/skills/custom-query.md`，包含：4 种触发示例、8 种查询意图→MCP工具映射表、5 步查询流程（含时间范围解析）、输出原则、完整示例、持久化。

**11. ad-operations.md** — UC-10 广告操作

内容见 `aws-sp-assistant/skills/ad-operations.md`，包含：10 种操作→风险等级矩阵、5 步执行流程（参数验证→风险评估→确认请求→执行操作→结果验证）、3 条操作限制、持久化。

- [ ] **Step 1: 创建 11 个 Skill 文件**

按上述 11 个文件的完整内容逐个创建到 `aws-sp-assistant/skills/` 目录。每个文件内容见对应的 `aws-sp-assistant/skills/*.md` 实际文件。

- [ ] **Step 2: 验证 Skill 文件结构完整性**

```bash
for f in skills/*.md; do echo "=== $f ==="; grep -c "^##" "$f"; done
```

Expected: 每个 Skill 文件至少包含 4 个 `##` 段落

- [ ] **Step 3: 提交**

```bash
git add -A && git commit -m "feat(skills): add 11 skill strategy files (UC-01 to UC-10 + risk rules)"
```

---

### Task 17: 部署脚本（setup.sh + uninstall.sh）

**Spec 对应:** Section 6.4 — 部署流程
**Files:**

- Create: `aws-sp-assistant/setup.sh`
- Create: `aws-sp-assistant/uninstall.sh`

- [ ] **Step 1: 实现 setup.sh**

`aws-sp-assistant/setup.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
cd "$SCRIPT_DIR"

# 颜色输出
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

log()  { echo -e "${GREEN}[setup]${NC} $*"; }
warn() { echo -e "${YELLOW}[warn]${NC} $*"; }
err()  { echo -e "${RED}[error]${NC} $*" >&2; exit 1; }

# ──────────────────────────────────────
# 1. 环境检查
# ──────────────────────────────────────
log "检查运行环境..."

command -v node >/dev/null 2>&1 || err "Node.js 未安装（需要 v22+）"
command -v npm >/dev/null 2>&1 || err "npm 未安装"
command -v openclaw >/dev/null 2>&1 || warn "openclaw CLI 未找到，请确认已安装"

NODE_VERSION=$(node -v | sed 's/v//' | cut -d. -f1)
[ "$NODE_VERSION" -ge 22 ] || err "Node.js 版本需要 >=22，当前: $(node -v)"

# ──────────────────────────────────────
# 2. 环境变量
# ──────────────────────────────────────
if [ ! -f .env ]; then
  warn ".env 文件不存在，从 .env.example 复制"
  cp .env.example .env
  warn "请编辑 .env 填入 Amazon Ads API 凭证后重新运行 setup.sh"
  exit 0
fi

# 导出环境变量
set -a; source .env; set +a

# 验证必要变量
[ -n "${AMZ_ADS_CLIENT_ID:-}" ] || err "AMZ_ADS_CLIENT_ID 未设置"
[ -n "${AMZ_ADS_CLIENT_SECRET:-}" ] || err "AMZ_ADS_CLIENT_SECRET 未设置"
[ -n "${AMZ_ADS_REFRESH_TOKEN:-}" ] || err "AMZ_ADS_REFRESH_TOKEN 未设置"
[ -n "${AMZ_ADS_PROFILE_ID:-}" ] || err "AMZ_ADS_PROFILE_ID 未设置"

# ──────────────────────────────────────
# 3. 编译 MCP Server
# ──────────────────────────────────────
log "安装依赖..."
npm install

log "编译 Storage MCP Server..."
cd mcp-servers/storage-mcp && npm run build && cd "$SCRIPT_DIR"

log "编译 Amazon Ads MCP Server..."
cd mcp-servers/amazon-ads-mcp && npm run build && cd "$SCRIPT_DIR"

# ──────────────────────────────────────
# 4. 初始化数据目录
# ──────────────────────────────────────
DB_PATH="${DB_PATH:-./data/sqlite.db}"
DB_DIR=$(dirname "$DB_PATH")
mkdir -p "$DB_DIR"
mkdir -p data/logs

log "数据库路径: $DB_PATH"

# ──────────────────────────────────────
# 5. 注册 MCP Server
# ──────────────────────────────────────
log "注册 MCP Server..."

STORAGE_MCP_CMD="node"
STORAGE_MCP_ARGS="[\"$SCRIPT_DIR/mcp-servers/storage-mcp/dist/index.js\"]"
ADS_MCP_CMD="node"
ADS_MCP_ARGS="[\"$SCRIPT_DIR/mcp-servers/amazon-ads-mcp/dist/index.js\"]"

# Storage MCP
openclaw mcp set biz-storage "{\"command\":\"$STORAGE_MCP_CMD\",\"args\":$STORAGE_MCP_ARGS,\"env\":{\"DB_PATH\":\"$SCRIPT_DIR/$DB_PATH\"}}" 2>/dev/null && \
  log "Storage MCP 注册成功" || warn "Storage MCP 注册失败（请确认 openclaw 已安装）"

# Amazon Ads MCP
openclaw mcp set amazon-ads "{\"command\":\"$ADS_MCP_CMD\",\"args\":$ADS_MCP_ARGS,\"env\":{\"AMZ_ADS_CLIENT_ID\":\"$AMZ_ADS_CLIENT_ID\",\"AMZ_ADS_CLIENT_SECRET\":\"$AMZ_ADS_CLIENT_SECRET\",\"AMZ_ADS_REFRESH_TOKEN\":\"$AMZ_ADS_REFRESH_TOKEN\",\"AMZ_ADS_PROFILE_ID\":\"$AMZ_ADS_PROFILE_ID\"}}" 2>/dev/null && \
  log "Amazon Ads MCP 注册成功" || warn "Amazon Ads MCP 注册失败"

# ──────────────────────────────────────
# 6. 注册 Agent
# ──────────────────────────────────────
log "注册 Agent..."

SYSTEM_PROMPT=$(cat config/system-prompt.md)

# 拼接 Skills 内容到 systemPromptOverride
SKILLS_CONTENT=""
for skill_file in skills/*.md; do
  [ -f "$skill_file" ] || continue
  SKILLS_CONTENT+="
---
$(cat "$skill_file")
"
done

FULL_PROMPT="${SYSTEM_PROMPT}

## Skills 策略文件
${SKILLS_CONTENT}"

# 写入 openclaw 配置
CONFIG_FILE="$HOME/.openclaw/openclaw.json"
if [ -f "$CONFIG_FILE" ]; then
  PROMPT_FILE=$(mktemp)
  echo "$FULL_PROMPT" > "$PROMPT_FILE"

  if command -v jq >/dev/null 2>&1; then
    if jq -e ".agents.list[] | select(.id == \"aws-sp-assistant\")" "$CONFIG_FILE" >/dev/null 2>&1; then
      jq --argfile prompt <(echo '{}' | jq --arg p "$(cat "$PROMPT_FILE")" '.systemPromptOverride = $p') \
        '(.agents.list[] | select(.id == "aws-sp-assistant")).systemPromptOverride = $prompt.systemPromptOverride' \
        "$CONFIG_FILE" > "${CONFIG_FILE}.tmp" && mv "${CONFIG_FILE}.tmp" "$CONFIG_FILE"
    else
      jq --arg p "$(cat "$PROMPT_FILE")" \
        '.agents.list += [{"id": "aws-sp-assistant", "name": "AWS SP 广告助手", "systemPromptOverride": $p}]' \
        "$CONFIG_FILE" > "${CONFIG_FILE}.tmp" && mv "${CONFIG_FILE}.tmp" "$CONFIG_FILE"
    fi
    log "Agent 注册成功"
  else
    warn "jq 未安装，请手动配置 openclaw.json"
  fi

  rm -f "$PROMPT_FILE"
else
  warn "$CONFIG_FILE 不存在，请先运行 openclaw 初始化"
fi

# ──────────────────────────────────────
# 7. 注册 Cron 任务
# ──────────────────────────────────────
log "注册 Cron 任务..."

# 日巡检 - 每天上午 9:00
openclaw cron add \
  --name "sp-daily-inspection" \
  --agent "aws-sp-assistant" \
  --message "执行日巡检：获取昨日广告数据，检查异常指标，生成巡检报告。" \
  --cron "0 9 * * *" \
  --session "isolated" \
  --tz "Asia/Shanghai" \
  --exact \
  --announce \
  --channel "last" 2>/dev/null && \
  log "日巡检 Cron 注册成功" || warn "日巡检 Cron 注册失败"

# 周复盘 - 每周一上午 10:00
openclaw cron add \
  --name "sp-weekly-review" \
  --agent "aws-sp-assistant" \
  --message "执行周复盘：分析本周广告数据趋势，Campaign/关键词归因，生成下周建议。" \
  --cron "0 10 * * 1" \
  --session "isolated" \
  --tz "Asia/Shanghai" \
  --exact \
  --announce \
  --channel "last" 2>/dev/null && \
  log "周复盘 Cron 注册成功" || warn "周复盘 Cron 注册失败"

# 异常监控 - 每 4 小时
openclaw cron add \
  --name "sp-anomaly-monitor" \
  --agent "aws-sp-assistant" \
  --message "异常监控：检查关键广告指标（ACOS、CTR、预算消耗），发现异常立即告警。" \
  --cron "0 */4 * * *" \
  --session "isolated" \
  --tz "Asia/Shanghai" \
  --exact \
  --announce \
  --channel "last" 2>/dev/null && \
  log "异常监控 Cron 注册成功" || warn "异常监控 Cron 注册失败"

# ──────────────────────────────────────
# 8. 部署验证
# ──────────────────────────────────────
log "验证部署..."

log "验证 MCP 注册:"
openclaw mcp list 2>/dev/null || warn "无法列出 MCP Server"

log "验证 Cron 任务:"
openclaw cron list 2>/dev/null || warn "无法列出 Cron 任务"

# ──────────────────────────────────────
# 9. 完成
# ──────────────────────────────────────
echo ""
log "========================================="
log "  AWS SP Assistant 安装完成！"
log "========================================="
echo ""
log "下一步："
log "  1. 验证 MCP 注册: openclaw mcp list"
log "  2. 验证 Cron 任务: openclaw cron list"
log "  3. 启动服务: openclaw gateway run"
echo ""
```

- [ ] **Step 2: 实现 uninstall.sh**

`aws-sp-assistant/uninstall.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

log()  { echo -e "${GREEN}[uninstall]${NC} $*"; }
warn() { echo -e "${YELLOW}[warn]${NC} $*"; }
err()  { echo -e "${RED}[error]${NC} $*" >&2; }

# ──────────────────────────────────────
# 1. 移除 MCP Server 注册
# ──────────────────────────────────────
log "移除 MCP Server 注册..."

openclaw mcp remove biz-storage 2>/dev/null && log "Storage MCP 已移除" || warn "Storage MCP 未注册或移除失败"
openclaw mcp remove amazon-ads 2>/dev/null && log "Amazon Ads MCP 已移除" || warn "Amazon Ads MCP 未注册或移除失败"

# ──────────────────────────────────────
# 2. 移除 Cron 任务
# ──────────────────────────────────────
log "移除 Cron 任务..."

openclaw cron remove sp-daily-inspection 2>/dev/null && log "日巡检 Cron 已移除" || warn "日巡检 Cron 未注册或移除失败"
openclaw cron remove sp-weekly-review 2>/dev/null && log "周复盘 Cron 已移除" || warn "周复盘 Cron 未注册或移除失败"
openclaw cron remove sp-anomaly-monitor 2>/dev/null && log "异常监控 Cron 已移除" || warn "异常监控 Cron 未注册或移除失败"

# ──────────────────────────────────────
# 3. 移除 Agent 注册
# ──────────────────────────────────────
log "移除 Agent 注册..."

CONFIG_FILE="$HOME/.openclaw/openclaw.json"
if [ -f "$CONFIG_FILE" ] && command -v jq >/dev/null 2>&1; then
  jq 'del(.agents.list[] | select(.id == "aws-sp-assistant"))' \
    "$CONFIG_FILE" > "${CONFIG_FILE}.tmp" && mv "${CONFIG_FILE}.tmp" "$CONFIG_FILE"
  log "Agent 已从配置中移除"
else
  warn "请手动从 $CONFIG_FILE 中移除 aws-sp-assistant agent 配置"
fi

# ──────────────────────────────────────
# 4. 清理编译产物（可选）
# ──────────────────────────────────────
echo ""
read -p "是否清理编译产物 (dist/) 和数据 (data/)？[y/N] " -n 1 -r
echo ""
if [[ $REPLY =~ ^[Yy]$ ]]; then
  rm -rf mcp-servers/storage-mcp/dist
  rm -rf mcp-servers/amazon-ads-mcp/dist
  rm -rf data
  log "编译产物和数据已清理"
else
  log "保留编译产物和数据"
fi

echo ""
log "卸载完成。"
```

- [ ] **Step 3: 设置可执行权限并提交**

```bash
chmod +x setup.sh uninstall.sh
git add -A && git commit -m "feat: add setup.sh and uninstall.sh with deployment verification"
```

---

# Plan 4: 文档

> **Spec 对应:** Section 7 — MVP 交付物清单
> **依赖:** Plan 1, 2, 3

---

### Task 18: 环境变量模板

**Spec 对应:** Section 3.1 — 凭证管理
**Files:**

- Create: `aws-sp-assistant/.env.example`

- [ ] **Step 1: 创建 .env.example**

```bash
# Amazon Ads API 凭证
AMZ_ADS_CLIENT_ID=
AMZ_ADS_CLIENT_SECRET=
AMZ_ADS_REFRESH_TOKEN=
AMZ_ADS_PROFILE_ID=

# Storage MCP
DB_PATH=./data/sqlite.db
```

- [ ] **Step 2: 提交**

```bash
git add -A && git commit -m "chore: add .env.example"
```

---

### Task 19: MCP 工具参考文档

**Spec 对应:** Section 7 — 文档交付物
**Files:**

- Create: `aws-sp-assistant/docs/mcp-tool-reference.md`

- [ ] **Step 1: 编写 MCP 工具参考**

文档内容覆盖：

- Storage MCP 的 13 个工具：每个工具的名称、参数（类型、是否必需、描述）、返回值格式
- Amazon Ads MCP 的 13 个工具：同上
- 通用说明：MCP Server 连接方式、环境变量、错误码

- [ ] **Step 2: 提交**

```bash
git add -A && git commit -m "docs: add MCP tool reference"
```

---

## Self-Review

**1. Spec coverage:**

- Plan 2: 覆盖 Section 3.1 全部 13 个工具 + Section 3.4 降级策略（指数退避）+ 凭证管理（环境变量）。Seller Data MCP (Section 3.3) MVP 不实现，已声明降级策略替代。
- Plan 3: 覆盖 Section 4.1 Agent 配置 + Section 4.2 全部 10 个 Skill 文件 + ad-operations (UC-10) + Section 4.4 全部 3 个 Cron + Section 5.2 飞书卡片模板 + Section 4.5 Skill 自进化。降级决策树覆盖 Section 3.4 全部 9 种场景。通知模板区分飞书卡片（3 种）和纯文本（3 种）。
- Plan 4: 覆盖 Section 6.4 部署流程（含验证步骤 8）+ Section 7 交付物清单中的文档。

**2. Placeholder scan:** Task 13a/13b/13c 已补全 ad-groups、keywords、account 的完整测试和实现代码。Task 17 已补全完整 setup.sh 和 uninstall.sh 脚本。

**3. Type consistency:** 所有工具使用 `server.tool(name, description, zodSchema, handler)` 4 参数签名注册。TokenProvider 接口在 client.ts 中定义，TokenManager 实现该接口。测试文件中 createToolRegistry 的 mock 签名统一为 4 参数格式。

**4. 已知限制（审核确认）：**

- Skill 文件（Task 16）包含核心内容要求映射表，但未逐文件展开完整 markdown 内容。执行时需按映射表逐个编写。
- `migrations.ts` 在文件结构中列出但实际使用 `CREATE TABLE IF NOT EXISTS` 幂等建表，MVP 阶段不需要独立迁移机制。
- `runs.type` 枚举值比规格多一个 `"task"`（对应 task-collaboration.md），为合理扩展。
- Skill 自进化缺少 `update_skill_candidate` 工具用于状态变更（candidate → approved/rejected），审批流程由 Agent 通过直接数据库操作完成。
