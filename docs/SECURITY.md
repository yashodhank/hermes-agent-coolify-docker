# SECURITY.md

Security policy and hardening guide for the Hermes Agent Coolify Stack.

This document is written for operators deploying Hermes Agent, Hermes Workspace, Honcho Memory, Browserless, Webhooks, Telegram, Discord, Email, and optional Open WebUI through Coolify v4.

---

## 1. Security Philosophy

This stack is powerful because it connects an AI agent to memory, messaging platforms, browser automation, webhooks, dashboards, and external LLM providers. That also means a weak deployment can become a serious security risk.

The safest default model is:

| Component          |  Public Exposure | Protection                       |
| ------------------ | ---------------: | -------------------------------- |
| Hermes Workspace   |              Yes | Workspace password + HTTPS       |
| Hermes Dashboard   |         Optional | Basic Auth proxy only            |
| Hermes Gateway API |               No | Internal Docker network only     |
| Hermes Webhooks    |              Yes | HMAC secret / route secret       |
| Honcho API         |               No | Internal Docker network only     |
| Honcho PostgreSQL  |               No | Internal Docker network only     |
| Redis              |               No | Internal Docker network only     |
| Browserless        |               No | Internal only + token            |
| Open WebUI         |         Optional | Open WebUI auth + HTTPS          |
| Telegram           |     External bot | Allowed user IDs                 |
| Discord            |     External bot | Allowed users / roles / channels |
| Email              | External mailbox | App password + allowed senders   |

Primary rule:

> Expose only what a human or external service must reach. Everything else should stay inside the Docker network.

---

## 2. Threat Model

### Main assets to protect

| Asset                                 | Why it matters                                     |
| ------------------------------------- | -------------------------------------------------- |
| `SERVICE_PASSWORD_64_HERMESBRIDGE`    | Gives access to Hermes gateway API                 |
| `SERVICE_PASSWORD_64_HERMESWORKSPACE` | Logs into Workspace                                |
| `SERVICE_PASSWORD_64_HERMESBROWSER`   | Controls Browserless / browser automation          |
| `SERVICE_PASSWORD_64_HERMESWEBHOOKS`  | Authenticates webhook requests                     |
| Provider API keys                     | Can create direct financial cost and data exposure |
| Telegram / Discord bot tokens         | Can allow impersonation or bot takeover            |
| Email credentials                     | Can allow mailbox takeover                         |
| Honcho PostgreSQL data                | Long-term memory, potentially sensitive            |
| Hermes data volume                    | Config, sessions, skills, state, runtime artifacts |
| Workspace override file               | Can override connection/security settings          |

### Primary attackers

| Attacker                                    | Risk                                  |
| ------------------------------------------- | ------------------------------------- |
| Internet scanner                            | Finds exposed dashboards/APIs         |
| Curious user with URL                       | Attempts dashboard or Workspace login |
| Compromised webhook sender                  | Sends malicious payloads              |
| Compromised Telegram/Discord user           | Uses bot as remote agent interface    |
| Leaked `.env` / logs                        | Full stack compromise                 |
| Malicious web page visited by browser agent | Browser/session abuse                 |
| Over-permissive email integration           | Agent triggered by unwanted senders   |

---

## 3. Public Surface Area

### Recommended public URLs

| Public URL         |          Should exist? | Notes                                  |
| ------------------ | ---------------------: | -------------------------------------- |
| Workspace URL      |                    Yes | Main user interface                    |
| Dashboard URL      |               Optional | Must be behind Basic Auth              |
| Webhook URL        | Yes, if using webhooks | Must use HMAC / route secret           |
| Open WebUI URL     |               Optional | Must have Open WebUI auth enabled      |
| Browserless URL    |                     No | Keep internal unless debugging briefly |
| Hermes Gateway URL |                     No | Do not expose `8642` directly          |
| Honcho URL         |                     No | Keep internal                          |

### Never expose directly

Do not create public Coolify URLs for these internal services:

```yaml
hermes:8642
honcho-api:8000
honcho-database:5432
honcho-redis:6379
hermes-browser:3000
```

If you must expose Browserless for manual login/debugging, do it temporarily, use a strong token, and remove the public URL after use.

---

## 4. Coolify Security Rules

### 4.1 Use Coolify magic secrets

Use Coolify-generated values instead of hand-written weak secrets.

Recommended:

