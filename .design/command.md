# OpenClaw 常用命令参考

> 📚 **使用提示**: 新用户建议先运行 `pnpm openclaw onboard` 完成初始配置。

---

## 🧙 向导和设置（最重要）

### Setup - 初始化配置向导

```bash
# 基本初始化
pnpm openclaw setup

# 指定工作空间
pnpm openclaw setup --workspace ~/my-workspace

# 运行交互式入门向导
pnpm openclaw setup --wizard

# 非交互模式
pnpm openclaw setup --non-interactive

# 指定模式（local/remote）
pnpm openclaw setup --mode local
pnpm openclaw setup --mode remote --remote-url ws://localhost:18789
```

### Onboard - 交互式入门向导（强烈推荐）

```bash
# 运行完整入门向导
pnpm openclaw onboard

# 快速入门流程
pnpm openclaw onboard --flow quickstart

# 高级配置流程
pnpm openclaw onboard --flow advanced

# 手动配置流程
pnpm openclaw onboard --flow manual

# 重置配置后重新入门
pnpm openclaw onboard --reset
pnpm openclaw onboard --reset --reset-scope config+creds+sessions

# 指定认证方式
pnpm openclaw onboard --auth-choice anthropic
pnpm openclaw onboard --auth-choice openai
pnpm openclaw onboard --auth-choice browser

# 使用 token 认证
pnpm openclaw onboard --auth-choice token --token-provider anthropic --token "sk-ant-..."

# Gateway 配置
pnpm openclaw onboard --gateway-port 18789
pnpm openclaw onboard --gateway-bind loopback
pnpm openclaw onboard --gateway-auth token --gateway-token "your-token"

# 跳过特定步骤
pnpm openclaw onboard --skip-channels    # 跳过渠道设置
pnpm openclaw onboard --skip-skills      # 跳过技能设置
pnpm openclaw onboard --skip-search      # 跳过搜索设置
pnpm openclaw onboard --skip-health      # 跳过健康检查

# 远程 Gateway
pnpm openclaw onboard --mode remote --remote-url ws://remote-host:18789 --remote-token "token"

# 输出 JSON 摘要
pnpm openclaw onboard --json
```

### Configure - 配置向导

```bash
# 运行配置向导
pnpm openclaw configure

# 指定配置节
pnpm openclaw configure --section credentials
pnpm openclaw configure --section channels
pnpm openclaw configure --section gateway
pnpm openclaw configure --section agents

# 可用配置节：credentials, channels, gateway, agents, skills, search
```

---

## 核心配置管理

### 查看配置

```bash
# 列出所有配置
pnpm openclaw config list

# 查看特定配置项
pnpm openclaw config get <key>

# 常用配置项
pnpm openclaw config get model
pnpm openclaw config get models
pnpm openclaw config get gateway.mode
```

### 设置配置

```bash
# 设置基本配置
pnpm openclaw config set <key> <value>

# 模型配置
pnpm openclaw config set model claude-sonnet-4.6
pnpm openclaw config set model claude-opus-4.6
pnpm openclaw config set model gpt-4o

# Gateway 模式
pnpm openclaw config set gateway.mode local
pnpm openclaw config set gateway.mode remote
```

### 删除配置

```bash
pnpm openclaw config delete <key>
```

---

## 渠道管理

### 查看渠道状态

```bash
# 查看所有渠道状态
pnpm openclaw channels status

# 探测渠道连接状态
pnpm openclaw channels status --probe

# 查看特定渠道
pnpm openclaw channels status --channel telegram
```

### 渠道配对

```bash
# 开始配对流程
pnpm openclaw channels pair

# 特定渠道配对
pnpm openclaw channels pair --channel discord
pnpm openclaw channels pair --channel whatsapp
```

### 渠道操作

```bash
# 发送测试消息
pnpm openclaw channels send

# 同步渠道数据
pnpm openclaw channels sync
```

---

## Gateway 管理

### Gateway 运行

```bash
# 启动 gateway
pnpm openclaw gateway run

# 后台运行
pnpm openclaw gateway run --background

# 指定端口和绑定
pnpm openclaw gateway run --bind loopback --port 18789

# 强制启动
pnpm openclaw gateway run --force
```

### Gateway 控制

```bash
# 停止 gateway
pnpm openclaw gateway stop

# 重启 gateway
pnpm openclaw gateway restart

# 查看 gateway 状态
pnpm openclaw gateway status
```

---

## Agent 管理

### Agent 列表

```bash
# 列出所有 agent
pnpm openclaw agents list

# 查看 agent 详情
pnpm openclaw agents info <agent-id>
```

### Agent 操作

```bash
# 发送消息给 agent
pnpm openclaw agent --message "your message"

# 指定 agent
pnpm openclaw agent --agent <agent-id> --message "your message"

# 设置思考级别
pnpm openclaw agent --message "your message" --thinking high
```

