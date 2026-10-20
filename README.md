# SRL Discord Bridge

[English](README.md) | [简体中文](README.zh-CN.md) | [隐私与数据处理说明](PRIVACY.md)

A small self-hosted Cloudflare Worker used by SRL to save selected Discord messages into the user's local resource library.

The Bridge is intentionally isolated from the SRL application. Each user deploys their own Worker and D1 database in their own Cloudflare account.

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https%3A%2F%2Fgithub.com%2Fjixiangruyi117%2FSRL-Discord-Bridge)

## What it does

For Bot replies without a message Apps menu, copy the actual Discord attachment URL
and run `/下载直链 链接:<download URL>`. This slash command uses the same paired resource
queue and attachment deduplication as the message download command; it does not save
post text. Only Discord CDN URLs are accepted, not message links or arbitrary websites.
The URL validator matches SRL Android's share parser, including Discord attachment URL
path variants. Expired pasted links require a fresh URL; ephemeral messages cannot be
reread by the Bot. Register commands again after updating the Worker.

If you cannot use Discord's message Apps menu, copy the Bot message text and run
`/粘贴收件 正文内容:<copied message text>` in a Discord DM, server channel, or thread.
The command extracts Discord CDN attachment links and queues them in the paired library;
it does not save the pasted post text. It is a user-installed slash command, so the Bot
does not need access to the original server. Expired signed links must be copied again.
`/下载直链` remains available for a single copied attachment URL.

The unsupported `/保存首楼帖子` and `/保存所有已标注信息` commands have been removed.
Register commands again after deploying this update to remove any old entries from the
Discord App.

SRL supports paired temporary queues: `保存帖子到SRL（云端暂存）` saves a selected post;
`下载资源到SRL（云端暂存）` extracts supported Discord attachment links into a separate resource
queue. `/绑定资源库` uses a ten-minute one-time pairing code from SRL. Both queues
target the paired library and retain tasks for seven days. Legacy `保存到资源库`
and its one-time handoff remain available.

Cloud acceptance and local saving are separate states. SRL confirms a post only
after saving and reading it back locally. Reclaiming a task or using its manual
handoff does not create a second local copy of the same message.

Resource files use the existing Android WorkManager download transport or an
authenticated streaming Worker response for Web/iOS. Binary files are not stored
in D1. The local importer owns content deduplication and version decisions; a
downloaded file is not reported as imported until that importer commits.
iOS requires an open page. Android can continue an already claimed native download;
receiving new jobs and resuming parsing after process termination require reopening
SRL. No push or headless import service is included.

```text
Discord Message Context Command
  → your Cloudflare Worker
  → short-lived one-time D1 handoff
  → your local SRL
```

The Worker:

- verifies Discord interaction signatures with Ed25519;
- accepts only the selected message from the Message Context Command;
- stores the normalized capture in D1 under a high-entropy one-time token;
- expires handoffs after a short TTL;
- allows the handoff to be consumed once by SRL;
- does not act as a permanent Discord archive.

A user-installed Message Command receives the selected interaction snapshot even when the Bot is not a member of that community. To update saved content, run the save command on that message again. The Bridge provides no source-update checks or thread-history scan endpoints.

## Required values

Cloudflare deployment asks for the Discord values belonging to **your own Discord App**:

- `DISCORD_APPLICATION_ID`
- `DISCORD_PUBLIC_KEY`
- `DISCORD_BOT_TOKEN` — store this as a secret

The D1 binding name is fixed to `DB`.

## Configure the privacy policy

1. Read [PRIVACY.md](PRIVACY.md) and verify it matches your personal self-hosted instance, including logging and backups.
2. For the unchanged default implementation, use `https://github.com/jixiangruyi117/SRL-Discord-Bridge/blob/main/PRIVACY.md`. If your data handling differs, publish an accurate policy and use its public URL. Before serving other users, add an effective private data-request channel.
3. In Discord Developer Portal, select the **same application** that owns your Application ID and Bot Token, then open **General Information**.
4. Paste the policy link into **Privacy Policy URL**, click **Save Changes**, and confirm it saved. The Worker URL ending in `/interactions` belongs in the separate **Interactions Endpoint URL** field.

