---
name: pocketbase-ai-mvp-builder
description: Build or extend PocketBase-based MVPs, internal tools, admin systems, and small SaaS prototypes with AI agents. Use when the task involves PocketBase local macOS setup, Go/JavaScript secondary development, collections and migrations, auth and API rules, realtime frontend synchronization, phone number identity, SMS verification, Aliyun Phone Number Verification Service/号码认证服务, login/registration by mobile phone, file uploads or Aliyun OSS/S3-compatible object storage, scheduled jobs with PocketBase app.Cron, GitHub CLI repository/version workflows, Caddy HTTPS, Aliyun ECS/Linux server deployment and operations, systemd autostart, logging, database backup, rollback, packaging, superuser/admin setup, admin dashboard workflows, SDK/API integration, or teaching non-engineers how to drive an agent to build professional small systems.
---

# PocketBase AI MVP Builder

## Core Posture

Treat PocketBase as the operating base for small professional systems: SQLite data storage, admin UI, auth, API rules, REST API, realtime subscriptions, file storage, migrations, and Go extension in one compact binary. Build the actual usable product, not a landing page or a pile of explanations.

Default to a simple architecture:

- PocketBase backend extended with Go when custom behavior, routes, hooks, cron jobs, or production-grade migrations are needed.
- A Vite/React or existing frontend client using the official PocketBase JS SDK.
- Schema and permission changes captured as migrations, not only manual admin clicks.
- Phone-number identity treated as a first-class auth path when real users register, log in, bind accounts, or submit forms that require verified identity.
- Realtime treated as a first-class product capability: initialize state from API, subscribe to changes, merge create/update/delete events into the UI, and verify with two browser sessions.
- Scheduled/background work implemented with PocketBase's built-in `app.Cron()` when the job belongs to the application.
- GitHub CLI (`gh`) used to create and maintain the repository, branches, commits, tags, and releases.
- Local macOS development and build, then deployment to a Linux server, preferably Aliyun ECS for the classroom path.
- Caddy used as the default reverse proxy for automatic free HTTPS certificates.
- systemd, journald, Caddy access logs, PocketBase Dashboard logs, backups, rollback scripts, and first superuser setup treated as part of the product, not optional ops trivia.

Before implementing, verify the current PocketBase release and Go requirement from official sources. As of 2026-06-08, GitHub latest release is `v0.39.2` and the release `go.mod` requires Go `1.25.0`; do not hard-code this forever.

## Reference Loading

Read these only when needed:

- `references/agent-prompt-template.md`: copy-ready Chinese prompt for students to give an AI agent.
- `references/pocketbase-patterns.md`: setup commands, schema patterns, auth rule recipes, realtime patterns, file/S3 guidance, and deployment checklist.
- `references/phone-sms-auth-aliyun.md`: Aliyun Phone Number Verification Service integration, SMS code registration/login, verified phone binding, form identity verification, data model, custom PocketBase endpoints, rate limiting, security and testing.
- `references/aliyun-deployment-github-caddy.md`: Aliyun OSS S3-compatible configuration, GitHub CLI workflow, Aliyun ECS Linux setup, Caddy HTTPS, systemd, logs, superuser setup, build/deploy/rollback/backup commands.

## Workflow

### 1. Clarify the Product

Ask only the missing questions that block implementation. For non-engineers, translate answers into product language.

Capture:

- Users and roles: anonymous visitor, registered user, operator, reviewer, admin, superuser.
- Identity requirements: whether phone number must be the primary login identifier, whether SMS code login is supported, whether password login is optional after phone verification, and which forms require verified phone ownership.
- Core records: what each record represents, who owns it, and who can see/change it.
- Realtime moments: where two users or tabs should update instantly.
- Files: upload types, privacy, size expectations, and whether S3-compatible object storage is required.
- Scheduled jobs: reminders, periodic sync, report generation, stale-status cleanup, backup verification, or other work that should run automatically.
- Admin needs: which data is managed through PocketBase dashboard vs a custom frontend.
- Delivery target: local demo, internal LAN, Aliyun ECS, other VPS, cloud deployment, or classroom exercise.
- Repository/deployment state: whether `gh` is authenticated, remote repo exists, domain/DNS exists, and whether the agent has SSH access to the Linux server.
- Operations state: superuser creation plan, log access method, backup location, rollback strategy, package format, and whether the user permits the agent to reboot/restart services.

