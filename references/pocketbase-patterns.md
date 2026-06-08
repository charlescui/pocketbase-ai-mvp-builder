# PocketBase 实战模式参考

## 目录

- [能力地图](#能力地图)
- [macOS 本地环境](#macos-本地环境)
- [项目结构](#项目结构)
- [数据建模](#数据建模)
- [API Rules](#api-rules)
- [手机号短信认证](#手机号短信认证)
- [Realtime 前后端同步](#realtime-前后端同步)
- [文件和 S3 兼容存储](#文件和-s3-兼容存储)
- [Go 扩展点](#go-扩展点)
- [PocketBase 定时任务](#pocketbase-定时任务)
- [GitHub CLI 和版本习惯](#github-cli-和版本习惯)
- [部署和备份](#部署和备份)
- [数据库备份策略](#数据库备份策略)
- [日志和日常运维](#日志和日常运维)
- [首次 superuser 和上线提醒](#首次-superuser-和上线提醒)
- [验收清单](#验收清单)

## 能力地图

PocketBase 适合小型 MVP、内部系统、设计/运营/测试团队工具、低到中等访问量产品原型。它的关键价值是把很多中后台基础能力放进一个轻量系统：

- SQLite 数据存储和 collections。
- 内置 Admin UI，用于管理 collections、数据、用户、文件、设置。
- Auth collections，用于普通用户、员工、客户等登录身份。
- 手机号短信认证，用于注册、登录、账号绑定和强身份表单。
- API rules，用声明式规则控制 list/view/create/update/delete。
- 自动 REST API 和 SDK 访问。
- Realtime subscriptions，用于前后端即时同步。
- File fields 和本地文件存储，也可接 S3-compatible object storage。
- Go 扩展能力：hooks、custom routes、migrations、commands、cron-like jobs、事件处理。
- 内置 jobs scheduling：用 `app.Cron()` 开发应用自己的定时任务。
- 单文件二进制部署，资源占用低，适合 VPS 和小团队。

不要把 PocketBase 当成无限扩展的大型分布式后端。遇到高并发、多区域、多写节点、复杂数据仓库、强审计合规、复杂队列工作流时，要明确边界并建议拆分或换架构。

## macOS 本地环境

基础安装：

```bash
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew update
brew install go node git gh rsync
```

国内网络常用配置：

```bash
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOSUMDB=sum.golang.google.cn
npm config set registry https://registry.npmmirror.com
```

验证：

```bash
go version
go env GOPROXY GOSUMDB
node -v
npm -v
git --version
gh --version
gh auth login -p ssh -w
gh auth status
```

如果 GitHub 仍然慢，优先使用用户自己的 VPN、公司代理、学校代理或可信镜像。不要默认写入第三方 GitHub `insteadOf` 规则，因为这会影响全局 git 行为并可能带来供应链风险。

## 项目结构

推荐结构：

```text
my-pocketbase-app/
  backend/
    go.mod
    go.sum
    main.go
    pb_migrations/
    internal/
      hooks/
      services/
  frontend/
    package.json
    src/
      lib/pocketbase.ts
      routes/
      components/
  README.md
```

后端初始化：

```bash
mkdir -p my-pocketbase-app/backend
cd my-pocketbase-app/backend
go mod init example.com/my-pocketbase-app
go get github.com/pocketbase/pocketbase@latest
```

前端初始化示例：

```bash
cd ..
npm create vite@latest frontend -- --template react-ts
cd frontend
npm install
npm install pocketbase
```

本地启动常见端口：

- PocketBase: `http://127.0.0.1:8090`
- Admin UI: `http://127.0.0.1:8090/_/`
- Frontend: Vite 默认 `http://127.0.0.1:5173`

## 数据建模

从角色和流程倒推 collections：

- `users`: auth collection。普通用户身份、昵称、头像、角色字段等。
- `organizations`: 多团队/部门场景。
- `memberships`: 用户与组织关系，包含 role/status。
- 业务 collection: 例如 `projects`、`tasks`、`tickets`、`events`、`registrations`。
- `comments`: 需要实时讨论时拆成独立 collection。
- `notifications`: 需要未读、提醒、操作反馈时使用。
- `audit_logs`: 管理员操作或状态变化需要追踪时使用。

字段建议：

- 所有权：`owner` relation -> `users`。
- 手机号身份：在 auth collection 中保存 normalized phone、verified state 和 phone login flags。
- 组织隔离：`organization` relation -> `organizations`。
- 工作流：`status` select，比如 `draft/submitted/approved/rejected/done`。
- 指派：`assignee` relation -> `users`。
- 文件：`attachments` file field，可多文件。
- 搜索：对常用筛选字段建立清晰字段，不要只靠 JSON。

迁移原则：

- 探索阶段可以在 Admin UI 点 collection。
- 一旦进入可交付项目，把 schema 写成 migration。
- 在迁移里设置字段、索引、API rules 和初始必要配置。
- 迁移文件要能从空 `pb_data` 重建系统结构。

## API Rules

规则思维：

- Public access: rule 是空字符串，谨慎使用。
- No client access: rule 为 null/locked，适合只允许服务端 hook 操作。
- Authenticated: `@request.auth.id != ""`。
- Owner-only: `owner = @request.auth.id`。
- Admin role: `@request.auth.role = "admin"`，前提是 `role` 是服务端可信字段。
- Related users: relation 多选常用 `allowed_users.id ?= @request.auth.id` 这类表达式。
- Request body guard: 防止客户端改敏感字段，例如拒绝 `role`、`owner`、`approved_by` 被普通用户提交。

常见规则草案：

```text
# 只有登录用户能看列表
@request.auth.id != ""

# 只能看自己的记录
owner = @request.auth.id

# 用户看自己的，管理员看全部
owner = @request.auth.id || @request.auth.role = "admin"

# 创建时必须登录，且 owner 必须是自己
@request.auth.id != "" && @request.body.owner = @request.auth.id

# 普通用户不能更新已审批记录
owner = @request.auth.id && status != "approved"
```

每次实现后至少测试：

- 未登录请求是否被拒绝。
- 登录用户是否只能访问自己的数据。
- 非拥有者是否无法查看/修改。
- 管理员/运营是否有预期权限。
- 前端隐藏按钮之外，直接调 API 是否也被拒绝。

## 手机号短信认证

真实用户注册和登录时，默认优先考虑手机号作为基础身份能力。Agent 遇到“手机号注册、短信登录、绑定手机号、强身份表单、找回密码、敏感操作验证”时，应读取 `phone-sms-auth-aliyun.md`，并使用 PocketBase Go 后端集成阿里云号码认证服务。

必须遵守：

- 短信发送和验证码校验只在服务端做。
- 默认使用阿里云生成并核验验证码：`SendSmsVerifyCode` 的模板参数使用动态占位，例如 `{"code":"##code##","min":"5"}`，并设置 `CodeType=1`、`CodeLength=6`、`ValidTime=300`、`Interval=60` 等生成和频控参数。
- 用户输入验证码后，服务端再使用阿里云 `CheckSmsVerifyCode` 校验验证码。
- 当前阿里云文档中，校验成功要确认 `Model.VerifyResult = PASS`，不能只看 HTTP 成功或 `Code=OK`。
- PocketBase 不需要自己生成或保存验证码；只保存 challenge、attempt、purpose、phone hash、阿里云 request id 等审计信息。
- 如果项目明确改成 PocketBase 自己生成验证码，则阿里云只是短信发送通道，PocketBase 必须自己实现验证码哈希存储、TTL、错误次数、频控、消费和审计。
- AccessKey 不能进前端、不能进 GitHub。
- 验证码不能明文入库或写日志。
- 生产环境不要启用验证码返回到响应、数据库或日志。
- 客户端不能直接设置 `phone_verified`、`phone_verified_at`、`phone_verification_id`。
- 错误文案不能暴露“手机号是否已注册”，防止枚举账号。

推荐数据模型：

- `users` 增加 `phone_e164`、`phone_national`、`phone_verified`、`phone_verified_at`、`phone_login_enabled`、`password_login_enabled`。
- 新增 locked/server-only collection：`sms_challenges`，记录 purpose、phone hash、out id、阿里云 request id、IP hash、attempts、expires_at、verified_at、status。

推荐后端接口：

```text
POST /api/phone/request-code
POST /api/phone/verify-code
POST /api/auth/phone/register
POST /api/auth/phone/login
POST /api/account/phone/bind
POST /api/forms/:collection/:id/verify-phone
```

推荐流程：

- 注册：手机号 -> 发送验证码 -> 校验验证码 -> 创建 user -> 标记 phone_verified -> 签发 auth token 或引导设置密码。
- 登录：手机号 -> 发送验证码 -> 校验验证码 -> 查找 user -> 签发 auth token。
- 手机号 + 密码：只有 `phone_verified = true` 且 `password_login_enabled = true` 才允许。
- 表单验证：表单手机号必须经过 `form_verify` purpose 的短信校验，后端写入 verified 标记。

测试清单：

- 正常注册、正常登录、重复注册、错误验证码、过期验证码、已消费验证码。
- 同手机号和同 IP 频控。
- 绑定/更换手机号。
- 强身份表单未验证不能提交。
- API rules 不能直接写 verified 字段。
- 日志里没有完整手机号、验证码、AccessKey。

## Realtime 前后端同步

核心原则：先 API 拉取初始数据，再订阅变更，再把事件合并到本地状态。

JS SDK 基础模式：

```ts
import { pb } from "./pocketbase";

export async function watchTasks(onChange: (records: any[]) => void) {
  let records = await pb.collection("tasks").getFullList({
    sort: "-updated",
  });

  onChange(records);

  const unsubscribe = await pb.collection("tasks").subscribe("*", (event) => {
    if (event.action === "create") {
      records = [event.record, ...records];
    }

    if (event.action === "update") {
      records = records.map((item) =>
        item.id === event.record.id ? event.record : item,
      );
    }

    if (event.action === "delete") {
      records = records.filter((item) => item.id !== event.record.id);
    }

    onChange(records);
  });

  return unsubscribe;
}
```

React 组件中要在 cleanup 里退订：

```ts
useEffect(() => {
  let unsubscribe: (() => void) | undefined;

  watchTasks(setTasks).then((fn) => {
    unsubscribe = fn;
  });

  return () => {
    unsubscribe?.();
  };
}, []);
```

Realtime 验收：

- 两个浏览器/标签页同时打开同一列表。
- A 新增记录，B 不刷新能看到。
- A 改状态，B 不刷新能看到状态变化。
- A 删除记录，B 不刷新能消失。
- 非授权用户收不到不该看到的数据。
- 登录状态变化后重新订阅，避免旧权限残留。

产品上优先使用 realtime 的场景：

- 看板状态、审核状态、排队状态、任务分配。
- 评论、消息、通知。
- 操作员后台、客服后台、测试任务台。
- 文件处理进度、导入进度、审批结果。

## 文件和 S3 兼容存储

本地 demo 默认用 PocketBase 本地文件存储，文件在 `pb_data` 下。生产部署如果容器会重建、磁盘不可靠、需要 CDN 或多实例读取，使用 S3-compatible object storage。

实现建议：

- 用 file field 管理上传，不要自己拼路径。
- 公开文件可以直接展示 URL；私有文件要遵守 view rule，并使用 PocketBase 的受保护文件访问方式。
- 上传前端用 SDK 或 `FormData`。
- 大文件、敏感文件、长期归档文件，要在需求阶段确认大小、保留周期和权限。

S3 兼容配置要点：

- endpoint
- region
- bucket
- access key id
- secret access key
- force path style 或 provider-specific 选项
- public base URL/CDN URL
- CORS

阿里云 OSS 场景：

- 使用官方 S3 兼容访问方式，endpoint 形如 `https://s3.oss-cn-hangzhou.aliyuncs.com`。
- bucket region 要与 endpoint region 匹配，例如杭州 region 使用 `oss-cn-hangzhou`。
- 使用 RAM 用户 AccessKey，并给最小 bucket 权限。
- AWS SDK v2 兼容场景通常需要 path-style 访问；如果 PocketBase 存储设置里有 force path style 选项，优先按阿里云官方 S3 兼容要求配置。
- 遇到 checksum 或签名兼容错误时，按阿里云官方文档调整 checksum validation/request checksum calculation 等 SDK 配置；不要盲目换 endpoint。
- 普通 OSS endpoint 与 S3-compatible endpoint/签名规则可能不同，按阿里云当前文档核对。
- 先用一个测试 collection 上传小文件，确认 PocketBase 能写入、读取、删除。

备份提醒：

- 只备份 SQLite 数据库不够，文件对象也要备份。
- 只备份对象存储不够，PocketBase 记录里的文件元数据也要备份。
- 迁移文件负责结构，`pb_data`/数据库和对象存储负责数据。

## Go 扩展点

只有需要服务端能力时才写 Go，避免把普通 CRUD 复杂化。适合写 Go 的场景：

- 自定义业务 API。
- 写入前后校验、自动生成编号、状态机。
- 审批、通知、审计日志。
- 定时任务、同步外部系统。
- 服务端聚合查询和导出。
- 自定义命令、部署健康检查。

最小 `main.go`：

```go
package main

import (
	"log"

	"github.com/pocketbase/pocketbase"
)

func main() {
	app := pocketbase.New()

	if err := app.Start(); err != nil {
		log.Fatal(err)
	}
}
```

自定义 route、hook、migration 的具体 API 要以当前官方 Go docs 为准。PocketBase Go API 会演进，遇到编译错误先查对应版本 docs 和 examples，不要凭旧代码硬改。

## PocketBase 定时任务

PocketBase 已经具备定时任务能力。Agent 在用户提出“每天/每小时/定期/自动清理/自动同步/自动提醒/定时报表/过期处理/自动备份检查”时，优先考虑 PocketBase 的 `app.Cron()`，不要一上来让用户学 Linux cron。

适合做成 PocketBase 定时任务的场景：

- 每天生成运营报表或发送摘要。
- 每小时同步外部系统数据。
- 定期清理过期草稿、临时文件、无效 token 或过期通知。
- 定时扫描超时任务并变更状态。
- 定时发送提醒、催办、日报、周报。
- 定时检查备份是否成功生成。
- 定期重算小规模统计缓存。

Go 扩展里注册任务：

```go
// main.go
package main

import (
	"log"

	"github.com/pocketbase/pocketbase"
)

func main() {
	app := pocketbase.New()

	app.Cron().MustAdd("daily-summary", "0 9 * * *", func() {
		app.Logger().Info("daily-summary job started")
		// TODO: query records, generate report, send notifications.
	})

	if err := app.Start(); err != nil {
		log.Fatal(err)
	}
}
```

使用规则：

- `app.Cron()` 会在 `serve` 时自动启动。
- 用 `app.Cron().Add(id, cronExpr, handler)` 或 `MustAdd` 注册任务。
- job id 要稳定、可读，例如 `daily-summary`、`expire-tasks`、`backup-check`。
- cron 表达式示例：`*/5 * * * *` 每 5 分钟，`0 * * * *` 每小时，`0 9 * * *` 每天 9 点。
- 每个任务要写开始、成功、失败日志，使用 `app.Logger()`。
- 任务失败不要让整个服务崩溃；捕获错误并记录。
- 长任务要防重入，避免上一次没跑完下一次又开始。
- 涉及外部 API 的任务要设置超时、重试上限和错误日志。
- 定时任务可以在 `Dashboard > Settings > Crons` 里预览和手动触发。
- 不要随意 `RemoveAll()` 或 `Stop()` app 级 cron。PocketBase 自己也用它运行系统任务，例如日志清理和自动备份，系统 job id 通常是 `__pb*__` 形式。

验收方式：

- 临时把 cron 表达式改成每 1-2 分钟运行一次，在本地和服务器各验证一次。
- 在 `Dashboard > Settings > Crons` 手动触发一次。
- 查看 `Dashboard > Logs` 和 `journalctl -u pbapp` 确认任务日志。
- 验证任务结果是否真的写入记录、发送通知或完成清理。
- 验收后改回生产周期并提交代码。

## GitHub CLI 和版本习惯

非工程同学通常没有“什么时候保存一个稳定版本”的直觉，所以 agent 要主动维护仓库：

- 本机安装并登录 `gh`。
- 新项目用 `gh repo create` 创建私有仓库并推送。
- `main` 保持可运行；新功能用 `feat/...` 分支，修复用 `fix/...` 分支。
- 每个可运行里程碑 commit 一次：脚手架、迁移、权限规则、realtime、文件/OSS、部署配置、远程 smoke test。
- 每次 commit 后 push 到远程。
- 上线成功后打 tag，例如 `v0.1.0`，必要时创建 GitHub Release。
- 永远不要提交 `.env`、`pb_data`、数据库文件、AccessKey、SSH 私钥、superuser 密码。
- 为非工程同学创建常用脚本：`scripts/build.sh`、`scripts/deploy.sh`、`scripts/logs.sh`、`scripts/rollback.sh`、`scripts/backup.sh`。

常用命令：

```bash
git checkout -b feat/realtime-workflow
git add backend frontend README.md
git commit -m "feat: add realtime workflow"
git push -u origin feat/realtime-workflow
gh pr create --title "feat: add realtime workflow" --body "Adds PocketBase realtime flow and API rules."

git checkout main
git merge --ff-only feat/realtime-workflow
git push origin main

git tag -a v0.1.0 -m "Release v0.1.0"
git push origin v0.1.0
gh release create v0.1.0 --notes "First runnable PocketBase MVP release."
```

## 部署和备份

小团队推荐：

- 单台 Aliyun ECS、VPS 或内网服务器。
- PocketBase 监听本地端口，例如 `127.0.0.1:8090`。
- Caddy 反向代理 HTTPS；课堂默认不要用 Nginx，降低复杂度。
- systemd 或进程管理器守护。
- `pb_data` 放到持久磁盘。
- 定期备份数据库、文件、对象存储和迁移代码。

推荐上线路径：

- 本机 macOS 开发、测试、编译。
- 本机用 `GOOS=linux GOARCH=amd64` 或 `arm64` 构建 Linux 二进制。
- 本机 `npm run build` 构建前端静态文件。
- 使用 `ssh` + `rsync` 上传到服务器 `/opt/pbapp/releases/<version>`。
- 服务器 systemd 运行 PocketBase。
- Caddy 提供前端静态文件，并把 `/api/*` 和 `/_/*` 代理到 `127.0.0.1:8090`。
- 域名 DNS A 记录指向 ECS 公网 IP，阿里云安全组开放 22/80/443。
- Caddy 自动申请和续期免费 HTTPS 证书，前提是域名和 80/443 都可公网访问。
- 部署前备份 `/var/lib/pocketbase/pb_data`。
- 如果部署包含 migration，agent 必须提醒：失败回滚可能需要恢复备份，不只是切回旧二进制。

最小 Caddyfile：

```caddyfile
app.example.com {
	encode zstd gzip

	log {
		output file /var/log/caddy/pbapp.access.log {
			roll_size 100mb
			roll_keep 10
			roll_keep_for 720h
		}
		format json
	}

	@pocketbase path /api/* /_ /_/*
	handle @pocketbase {
		reverse_proxy 127.0.0.1:8090
	}

	handle {
		root * /opt/pbapp/current/frontend
		try_files {path} /index.html
		file_server
	}
}
```

最小 systemd service：

```ini
[Unit]
Description=PocketBase App
After=network.target

[Service]
User=pocketbase
Group=pocketbase
WorkingDirectory=/opt/pbapp/current
Environment=PB_PUBLIC_URL=https://app.example.com
ExecStart=/opt/pbapp/current/pbapp serve --http=127.0.0.1:8090 --dir=/var/lib/pocketbase/pb_data
Restart=always
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

给 agent 服务器权限时：

- 首次 bootstrap 可以临时使用 root SSH；之后创建 `deploy` 用户和 `pocketbase` 运行用户。
- 日常部署使用 SSH key，不要发 root 密码。
- agent 执行删除、重装、清空数据、改防火墙之前必须明确确认。
- SSH key、AccessKey、superuser 密码只保存在本机或服务器安全位置，不进 GitHub。

部署前检查：

- 创建独立 superuser，不要把课堂 demo 密码带到生产。
- 设置 Caddy 反向代理和 HTTPS。
- 限制服务器防火墙，仅开放必要端口。
- 配置邮件服务，否则密码重置/验证邮件不可用。
- 如果使用 S3 兼容存储，确认 credentials 不在 git 中。
- 记录恢复步骤：从代码迁移 + 数据库 + 文件对象恢复。

注意：PocketBase 官方通常强调自己构建/运行二进制即可；Docker 可以用，但要确认镜像来源、持久卷和备份策略，不要把临时容器文件当成持久数据。

## 数据库备份策略

数据库备份是上线交付的一部分。Agent 必须帮用户建立“能备份、知道备份在哪里、能恢复”的闭环。

PocketBase 内置备份：

- 后台路径：`Dashboard > Settings > Backups`。
- 适合非工程同学优先使用。
- 可以做备份和恢复；当前版本支持把备份存在本地或 S3-compatible storage。
- 备份建议放到独立 bucket，不要和业务文件 bucket 混在一起。
- 内置备份是 `pb_data` 的 ZIP 快照，包含本地存储的上传文件，但不包含本地备份目录本身，也不包含已经放到 S3 的业务文件。
- 生成 ZIP 期间应用会短暂进入只读状态；`pb_data` 很大时会慢，要提前告知用户。
- 如果 `pb_data` 达到 GB 级，考虑 SQLite `.backup` 加文件同步或服务暂停冷备份。

生产建议：

- 上线前至少做一次手动备份，并记录备份文件名。
- 每次运行 migration 前必须备份。
- 每天自动备份，至少保留 7-14 天；重要系统保留 30 天。
- OSS/S3 业务文件要开启版本控制或生命周期策略。
- 定期做恢复演练：找一个临时目录或测试服务器恢复，确认备份可用。
- 备份文件可能包含用户数据和系统密钥，不能提交到 GitHub。
- 如果启用了 settings encryption，必须单独保存 `PB_ENCRYPTION_KEY`；没有它，数据库里的加密配置无法恢复。

小系统最安全的冷备份命令：

```bash
ssh $SERVER "
  TS=\$(date +%Y%m%d%H%M%S) &&
  sudo mkdir -p /var/backups/pbapp &&
  if systemctl is-active --quiet pbapp; then sudo systemctl stop pbapp; fi &&
  sudo tar -C /var/lib/pocketbase -czf /var/backups/pbapp/pb_data-\$TS.tar.gz pb_data &&
  sudo systemctl start pbapp &&
  sudo ls -lh /var/backups/pbapp/pb_data-\$TS.tar.gz
"
```

数据库级热备份适合减少停机时间，但仍要同步上传文件：

```bash
ssh $SERVER "
  TS=\$(date +%Y%m%d%H%M%S) &&
  sudo mkdir -p /var/backups/pbapp/sqlite-\$TS &&
  sudo sqlite3 /var/lib/pocketbase/pb_data/data.db \".backup '/var/backups/pbapp/sqlite-\$TS/data.db'\" &&
  if [ -d /var/lib/pocketbase/pb_data/storage ]; then sudo tar -C /var/lib/pocketbase/pb_data -czf /var/backups/pbapp/sqlite-\$TS/pb_files.tar.gz storage; fi &&
  sudo tar -C /var/backups/pbapp -czf /var/backups/pbapp/sqlite-\$TS.tar.gz sqlite-\$TS &&
  sudo rm -rf /var/backups/pbapp/sqlite-\$TS &&
  sudo ls -lh /var/backups/pbapp/sqlite-\$TS.tar.gz
"
```

恢复冷备份或 PocketBase 内置导出的完整 `pb_data` 备份前必须停服务并征求用户确认。SQLite 热备份压缩包不是完整 `pb_data` 包，恢复时要先解包并按实际文件结构恢复数据库和本地文件：

```bash
ssh $SERVER "
  if systemctl is-active --quiet pbapp; then sudo systemctl stop pbapp; fi &&
  sudo mv /var/lib/pocketbase/pb_data /var/lib/pocketbase/pb_data.before-restore-\$(date +%Y%m%d%H%M%S) &&
  sudo tar -C /var/lib/pocketbase -xzf /var/backups/pbapp/<backup-file>.tar.gz &&
  sudo chown -R pocketbase:pocketbase /var/lib/pocketbase/pb_data &&
  sudo systemctl start pbapp &&
  sudo systemctl status pbapp --no-pager
"
```

## 日志和日常运维

Agent 要把日志管理当作交付内容，而不是只把服务跑起来：

- PocketBase 应用日志：在 Admin UI 的 `Dashboard > Logs` 查看。Go 扩展里用 `app.Logger()` 记录重要业务事件和错误。
- PocketBase 服务日志：通过 systemd/journald 查看，例如 `journalctl -u pbapp`。
- Caddy 服务日志：看证书、配置、反向代理错误，例如 `journalctl -u caddy`。
- Caddy access log：记录公网请求、4xx/5xx、代理路径，例如 `/var/log/caddy/pbapp.access.log`。

交付时必须提供这些命令：

```bash
ssh $SERVER 'sudo systemctl status pbapp caddy --no-pager'
ssh $SERVER 'sudo journalctl -u pbapp -n 200 --no-pager'
ssh $SERVER 'sudo journalctl -u pbapp -f'
ssh $SERVER 'sudo journalctl -u caddy -n 200 --no-pager'
ssh $SERVER 'sudo tail -n 200 /var/log/caddy/pbapp.access.log'
ssh $SERVER 'df -h; sudo du -sh /var/lib/pocketbase/pb_data /var/log/caddy /var/backups/pbapp 2>/dev/null'
```

日常操作：

- 重启应用：`sudo systemctl restart pbapp`。
- 重载 Caddy：`sudo systemctl reload caddy`。
- 确认自启动：`sudo systemctl is-enabled pbapp caddy`。
- 查看是否运行：`sudo systemctl status pbapp caddy --no-pager`。
- 清理 journal：`sudo journalctl --vacuum-time=14d`。
- 回滚前先保存错误日志，不要立刻覆盖线索。

Agent 写业务日志时不要记录密码、token、AccessKey、完整身份证、手机号全量、私密文件 URL 或用户隐私原文。

## 首次 superuser 和上线提醒

PocketBase 把超级管理员称为 `superuser`，课堂上说的 superadmin 通常就是它。第一次启动后会在日志里出现 Web UI installer 链接，也可以用命令创建：

```bash
sudo -u pocketbase /opt/pbapp/current/pbapp superuser create ADMIN_EMAIL STRONG_TEMP_PASSWORD --dir=/var/lib/pocketbase/pb_data
```

Agent 必须提醒用户：

- 第一次部署后立刻设置自己的 superuser 邮箱和强密码。
- 如果 agent 临时创建密码，用户登录后必须马上修改。
- superuser 密码、AccessKey、SSH key、`.env`、`pb_data` 绝不能进 GitHub。
- 登录后台：`https://app.example.com/_/`。
- 配置 SMTP，否则重置密码、验证邮件、MFA 邮件可能不可用。
- 开启 rate limiter，减少暴力登录和 API 滥用。
- 配置 superuser IP whitelist；如果把自己锁住，可用当前二进制的 `superuser ips --dir=/var/lib/pocketbase/pb_data` 命令重置。
- 可选启用 superuser MFA/OTP。
- 在反向代理后配置真实客户端 IP 头，常见是 `X-Real-IP` 和 `X-Forwarded-For`。
- 检查 public API rules，确认敏感数据没有公开。
- 确认备份恢复路径：代码在 GitHub，数据在 `pb_data`，对象在 OSS 或本地文件目录。

## 验收清单

交付前逐项确认：

- 本地命令能从干净终端启动后端和前端。
- Admin UI 可以打开，但普通业务不依赖手工后台操作才能跑。
- 迁移能创建 collections、字段、索引和 API rules。
- 普通用户能完成核心流程。
- 管理员/运营能完成审核、分配或管理流程。
- API rules 直接用 SDK/API 测试通过。
- realtime 在两窗口验证通过。
- 文件上传、预览、删除、权限访问通过。
- 重启 PocketBase 后数据仍在。
- `systemctl is-enabled pbapp caddy` 通过，服务可自启动。
- `journalctl -u pbapp`、`journalctl -u caddy`、Caddy access log 都能查看。
- 第一个 superuser 已创建，用户已被提醒更换强密码并配置 SMTP/rate limiter/IP whitelist。
- 部署前备份已生成，备份文件名已记录，恢复命令已写清楚。
- 如果需要定时任务，已使用 `app.Cron()` 实现并在 `Dashboard > Settings > Crons` 验证。
- `pb_data` 和对象存储备份策略已写清楚。
- 最终说明里没有暴露真实密钥、邮箱密码、superuser 密码或对象存储 secret。