```env
SERVICE_PASSWORD_64_HERMESBRIDGE
SERVICE_PASSWORD_64_HERMESWORKSPACE
SERVICE_PASSWORD_64_HERMESBROWSER
SERVICE_PASSWORD_64_HERMESWEBHOOKS
SERVICE_PASSWORD_HONCHODB
```

Avoid:

```env
HERMES_BRIDGE_KEY=hermes_secure_bridge_key_123
WORKSPACE_PASSWORD=admin123
BROWSER_TOKEN=
```

### 4.2 Avoid `container_name`

Do not use fixed container names:

```yaml
container_name: hermes
```

Coolify manages naming. Fixed names can break redeployments, previews, duplicated stacks, and migrations.

### 4.3 Avoid host `ports:` unless deliberately bypassing Coolify

Prefer:

```yaml
expose:
  - "3000"
```

Avoid:

```yaml
ports:
  - "3000:3000"
```

`ports:` may expose services directly through the Docker host and bypass Coolify proxy protections.

### 4.4 Avoid custom Docker networks

Coolify creates and manages the deployment network. Custom networks can cause proxy routing problems and may accidentally expose or isolate services incorrectly.

### 4.5 Use clean magic variable identifiers

Good:

```yaml
- SERVICE_URL_HERMESWORKSPACE_3000
```

Avoid identifiers with underscores before port suffix:

```yaml
- SERVICE_URL_HERMES_WORKSPACE_3000
```

---

## 5. Dashboard Protection

The Hermes dashboard is sensitive. It can expose sessions, configuration, jobs, and API-key-adjacent controls.

### 5.1 Do not expose dashboard directly from `hermes`

Bad model:

```text
Internet -> hermes:9119 directly
```

Good model:

```text
Internet -> hermes-dashboard-proxy:8080 -> hermes:9119
                 |
                 +-- Basic Auth middleware
```

### 5.2 Why use a separate dashboard proxy?

Because Basic Auth middleware should protect only the dashboard, not webhooks.

If Basic Auth is applied directly to the main `hermes` service, webhook callbacks may also receive Basic Auth challenges and fail.

Correct separation:

```text
hermes-dashboard-proxy -> Basic Auth -> hermes:9119
hermes-webhook-proxy   -> No Basic Auth -> hermes:8644
```

### 5.3 Generate Basic Auth credentials

Install htpasswd:

```bash
# macOS
brew install httpd

# Ubuntu / Debian
sudo apt-get update
sudo apt-get install -y apache2-utils
```

Generate:

```bash
htpasswd -nbB admin 'your-strong-dashboard-password'
```

Example output shape:

```text
admin:$2y$05$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Add to Coolify Developer View:

```env
DASHBOARD_BASIC_AUTH_USERS=admin:$2y$05$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

In `.env` / Coolify Developer View, keep single `$` characters. If you place the hash directly inside a Compose label instead of using `.env`, escape `$` as `$$`.

### 5.4 Dashboard checklist

* [ ] Dashboard is exposed only through `hermes-dashboard-proxy`.
* [ ] `DASHBOARD_BASIC_AUTH_USERS` is set.
* [ ] Dashboard URL asks for username/password.
* [ ] Main `hermes` service does not have a direct public dashboard URL unless intentionally used for debugging.
* [ ] Webhooks still work without Basic Auth.

---

## 6. Workspace Security

Workspace is the main public UI.

Required:

```env
HERMES_PASSWORD=${SERVICE_PASSWORD_64_HERMESWORKSPACE}
COOKIE_SECURE=1
TRUST_PROXY=1
HOST=0.0.0.0
PORT=3000
```

### 6.1 Cookie behavior

For Coolify HTTPS production:

```env
COOKIE_SECURE=1
TRUST_PROXY=1
```

For local HTTP-only testing:

```env
COOKIE_SECURE=0
```

Do not use `COOKIE_SECURE=0` in production.

### 6.2 Workspace override risk

Workspace may persist connection settings in:

```text
/home/workspace/.hermes/workspace-overrides.json
```

Bad overrides can silently override Compose values.

If Workspace behaves strangely after changing Compose envs, clear it:

```bash
rm -f /home/workspace/.hermes/workspace-overrides.json
```

Restart `hermes-workspace` afterward.

### 6.3 Dashboard token 401 issue

If Workspace shows:

```text
Failed to load sessions.
Hermes Agent dashboard /api/sessions?limit=50&offset=0: 401 {"detail":"Unauthorized"}
```

