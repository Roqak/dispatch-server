# dispatch-server on Cloudflare Workers

The [Deploy to Cloudflare button](https://deploy.workers.cloudflare.com/?url=https://github.com/Roqak/dispatch/tree/main/worker) deploys this Worker and its D1 database, and prompts for the secrets. Leave `ADMIN_PASSWORD` as anything strong: it's only used when `REQUIRE_AUTH` is `"false"`. Manual deployment is below.

This folder must stay self-contained, because the button treats it as the root of a new repository. Nothing in it can point outside `worker/`.

The first account registered becomes admin and registration is open by default, so register yours straight after deploying.

```sh
npm run deploy:server   # from the repo root: provisions D1, binds it, deploys;
                        # wrangler creates the database on first deploy
cd worker
npx wrangler secret put POSTMAN_API_KEY   # the one shared Postman key
# REQUIRE_AUTH=false deploys additionally need:
npx wrangler secret put ADMIN_PASSWORD
```

No schema step is required: on the first request the Worker migrates itself (the same idempotent `CREATE TABLE IF NOT EXISTS` set `server/src/db.rs` runs at boot). `worker/schema.sql` exists for manual inspection/re-apply:

```sh
(cd worker && npx wrangler d1 execute dispatch-server --remote --file schema.sql)
```

## What is configured where

- `worker/wrangler.jsonc`: Worker name, D1 binding `DB` (auto-provisioned; to bind an existing database add `database_id` yourself), the admin panel assets (`admin/`, a copy of the server's, kept identical by `npm test`), and the `REQUIRE_AUTH` / `POSTMAN_REGION` vars.
- `worker/.dev.vars.example`: the secrets this Worker uses — `POSTMAN_API_KEY`, and `ADMIN_PASSWORD` when running without auth enforcement. Never commit `.dev.vars`.
- `worker/src/index.js`: the whole Worker (same contract, roles, redaction and admin API as the Rust binary).

## Local development

```sh
cd worker && npx wrangler dev   # local D1, panel on /admin/
```

Local state lives in `.wrangler/` (gitignored). If a request answers `Server database failed to initialize`, the previous section's `d1 execute --file schema.sql --local` (from `worker/`) re-applies by hand.

## Behavior notes

- Everything from the binary contract carries over: roles `viewer < editor < creator < admin`, workspace grants, admin-only deletes, secret redaction for viewers, audit log, verbatim Postman passthrough, `/admin/` panel — including its login (the same `POST /auth/login` / `POST /auth/admin-login` endpoints).
- Passwords are PBKDF2-SHA256 (native WebCrypto) instead of argon2 because Workers' CPU budget can't run argon2; `PBKDF2_ITERATIONS` raises the rounds on paid plans.
- Postman's API rate limits still attach to the one shared key, and workspace-list merges for scope-restricted users fan out per granted workspace — the free plan's 50-subrequests-per-request cap matters if you grant many workspaces. Heavy use wants a paid Workers plan.
- Register/login behave identically to the binary (first registered user becomes admin); admin user management, settings and the audit log live in the same `/admin/` panel.