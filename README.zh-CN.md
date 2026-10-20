# SRL Discord Bridge 中文说明

[English](README.md) | [简体中文](README.zh-CN.md) | [隐私与数据处理说明](PRIVACY.md)

SRL Discord Bridge 是一个个人自部署的 Cloudflare Worker，用来把你主动选中的 Discord 帖子和资源送到自己的 SRL 本地资源库。每个人使用自己的 Discord App、Cloudflare Worker 和 D1 数据库，项目不提供统一的公共消息中转服务。

## 可以做什么

- **保存帖子**：在 Discord 消息的应用菜单中选择 `保存帖子到SRL（云端暂存）`，暂存选中消息的正文、来源和附件记录，再由 SRL 领取、保存并关联本地资源。
- **下载资源**：选择 `下载资源到SRL（云端暂存）`，将选中消息里的 Discord 附件加入独立下载队列。文件下载后仍需解析并完成本地导入，才算资源已入库。
- **粘贴收件**：没有消息应用菜单时，在私聊、频道或线程运行 `/粘贴收件 正文内容:<复制的文字>`。它只提取 Discord 附件链接，不把粘贴正文保存为帖子。
- **下载单条直链**：运行 `/下载直链 链接:<Discord 附件下载地址>`。只接受 Discord CDN 附件地址，不接受普通网站或帖子链接；过期链接需要重新复制。
- **配对资源库**：在 SRL 连接设置中生成一次性配对码，再用 `/绑定资源库` 将命令的收件目标绑定到那份库。配对码十分钟有效；任务保持原目标，不会因后来更换默认配对而转到另一份库。

旧的 `保存到资源库` 一次性交付方式仍可使用。当前不支持的 `/保存首楼帖子`、`/保存所有已标注信息` 已从注册列表移除；更新 Worker 后重新注册命令，才能清除 Discord 中的旧菜单项。

## 数据怎样流转

```text
你在 Discord 主动选择消息或附件
  → 自己的 Cloudflare Worker / D1 临时队列
  → 自己的 SRL 领取并保存到本机
  → 本机保存成功并读回校验后，确认并清理相应云端载荷
```

“云端已暂存”不等于“本机已保存”。帖子正文与资源附件使用不同队列；重复领取或手动交付回退复用已有去重逻辑。

Android 已领取的资源使用现有 WorkManager 下载通道；Web/iOS 可以通过经过认证的 Worker 响应流式下载。D1 不永久保存资源二进制文件。iOS 网页需要保持运行；Android 收取新任务及进程结束后的解析恢复仍受应用生命周期限制，本项目不提供云端推送或无界后台导入。

更新已保存内容时，请回到 Discord，对同一消息再次执行保存命令。SRL 会合并到已有来源，不重复创建。Bridge 不提供检查来源更新或扫描帖子历史的接口。

## 部署与配置

开始前请阅读[隐私与数据处理说明](PRIVACY.md)，核对自己部署的数据范围、日志和备份设置。公开隐私说明不等于公开 Bot 或保存的数据，也不要求公开真名、私人邮箱或个人账号。将符合实际的说明发布到公开可读的网址，并在自己的 Discord App 中填写 **Privacy Policy URL**；以后向他人提供服务时，还需要有效的私密数据请求渠道和应用内入口。

### 方式一：从 Git 仓库部署

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https%3A%2F%2Fgithub.com%2Fjixiangruyi117%2FSRL-Discord-Bridge)

在 SRL 的“功能 → 收件箱 → 连接设置”打开一键部署或 GitHub 部署教程，使用自己的仓库和 Cloudflare 账号。

准备同一个 Discord App 的三个值：

- `DISCORD_APPLICATION_ID`：Developer Portal → General Information → Application ID。
- `DISCORD_PUBLIC_KEY`：同一页面的 Public Key。
- `DISCORD_BOT_TOKEN`：Bot 页面生成的 Bot Token，在 Cloudflare 中保存为 Secret，不要提交到 Git。

D1 的绑定名称固定为 `DB`。Cloudflare Workers Builds 的生产分支使用 `main`，构建命令为 `npm run typecheck`，部署命令为 `npm run deploy`；部署脚本会先应用待执行的 D1 迁移。除非确实需要预览 Worker，否则关闭非生产分支部署。

自己 Fork 的仓库不会自动同步上游更新；需要先同步自己的生产分支，Cloudflare 才会部署那份内容。

### 方式二：浏览器手动部署

不使用 GitHub/GitLab 或命令行时，可在 SRL 连接设置中打开“浏览器手动部署”教程。教程从相同 Worker 源码生成 Cloudflare Quick Editor 版本，并补上幂等 D1 建表初始化；按步骤配置自己的数据库、三个 Discord 参数和 Secret。

### 部署后的连接

1. 将自己的 Worker 根网址填入 SRL 的 Cloudflare 部署链接，测试连接并检查部署配置。
2. SRL 会生成 Interactions Endpoint URL，即 Worker 根网址加 `/interactions`；将它填入同一个 Discord App 的对应字段。
3. 按 SRL 教程完成 Discord App 的安装授权，然后在连接设置中注册命令并检查状态。
4. 配对本机资源库后，回 Discord 主动保存消息或下载附件，再到 SRL 收件箱确认领取结果。

健康检查 `/health` 和配置诊断 `/setup/status` 用于检查部署；它们不能替代本地保存结果。路由清单与接口细节见[英文说明的 Routes](README.md#routes)。

## 保留与删除

- 普通一次性交付默认二十分钟有效；已配对帖子和资源任务七天有效。完成本地保存并确认后，会清除相应云端正文或资源下载链接；部分状态、标题或文件名等回执保留至原有效期结束。
- 小时定时任务和请求触发的清理会删除到期内容。数据库故障或缺少定时任务可能延迟物理删除；配对记录不随任务到期全部删除。
- **Discord 原帖被编辑或删除，不会反向删除已经保存的本地副本。** 旧版已保存的历史版本仍可查看和恢复；本地内容、独立附件、已导入资源及备份需要分别管理。
- 收件箱“清理云端”不删除本机。永久删除整个本地来源会删除其消息、历史版本和资源关联，但不会替你清理所有独立资源、导出及备份副本。
- 不要公开 Bot Token、配对码、库密钥或未过期临时链接。临时链接含领取凭据，获得链接的人可能读取对应内容。

完整数据范围、删除入口、数据请求及各实例的责任见[隐私说明](PRIVACY.md)。个人收藏仍须遵守作者授权、社区规则及 Discord 条款。

## 开发与命令行部署

```bash
npm ci
npm run typecheck
```

确认 Wrangler 指向自己的目标账号、Worker、D1 和 Secret 后，运行：

```bash
npm run deploy
```

该命令会执行远端 D1 迁移和 Worker 部署。仅修改本地文件或通过类型检查，并不代表线上已更新；Worker 部署也不会自动注册 Discord 命令。
