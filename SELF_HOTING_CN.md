# 自托管指南

在你自己的基础设施上几分钟内部署 Multica。

## 架构

| 组件 | 描述 | 技术栈 |
|-----------|-------------|------------|
| **后端** | REST API + WebSocket 服务器 | Go（单个二进制文件） |
| **前端** | Web 应用 | Next.js 16 |
| **数据库** | 主数据存储 | PostgreSQL 17 + pgvector |

每个在本地运行 AI agent 的用户还需要在自己的机器上安装 **`multica` CLI** 并运行 **agent daemon**。

## 快速安装（推荐）

只需两条命令即可完成所有配置 — 服务器、CLI 和配置：

```bash
# 1. 安装 CLI 并部署自托管服务器
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --with-server

# 2. 配置 CLI、认证并启动 daemon
multica setup self-host
```

此命令会安装 `multica` CLI，检出最新的自托管资源，从 GHCR 拉取 Multica 官方镜像，并完成 localhost 的所有配置。

打开 http://localhost:3000。要登录，请在 `.env` 中配置 `RESEND_API_KEY` 以使用邮件验证码（推荐方式），或者不设置 Resend 并从后端日志中复制生成的验证码。详见 [步骤 2 — 登录](#步骤-2--登录)。

> **前置条件：** 必须已安装 Docker 和 Docker Compose。脚本会检查并提供缺失依赖的安装链接。
>
> **只需要 CLI？** 如果自托管服务器已经在运行，你只需要在 macOS/Linux 机器上安装 CLI，可使用 Homebrew：
>
> ```bash
> brew install multica-ai/tap/multica
> ```

---

## 分步设置（可选）

如果你更喜欢手动逐步执行：

### 步骤 1 — 启动服务器

**前置条件：** Docker 和 Docker Compose。

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
make selfhost
```

`make selfhost` 会自动从示例创建 `.env`，生成随机的 `JWT_SECRET`，并通过 Docker Compose 启动所有服务。

默认情况下，它会从 GHCR 拉取最新稳定版的发布镜像。若想用当前检出代码构建 backend/web，请运行 `make selfhost-build`。
如果所选的 GHCR tag 尚未发布，`make selfhost` 现在会提示你回退到 `make selfhost-build`。
`make selfhost-build` 使用本地的 `multica-backend:dev` / `multica-web:dev` tag，因此不会覆盖已拉取的 `:latest` 镜像。

就绪后：

- **前端：** http://localhost:3000
- **后端 API：** http://localhost:8080

> **注意：** 如果你倾向于手动执行 Docker Compose 步骤，请参考下方的 [手动 Docker Compose 设置](#手动-docker-compose-设置)。

### 步骤 2 — 登录

在浏览器中打开 http://localhost:3000。Docker 自托管 stack 默认 `APP_ENV=production`（在 `docker-compose.selfhost.yml` 中设置），且默认没有固定的验证码。请选择以下任一方式登录：

- **推荐（生产环境）：** 在 `.env` 中配置 `RESEND_API_KEY`，然后重启后端。真实的验证码会发送到你输入的邮箱。详见 [高级配置 → Email](SELF_HOSTING_ADVANCED.md#email-required-for-authentication)。
- **未配置邮件的情况：** 验证码由服务器端生成并打印到后端容器日志（查找 `[DEV] Verification code for ...:`）。适用于单机一次性测试。
- **确定性的本地/私有测试：** 在 `.env` 中设置 `APP_ENV=development` 和 `MULTICA_DEV_VERIFICATION_CODE=888888`，然后重启后端。当 `APP_ENV=production` 时，该固定验证码会被忽略。

修改 `ALLOW_SIGNUP` 和 `GOOGLE_CLIENT_ID` 同样需要重启后端 / compose stack 后才会生效。Web UI 在运行时通过 `/api/config` 读取这两个配置，因此无需重新构建 web。

> **警告：** **不要** 在公网可访问的实例上设置 `MULTICA_DEV_VERIFICATION_CODE` — 任何人只要知道一个邮箱地址就能用该固定验证码登录。

### 步骤 3 — 安装 CLI 并启动 Daemon

Daemon 运行在你的本地机器上（不在 Docker 内）。它会检测已安装的 AI agent CLI，将其注册到服务器，并在 agent 被分配任务时执行任务。

每个想在本地运行 AI agent 的团队成员需要：

### a) 安装 CLI 和 AI agent

```bash
brew install multica-ai/tap/multica
```

你还需要至少安装一个 AI agent CLI：
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)（`claude` 在 PATH 中）
- [Codex](https://github.com/openai/codex)（`codex` 在 PATH 中）
- [GitHub Copilot CLI](https://docs.github.com/en/copilot)（`copilot` 在 PATH 中）
- [OpenClaw](https://github.com/openclaw/openclaw)（`openclaw` 在 PATH 中）
- [OpenCode](https://github.com/anomalyco/opencode)（`opencode` 在 PATH 中）
- [Hermes](https://github.com/NousResearch/hermes)（`hermes` 在 PATH 中）
- Gemini（`gemini` 在 PATH 中）
- [Pi](https://pi.dev/)（`pi` 在 PATH 中）
- [Cursor Agent](https://cursor.com/)（`cursor-agent` 在 PATH 中）
- Kimi（`kimi` 在 PATH 中）
- Kiro CLI（`kiro-cli` 在 PATH 中）

### b) 一键设置

```bash
multica setup self-host
```

此命令会自动：
1. 将 CLI 配置为连接到 `localhost`（端口 8080/3000）
2. 打开浏览器进行认证
3. 发现你的 workspaces
4. 在后台启动 daemon

对于使用自定义域名的本地部署：

```bash
multica setup self-host --server-url https://api.example.com --app-url https://app.example.com
```

要验证 daemon 是否在运行：

```bash
multica daemon status
```

> **替代方式：** 如果你更喜欢手动操作，请参考下方的 [手动 CLI 配置](#手动-cli-配置)。

### 步骤 4 — 验证并开始使用

1. 在 http://localhost:3000 打开你的 workspace
2. 进入 **Settings → Runtimes** — 你应该能看到自己的机器列表
3. 进入 **Settings → Agents** 并创建一个新 agent
4. 创建一个 issue 并分配给你的 agent — 它将自动接管任务

## 停止服务

如果你通过安装脚本安装：

```bash
curl -fsSL https://raw.githubusercontent.com/multica-ai/multica/main/scripts/install.sh | bash -s -- --stop
```

如果你是手动克隆仓库：

```bash
# 停止 Docker Compose 服务（后端、前端、数据库）
make selfhost-stop

