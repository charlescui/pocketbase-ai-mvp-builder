# Aliyun + GitHub CLI + Caddy 部署参考

## 目录

- [默认部署模型](#默认部署模型)
- [让 AI agent 操作 Linux 服务器](#让-ai-agent-操作-linux-服务器)
- [macOS 本机工具](#macos-本机工具)
- [GitHub CLI 仓库和版本策略](#github-cli-仓库和版本策略)
- [阿里云 ECS 服务器准备](#阿里云-ecs-服务器准备)
- [Caddy 免费 HTTPS](#caddy-免费-https)
- [systemd 运行 PocketBase](#systemd-运行-pocketbase)
- [日志管理](#日志管理)
- [首次 superuser 和上线后安全设置](#首次-superuser-和上线后安全设置)
- [本机编译并部署到服务器](#本机编译并部署到服务器)
- [打包发布脚本](#打包发布脚本)
- [PocketBase 定时任务](#pocketbase-定时任务)
- [备份恢复和回滚](#备份恢复和回滚)
- [阿里云 OSS S3 兼容配置](#阿里云-oss-s3-兼容配置)
- [上线验收](#上线验收)

## 默认部署模型

课堂默认建议：

- 本机 macOS 做二次开发、测试、编译和打包。
- GitHub 托管代码和版本，使用 `gh` CLI 自动创建仓库、分支、PR、tag 和 release。
- 阿里云 ECS 作为 Linux 服务器。
- Caddy 作为公网入口，自动申请和续期免费 HTTPS 证书。
- PocketBase 监听本机端口 `127.0.0.1:8090`，不直接暴露到公网。
- 前端静态文件由 Caddy 提供，`/api/*` 和 `/_/*` 反向代理到 PocketBase。
- `pb_data` 放在服务器持久目录，例如 `/var/lib/pocketbase/pb_data`。
- 上传文件简单场景放本地 `pb_data`，正式场景使用阿里云 OSS 的 S3 兼容接口。
- systemd 负责开机自启动、崩溃重启和服务日志。
- Caddy access log 负责记录公网请求和 HTTPS/代理问题。
- PocketBase Dashboard > Logs 负责查看应用内日志和请求日志。
- 每次上线前备份，失败可回滚到上一版；如果有数据库迁移，必要时还要恢复部署前 `pb_data`。
- 应用内周期性工作使用 PocketBase `app.Cron()`，并在 Dashboard > Settings > Crons 验证。
- 数据备份优先配置 PocketBase 内置 Backups；部署脚本再做上线前安全快照。

## 让 AI agent 操作 Linux 服务器

非工程同学可以让自己的 AI agent 执行服务器部署运维，但要给权限给得清楚：

1. 在阿里云控制台创建 ECS、域名解析、安全组后，把这些信息交给 agent：
   - 服务器公网 IP。
   - 域名，例如 `app.example.com`。
   - Linux 发行版和 CPU 架构，推荐 Ubuntu 24.04 LTS x86_64。
   - SSH 用户名，例如 `root` 或 `deploy`。
   - SSH 私钥路径，例如 `~/.ssh/aliyun_pocketbase`.
2. 首次 bootstrap 可以临时给 agent root SSH 权限；完成后让 agent 创建专用用户并关闭密码登录。
3. 日常部署使用专用用户，例如 `deploy`，只给必要 sudo 权限。
4. 永远不要把 SSH 私钥、AccessKey、数据库文件、superuser 密码提交到 GitHub。
5. 让 agent 每次远程操作前说明要执行的服务器命令；涉及删除、重装、清空数据、改防火墙时必须再次确认。

推荐本机 SSH 配置：

```sshconfig
Host aliyun-pocketbase
  HostName 1.2.3.4
  User deploy
  IdentityFile ~/.ssh/aliyun_pocketbase
  IdentitiesOnly yes
```

验证：

```bash
ssh aliyun-pocketbase 'hostname; whoami; uname -a'
```

## macOS 本机工具

安装：

```bash
brew install go node git gh rsync
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOSUMDB=sum.golang.google.cn
npm config set registry https://registry.npmmirror.com
gh auth login -p ssh -w
gh auth status
```

建议检查：

```bash
go version
node -v
npm -v
git --version
gh --version
ssh -V
rsync --version
```

## GitHub CLI 仓库和版本策略

初始化新项目：

```bash
git init
git branch -M main
cat > .gitignore <<'EOF'
.DS_Store
.env
.env.*
node_modules/
dist/
build/
pb_data/
*.db
*.sqlite
*.sqlite3
*.pem
*.key
EOF
git add .
git commit -m "chore: scaffold pocketbase app"
gh repo create my-pocketbase-app --private --source=. --remote=origin --push
```

已有项目连接远程：

```bash
gh repo view --web
git remote -v
git status --short
```

默认分支策略：

- `main`: 始终保持可运行、可部署。
- `feat/<short-name>`: 新功能，例如 `feat/realtime-kanban`。
- `fix/<short-name>`: bugfix。
- `deploy/<target>`: 只在需要隔离部署脚本或服务器配置变更时使用。

Agent 应主动 commit 的时机：

- 项目脚手架能启动后。
- 新 migration 或 collection/API rules 完成并跑通后。
- 登录/权限流程完成并测试后。
- realtime 流程完成并双窗口验证后。
- 文件上传或 OSS 配置完成并验证后。
- 部署脚本、Caddyfile、systemd unit 完成后。
- 修复一个明确 bug 并验证后。
- 远程部署 smoke test 通过后，打 tag。

不要 commit 的内容：

- `.env`、AccessKey、SSH key、superuser 密码。
- `pb_data`、生产数据库、上传文件。
- `node_modules`、临时 build 产物。
- 服务器日志或包含用户隐私的数据导出。

常用自动化命令：

```bash
git checkout -b feat/realtime-kanban
git add backend frontend README.md
git commit -m "feat: add realtime kanban workflow"
git push -u origin feat/realtime-kanban
gh pr create --title "feat: add realtime kanban workflow" --body "Adds PocketBase collections, API rules, and realtime UI."
```

课堂项目如果不使用 PR，可以直接合并到 `main`，但每个可运行里程碑仍要提交并推送：

```bash
git checkout main
git merge --ff-only feat/realtime-kanban
git push origin main
```

发布 tag：

```bash
VERSION=v0.1.0
git tag -a "$VERSION" -m "Release $VERSION"
git push origin "$VERSION"
gh release create "$VERSION" --notes "PocketBase MVP release $VERSION"
```

版本号建议：

- `v0.1.0`: 第一次能完整跑通。
- `v0.1.1`: 小修小补。
- `v0.2.0`: 增加一个明显新模块或新流程。

## 阿里云 ECS 服务器准备

控制台操作给非工程同学的 checklist：

1. 创建 ECS 实例，推荐 Ubuntu 24.04 LTS 或 Debian 12。
2. 规格选择轻量即可，课堂/小 MVP 可从 1-2 vCPU、1-4 GB 内存开始。
3. 绑定公网 IP。
4. 安全组开放：
   - TCP 22: SSH。能限制到自己的办公网络 IP 更好。
   - TCP 80: Caddy 申请 HTTP-01 证书和跳转。
   - TCP 443: HTTPS。
5. 域名 DNS 添加 A 记录到 ECS 公网 IP，例如 `app.example.com -> 1.2.3.4`。
6. 等待 DNS 生效后，本机测试：

```bash
dig +short app.example.com
ssh root@1.2.3.4 'uname -a'
```

服务器 bootstrap：

```bash
apt update
apt -y upgrade
apt -y install curl wget git rsync unzip tar sqlite3 ca-certificates gnupg lsb-release ufw
timedatectl set-timezone Asia/Shanghai
adduser --disabled-password --gecos "" deploy
usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

可选防火墙：

```bash
ufw allow OpenSSH
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
ufw status
```

创建应用目录和运行用户：

```bash
useradd --system --home /var/lib/pocketbase --shell /usr/sbin/nologin pocketbase || true
mkdir -p /opt/pbapp/releases /opt/pbapp/shared /var/lib/pocketbase/pb_data /var/backups/pbapp /var/log/caddy
chown -R deploy:deploy /opt/pbapp
chown -R pocketbase:pocketbase /var/lib/pocketbase
chown -R caddy:caddy /var/log/caddy 2>/dev/null || true
```

## Caddy 免费 HTTPS

Caddy 是默认推荐入口。它能在域名 DNS 指向服务器、80/443 可访问时自动申请和续期 HTTPS 证书。

Debian/Ubuntu 安装：

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install -y caddy
mkdir -p /var/log/caddy
chown -R caddy:caddy /var/log/caddy
systemctl enable --now caddy
```

单域名、前端静态文件 + PocketBase API/Admin 的 Caddyfile：

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

写入并重载：

```bash
caddy fmt --overwrite /etc/caddy/Caddyfile
caddy validate --config /etc/caddy/Caddyfile
systemctl reload caddy
systemctl status caddy --no-pager
```

如果只部署 PocketBase 后端，没有前端静态站：

```caddyfile
app.example.com {
	reverse_proxy 127.0.0.1:8090
}
```

排错重点：

- DNS A 记录必须指向当前服务器公网 IP。
- 阿里云安全组必须开放 80/443。
- 服务器本机防火墙必须开放 80/443。
- Caddy 首次签证书时 80/443 不能被 Nginx/Apache 占用。
- PocketBase 只监听 `127.0.0.1:8090`，公网入口交给 Caddy。

## systemd 运行 PocketBase

服务文件：

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
LimitNOFILE=65535
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

安装：

```bash
tee /etc/systemd/system/pbapp.service >/dev/null <<'EOF'
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
LimitNOFILE=65535
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable pbapp
```

常用运维：

```bash
systemctl status pbapp --no-pager
journalctl -u pbapp -n 200 --no-pager
systemctl restart pbapp
systemctl is-enabled pbapp
```

开机自启动检查：

```bash
systemctl is-enabled pbapp
systemctl is-enabled caddy
systemctl status pbapp caddy --no-pager
```

不要在未确认的情况下让 agent 执行 `reboot`。如果用户允许验证自启动，先确认当前部署可回滚，再执行：

```bash
sudo reboot
# 等待 30-90 秒后
ssh aliyun-pocketbase 'systemctl status pbapp caddy --no-pager'
```

## 日志管理

Agent 要把日志分成三层看，排障时按顺序定位：

1. Caddy 层：证书、域名、HTTPS、反向代理、公网访问。
2. systemd/PocketBase 进程层：服务是否启动、是否崩溃、端口是否监听、迁移是否报错。
3. PocketBase 应用层：Dashboard > Logs 中的请求日志、业务日志、文件存储错误、权限错误。

常用查看命令：

```bash
# 服务当前状态，包含最近日志摘要
ssh $SERVER 'sudo systemctl status pbapp caddy --no-pager'

# PocketBase 服务日志，最近 200 行
ssh $SERVER 'sudo journalctl -u pbapp -n 200 --no-pager'

# 实时跟踪 PocketBase 服务日志
ssh $SERVER 'sudo journalctl -u pbapp -f'

# Caddy 服务日志，重点看证书和配置错误
ssh $SERVER 'sudo journalctl -u caddy -n 200 --no-pager'

# Caddy access log，重点看 4xx/5xx、代理失败、访问路径
ssh $SERVER 'sudo tail -n 200 /var/log/caddy/pbapp.access.log'
ssh $SERVER 'sudo tail -f /var/log/caddy/pbapp.access.log'

# 磁盘和数据目录大小
ssh $SERVER 'df -h; sudo du -sh /var/lib/pocketbase/pb_data /var/log/caddy /var/backups/pbapp 2>/dev/null'
```

PocketBase 业务代码要使用 `app.Logger()` 记录关键事件：

- migration 开始/完成/失败。
- 自定义 route 的重要错误。
- 文件上传和 OSS 写入失败。
- 重要状态流转，例如审批通过、任务分配、支付/导入/通知失败。
- 不记录密码、token、AccessKey、身份证、手机号全量、个人隐私原文。

PocketBase 官方日志会写入数据库，可在 `Dashboard > Logs` 查看。上线后让用户设置：

- logs retention period：小 MVP 可保留 7-30 天。
- minimal log level：生产默认 `info` 或 `warn`，排障时临时调低。
- request IP logging：启用前先确认反向代理真实 IP 头配置正确。

日志清理：

```bash
# 查看 journald 占用
ssh $SERVER 'sudo journalctl --disk-usage'

# 只保留最近 14 天 journal，课堂项目够用
ssh $SERVER 'sudo journalctl --vacuum-time=14d'
```

Agent 最终交付时必须给用户一组“复制即可看日志”的命令，不能只说“查看日志”。

## 首次 superuser 和上线后安全设置

PocketBase 官方称后台超级管理员为 `superuser`。很多课堂用户会叫它 superadmin，两者在这里指同一件事。

第一次启动后，PocketBase 会在日志里生成 Web UI installer 链接用于设置第一个 superuser。用 systemd 运行时，从日志里找：

```bash
ssh $SERVER 'sudo journalctl -u pbapp -n 200 --no-pager'
```

也可以显式创建第一个 superuser。因为本部署把数据目录放在 `/var/lib/pocketbase/pb_data`，命令必须带同一个 `--dir`：

```bash
ssh $SERVER
sudo -u pocketbase /opt/pbapp/current/pbapp superuser create ADMIN_EMAIL STRONG_TEMP_PASSWORD --dir=/var/lib/pocketbase/pb_data
```

安全要求：

- Agent 不要替用户编造永久管理员密码。
- 如果 agent 创建临时密码，必须提示用户登录后立刻更换。
- superuser 邮箱和密码不要写入 GitHub、README、commit message、issue、PR、release note。
- 登录地址通常是 `https://app.example.com/_/`。

创建 superuser 后，agent 要提醒用户立刻完成：

- 更改为强密码，保存到自己的密码管理器。
- 配置 SMTP 邮件服务，否则重置密码、验证邮件、MFA 邮件可能不可用。
- 启用 rate limiter，降低暴力登录和 API 滥用风险。
- 配置 Superuser IPs whitelist，只允许自己的办公/VPN IP 访问 superuser API。IP 会变的用户要知道如何解除：

```bash
# 清空 superuser IP 白名单，用户被锁住时才用
sudo -u pocketbase /opt/pbapp/current/pbapp superuser ips --dir=/var/lib/pocketbase/pb_data

# 设置白名单，替换成真实 IP 或网段
sudo -u pocketbase /opt/pbapp/current/pbapp superuser ips 1.2.3.4 --dir=/var/lib/pocketbase/pb_data
```

- 可选启用 superuser MFA/OTP。启用后如果邮件不可用，可用当前版本支持的 `superuser otp` 命令临时生成 OTP。
- 在 Settings 里配置 Application URL/Public URL 为正式 HTTPS 域名。
- 在反向代理后配置真实客户端 IP 头，常见是 `X-Real-IP` 和 `X-Forwarded-For`，否则日志里看到的可能都是 Caddy/本机 IP。
- 检查所有 public API rules，确认没有误把敏感 collection 设成公开。

## 本机编译并部署到服务器

本机设置变量：

```bash
export APP_NAME=pbapp
export SERVER=aliyun-pocketbase
export DOMAIN=app.example.com
export VERSION=$(git rev-parse --short HEAD)
export GOOS=linux
export GOARCH=amd64
```

如果 ECS 是 ARM 架构，把 `GOARCH` 改成 `arm64`。

本机构建：

```bash
mkdir -p dist/deploy
(cd backend && CGO_ENABLED=0 GOOS=$GOOS GOARCH=$GOARCH go build -trimpath -ldflags="-s -w" -o ../dist/deploy/$APP_NAME .)
(cd frontend && npm ci && npm run build)
cp -R frontend/dist dist/deploy/frontend
tar -C dist/deploy -czf "dist/${APP_NAME}-${VERSION}-linux-${GOARCH}.tar.gz" .
```

上传到服务器 release 目录：

```bash
ssh $SERVER "mkdir -p /opt/pbapp/releases/$VERSION"
rsync -avz --delete dist/deploy/ "$SERVER:/opt/pbapp/releases/$VERSION/"
```

部署前备份当前数据。小型课堂系统优先使用短暂停机冷备份，最容易理解、恢复也最稳：

```bash
ssh $SERVER "
  TS=\$(date +%Y%m%d%H%M%S) &&
  sudo mkdir -p /var/backups/pbapp &&
  if systemctl is-active --quiet pbapp; then sudo systemctl stop pbapp; fi &&
  sudo tar -C /var/lib/pocketbase -czf /var/backups/pbapp/pb_data-before-${VERSION}-\$TS.tar.gz pb_data &&
  sudo systemctl start pbapp &&
  sudo ls -lh /var/backups/pbapp/pb_data-before-${VERSION}-\$TS.tar.gz
"
```

切换版本并重启：

```bash
ssh $SERVER "
  ln -sfn /opt/pbapp/releases/$VERSION /opt/pbapp/current &&
  sudo chown -R pocketbase:pocketbase /opt/pbapp/releases/$VERSION &&
  sudo systemctl restart pbapp &&
  sudo systemctl reload caddy &&
  sudo systemctl status pbapp --no-pager &&
  sudo journalctl -u pbapp -n 80 --no-pager
"
```

Smoke test：

```bash
curl -I https://$DOMAIN
curl -I https://$DOMAIN/api/health
```

如果 `api/health` 不存在，改用项目实际健康检查 route，或至少访问 `https://$DOMAIN/_/` 确认 Admin UI 可达。

上传 release artifact 到 GitHub：

```bash
gh release upload "$VERSION" "dist/${APP_NAME}-${VERSION}-linux-${GOARCH}.tar.gz" --clobber
```

## 打包发布脚本

Agent 应该在项目里生成这些脚本，方便非工程同学重复使用：

```text
scripts/
  build.sh      # 本机构建 Linux 二进制和前端 dist，并生成 tar.gz
  deploy.sh     # 上传到 ECS release 目录，先备份 pb_data，切换 current，重启服务
  logs.sh       # 查看 pbapp/caddy/journal/access log
  rollback.sh   # 列出 releases，切回指定版本
  backup.sh     # 手动备份 pb_data 或 SQLite 数据库，并提醒 OSS 另备
  restore.sh    # 恢复备份，必须二次确认
```

脚本要求：

- 顶部集中配置 `APP_NAME`、`SERVER`、`DOMAIN`、`GOARCH`。
- 使用 `set -euo pipefail`。
- 部署前检查 `git status --short`，提醒未提交变更。
- 部署前运行测试和 build。
- 部署前备份 `pb_data`，记录备份文件名。
- 部署后输出 `systemctl status`、`journalctl` 摘要、`curl` smoke test。
- 禁止把密码或 AccessKey 写进脚本。

## PocketBase 定时任务

PocketBase 自带定时任务能力，适合应用自己的周期性工作。Agent 遇到这些需求时要优先使用 `app.Cron()`：

- 每天/每周发送运营报表。
- 每小时同步外部系统。
- 定期清理过期草稿、临时文件、通知、验证码。
- 定时扫描超时任务并自动变更状态。
- 定时发送提醒、催办、日报。
- 定时检查最近一次备份是否存在。

Go 代码模式：

```go
app.Cron().MustAdd("backup-check", "0 10 * * *", func() {
	app.Logger().Info("backup-check job started")
	// TODO: check backup records/files and notify admin if missing.
})
```

规则：

- `app.Cron()` 在 `serve` 时自动启动，任务应在 `app.Start()` 前注册。
- 用稳定 id，例如 `backup-check`、`daily-summary`、`expire-tasks`。
- cron 表达式示例：`*/10 * * * *` 每 10 分钟，`0 * * * *` 每小时，`0 9 * * *` 每天 9 点。
- 每个任务都要写开始、成功、失败日志。
- 任务失败要记录错误，不要让服务崩溃。
- 长任务要防重入，例如使用数据库锁、状态记录或任务运行记录。
- 任务可以在 `Dashboard > Settings > Crons` 预览和手动触发。
- 不要删除或停止 app 级所有 cron；PocketBase 系统任务也使用它，包括日志清理和自动备份等系统任务，系统 id 通常是 `__pb*__`。

验证：

```bash
ssh $SERVER 'sudo journalctl -u pbapp -n 200 --no-pager | grep -i "backup-check\\|daily-summary\\|expire-tasks" || true'
```

同时在 Admin UI 的 `Dashboard > Settings > Crons` 手动触发一次任务，并在 `Dashboard > Logs` 看结果。

## 备份恢复和回滚

PocketBase 的业务数据、上传文件和配置通常在 `pb_data`。生产备份必须覆盖：

- `/var/lib/pocketbase/pb_data`。
- 如果使用本地文件存储，`pb_data` 中的文件对象。
- 如果使用 OSS，OSS bucket 对象也要有备份/版本控制策略。
- GitHub 仓库里的迁移文件和应用代码。
- 如果启用 settings encryption，单独保存 `PB_ENCRYPTION_KEY`。

### PocketBase 内置备份

非工程同学优先使用 PocketBase 后台内置备份：

- 入口：`Dashboard > Settings > Backups`。
- 备份/恢复走 PocketBase 内置能力，不需要用户直接操作数据库文件。
- 备份可存本地或 S3-compatible storage；建议备份使用独立 bucket。
- 备份是 `pb_data` 的 ZIP 快照，包含本地上传文件，但不包含本地备份目录本身，也不包含已经放到 S3 的业务文件。
- 生成备份 ZIP 期间应用会临时只读；`pb_data` 很大时会慢。
- 后台如果启用 auto backups，agent 必须记录频率、保留时间、存储位置，并做一次恢复演练。

Agent 上线时必须提醒用户：

- 至少配置每日备份，保留 7-14 天；重要系统保留 30 天。
- 备份和业务文件尽量放不同 bucket。
- OSS bucket 建议开启版本控制或生命周期策略。
- 备份文件含敏感数据，不进 GitHub，不发到公开群。

### 冷备份

冷备份最稳，适合课堂、小 MVP 和部署前快照。缺点是服务会短暂停止：

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

### SQLite 热备份

如果不想停机，可以对数据库做 SQLite `.backup`，但还要同步本地上传文件；如果业务文件已经在 OSS，也要备份 OSS 或开启 bucket 版本控制。

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

如果当前 PocketBase 版本或项目数据目录不是 `data.db`/`storage`，agent 要先 `find /var/lib/pocketbase/pb_data -maxdepth 2 -type f` 确认实际路径。

### 恢复

恢复冷备份或 PocketBase 内置导出的完整 `pb_data` 备份会覆盖线上数据，agent 必须先征求确认。SQLite 热备份压缩包不是完整 `pb_data` 包，恢复时要先解包并按实际文件结构恢复数据库和本地文件：

```bash
ssh $SERVER "
  if systemctl is-active --quiet pbapp; then sudo systemctl stop pbapp; fi &&
  sudo mv /var/lib/pocketbase/pb_data /var/lib/pocketbase/pb_data.broken-$(date +%Y%m%d%H%M%S) &&
  sudo tar -C /var/lib/pocketbase -xzf /var/backups/pbapp/<backup-file>.tar.gz &&
  sudo chown -R pocketbase:pocketbase /var/lib/pocketbase/pb_data &&
  sudo systemctl start pbapp &&
  sudo systemctl status pbapp --no-pager
"
```

只回滚程序版本：

```bash
ssh $SERVER "
  ls -1 /opt/pbapp/releases &&
  ln -sfn /opt/pbapp/releases/<previous-version> /opt/pbapp/current &&
  sudo chown -R pocketbase:pocketbase /opt/pbapp/releases/<previous-version> &&
  sudo systemctl restart pbapp &&
  sudo systemctl status pbapp --no-pager
"
```

重要提醒：

- 如果新版本只改前端或 Go 逻辑，通常切回旧 release 即可。
- 如果新版本运行了 migration 并改变数据库结构，回滚旧二进制可能还不够，可能需要 migration down 或恢复部署前 `pb_data` 备份。
- 回滚前先查看日志，保存错误信息，避免把线索覆盖掉。
- 回滚后打一个修复分支和 hotfix commit，不要只在服务器上手改。
- 每月至少做一次恢复演练：在临时目录或测试服务器恢复备份，确认能打开 Admin UI 和核心数据。

## 阿里云 OSS S3 兼容配置

阿里云 OSS 可以通过 AWS SDK 的 S3 兼容方式访问。配置时要使用 OSS S3 格式 endpoint，而不是随手填写普通域名。

关键规则：

- S3 兼容 endpoint 形如：`https://s3.oss-cn-hangzhou.aliyuncs.com`。
- bucket 必须在对应 region，例如 `oss-cn-hangzhou`。
- 使用 OSS 支持的 region id。
- 使用 RAM 用户 AccessKey，授予最小 OSS bucket 权限。
- 阿里云文档提示 AWS SDK v2 需要开启 path-style 访问。
- 阿里云文档提示某些 SDK/场景可能需要关闭默认校验，例如 checksum validation 或 request checksum calculation。只有遇到兼容错误时再按官方文档调整。
- AccessKey 只放服务器环境变量或 PocketBase 管理后台设置，不放前端、不放 GitHub。

PocketBase 管理后台配置建议：

1. 打开 Admin UI。
2. 进入 Settings / File storage 或对应版本的存储设置。
3. 启用 S3 storage。
4. 填写：
   - Bucket: `your-bucket-name`
   - Region: `oss-cn-hangzhou`
   - Endpoint: `https://s3.oss-cn-hangzhou.aliyuncs.com`
   - Access key: RAM AccessKey ID
   - Secret: RAM AccessKey Secret
   - Force path style: 如果 PocketBase 版本提供该选项，按阿里云 AWS SDK v2 兼容要求优先开启；若上传失败，再按当前 PocketBase/OSS 错误信息调整。
5. 保存后用一个测试 collection 上传、读取、删除小文件。

如果用环境变量或部署密钥保存配置：

```bash
sudo install -d -m 700 -o pocketbase -g pocketbase /etc/pbapp
sudo tee /etc/pbapp/pbapp.env >/dev/null <<'EOF'
ALIYUN_OSS_ENDPOINT=https://s3.oss-cn-hangzhou.aliyuncs.com
ALIYUN_OSS_REGION=oss-cn-hangzhou
ALIYUN_OSS_BUCKET=your-bucket-name
ALIYUN_OSS_ACCESS_KEY_ID=replace-me
ALIYUN_OSS_ACCESS_KEY_SECRET=replace-me
ALIBABA_CLOUD_ACCESS_KEY_ID=replace-me
ALIBABA_CLOUD_ACCESS_KEY_SECRET=replace-me
ALIYUN_PNVS_ENDPOINT=dypnsapi.aliyuncs.com
ALIYUN_SMS_SCHEME_NAME=replace-me
ALIYUN_SMS_DEFAULT_SIGN_NAME=速通互联验证码
ALIYUN_SMS_DEFAULT_TEMPLATE_CODE=100001
ALIYUN_SMS_CODE_TYPE=1
SMS_CODE_LENGTH=6
SMS_CODE_TTL_SECONDS=300
SMS_CODE_COOLDOWN_SECONDS=60
EOF
sudo chown pocketbase:pocketbase /etc/pbapp/pbapp.env
sudo chmod 600 /etc/pbapp/pbapp.env
```

在 systemd service 中引用：

```ini
EnvironmentFile=/etc/pbapp/pbapp.env
```

CORS 和访问：

- 如果浏览器直接访问 OSS/CDN 域名，需要在 OSS bucket CORS 中允许你的 HTTPS 域名。
- 私有文件优先让 PocketBase 控制访问，不要把 bucket 整体公开。
- 用 CDN 时确认缓存策略不会泄漏受保护文件。
- 阿里云短信认证 AccessKey 也放在服务器环境文件或密钥管理系统中，不能放进前端构建产物、README、GitHub Actions 明文变量或客户端可读 API。

备份：

- SQLite/PocketBase 元数据和 OSS 对象必须一起备份。
- 迁移文件只能恢复结构，不能恢复业务数据和上传文件。
- 定期导出数据库快照，并配置 OSS bucket 版本控制或生命周期策略。

## 上线验收

Agent 部署完成后必须回报：

- GitHub repo URL、当前 branch、最新 commit hash、tag/release。
- ECS IP、域名、Caddy HTTPS 状态。
- systemd 服务状态和最近日志摘要。
- PocketBase Admin UI URL。
- 前端 URL。
- API rules 权限测试结果。
- realtime 双窗口测试结果。
- PocketBase `app.Cron()` 定时任务列表、job id、cron 表达式、Dashboard 触发/日志验证结果。
- 文件上传、本地存储或 OSS S3 兼容存储测试结果。
- PocketBase 内置 Backups 配置、最近一次备份文件名、备份存储位置、恢复步骤和恢复演练结果。
- 部署前快照备份文件名，migration 回滚风险说明。
- 没有提交或泄漏任何密钥的确认。