Return a short build plan with data model, permissions, screens, realtime flows, and validation steps before editing.

### 2. Prepare macOS Development Environment

Use Homebrew unless the workspace already has pinned tool versions. For China-mainland networks, configure mirrors without using untrusted GitHub rewrite mirrors by default.

Core commands:

```bash
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install go node git gh
go env -w GOPROXY=https://goproxy.cn,direct
go env -w GOSUMDB=sum.golang.google.cn
npm config set registry https://registry.npmmirror.com
gh auth login -p ssh -w
gh auth status
go version
node -v
npm -v
gh --version
```

If GitHub downloads are blocked, ask the user for their company/VPN proxy or an approved mirror. Do not silently configure random acceleration domains.

### 3. Choose the PocketBase Mode

Use one of these modes:

- **Binary-first demo**: download the PocketBase release binary when the task is mostly dashboard configuration, schema exploration, or teaching the basic product capability.
- **Go extension project**: default for professional secondary development. Create a Go module, import PocketBase, add migrations, hooks, custom endpoints, and build a single deployable backend.
- **Existing project extension**: inspect current `go.mod`, migrations, hooks, frontend SDK usage, and deployment scripts before changing anything.

For new Go extension projects:

```bash
mkdir backend
cd backend
go mod init example.com/app
go get github.com/pocketbase/pocketbase@latest
```

Use a minimal `main.go` pattern:

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

Prefer official docs examples for current hook/router signatures because PocketBase's Go extension API evolves.

### 4. Model Data Deliberately

Create collections from the product model:

- Use auth collections for login identities such as `users`, `staff`, or `clients`.
- For Chinese public/internal systems, default to a verified phone-number identity model when the user base is real people. Store normalized phone fields and verification state in the auth collection; do not trust a client-submitted phone number as verified.
- Use relation fields for ownership and organization membership instead of duplicating user names.
- Use select fields for stable statuses and roles; document the allowed values.
- Use file fields for user-uploaded files; store external URLs only when the file is truly managed elsewhere.
- Avoid stuffing queryable business data into JSON fields.
- Add `created`, `updated`, ownership, status, and audit fields where workflow matters.

For production-like work, implement schema as migrations and run them locally. Keep admin-dashboard edits only for exploration unless the user explicitly wants a no-code classroom path.

### 4.5. Add Phone SMS Identity When Needed

Use Aliyun Phone Number Verification Service/号码认证服务 for SMS code flows when the product needs reliable phone ownership:

- New user registration by phone number.
- SMS-code login or phone+password login after verification.
- Binding/changing a phone number on an existing account.
- High-trust forms where users must prove they own the submitted phone number.
- Password reset or sensitive account actions that require SMS verification.

Implement this in the PocketBase backend, never only in the frontend:

