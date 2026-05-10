# TROUBLESHOOTING.md

Troubleshooting guide for the Hermes Agent Coolify Stack.

This document covers common deployment, routing, authentication, dashboard, Workspace, Honcho Memory, Browserless, Telegram, Discord, Email, Webhook, and optional Open WebUI issues.

---

## 1. Quick Diagnosis Flow

Use this sequence before changing configuration randomly.

```text
1. Did Coolify deploy all services?
   |
   +-- No  -> Check YAML, env vars, image/build errors.
   |
   +-- Yes
        |
        v
2. Is hermes healthy?
   |
   +-- No  -> Check hermes logs and provider/env config.
   |
   +-- Yes
        |
        v
3. Is Workspace reachable publicly?
   |
   +-- No  -> Check Coolify URL, expose port, HOST/PORT, proxy logs.
   |
   +-- Yes
        |
        v
4. Can Workspace reach hermes:8642 and hermes:9119 internally?
   |
   +-- No  -> Check service names, health, network, depends_on.
   |
   +-- Yes
        |
        v
5. Are sessions/dashboard APIs loading?
   |
   +-- No  -> Check dashboard token / Workspace overrides.
   |
   +-- Yes
        |
        v
6. Are optional integrations working?
   |
   +-- No  -> Debug Telegram/Discord/Email/Webhooks/Honcho/Open WebUI individually.
```

---

## 2. First Commands to Run

### 2.1 Check running containers

From server shell:

```bash
docker ps
```

Look for services similar to:

```text
hermes
hermes-workspace
hermes-dashboard-proxy
hermes-webhook-proxy
hermes-browser
honcho-api
honcho-deriver
honcho-database
honcho-redis
```

In Coolify, check each service status under the deployment/resource page.

---

### 2.2 Check logs

From server shell:

```bash
docker compose logs -f hermes
```

```bash
docker compose logs -f hermes-workspace
```

```bash
docker compose logs -f honcho-api
```

```bash
docker compose logs -f honcho-deriver
```

If using Coolify UI, open service logs from the resource page.

---

### 2.3 Check Hermes gateway

Inside the `hermes` container:

```bash
curl -fsS http://localhost:8642/health
```

Expected: HTTP 200 / healthy JSON or healthy text.

---

### 2.4 Check Hermes dashboard side-process

Inside the `hermes` container:

```bash
curl -fsS http://localhost:9119/api/status || curl -fsS http://localhost:9119/
```

Expected: dashboard status JSON or dashboard HTML.

---

### 2.5 Check Hermes webhooks

Inside the `hermes` container:

```bash
curl -fsS http://localhost:8644/health
```

Expected shape:

```json
{"status":"ok","platform":"webhook"}
```

If webhooks are disabled, this may fail; check `WEBHOOK_ENABLED=true`.

---

### 2.6 Check Workspace internal access

Inside the `hermes-workspace` container:

```bash
curl -fsS http://hermes:8642/health
```

```bash
curl -fsS http://hermes:9119/api/status || curl -fsS http://hermes:9119/
```

If these fail, Workspace cannot reach Hermes internally.

---

### 2.7 Check Honcho

Inside `hermes` or `honcho-api`:

```bash
curl -fsS http://honcho-api:8000/docs
```

Expected: Swagger/OpenAPI HTML.

---

### 2.8 Check OpenAI-compatible Hermes API

Inside any service that can reach `hermes`:

```bash
curl -fsS http://hermes:8642/v1/models \
  -H "Authorization: Bearer <SERVICE_PASSWORD_64_HERMESBRIDGE>"
```

Expected: model list or compatible response.

---

## 3. Coolify Deployment Issues

### 3.1 Invalid YAML: sequence item in mapping

Error:

```text
Invalid YAML format: You cannot define a sequence item when in a mapping
```

Cause:

```yaml
environment:
  KEY: value
  - SERVICE_URL_APP_3000
```

Fix:

```yaml
environment:
  - KEY=value
  - SERVICE_URL_APP_3000
```