Publishing the policy does not publish your Bot or local collection. Do not upload messages, databases or credentials. Existing deployments can add this field without redeploying, reinstalling the Bot or pairing again.

## Routes

- `POST /interactions` — Discord Interactions Endpoint
- `GET /health` — checks that the D1 `handoffs` table is queryable and reports whether each Discord variable is present (never returns secret values)
- `GET /setup/status` — checks that the Bot Token is valid, verifies the configured Application ID and Public Key belong to that Discord App, and reports Message Command registration
- `POST /setup/register` — registers the legacy/post/resource Message Commands and the pairing Slash Command
- `GET /handoff/:token` — one-time SRL handoff
- `GET /open/:token` — opens SRL with the handoff token
- `GET /handoff/:token/status` — reads the handoff receipt state
- `POST /handoff/:token/ack` — confirms a paired post after local saving
- `POST /inbox/pair` — creates a library endpoint and one-time pairing code
- `GET /inbox/status` — checks the paired library endpoint
- `DELETE /inbox/pair` — revokes that endpoint
- `DELETE /inbox/cleanup` — clears completed post and resource cloud records
- `DELETE /inbox/cleanup-scoped` — clears only the requested `posts` or `resources` records
- `GET /inbox/jobs` — lists pending posts and recent receipts in bounded pages
- `GET /inbox/jobs/:id` — reads a post without consuming it before local saving
- `DELETE /inbox/jobs/:id` — cancels a post that is still pending and removes its temporary payload
- `POST /inbox/jobs/:id/ack` — confirms `saved` or `waiting_binding`
- `GET /inbox/waiting-sources` — lists saved posts awaiting a local resource binding
- `POST /inbox/sources/:sourceKeyHash/ack-bound` — updates receipts after local binding
- `GET /inbox/resources` — lists resource download tasks and recent receipts
- `GET /inbox/resources/:id` — reads a task and its validated/refreshed Discord URL
- `DELETE /inbox/resources/:id` — clears a completed, failed, or cancelled resource task
- `GET /inbox/resources/:id/file` — streams the authenticated CDN response without redirects
- `POST /inbox/resources/:id/ack` — updates local download/import/version-choice status

Authenticated inbox routes require the endpoint secret and `X-SRL-Library-ID`
header. Creating a pairing code requires the configured Bot Token. Tasks keep
their original target library even when the default pairing changes.

Resource downloads accept only HTTPS Discord CDN attachment paths. Refreshing an
expired URL requires Bot access to the original message; otherwise SRL asks the
user to resend. Apply migrations `0002_inbox.sql` and `0003_resources.sql` along
with `0001_handoffs.sql` before deploying. The existing deploy script applies all
pending migrations; Dashboard-generated code initializes the same schema.

Users normally do not need to type these routes. SRL derives them automatically from the Worker root URL. For Android and iOS web apps / PWAs, the `/open/:token` page can copy a temporary link to paste inside the app; this keeps the handoff in that app's own local storage context and does not require an SRL site URL setting. Large handoffs are split across temporary D1 rows, so there is no Bridge-specific per-handoff size cap; Cloudflare account and database quotas still apply.

The SRL deployment diagnostics use the expanded `/health` and `/setup/status` responses from this Worker. After syncing this repository into a GitHub fork, Cloudflare Workers Builds must deploy the updated production branch before those diagnostics can inspect the D1 migration table and compare Discord credentials. Older Bridge deployments continue to serve the existing interaction and handoff routes.

## Deploy without GitHub / GitLab

SRL also provides a browser-only Cloudflare deployment guide. It generates a Quick Editor version from this same Worker source and adds idempotent D1 schema initialization, so users without a Git account do not need Wrangler or manual SQL.