Do not set these unless you know the exact dashboard bearer:

```env
HERMES_DASHBOARD_TOKEN=
CLAUDE_DASHBOARD_TOKEN=
```

Safe default:

```env
HERMES_DASHBOARD_URL=http://hermes:9119
```

No dashboard token.

---

## 7. Hermes Gateway API Security

The Hermes gateway runs internally on:

```text
http://hermes:8642
```

It should not be publicly exposed.

Required shared token wiring:

```env
API_SERVER_KEY=${SERVICE_PASSWORD_64_HERMESBRIDGE}
HERMES_API_TOKEN=${SERVICE_PASSWORD_64_HERMESBRIDGE}
CLAUDE_API_TOKEN=${SERVICE_PASSWORD_64_HERMESBRIDGE}
```

### 7.1 OpenAI-compatible endpoint

Internal endpoint:

```text
http://hermes:8642/v1
```

For another container on the same Docker host but outside the stack:

```text
http://host.docker.internal:8642/v1
```

Required Compose support:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Do not expose `8642` to the internet unless you add strong external authentication, IP restrictions, and rate limiting.

---

## 8. Webhook Security

Hermes webhook listener runs internally on:

```text
http://hermes:8644
```

Public webhook traffic should go through:

```text
hermes-webhook-proxy:8080
```

### 8.1 Required secret

Use Coolify-generated secret:

```env
WEBHOOK_SECRET=${SERVICE_PASSWORD_64_HERMESWEBHOOKS}
```

### 8.2 Webhook endpoint shape

```text
https://your-webhook-url/webhooks/<route-name>
```

### 8.3 Webhook hardening checklist

* [ ] Use HMAC / route secret.
* [ ] Do not disable webhook verification.
* [ ] Keep webhook proxy separate from dashboard proxy.
* [ ] Limit request body size in Nginx.
* [ ] Route only required webhook paths.
* [ ] Do not log full payloads if they contain secrets.
* [ ] Rotate webhook secret if leaked.

### 8.4 Example Nginx hardening for webhook proxy

```nginx
client_max_body_size 2m;
proxy_read_timeout 60s;
proxy_send_timeout 60s;
```

If your provider sends large payloads, increase `client_max_body_size` intentionally.

---

## 9. Browserless / Chromium Security

Browserless can control a browser session. Treat it as sensitive.

Internal endpoint:

```text
ws://hermes-browser:3000/chromium?token=<SERVICE_PASSWORD_64_HERMESBROWSER>
```

Do not expose Browserless publicly by default.

### 9.1 Risks

| Risk                 | Impact                      |
| -------------------- | --------------------------- |
| Browser token leak   | Remote browser control      |
| Public debug UI      | Manual browser takeover     |
| Persistent cookies   | Account/session compromise  |
| Malicious pages      | Browser exploit / data leak |
| Untrusted automation | SSRF / unwanted browsing    |

### 9.2 Recommended settings

```env
BROWSER_KEEP_ALIVE=true
BROWSER_CONCURRENT=3
BROWSER_QUEUED=3
BROWSER_TIMEOUT_MS=300000
BROWSER_INACTIVITY_TIMEOUT=900
```

### 9.3 Public debug exception

Only if needed, temporarily add:

```yaml
- SERVICE_URL_HERMESBROWSER_3000
```

Open:

```text
https://browserless-url/?token=<SERVICE_PASSWORD_64_HERMESBROWSER>
```

Remove the public URL after debugging.

---

## 10. Honcho Memory Security

Honcho stores long-term memory. Treat its database as sensitive.

Internal Hermes setting:

```env
HONCHO_BASE_URL=http://honcho-api:8000
```

Honcho should not be public.

### 10.1 Honcho database

PostgreSQL password:

```env
SERVICE_PASSWORD_HONCHODB
```

Database service should not expose host ports.

### 10.2 Honcho provider keys

Honcho requires at least one LLM provider key:

```env
HONCHO_LLM_OPENAI_API_KEY=
HONCHO_LLM_ANTHROPIC_API_KEY=
HONCHO_LLM_GEMINI_API_KEY=
```

For OpenRouter:

```env
HONCHO_LLM_OPENAI_API_KEY=sk-or-v1-...
HONCHO_OPENAI_BASE_URL=https://openrouter.ai/api/v1
HONCHO_MODEL=openai/gpt-4.1-mini
HONCHO_EMBEDDING_MODEL=openai/text-embedding-3-small
```