Use list-style `environment` blocks everywhere when using Coolify magic envs.

---

### 3.2 Coolify does not generate public URL

Symptoms:

* No generated URL for Workspace, Dashboard proxy, or Webhook proxy.
* Resource deploys but has no public endpoint.

Check that the service has a magic URL variable inside its `environment` block.

Workspace:

```yaml
- SERVICE_URL_HERMESWORKSPACE_3000
```

Dashboard proxy:

```yaml
- SERVICE_URL_HERMESDASHBOARD_8080
```

Webhook proxy:

```yaml
- SERVICE_URL_HERMESWEBHOOKS_8080
```

Also verify:

```yaml
expose:
  - "3000"
```

or for proxy services:

```yaml
expose:
  - "8080"
```

Avoid identifiers with underscores:

Bad:

```yaml
- SERVICE_URL_HERMES_WORKSPACE_3000
```

Good:

```yaml
- SERVICE_URL_HERMESWORKSPACE_3000
```

---

### 3.3 Magic secrets are blank

Symptoms:

* `SERVICE_PASSWORD_64_HERMESBRIDGE` appears empty.
* Workspace cannot authenticate to Hermes.
* Browserless token missing.

Checklist:

* [ ] Magic variable appears in a service `environment` block.
* [ ] Coolify version supports Compose magic variables.
* [ ] You redeployed after adding the variable.
* [ ] Identifier has no underscore before port suffix.
* [ ] You did not define it manually as blank in Developer View.

Do not manually create blank values like:

```env
SERVICE_PASSWORD_64_HERMESBRIDGE=
```

Let Coolify generate it.

---

### 3.4 502 Bad Gateway from Coolify URL

Symptoms:

* Workspace URL returns 502.
* Dashboard proxy URL returns 502.
* Webhook URL returns 502.

Common causes:

| Cause                            | Fix                                        |
| -------------------------------- | ------------------------------------------ |
| Service not running              | Check logs and health                      |
| Wrong internal port              | Match `SERVICE_URL_*_<PORT>` with `expose` |
| App binds to localhost only      | Set `HOST=0.0.0.0`                         |
| Missing `PORT` env               | Set correct `PORT`                         |
| Custom networks                  | Remove custom networks                     |
| Fixed `container_name`           | Remove `container_name`                    |
| Proxy target service not healthy | Check upstream service                     |

Workspace should have:

```env
HOST=0.0.0.0
PORT=3000
```

and:

```yaml
expose:
  - "3000"
```

Dashboard proxy should expose:

```yaml
expose:
  - "8080"
```

Webhook proxy should expose:

```yaml
expose:
  - "8080"
```

---

### 3.5 Coolify deployment hangs or HTTPS intermittently fails

Likely causes:

* Custom Docker networks.
* Fixed `container_name`.
* Host `ports:` bypassing proxy.
* Multiple services exposing same magic identifier.

Fix:

* Remove custom `networks:` unless absolutely required.
* Remove all `container_name:` values.
* Use `expose`, not `ports`, for Coolify-routed services.
* Use unique magic identifiers.

---

## 4. Hermes Gateway Issues

### 4.1 Hermes container unhealthy

Check logs:

```bash
docker compose logs -f hermes
```

Common causes:

| Symptom                | Cause                 | Fix                                          |
| ---------------------- | --------------------- | -------------------------------------------- |
| Missing provider key   | No LLM configured     | Add OpenRouter/OpenAI/Anthropic/Google key   |
| Dashboard health fails | Dashboard not started | Check `HERMES_DASHBOARD=1`                   |
| Port conflict          | Wrong command or env  | Use `API_SERVER_PORT=8642`                   |
| Permission issue       | Volume ownership      | Check `/opt/data` volume                     |
| Honcho dependency slow | Honcho not ready      | Increase health start period or check Honcho |

Required Hermes basics:

```env
API_SERVER_ENABLED=true
API_SERVER_HOST=0.0.0.0
API_SERVER_PORT=8642
API_SERVER_KEY=${SERVICE_PASSWORD_64_HERMESBRIDGE}
```