## Privacy boundary

Read the [privacy and data handling notice](PRIVACY.md) before configuring an instance.
It covers personal self-hosting, Cloudflare transit, local copies, revisions,
backups and deletion. It does not require publishing your real name or personal
account. Review it against your deployment, publish an accurate policy at a public
URL, and enter that URL in your own Discord App's **Privacy Policy URL** field.
If you serve other users, include a working private data-request contact and an
accessible policy link in your app; the upstream template alone cannot identify
or manage your instance. Never publish messages, credentials or deletion requests
containing private data with the policy.

This repository contains no SRL library data, no Discord credentials and no user content. Credentials are supplied by each user to their own Cloudflare deployment. Discord Message Context Command payloads are kept only long enough to complete the one-time handoff to the user's local SRL. Large payloads use multiple temporary D1 rows and are deleted under the same expiry and one-time-use rules. The Bridge can read up to two `.txt` attachments per message, at up to 1 MB each and 2 MB total per request, from Discord's HTTPS CDN; the client keeps that text in the local source record. Other attachments remain links.

Paired post captures and resource task metadata expire after seven days. Library
endpoint secrets are stored as hashes in D1. Downloaded binary files and local
resource bindings stay on the client; the Bridge stores no permanent resource files.

After updating the Worker, register commands in SRL connection settings to apply
the names. Registration renames the two previous cloud commands in place and
removes only their duplicate old names if both versions exist. Legacy direct
handoff and unrelated commands are preserved. Already cached old cloud command
interactions remain accepted. Worker backend updates alone do not require a new
SRL APK or Discord App installation.

The handoff page provides a compact progress receipt, an Android deep link and
copy-to-Web/PWA actions. Status refresh does not claim the post. If clipboard
access fails, a selectable temporary link remains available.

After SRL confirms that a post has been saved and read back locally, the Worker
deletes its cloud body, attachment metadata and all handoff payload chunks/aliases
atomically with the saved/waiting-binding receipt. After a resource is fully
imported, its download URL and Discord channel/message identifiers are cleared.
Downloading, failed imports and pending version choices retain their transport
data for retry. Small status/deduplication receipts keep their existing expiry;
completed tasks no longer provide a body or download URL, and repeated sharing
does not restore the cleared data. Older completed tasks are also cleared by the
next cleanup-triggering request. An hourly Cron also deletes expired payloads,
tasks, pairing codes and receipts without user activity. Expiry deletion runs
independently of completed-data cleanup, so one failure does not prevent the other
cleanup branches. Database failures are retried on the next scheduled run and
incomplete runs fail visibly in Cloudflare. Active library pairings remain available.

## Development

```bash
npm ci
npm run typecheck
```

For a normal Wrangler deployment:

```bash
npm run deploy
```

The deploy script applies D1 migrations and then deploys the Worker.

For automatic deployment, connect this repository to the Worker with Cloudflare Workers Builds,
use `main` as the production branch, `npm run typecheck` as the build command, and
`npm run deploy` as the deploy command. Keep non-production branch deployments disabled unless
you explicitly need preview Workers. Discord credentials remain Cloudflare secrets and must never
be committed to Git.

> **Deployment setup:** The production Worker is connected to this repository's `main` branch.
> Cloudflare Workers Builds runs `npm run typecheck` and then `npm run deploy`; preview builds are
> disabled. The first post-reconnection build succeeded: typecheck passed, no D1 migrations were
> pending, and Worker version `a7835cb5-73bd-417e-962d-d7d4d3685f1d` was deployed.

After updating this repository and SRL, use SRL's connection settings to register
the Discord commands again. Deploying the Worker alone does not register commands
with Discord. Update the Android shell before enabling native resource intake.

This repository does not provide a shared production Worker. Deploy the Worker to your own Cloudflare account and configure your Worker root URL in SRL.