- Add locked/server-only audit collections for SMS challenges and verification attempts.
- Add custom Go routes for requesting and checking codes.
- Prefer Aliyun-generated verification codes: call `SendSmsVerifyCode` with the dynamic code placeholder such as `{"code":"##code##","min":"5"}`, set code-generation parameters such as `CodeType=1`, `CodeLength=6`, `ValidTime=300`, and `Interval=60`, then call `CheckSmsVerifyCode` to verify user input.
- Treat Aliyun-mode verification as successful only when the Aliyun check result is explicitly successful, including `Model.VerifyResult = PASS` in the current API.
- If the project intentionally uses PocketBase-generated codes and only sends them through Aliyun, do not call Aliyun verification as the source of truth; PocketBase must hash, expire, compare, limit, consume, and audit codes itself.
- Do not enable returning the verification code in production responses.
- Use the approved classroom defaults from `references/phone-sms-auth-aliyun.md`: default `SignName=速通互联验证码`; default `TemplateCode=100001` for register/login/form verification; `100002` for changing bound phone, `100003` for password reset, `100004` for binding a new phone, and `100005` for verifying a bound phone or sensitive account action.
- Create locked/server-only PocketBase config collections such as `system_sms_configs`, `system_sms_signatures`, and `system_sms_templates` so admins can maintain sign/template/purpose/rate-limit settings in the Dashboard.
- Keep Aliyun credentials in server environment variables, ECS RAM role, or encrypted server-side secret storage; never store AccessKey/Secret as ordinary editable text records.
- Never store raw SMS codes, never return whether a phone number already exists, and never expose Aliyun AccessKeys to the frontend.
- Rate limit by phone, IP, purpose, and user agent; add cooldowns and attempt limits.
- Log masked phone numbers or hashes only.

After verification, either create/login the user through the current PocketBase auth-token API, or require the user to set a password before enabling phone+password login. Read `references/phone-sms-auth-aliyun.md` before implementing.

### 5. Design API Rules Before UI

PocketBase API rules are the product's security boundary. Write them before relying on frontend checks.

Use deny-by-default thinking:

- Empty rule means public access; use it only intentionally.
- Locked or null rule means no client API access for that operation.
- Common authenticated rule: `@request.auth.id != ""`.
- Common owner rule: `owner = @request.auth.id`.
- Common role rule: `@request.auth.role = "admin"` when the auth collection has a trusted `role` field.
- Prevent client-side privilege changes with create/update rules that reject protected fields.

After every rule change, test as anonymous, normal user, record owner, non-owner, and admin/operator.

### 6. Build Realtime as a Product Feature

For every realtime surface, implement the loop:

1. Authenticate if the collection is private.
2. Fetch initial records with the same filter/sort used by the screen.
3. Subscribe with the JS SDK to the collection or record.
4. Merge `create`, `update`, and `delete` events into local state.
5. Unsubscribe on route/component cleanup.
6. Verify with two browsers, two users, or two tabs.

Do not call realtime "done" after only adding `subscribe`. The UI must visibly update without refresh and obey API rules.

Use realtime for:

- Live dashboards and kanban boards.
- Collaborative status, assignment, approval, and queue changes.
- Chat-like comments, notifications, logs, and progress updates.
- Admin/operator screens where immediate feedback matters.

Avoid realtime for high-volume analytics streams or data the current user cannot legitimately access.

### 7. Handle Files and Object Storage

Use PocketBase file fields for uploads. Keep local `pb_data` storage for simple local demos. Use S3-compatible storage when deployment needs durable external object storage, shared instances, or larger files.

For S3-compatible providers such as AWS S3, MinIO, Cloudflare R2, or Aliyun OSS S3-compatible endpoints:

- Configure endpoint, region, bucket, access key, secret key, and public/private behavior in PocketBase settings or environment-supported deployment setup.
- Keep credentials out of the repo and out of frontend code.
- Test upload, download, thumbnail/preview, and permission-restricted file access.
- Configure provider CORS when browser uploads or direct file reads require it.
- Back up metadata database and object storage together; one without the other is not a complete restore.
- For Aliyun OSS, use the official S3-compatible endpoint format such as `https://s3.oss-cn-hangzhou.aliyuncs.com`, match the bucket region, use RAM-created OSS AccessKey credentials, and do not hard-code permanent keys in browser code.

### 8. Implement Frontend with the SDK

Create one PocketBase client singleton, keep auth state in a central store/context, and pass records through typed or well-named data access functions.

Install:

```bash
npm install pocketbase
```

Client pattern:

```ts
import PocketBase from "pocketbase";

export const pb = new PocketBase(import.meta.env.VITE_PB_URL ?? "http://127.0.0.1:8090");
```