---

### 4.2 Gateway unauthorized / 401

Symptoms:

```text
401 Unauthorized
```

Cause:

Workspace/Open WebUI token does not match `API_SERVER_KEY`.

Correct wiring:

```env
API_SERVER_KEY=${SERVICE_PASSWORD_64_HERMESBRIDGE}
HERMES_API_TOKEN=${SERVICE_PASSWORD_64_HERMESBRIDGE}
CLAUDE_API_TOKEN=${SERVICE_PASSWORD_64_HERMESBRIDGE}
OPENAI_API_KEY=${SERVICE_PASSWORD_64_HERMESBRIDGE}   # for Open WebUI style clients
```

Verify inside Workspace:

```bash
printenv | grep HERMES_API_TOKEN
```

Verify inside Hermes:

```bash
printenv | grep API_SERVER_KEY
```

They must match.

---

### 4.3 `/v1/models` fails

Check:

```bash
curl -i http://hermes:8642/v1/models \
  -H "Authorization: Bearer <SERVICE_PASSWORD_64_HERMESBRIDGE>"
```

Common causes:

* Missing `/v1` in client URL.
* Wrong bearer token.
* Hermes gateway not healthy.
* No model/provider configured.

Correct OpenAI-compatible base URL:

```text
http://hermes:8642/v1
```

Not:

```text
http://hermes:8642
```

---

## 5. Hermes Dashboard Issues

### 5.1 Dashboard opens without password

Cause:

Dashboard exposed directly from `hermes` instead of through `hermes-dashboard-proxy`.

Fix:

Remove direct public magic URL from `hermes` service if present:

```yaml
- SERVICE_URL_HERMESDASHBOARD_9119
- SERVICE_FQDN_HERMESDASHBOARD_9119
```

Use only dashboard proxy:

```yaml
hermes-dashboard-proxy:
  environment:
    - SERVICE_URL_HERMESDASHBOARD_8080
    - SERVICE_FQDN_HERMESDASHBOARD_8080
```

with Basic Auth labels.

---

### 5.2 Dashboard URL asks for password, but login fails

Check your Basic Auth hash.

Generate again:

```bash
htpasswd -nbB admin 'your-password'
```

Paste full output:

```env
DASHBOARD_BASIC_AUTH_USERS=admin:$2y$05$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Rules:

* In Coolify Developer View, keep single `$`.
* In direct Compose labels, escape `$` as `$$`.
* Do not wrap the hash in quotes unless your env parser requires it.

---

### 5.3 Dashboard proxy returns 502

Check if dashboard side-process works inside `hermes`:

```bash
curl -fsS http://localhost:9119/api/status || curl -fsS http://localhost:9119/
```

If this fails, dashboard did not start.

Check Hermes env:

```env
HERMES_DASHBOARD=1
HERMES_DASHBOARD_HOST=0.0.0.0
HERMES_DASHBOARD_PORT=9119
```

Check proxy config points to:

```text
http://hermes:9119
```

---

### 5.4 Workspace shows dashboard `/api/sessions` 401

Error:

```text
Failed to load sessions.
Hermes Agent dashboard /api/sessions?limit=50&offset=0: 401 {"detail":"Unauthorized"}
```

Cause:

Wrong explicit dashboard token persisted in Workspace or Compose.

Fix Compose:

Do not set:

```env
HERMES_DASHBOARD_TOKEN=
CLAUDE_DASHBOARD_TOKEN=
```

Keep only:

```env
HERMES_DASHBOARD_URL=http://hermes:9119
```

Clear Workspace overrides:

```bash
rm -f /home/workspace/.hermes/workspace-overrides.json
```

Restart Workspace.

---

## 6. Hermes Workspace Issues

### 6.1 Workspace public URL returns 502

Check `hermes-workspace` logs:

```bash
docker compose logs -f hermes-workspace
```

Check env:

```env
HOST=0.0.0.0
PORT=3000
HERMES_PASSWORD=${SERVICE_PASSWORD_64_HERMESWORKSPACE}
```

Check Compose:

```yaml
expose:
  - "3000"