---

## 消息管理

### 发送消息

```bash
# 基本发送
pnpm openclaw message send "your message"

# 指定渠道
pnpm openclaw message send --channel telegram "your message"

# 指定对话
pnpm openclaw message send --conversation <conversation-id> "your message"
```

### 消息历史

```bash
# 查看历史消息
pnpm openclaw messages history

# 搜索消息
pnpm openclaw messages search <query>
```

---

## 插件管理

### 插件列表

```bash
# 列出所有插件
pnpm openclaw plugins list

# 查看插件详情
pnpm openclaw plugins info <plugin-id>
```

### 插件操作

```bash
# 安装插件
pnpm openclaw plugins install <plugin-spec>

# 卸载插件
pnpm openclaw plugins uninstall <plugin-id>

# 更新插件
pnpm openclaw plugins update <plugin-id>

# 启用/禁用插件
pnpm openclaw plugins enable <plugin-id>
pnpm openclaw plugins disable <plugin-id>
```

### 插件开发

```bash
# 验证插件
pnpm openclaw plugins validate <plugin-path>

# 打包插件
pnpm openclaw plugins package <plugin-path>
```

---

## 故障排除

### Doctor 命令

```bash
# 运行诊断
pnpm openclaw doctor

# 自动修复问题
pnpm openclaw doctor --fix

# 详细诊断
pnpm openclaw doctor --verbose
```

### 清理和重置

```bash
# 清理缓存
pnpm clean

# 重新安装依赖
rm -rf node_modules pnpm-lock.yaml
pnpm install

# 重置配置（谨慎使用）
pnpm openclaw config reset
```

---

## 开发和调试

### 开发模式

```bash
# 启动开发服务器
pnpm dev

# 运行 CLI 开发模式
pnpm openclaw --dev
```

### 日志查看

```bash
# 查看 gateway 日志
tail -f /tmp/openclaw-gateway.log

# 实时跟踪日志
./scripts/clawlog.sh follow

# 按类别过滤
./scripts/clawlog.sh --category gateway
./scripts/clawlog.sh --category agent
```

### 测试

```bash
# 运行所有测试
pnpm test

# 运行特定测试
pnpm test src/commands/config.test.ts

# 测试覆盖率
pnpm test:coverage

# Live 测试（需要真实 API 密钥）
OPENCLAW_LIVE_TEST=1 pnpm test:live
```

### 构建

```bash
# 构建项目
pnpm build

# TypeScript 类型检查
pnpm tsgo

# 代码检查和格式化
pnpm check          # 检查代码质量
pnpm format:fix     # 格式化代码
```

---

## macOS 特定命令

### 查看日志

```bash
# 统一日志查询
./scripts/clawlog.sh

# 实时跟踪
./scripts/clawlog.sh follow
```

---

## Docker 相关

### Docker 测试

```bash
# Live 模型测试
pnpm test:docker:live-models

# Gateway 测试
pnpm test:docker:live-gateway

# Onboarding E2E 测试
pnpm test:docker:onboard
```

---

## 快捷操作

### 验证 Gateway 运行

```bash
# 检查端口
ss -ltnp | rg 18789

# 检查进程
pgrep -fl openclaw-gateway

# 检查日志
tail -n 120 /tmp/openclaw-gateway.log
```

### Git 工作流

```bash
# 安装 pre-commit hooks
prek install

# 快速提交（跳过完整检查）
FAST_COMMIT=1 git commit ...

# 查看导入循环
pnpm check:import-cycles
pnpm check:madge-import-cycles
```

### 配置和元数据

```bash
# 生成配置文档
pnpm config:docs:gen
pnpm config:docs:check

# 生成 Plugin SDK API 文档
pnpm plugin-sdk:api:gen
pnpm plugin-sdk:api:check
```

---

## 环境变量

### 常用环境变量

```bash
# 本地检查模式
export OPENCLAW_LOCAL_CHECK=1
export OPENCLAW_LOCAL_CHECK_MODE=throttled  # 或 full

# Live 测试
export OPENCLAW_LIVE_TEST=1
export OPENCLAW_LIVE_TEST_QUIET=0  # 详细日志

# Vitest 测试工作线程
export OPENCLAW_VITEST_MAX_WORKERS=1
export OPENCLAW_VITEST_POOL=forks
```

---

## 注意事项

1. **Gateway 重启**: 推荐使用 `pnpm openclaw gateway restart`
2. **配置更改**: 更改模型或配置后需要重启 gateway 才能生效
3. **日志位置**: Gateway 日志默认在 `/tmp/openclaw-gateway.log`
4. **会话数据**: Pi 会话在 `~/.openclaw/sessions/`
5. **凭证存储**: Web provider 凭证在 `~/.openclaw/credentials/`