For local OpenAI-compatible provider on host:

```env
HONCHO_OPENAI_BASE_URL=http://host.docker.internal:8000/v1
```

### 10.3 Memory data risks

Honcho may store:

* User preferences
* Conversation-derived facts
* Agent observations
* Project details
* Sensitive personal/business context

Do not expose Honcho API publicly unless you add strong authentication, HTTPS, and authorization boundaries.

---

## 11. Telegram Security

Telegram bot token gives control of the bot.

Required:

```env
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_USERS=
```

Example:

```env
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ
TELEGRAM_ALLOWED_USERS=123456789,987654321
```

Use numeric Telegram user IDs, not usernames.

### Telegram checklist

* [ ] Bot token stored only in Coolify env.
* [ ] `TELEGRAM_ALLOWED_USERS` is not blank.
* [ ] Group chat IDs are restricted if using groups.
* [ ] Bot privacy mode reviewed for group behavior.
* [ ] Token rotated if accidentally committed.

Optional:

```env
TELEGRAM_GROUP_ALLOWED_USERS=
TELEGRAM_GROUP_ALLOWED_CHATS=
```

---

## 12. Discord Security

Required:

```env
DISCORD_BOT_TOKEN=
DISCORD_ALLOWED_USERS=
```

Optional:

```env
DISCORD_ALLOWED_ROLES=
DISCORD_HOME_CHANNEL=
```

### Discord checklist

* [ ] Bot has only required permissions.
* [ ] Allowed users/roles are set.
* [ ] Home channel is restricted.
* [ ] Bot token is never committed.
* [ ] Bot token rotated if leaked.

---

## 13. Email Security

Email credentials can expose an entire mailbox.

Required:

```env
EMAIL_ADDRESS=
EMAIL_PASSWORD=
EMAIL_IMAP_HOST=
EMAIL_SMTP_HOST=
```

Recommended:

```env
EMAIL_ALLOWED_USERS=
EMAIL_ALLOW_ALL_USERS=false
```

### Email provider guidance

Use app passwords where available. Do not use your primary mailbox password.

Gmail / Google Workspace example:

```env
EMAIL_ADDRESS=agent@example.com
EMAIL_PASSWORD=google-app-password
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_IMAP_PORT=993
EMAIL_SMTP_PORT=587
EMAIL_ALLOWED_USERS=you@example.com
EMAIL_ALLOW_ALL_USERS=false
```

### Email checklist

* [ ] App password used instead of main password.
* [ ] IMAP enabled only if required.
* [ ] Allowed senders are restricted.
* [ ] Mailbox does not contain unnecessary sensitive data.
* [ ] Password rotated if logs or env are exposed.

---

## 14. Open WebUI Security

Open WebUI is optional.

Same-stack Hermes endpoint:

```env
OPENWEBUI_OPENAI_API_BASE_URL=http://hermes:8642/v1
OPENAI_API_KEY=${SERVICE_PASSWORD_64_HERMESBRIDGE}
```

Separate container on same host:

```env
OPENWEBUI_OPENAI_API_BASE_URL=http://host.docker.internal:8642/v1
```

### Open WebUI checklist

* [ ] Open WebUI auth is enabled.
* [ ] Admin account password is strong.
* [ ] Hermes API key is not shown to normal users.
* [ ] Open WebUI is not configured to expose unwanted models/providers.
* [ ] If env values changed after first launch, update connection in Admin UI or reset Open WebUI volume.

---

## 15. Secrets Handling

### Never commit

Do not commit:

```text
.env
.env.production
.env.local
*.key
*.pem
*.p12
*.pfx
secrets.json
workspace-overrides.json
```

### Recommended `.gitignore`

```gitignore
.env
.env.*
!.env.example
*.key
*.pem
*.p12
*.pfx
secrets.json
workspace-overrides.json
*.sqlite
*.db
backups/
```

### Secret rotation events

Rotate secrets after:

* Accidental Git commit
* Screenshot shared publicly
* Logs exposed
* Coolify admin account compromise
* Team member leaves
* Suspicious API usage
* Unauthorized dashboard access attempt

---

## 16. Secret Rotation Playbooks

### 16.1 Rotate Hermes gateway token

Coolify magic value:

```env
SERVICE_PASSWORD_64_HERMESBRIDGE
```