```

Check Coolify magic URL:

```yaml
- SERVICE_URL_HERMESWORKSPACE_3000
```

---

### 6.2 Workspace login works but immediately logs out

Usually cookie/proxy mismatch.

For Coolify HTTPS:

```env
COOKIE_SECURE=1
TRUST_PROXY=1
```

For local HTTP testing only:

```env
COOKIE_SECURE=0
```

Restart Workspace after changing.

---

### 6.3 Workspace cannot connect to agent

Inside `hermes-workspace`:

```bash
curl -fsS http://hermes:8642/health
```

If this fails:

* `hermes` is not healthy.
* service name is wrong.
* custom networks broke resolution.
* `depends_on` condition not met.

Correct env:

```env
HERMES_API_URL=http://hermes:8642
HERMES_API_TOKEN=${SERVICE_PASSWORD_64_HERMESBRIDGE}
```

Do not use:

```env
HERMES_API_URL=http://localhost:8642
```

Inside Workspace, `localhost` means the Workspace container itself.

---

### 6.4 Workspace still uses old settings after redeploy

Cause:

Workspace saved override file.

Fix:

```bash
rm -f /home/workspace/.hermes/workspace-overrides.json
```

Restart:

```bash
docker compose restart hermes-workspace
```

or restart the service in Coolify.

---

## 7. Honcho Memory Issues

### 7.1 Honcho API fails to start

Check logs:

```bash
docker compose logs -f honcho-api
```

Common causes:

| Cause                    | Fix                                                   |
| ------------------------ | ----------------------------------------------------- |
| No LLM provider key      | Set `HONCHO_LLM_OPENAI_API_KEY` or another provider   |
| Bad DB connection        | Check `SERVICE_PASSWORD_HONCHODB` and Postgres health |
| pgvector not initialized | Check `honcho-db-init` logs                           |
| Redis unavailable        | Check `honcho-redis` health                           |
| Build failed             | Vendor Honcho locally or fix build context            |

Honcho requires at least one usable LLM provider key.

OpenAI example:

```env
HONCHO_LLM_OPENAI_API_KEY=sk-...
HONCHO_OPENAI_BASE_URL=
HONCHO_MODEL=gpt-5.4-mini
```

OpenRouter example:

```env
HONCHO_LLM_OPENAI_API_KEY=sk-or-v1-...
HONCHO_OPENAI_BASE_URL=https://openrouter.ai/api/v1
HONCHO_MODEL=openai/gpt-4.1-mini
```

---

### 7.2 Honcho remote Git build fails in Coolify

Symptoms:

* Build fails at `context: https://github.com/plastic-labs/honcho.git#main`.
* Coolify cannot fetch/build remote context.

Fix option A: Vendor Honcho into your repo.

```bash
mkdir -p vendor
git clone https://github.com/plastic-labs/honcho.git vendor/honcho
```

Change Compose:

```yaml
build:
  context: ./vendor/honcho
  dockerfile: Dockerfile
```

Fix option B: Use `honcho-self-hosted` overlay.

```bash
git clone https://github.com/elkimek/honcho-self-hosted.git
```

Adapt its files into your deployment repo.

---

### 7.3 `pgvector` extension error

Symptoms:

```text
extension "vector" does not exist
```

Fix:

Make sure database image is:

```yaml
image: pgvector/pgvector:pg15
```

Run init SQL:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

Check `honcho-db-init` logs:

```bash
docker compose logs honcho-db-init
```

---

### 7.4 Hermes cannot reach Honcho

Inside `hermes`:

```bash
curl -fsS http://honcho-api:8000/docs
```

Correct Hermes env:

```env
HONCHO_BASE_URL=http://honcho-api:8000
```

If this fails:

* Honcho API is down.
* Service name changed.
* Custom networks broke DNS.
* Honcho build failed.