# 停止本地 daemon
multica daemon stop
```

## 切换到 Multica Cloud

如果你一直在自托管，并想将 CLI 切换到 [Multica Cloud](https://multica.ai)：

```bash
multica setup
```

此命令会将 CLI 重新配置为指向 multica.ai，重新认证并重启 daemon。在覆盖已有配置前会提示你确认。

> 本地的 Docker 服务不受影响。如果你不再需要它们，请单独停止。

## 升级

```bash
docker compose -f docker-compose.selfhost.yml pull
docker compose -f docker-compose.selfhost.yml up -d
```

如果你想固定在某个特定版本，可以在 `.env` 中将 `MULTICA_IMAGE_TAG` 设为精确版本号（例如 `v0.2.4`）。后端启动时会自动运行数据库迁移。
如果所选的 GHCR tag 尚未发布，请回退到 `make selfhost-build` 或 `docker compose -f docker-compose.selfhost.yml -f docker-compose.selfhost.build.yml up -d --build`。

---

## 手动 Docker Compose 设置

如果你倾向于手动运行 Docker Compose 步骤而非 `make selfhost`：

```bash
git clone https://github.com/multica-ai/multica.git
cd multica
cp .env.example .env
```

编辑 `.env` — 至少需要修改 `JWT_SECRET`：

```bash
JWT_SECRET=$(openssl rand -hex 32)
```

然后启动所有服务：

```bash
docker compose -f docker-compose.selfhost.yml pull
docker compose -f docker-compose.selfhost.yml up -d
```

## 手动 CLI 配置

如果你倾向于逐步配置 CLI 而非使用 `multica setup`：

```bash
# 将 CLI 指向你的本地服务器
multica config set server_url http://localhost:8080
multica config set app_url http://localhost:3000

# 登录（会打开浏览器）
multica login

# 启动 daemon
multica daemon start
```

对于使用 TLS 的生产部署：

```bash
multica config set app_url https://app.example.com
multica config set server_url https://api.example.com
multica login
multica daemon start
```

## 高级配置

环境变量、手动设置（不使用 Docker）、反向代理配置、数据库设置等更多内容，请参考 [高级配置指南](SELF_HOSTING_ADVANCED.md)。