Steps:

1. Regenerate/edit value in Coolify if supported.
2. Redeploy stack.
3. Restart Workspace and Open WebUI.
4. Clear Workspace overrides if needed:

```bash
rm -f /home/workspace/.hermes/workspace-overrides.json
```

5. Verify:

```bash
curl -fsS http://hermes:8642/health
curl -fsS http://hermes:8642/v1/models
```

### 16.2 Rotate Workspace password

Coolify magic value:

```env
SERVICE_PASSWORD_64_HERMESWORKSPACE
```

Steps:

1. Regenerate/edit value.
2. Redeploy `hermes-workspace`.
3. Confirm old password no longer works.
4. Store new password securely.

### 16.3 Rotate Browserless token

Coolify magic value:

```env
SERVICE_PASSWORD_64_HERMESBROWSER
```

Steps:

1. Regenerate/edit value.
2. Redeploy `hermes` and `hermes-browser`.
3. Confirm CDP endpoint works with new token.

### 16.4 Rotate Telegram bot token

1. Use BotFather to revoke/regenerate token.
2. Update `TELEGRAM_BOT_TOKEN` in Coolify.
3. Redeploy Hermes.
4. Test bot response.

### 16.5 Rotate provider API keys

1. Create new provider API key.
2. Update Coolify env.
3. Redeploy Hermes/Honcho as needed.
4. Revoke old key at provider.
5. Monitor usage dashboard.

---

## 17. Backup and Restore Security

Backups contain sensitive memory and config.

### Backup encryption recommendation

Use encrypted storage for backups. Do not leave raw database dumps on public web directories.

Example encrypted backup:

```bash
pg_dump -U honcho honcho > honcho-backup.sql
openssl enc -aes-256-cbc -salt -pbkdf2 -in honcho-backup.sql -out honcho-backup.sql.enc
rm honcho-backup.sql
```

### PostgreSQL backup

```bash
docker exec -t <honcho-database-container> pg_dump -U honcho honcho > honcho-backup.sql
```

### PostgreSQL restore

```bash
cat honcho-backup.sql | docker exec -i <honcho-database-container> psql -U honcho -d honcho
```

### Hermes data volume backup

```bash
docker run --rm \
  -v <project>_hermes-data:/data \
  -v "$PWD":/backup \
  alpine \
  tar czf /backup/hermes-data-backup.tar.gz -C /data .
```

### Backup checklist

* [ ] Backups encrypted.
* [ ] Backups stored off-server.
* [ ] Restore tested.
* [ ] Old backups pruned.
* [ ] Access to backups restricted.

---

## 18. Logging and Redaction

Avoid logging secrets and full webhook payloads.

High-risk log data:

* API keys
* Authorization headers
* Webhook signatures
* Email contents
* Telegram message contents
* Discord message contents
* Browser cookies
* Honcho memory payloads

### Log review commands

```bash
docker compose logs hermes | grep -iE 'key|token|secret|password|authorization'
```

If secrets appear in logs, rotate them.

---

## 19. Network Hardening

### Allowed internal flows

```text
hermes-workspace -> hermes:8642
hermes-workspace -> hermes:9119
hermes-dashboard-proxy -> hermes:9119
hermes-webhook-proxy -> hermes:8644
hermes -> honcho-api:8000
honcho-api -> honcho-database:5432
honcho-api -> honcho-redis:6379
hermes -> hermes-browser:3000
open-webui -> hermes:8642
```

### Disallowed public flows

```text
Internet -> hermes:8642
Internet -> hermes:9119 directly
Internet -> honcho-api:8000
Internet -> honcho-database:5432
Internet -> honcho-redis:6379
Internet -> hermes-browser:3000
```

---

## 20. Rate Limiting and Abuse Protection

Recommended future hardening:

* Add Traefik rate limiting middleware to Workspace.
* Add request size limits to webhook proxy.
* Use Cloudflare/WAF in front of public URLs if needed.
* Restrict dashboard URL by IP if your proxy stack supports it.
* Monitor provider API usage for sudden cost spikes.

Example conceptual Traefik rate-limit middleware:

```yaml
labels:
  - "traefik.http.middlewares.hermes-ratelimit.ratelimit.average=30"
  - "traefik.http.middlewares.hermes-ratelimit.ratelimit.burst=60"
```

Apply carefully through Coolify middleware support.

---