---

## 8. Browserless Issues

### 8.1 Browserless token unauthorized

Check Browserless env:

```env
TOKEN=${SERVICE_PASSWORD_64_HERMESBROWSER}
```

Check Hermes env:

```env
BROWSER_TOKEN=${SERVICE_PASSWORD_64_HERMESBROWSER}
BROWSER_CDP_URL=ws://hermes-browser:3000/chromium?token=${SERVICE_PASSWORD_64_HERMESBROWSER}
```

Both must use the same generated token.

---

### 8.2 Browserless sessions do not persist

Check volume:

```yaml
volumes:
  - browser-data:/data
```

Check env:

```env
DATA_DIR=/data
KEEP_ALIVE=true
```

Note: session persistence may also depend on how the CDP client launches Chromium and whether a persistent user-data profile is used.

---

### 8.3 Browserless debug page exposed publicly

This is risky.

Remove:

```yaml
- SERVICE_URL_HERMESBROWSER_3000
```

Keep Browserless internal by default.

---

## 9. Webhook Issues

### 9.1 Webhook URL returns 502

Check webhook proxy logs:

```bash
docker compose logs -f hermes-webhook-proxy
```

Check Hermes webhook listener inside `hermes`:

```bash
curl -fsS http://localhost:8644/health
```

If not healthy, check:

```env
WEBHOOK_ENABLED=true
WEBHOOK_PORT=8644
WEBHOOK_SECRET=${SERVICE_PASSWORD_64_HERMESWEBHOOKS}
```

---

### 9.2 GitHub/GitLab/Stripe webhook gets 401 Basic Auth

Cause:

Basic Auth middleware was applied to the main Hermes service or webhook proxy.

Fix:

Use separate proxies:

```text
hermes-dashboard-proxy -> Basic Auth -> hermes:9119
hermes-webhook-proxy   -> No Basic Auth -> hermes:8644
```

Webhook security should be HMAC/secret-based, not dashboard Basic Auth.

---

### 9.3 Webhook signature verification fails

Check:

* Correct global `WEBHOOK_SECRET`.
* Correct route-level secret.
* Provider sends signature header expected by Hermes route.
* Raw payload is not modified by proxy.
* Endpoint path matches route name.

Endpoint shape:

```text
https://your-webhook-url/webhooks/<route-name>
```

---

### 9.4 Webhook body too large

Nginx proxy may reject payload.

Increase:

```nginx
client_max_body_size 2m;
```

Example:

```nginx
client_max_body_size 10m;
```

Only increase intentionally.

---

## 10. Telegram Issues

### 10.1 Telegram bot does not respond

Check env:

```env
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_USERS=
```

Common causes:

| Cause                       | Fix                                           |
| --------------------------- | --------------------------------------------- |
| Wrong token                 | Regenerate in BotFather                       |
| User ID missing             | Add numeric user ID                           |
| Used username instead of ID | Use numeric ID                                |
| Bot blocked                 | Start bot manually in Telegram                |
| Group privacy mode          | Configure BotFather privacy or group settings |
| Group chat ID missing       | Add `TELEGRAM_GROUP_ALLOWED_CHATS`            |

---

### 10.2 How to find Telegram user ID

Options:

1. Message `@userinfobot`.
2. Temporarily check bot update payloads.
3. Use Telegram API `getUpdates` for your bot.

Example:

```bash
curl "https://api.telegram.org/bot<TELEGRAM_BOT_TOKEN>/getUpdates"
```

Look for:

```json
"from": { "id": 123456789 }
```

Use:

```env
TELEGRAM_ALLOWED_USERS=123456789
```

---

### 10.3 Telegram group messages ignored

Set:

```env
TELEGRAM_GROUP_ALLOWED_USERS=123456789
TELEGRAM_GROUP_ALLOWED_CHATS=-1001234567890
```

Group IDs are often negative and may start with `-100`.

---

## 11. Discord Issues

### 11.1 Discord bot does not respond

Check:

