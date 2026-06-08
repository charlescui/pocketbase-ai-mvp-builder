# PocketBase 二次开发 Agent 提示词模板

把下面模板发给 AI agent。将方括号内容替换成你的产品信息；不知道的地方就写“不确定，请帮我判断”。

```markdown
你是一名擅长 PocketBase 二次开发的全栈工程 agent。请基于 PocketBase 为我实现一个小型 MVP/内部系统，不要只给教程，要在当前工作区创建可运行项目、代码、迁移、前端页面和验证步骤。

## 我的系统想法

- 系统名称：[例如：活动报名管理系统 / 设备巡检系统 / 客诉处理看板]
- 使用人群：[例如：运营、设计师、测试同学、客户、管理员]
- 主要目标：[这个系统最重要解决什么问题]
- 核心流程：[用户从进入系统到完成任务的大概步骤]
- 访问量预期：[本地演示 / 少数内部用户 / 小规模真实用户]
- 上线目标：[只本地演示 / 部署到阿里云 ECS / 暂不确定]
- 域名：[例如 app.example.com，没有就写没有]
- 登录注册方式：[手机号短信注册登录 / 邮箱密码 / 管理员邀请 / 不确定]
- 手机号验证场景：[注册登录 / 绑定手机号 / 提交表单前验证 / 找回密码 / 不需要 / 不确定]
- 是否需要定时任务：[例如每天发报表 / 每小时同步 / 自动清理 / 不需要 / 不确定]

## 角色和权限

- 匿名用户能做什么：[例如：查看公开活动 / 什么都不能做]
- 登录用户能做什么：[例如：创建自己的记录、上传文件、查看自己的状态]
- 运营/管理员能做什么：[例如：审核、分配、导出、修改状态]
- 是否需要组织/部门隔离：[需要/不需要/不确定]

## 数据和页面

- 需要管理的数据：[列出对象，例如活动、报名、任务、附件、评论]
- 需要上传的文件：[图片、PDF、Excel、录音、无]
- 需要的页面：[登录页、列表页、详情页、编辑页、管理看板等]
- 是否需要移动端适配：[需要/不需要]
- 数据备份要求：[每天备份 / 重要数据需要 30 天备份 / 本地演示可简单备份 / 不确定]

## 实时同步

请特别利用 PocketBase realtime。以下场景希望不用刷新就更新：

- [例如：管理员修改报名状态后，用户页面立即变化]
- [例如：看板中新增任务、状态拖动、评论新增实时出现]
- [例如：后台处理队列实时刷新]

## 技术要求

- 使用 PocketBase 作为后端基础。
- 默认使用 Go 扩展 PocketBase，所有正式 schema 变更写成迁移。
- 默认使用官方 PocketBase JS SDK 连接前端。
- 在 macOS 本地可运行，并给出安装/启动命令。
- 如果网络在国内，请配置 Go 和 NPM 代理：GOPROXY、GOSUMDB、npm registry。
- 请安装并使用 GitHub CLI `gh` 维护仓库、分支、提交、tag 和 release。
- 权限必须通过 PocketBase API rules 实现，不要只在前端隐藏按钮。
- 如果系统面向真实用户，请默认把手机号作为基础身份能力，但不要把手机号当作唯一登录方式；PocketBase 仍可支持邮箱密码、管理员邀请、Google/GitHub OAuth 等。优先使用阿里云生成并核验验证码：服务端调用 `SendSmsVerifyCode`，模板参数使用动态占位例如 `{"code":"##code##","min":"5"}`，并设置 `CodeType=1`、`CodeLength=6`、`ValidTime=300`、`Interval=60` 等生成和频控参数；服务端再调用 `CheckSmsVerifyCode` 校验用户输入。只有阿里云校验结果明确成功，包括 `Model.VerifyResult = PASS`，才能注册、登录、绑定手机号或通过强身份表单。默认签名用 `速通互联验证码`，注册/登录/通用表单用模板 `100001`，修改绑定手机号用 `100002`，重置密码用 `100003`，绑定新手机号用 `100004`，验证绑定手机号/敏感操作用 `100005`。请创建 locked/server-only 的 `system_sms_configs`、`system_sms_signatures`、`system_sms_templates` 后台配置表来维护签名、模板、purpose 和频控参数；AccessKey/Secret 默认放服务端环境变量、ECS RAM 角色或加密后的服务端 secret，不要作为普通明文字段。如果改成 PocketBase 自己生成验证码，只能把阿里云当短信发送通道，PocketBase 必须自己完成哈希存储、过期、次数限制和比对。生产环境不要让验证码返回到接口响应或日志里。
- 手机号短信认证必须由 PocketBase Go 后端实现，不能只做前端页面。AccessKey 不能进前端、不能进 GitHub，验证码不能明文入库或写日志。
- 文件上传使用 PocketBase file field；如果需要对象存储，请按阿里云 OSS 的 S3 兼容方案配置，endpoint 示例：`https://s3.oss-cn-hangzhou.aliyuncs.com`，并提醒我不要泄漏 AccessKey。
- 如果部署到服务器，请使用阿里云 ECS + Caddy。Caddy 必须负责 HTTPS，使用免费自动证书。
- 本机负责开发、测试、编译和打包；Linux 服务器负责运行。请通过 SSH/rsync/scp 部署。
- 如果需要你操作 Linux 服务器，我会提供 SSH host、用户名和密钥路径；请先说明你要执行的远程命令，涉及删除数据或改防火墙时先征求确认。
- 如果我需要定时任务，请使用 PocketBase 内置 `app.Cron()`，并在 `Dashboard > Settings > Crons` 和日志里验证，不要默认让我学习 Linux cron。
- 数据库备份很重要。请配置 PocketBase `Dashboard > Settings > Backups`，说明本地备份、S3-compatible 备份、阿里云 OSS 独立备份 bucket 的建议；部署前必须生成备份，恢复步骤也要写出来。
- 如果启用 settings encryption，请提醒我单独保存 `PB_ENCRYPTION_KEY`，不要把它放进 GitHub。