For non-engineer users, deliver clear screens: login, list/detail/editor, role-specific admin/operator views, and visible realtime behavior. Do not hide essential workflows behind raw API calls.

### 9. Validate Before Handoff

Run the smallest meaningful verification set:

- `go test ./...` when a Go project exists.
- `go run . migrate up` or the project's migration command.
- `go run . serve` or the project's dev server.
- Frontend typecheck/build/test commands present in the repo.
- API rule checks for anonymous, owner, non-owner, operator/admin.
- Realtime check in two sessions.
- Scheduled jobs checked in `Dashboard > Settings > Crons` when jobs are registered.
- Phone SMS auth checked for request-code, verify-code, register, login, phone binding, duplicate phone, rate limit, wrong code, expired code, and form verification scenarios when enabled.
- File upload/download and protected-file access checks.
- Local and server logs checked for startup errors, 4xx/5xx bursts, migration errors, realtime errors, file storage errors, and Caddy certificate/proxy errors.
- Backup/restore proof: built-in PocketBase backup configured or manual backup script tested; restore path documented for `pb_data`, SQLite database, local files, and external object storage.

### 10. Version, Release, and Deploy

For non-engineer users, proactively maintain source control:

- Initialize or connect the GitHub repository with `gh`.
- Create a branch for each meaningful feature or deployment change.
- Commit after runnable milestones: scaffold, migrations, auth rules, realtime flow, file storage, deployment config, and test fixes.
- Push commits to the remote branch after each milestone commit.
- Tag releases after a successful local build and remote smoke test.
- Never commit `.env`, `pb_data`, production database files, SSH keys, AccessKeys, or server secrets.
- Include project scripts when useful: `scripts/build.sh`, `scripts/deploy.sh`, `scripts/logs.sh`, `scripts/rollback.sh`, `scripts/backup.sh`, and `scripts/restore.sh`, with variables at the top for non-engineers to edit.

For deployment, prefer local build and remote operation:

- Build Go/PocketBase and frontend assets on macOS.
- Use SSH/rsync/scp to upload versioned artifacts to the Linux server.
- Run PocketBase as a systemd service under a dedicated non-root user.
- Enable autostart with systemd and verify `systemctl is-enabled pbapp caddy`.
- Put `pb_data` on persistent storage.
- Put Caddy in front of PocketBase and frontend assets; require a real domain, DNS A record, and open ports `80` and `443` for automatic HTTPS.
- Give the AI agent SSH access only through a dedicated key and least-privilege deployment user when possible; avoid sharing root passwords after bootstrap.
- Before deploys that include migrations, create a verified database/`pb_data` backup and warn that rollback may require restoring the backup, not only switching the binary symlink.
- After first server start, ensure the user creates a PocketBase superuser account immediately. PocketBase calls this `superuser`; classroom users may call it superadmin. Use the Web UI installer link from logs or run the current binary's `superuser create` command with the same `--dir` as the service.
- Configure operational safety: Caddy access logs with rotation, service logs via journald, PocketBase app logs in Dashboard > Logs, log retention/minimum level, SMTP, rate limiter, superuser IP whitelist, optional superuser MFA, PocketBase built-in backups, separate backup storage, and backup schedule.

In the final response, state what was built, repo/branch/tag status, where it lives, how to run it, server URL, HTTPS state, superuser setup status, log commands, backup/rollback commands, which checks passed, and any credentials or manual admin steps the user must perform privately.

## Teaching Style

When helping classroom users, be concrete and calm:

- Explain PocketBase as "a ready-made backend control room plus database and API".
- Show copy-paste commands with expected success signals.
- Convert vague product ideas into collections, fields, roles, and screens.
- Give the agent a checklist it can execute, not a lecture.
- Emphasize that admin UI, API rules, realtime, and file storage are core advantages, not side features.
- Keep engineering options visible for experienced engineers without forcing them on non-engineers.