```env
DISCORD_BOT_TOKEN=
DISCORD_ALLOWED_USERS=
```

Common causes:

* Bot token wrong.
* Bot not invited to server.
* Required intents not enabled in Discord Developer Portal.
* User ID not allowed.
* Channel not allowed or bot lacks permissions.
* Role restrictions block the user.

---

### 11.2 Discord user/role ID confusion

Use numeric IDs, not display names.

Enable Developer Mode in Discord:

```text
User Settings -> Advanced -> Developer Mode
```

Right-click user, role, or channel → Copy ID.

---

## 12. Email Issues

### 12.1 Email receives nothing

Check:

```env
EMAIL_ADDRESS=
EMAIL_PASSWORD=
EMAIL_IMAP_HOST=
EMAIL_IMAP_PORT=993
```

For Gmail / Google Workspace:

* Enable IMAP.
* Use app password.
* Do not use account password.
* Check account security blocks.

---

### 12.2 Email sends nothing

Check:

```env
EMAIL_SMTP_HOST=
EMAIL_SMTP_PORT=587
EMAIL_ADDRESS=
EMAIL_PASSWORD=
```

Common SMTP ports:

| Port | Mode          |
| ---: | ------------- |
|  587 | STARTTLS      |
|  465 | SSL/TLS       |
|   25 | Often blocked |

---

### 12.3 Agent responds to unwanted senders

Restrict:

```env
EMAIL_ALLOWED_USERS=you@example.com,team@example.com
EMAIL_ALLOW_ALL_USERS=false
```

Do not use:

```env
EMAIL_ALLOW_ALL_USERS=true
```

unless the mailbox is intentionally public-facing.

---

## 13. Open WebUI Issues

### 13.1 Open WebUI cannot find Hermes models

Use `/v1`.

Correct:

```env
OPENWEBUI_OPENAI_API_BASE_URL=http://hermes:8642/v1
```

Wrong:

```env
OPENWEBUI_OPENAI_API_BASE_URL=http://hermes:8642
```

Check from Open WebUI container:

```bash
curl -fsS http://hermes:8642/v1/models \
  -H "Authorization: Bearer <SERVICE_PASSWORD_64_HERMESBRIDGE>"
```

---

### 13.2 Open WebUI is separate from the stack

Use:

```env
OPENWEBUI_OPENAI_API_BASE_URL=http://host.docker.internal:8642/v1
```

Add:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Important: this requires Hermes `8642` to be reachable from that container path. Same-stack service DNS is safer when possible:

```env
OPENWEBUI_OPENAI_API_BASE_URL=http://hermes:8642/v1
```

---

### 13.3 Open WebUI env changes are ignored

Open WebUI may persist connection settings after first startup.

Fix options:

1. Change connection in Open WebUI Admin UI.
2. Reset Open WebUI volume if acceptable.
3. Recreate service after clearing persisted settings.

---

## 14. `host.docker.internal` Issues

### 14.1 Host name does not resolve

Add this to services needing host access:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

Use cases:

* Open WebUI separate container → Hermes API.
* Honcho → local OpenAI-compatible LLM server.
* Hermes → host services.

Example:

```env
HONCHO_OPENAI_BASE_URL=http://host.docker.internal:8000/v1
```

---

### 14.2 `host.docker.internal:8642` does not work

If Hermes is not publishing host port `8642`, another container outside the Compose network may not reach it via host gateway.

Prefer same-stack internal DNS:

```text
http://hermes:8642/v1
```

Only use host gateway when the caller is outside the Compose stack and Hermes is reachable through host networking or a published port.

---

## 15. Provider API Issues

### 15.1 Hermes says no model/provider configured

Set at least one:

```env
OPENROUTER_API_KEY=
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
GOOGLE_API_KEY=
GEMINI_API_KEY=
GROQ_API_KEY=
MISTRAL_API_KEY=
```

For Gemini, prefer:

```env
GOOGLE_API_KEY=
```

Keep `GEMINI_API_KEY` only for compatibility.

---