## GitHub 和服务器信息

- GitHub 是否已登录 gh：[已登录 / 未登录 / 不确定]
- 希望仓库名：[例如 my-pocketbase-app]
- 仓库可见性：[private / public]
- ECS 公网 IP：[例如 1.2.3.4，没有就写没有]
- SSH 连接方式：[例如 ssh deploy@app.example.com 或 ssh -i ~/.ssh/xxx root@1.2.3.4]
- Linux 系统：[Ubuntu 24.04 / Debian 12 / 不确定]
- CPU 架构：[x86_64 / arm64 / 不确定]
- 阿里云 OSS：[不需要 / 需要，bucket 和 region 是 xxx / 不确定]
- 备份存储：[PocketBase 本地备份 / 阿里云 OSS 独立 bucket / 暂不确定]
- 阿里云号码认证服务：[已开通 / 未开通 / 不确定]
- 短信签名/模板/方案名称：[已有，分别是 xxx / 没有 / 不确定]

## 请按这个顺序工作

1. 如果信息不足，先问最多 8 个关键问题；能合理假设的就直接假设并标注。
2. 输出产品结构：角色、数据表/collection、字段、权限规则、页面、realtime 流程。
3. 搭建本地开发环境和项目结构，包括 `gh`。
4. 初始化或连接 GitHub 仓库；创建合理分支；每个可运行里程碑主动 commit 并 push。
5. 创建 PocketBase 后端：Go 项目、迁移、必要 hooks/custom routes、启动命令。
6. 如果需要真实用户账号，请实现手机号短信注册/登录：users 手机号字段、sms_challenges 审计表、发送验证码接口、校验验证码接口、注册/登录/绑定手机号流程、限流和日志。
7. 创建前端：手机号注册登录、核心业务页面、实时同步、文件上传。
8. 做权限测试：匿名、普通用户、数据拥有者、非拥有者、管理员。
9. 做短信认证测试：正常注册、正常登录、错误验证码、过期验证码、重复注册、频控、手机号绑定、强身份表单验证。
10. 做 realtime 测试：两个浏览器/两个账号同时打开，验证不刷新同步。
11. 如果需要定时任务，用 PocketBase `app.Cron()` 实现；给出 cron 表达式、job id、日志和 Dashboard 验证方式。
12. 做文件测试：上传、预览/下载、受保护访问；如果用 OSS，验证阿里云 S3 兼容上传下载。
13. 配置数据库备份：PocketBase 内置 Backups、部署前备份脚本、恢复脚本、OSS 备份提醒；至少做一次手动备份验证。
14. 本机编译打包，然后部署到阿里云 ECS：systemd 运行 PocketBase，Caddy 反向代理并自动 HTTPS。
15. 远程 smoke test 成功后打 tag，必要时创建 GitHub release。
16. 最后给我：运行命令、GitHub repo/branch/commit/tag、服务器 URL、测试结果、手机号认证方案、账号创建方式、定时任务说明、备份文件名和恢复方式、已知限制、下一步建议。

## 交付偏好

- 请用中文解释关键决策。
- 命令要可复制。
- 对非开发同学友好，但工程实现要专业。
- 遇到 PocketBase 版本/API 差异时，请先查官方文档或仓库，再调整代码。
```

## 课堂用极简版本

```markdown
请使用 PocketBase 帮我做一个 [系统名称]。你要像全栈工程师一样直接创建可运行项目：macOS 环境配置、GitHub CLI 仓库和分支维护、Go 扩展版 PocketBase 后端、collections/migrations、API rules 权限、手机号短信注册登录、阿里云号码认证服务 SendSmsVerifyCode/CheckSmsVerifyCode、官方 JS SDK 前端、realtime 实时同步、PocketBase app.Cron 定时任务、文件上传/阿里云 OSS S3 兼容存储说明、数据库备份和恢复、测试和运行命令都要完整。本机开发编译，部署到阿里云 ECS，用 Caddy 自动申请免费 HTTPS 证书。我的用户是 [角色]，核心数据是 [数据对象]，最重要的实时场景是 [实时场景]，需要的手机号验证场景是 [注册登录/表单验证/无]，需要的定时任务是 [定时任务，没有就写无]。如果信息不足，先问最多 5 个问题，然后开始实现。
```
