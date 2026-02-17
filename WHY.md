# Why This Fork Exists

The [upstream SigNoz Railway template](https://github.com/SigNoz/signoz-railway-template) does not deploy cleanly. This fork applies the minimum set of Dockerfile changes to make it work out of the box.

## What the fork fixes

| # | Gap | Symptom | Root cause | Fix in this fork |
|---|-----|---------|------------|------------------|
| 1 | `:latest` image tags | Unpredictable breakage after upstream pushes a new release | Dockerfiles use `FROM signoz/signoz:latest` — no version pinning | Pin to specific versions: `v0.111.0` (signoz), `v0.142.0` (otel-collector) |
| 2 | Missing `server` subcommand | `signoz` container exits immediately with usage error | Upstream `CMD` omits the required `server` subcommand | `CMD ["./signoz", "server"]` in `Dockerfile.signoz` |
| 3 | Stale feature gate | otel-collector crashes on startup with "unknown feature gate" | Upstream start command includes `--feature-gates=-pkg.translator.prometheus.NormalizeName` which was removed in recent collector versions | Override `CMD` in `Dockerfile.otel` without the flag |
| 4 | Schema migrator version | Column-not-found errors in collector after deploying with mismatched versions | Upstream template doesn't pin the schema migrator image; Railway dashboard defaults to `:latest` which may not match the app version | Documented version mapping — migrator must match app version |

## Manual steps still required after deploy

These items require Railway dashboard actions and cannot be fixed in the fork's Dockerfiles.

### 1. Set JWT secret

```bash
railway service signoz
railway variables set SIGNOZ_TOKENIZER_JWT_SECRET=$(openssl rand -hex 32)
```

Without this, sessions are signed with an empty string — anyone can forge valid tokens.

### 2. Rename deprecated env vars

Railway dashboard → **signoz** → **Variables**:

| Old (template default) | New | Value |
|------------------------|-----|-------|
| `TELEMETRY_ENABLED` | `SIGNOZ_ANALYTICS_ENABLED` | `true` |
| `STORAGE` | `SIGNOZ_TELEMETRYSTORE_PROVIDER` | `clickhouse` |

Delete the old variables after adding the new ones.

### 3. Fix schema migrator start commands

The template uses `tcp://[clickhouse]:9000` which is not valid Railway variable syntax — `[clickhouse]` is passed as a literal string, causing a DSN parse error.

Update both migrator start commands on the Railway dashboard:

| Migrator | Start Command |
|----------|--------------|
| **sync** | `/bin/sh -c "sleep 120 && exec ./signoz-schema-migrator sync --dsn=tcp://${{clickhouse.RAILWAY_PRIVATE_DOMAIN}}:9000 --up="` |
| **async** | `./signoz-schema-migrator async --dsn=tcp://${{clickhouse.RAILWAY_PRIVATE_DOMAIN}}:9000 --up=` |

Also remove the `npm run migrate` pre-deploy command on both migrators — it's a leftover from Railway template boilerplate and does nothing in a Go binary image.

### 4. Complete initial setup

Open the SigNoz UI and create the first user/organization. Until this is done, the otel-collector logs repeated errors:

```
[ERRO] Failed to find or create agent error="cannot create agent without orgId"
```

## Full runbook

For step-by-step operational guidance (deploy, upgrade, troubleshoot), see the [SigNoz Railway Deployment Runbook](https://github.com/credoqr/hexagon/wiki/SigNoz-Railway-Deployment-Runbook).