### 15.2 Honcho model calls fail but Hermes works

Hermes and Honcho use separate provider envs.

Hermes:

```env
OPENAI_API_KEY=
OPENROUTER_API_KEY=
```

Honcho:

```env
HONCHO_LLM_OPENAI_API_KEY=
HONCHO_OPENAI_BASE_URL=
HONCHO_MODEL=
```

Set both if both components need provider access.

---

### 15.3 OpenRouter with Honcho fails

Use OpenAI-compatible configuration:

```env
HONCHO_LLM_OPENAI_API_KEY=sk-or-v1-...
HONCHO_OPENAI_BASE_URL=https://openrouter.ai/api/v1
HONCHO_TRANSPORT=openai
HONCHO_MODEL=openai/gpt-4.1-mini
HONCHO_EMBEDDING_MODEL=openai/text-embedding-3-small
```

If the embedding model name is rejected, switch to a provider/model combination supported by your OpenRouter account or use OpenAI directly for embeddings.

---

## 16. Volume and Permission Issues

### 16.1 Workspace permission denied

The `workspace-init` service should create and chown:

```text
/home/workspace/.hermes
```

Check logs:

```bash
docker compose logs workspace-init
```

If needed, run inside a temporary root container or adjust volume ownership.

---

### 16.2 Hermes data permission denied

Check volume mount:

```yaml
volumes:
  - hermes-data:/opt/data
```

Check logs for permission errors.

If the image expects a specific user, align volume ownership accordingly. Do not blindly `chmod 777` in production.

---

### 16.3 Browserless cannot write to `/data`

The stack uses:

```yaml
user: "0:0"
```

and:

```yaml
volumes:
  - browser-data:/data
```

This avoids common named-volume write problems. If you harden to non-root later, adjust volume ownership first.

---

## 17. Backup and Restore Issues

### 17.1 Honcho backup fails

Use the actual container name from:

```bash
docker ps
```

Then:

```bash
docker exec -t <honcho-database-container> pg_dump -U honcho honcho > honcho-backup.sql
```

If password error occurs, check:

```env
SERVICE_PASSWORD_HONCHODB
```

---

### 17.2 Restore fails due to existing data

Options:

1. Restore into a fresh database.
2. Drop/recreate schema carefully.
3. Stop services that write to DB during restore.

Example cautious flow:

```bash
docker compose stop honcho-api honcho-deriver
cat honcho-backup.sql | docker exec -i <honcho-database-container> psql -U honcho -d honcho
docker compose start honcho-api honcho-deriver
```

---

## 18. Performance Issues

### 18.1 Browserless consumes too much RAM

Tune:

```env
BROWSER_CONCURRENT=1
BROWSER_QUEUED=2
BROWSER_TIMEOUT_MS=180000
```

Also reduce:

```yaml
shm_size: "1gb"
```

if memory constrained, though Chromium may become less stable.

---

### 18.2 Honcho is slow

Possible causes:

* Slow LLM provider.
* Embedding model latency.
* Database underpowered.
* Redis unavailable.
* Too many memories/messages.

Check:

```bash
docker compose logs -f honcho-deriver
```

Consider smaller/faster models for summary/deriver tasks.

---

### 18.3 Workspace streaming times out

Increase:

```env
STREAM_ACCEPTED_TIMEOUT_MS=120000
STREAM_HANDOFF_TIMEOUT_MS=300000
```

For very slow agents:

```env
STREAM_HANDOFF_TIMEOUT_MS=600000
```

---

## 19. Security Incident Symptoms

### 19.1 Unexpected provider API usage

Actions:

1. Rotate provider API keys.
2. Check dashboard/session logs.
3. Check Telegram/Discord/Email allowed users.
4. Check webhook endpoints.
5. Check Browserless exposure.
6. Review Coolify environment variable access.

---

### 19.2 Unknown dashboard access

Actions:

1. Rotate dashboard Basic Auth password.
2. Rotate `SERVICE_PASSWORD_64_HERMESBRIDGE`.
3. Rotate provider keys if dashboard had access.
4. Review sessions/jobs/config changes.
5. Confirm dashboard is not directly exposed.