## 21. Incident Response

### If dashboard was exposed without password

1. Remove public dashboard URL or route it only through proxy.
2. Add `DASHBOARD_BASIC_AUTH_USERS`.
3. Redeploy.
4. Rotate Hermes bridge token.
5. Rotate provider keys if dashboard could view them.
6. Review logs.
7. Check for unknown sessions/jobs/config changes.

### If `.env` leaked

1. Assume full compromise.
2. Rotate all provider API keys.
3. Rotate Telegram/Discord bot tokens.
4. Rotate email app password.
5. Regenerate Coolify service passwords.
6. Redeploy stack.
7. Review billing dashboards.
8. Review agent logs and memory.

### If browser token leaked

1. Rotate `SERVICE_PASSWORD_64_HERMESBROWSER`.
2. Redeploy `hermes-browser` and `hermes`.
3. Clear browser session data if needed.
4. Review accounts logged into browser profiles.

### If Honcho memory leaked

1. Take stack offline or isolate Honcho.
2. Rotate database password.
3. Review data exposure scope.
4. Purge sensitive memory if needed.
5. Restore from safe backup if tampered.

---

## 22. Security Validation Commands

### Check public routes from local machine

```bash
curl -I https://your-workspace-url
curl -I https://your-dashboard-url
curl -I https://your-webhook-url/health
```

Dashboard should return `401 Unauthorized` before Basic Auth credentials are supplied.

### Check gateway is not public

Try opening:

```text
https://your-gateway-url
```

There should be no public gateway URL.

### Check internal gateway from Workspace

```bash
curl -fsS http://hermes:8642/health
```

### Check internal dashboard from Workspace

```bash
curl -fsS http://hermes:9119/api/status || curl -fsS http://hermes:9119/
```

### Check Honcho internal only

```bash
curl -fsS http://honcho-api:8000/docs
```

Run from inside the Docker network, not from the public internet.

### Check OpenAI-compatible endpoint

```bash
curl -fsS http://hermes:8642/v1/models \
  -H "Authorization: Bearer <SERVICE_PASSWORD_64_HERMESBRIDGE>"
```

---

## 23. Pre-Production Security Checklist

* [ ] Workspace public URL works over HTTPS.
* [ ] Workspace requires password.
* [ ] Dashboard URL requires Basic Auth.
* [ ] Dashboard is not directly exposed from `hermes`.
* [ ] Gateway API has no public URL.
* [ ] Honcho API has no public URL.
* [ ] PostgreSQL has no public port.
* [ ] Redis has no public port.
* [ ] Browserless has no public URL.
* [ ] Webhook URL uses secret verification.
* [ ] Provider API keys are not committed.
* [ ] Telegram allowed users are configured.
* [ ] Discord allowed users/roles are configured.
* [ ] Email allowed users are configured.
* [ ] Backups are encrypted.
* [ ] Restore process tested.
* [ ] Logs checked for secrets.
* [ ] Coolify admin account protected with MFA if available.

---

## 24. Minimal Safe Defaults

Use these defaults unless you have a reason to change them:

```env
COOKIE_SECURE=1
TRUST_PROXY=1
EMAIL_ALLOW_ALL_USERS=false
WEBHOOK_ENABLED=true
BROWSER_KEEP_ALIVE=true
BROWSER_CONCURRENT=3
BROWSER_QUEUED=3
BROWSER_TIMEOUT_MS=300000
BROWSER_INACTIVITY_TIMEOUT=900
```

Keep these internal:

```text
hermes:8642
hermes:9119
honcho-api:8000
honcho-database:5432
honcho-redis:6379
hermes-browser:3000
```

Expose only through controlled proxies:

```text
hermes-workspace:3000
hermes-dashboard-proxy:8080
hermes-webhook-proxy:8080
open-webui:8080 optional
```

---

## 25. Responsible Disclosure

If you discover a security issue in this deployment template:

1. Do not open a public issue with secrets or exploit details.
2. Privately contact the repository maintainer.
3. Include affected component, reproduction steps, and suggested fix.
4. Redact all real credentials, tokens, URLs, and private payloads.

---

## 26. Final Operator Rule

Before exposing any new service publicly, ask:

1. Who needs to access this from the internet?
2. What authentication protects it?
3. What happens if the URL leaks?
4. What secret would need rotation after compromise?
5. Can this stay internal instead?

If the answer is unclear, keep it internal.