---

### 19.3 Browserless exposed publicly

Actions:

1. Remove public Browserless URL.
2. Rotate `SERVICE_PASSWORD_64_HERMESBROWSER`.
3. Clear browser data if needed.
4. Review accounts logged into persistent browser sessions.

---

## 20. Clean Recreate Procedure

Use this when the stack is stuck in a bad state after many edits.

### 20.1 Preserve data first

Back up:

* `honcho-pgdata`
* `hermes-data`
* `workspace-data`

### 20.2 Clear Workspace bad overrides

```bash
rm -f /home/workspace/.hermes/workspace-overrides.json
```

### 20.3 Force recreate

```bash
docker compose pull
docker compose build --no-cache honcho-api honcho-deriver
docker compose up -d --force-recreate
```

In Coolify:

```text
Redeploy -> Force Pull Latest Images
```

If Honcho is built from source, also force rebuild if Coolify provides that option.

---

## 21. Final Debug Checklist

Use this checklist before asking for help.

### Coolify

* [ ] No YAML syntax error.
* [ ] Magic URLs generated.
* [ ] Magic passwords generated.
* [ ] No `container_name`.
* [ ] No custom `networks`.
* [ ] No unnecessary host `ports:`.
* [ ] `expose` ports match magic URL suffixes.

### Hermes

* [ ] `curl http://localhost:8642/health` works inside `hermes`.
* [ ] `curl http://localhost:9119/api/status` or `/` works inside `hermes`.
* [ ] Provider key configured.
* [ ] `API_SERVER_KEY` set.

### Workspace

* [ ] Public Workspace URL loads.
* [ ] Login works.
* [ ] `HERMES_API_URL=http://hermes:8642`.
* [ ] `HERMES_DASHBOARD_URL=http://hermes:9119`.
* [ ] No wrong dashboard token.
* [ ] Override file cleared if needed.

### Dashboard

* [ ] Dashboard public URL uses proxy.
* [ ] Basic Auth prompt appears.
* [ ] Dashboard proxy points to `http://hermes:9119`.

### Honcho

* [ ] `honcho-database` healthy.
* [ ] `honcho-redis` healthy.
* [ ] `honcho-db-init` completed.
* [ ] `honcho-api` starts.
* [ ] Honcho provider key configured.
* [ ] Hermes can reach `http://honcho-api:8000/docs`.

### Webhooks

* [ ] Webhook proxy URL loads `/health`.
* [ ] `WEBHOOK_ENABLED=true`.
* [ ] Webhook secret generated.
* [ ] No Basic Auth on webhook proxy.

### Messaging

* [ ] Telegram token set.
* [ ] Telegram allowed users set.
* [ ] Discord token set if used.
* [ ] Discord allowed users/roles set.
* [ ] Email app password set if used.
* [ ] Email allowed users set.

### Open WebUI

* [ ] Base URL includes `/v1`.
* [ ] API key matches Hermes bridge token.
* [ ] Connection updated in Admin UI if env changes ignored.

---

## 22. Information to Collect for Support

When asking for help, provide sanitized versions of:

```bash
docker compose ps
```

```bash
docker compose logs --tail=200 hermes
```

```bash
docker compose logs --tail=200 hermes-workspace
```

```bash
docker compose logs --tail=200 honcho-api
```

```bash
docker compose logs --tail=200 hermes-dashboard-proxy
```

Also provide sanitized output of:

```bash
printenv | sort | grep -E 'HERMES|HONCHO|TELEGRAM|DISCORD|EMAIL|WEBHOOK|OPENWEBUI|SERVICE_URL|SERVICE_PASSWORD|COOKIE|TRUST|HOST|PORT'
```

Redact:

* API keys
* bot tokens
* passwords
* webhook secrets
* email credentials
* public URLs if sensitive

Do not paste real secrets into GitHub issues, chats, screenshots, or logs.
