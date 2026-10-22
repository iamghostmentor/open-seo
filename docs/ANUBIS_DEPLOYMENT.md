# Anubis OpenSEO deployment

Production URL: `https://oseo.anubistechholdings.com`  
MCP URL: `https://oseo.anubistechholdings.com/mcp`  
Cloudflare account: `7e47966ca7f26589c439ec5585cb197a`  
Deployment stage: `selfhost`

This fork's `codex/anubis-selfhost` branch adds a custom Worker domain, makes
the Access application cover that domain, disables the `workers.dev` route,
turns off scheduled paid research and JavaScript audit rendering, and uses
Alchemy's local state store. The Cloudflare-backed Alchemy state-store bootstrap
failed with a Cloudflare Secrets Store `internal_error` on 2026-10-22. Keep
`.alchemy/state` with the checkout; losing it can cause a later deployment to
try to recreate or replace existing resources.

## Resources and data

- D1: `open-seo-db-selfhost` (`deac1d57-a728-49e8-a7ed-b6f04f3089a0`)
- KV: `open-seo-kv-selfhost`, `open-seo-oauth-kv-selfhost`
- R2: `open-seo-r2-selfhost`
- Workers: `open-seo-selfhost`, `open-seo-selfhost-audit`
- Access application: `open-seo selfhost`, restricted to the email in
  `ACCESS_ALLOWED_EMAILS`
- Three Workflow resources for site audits, rank checks, and AI visibility

Project records, context, competitors, and saved keywords live in D1. R2 is
for object artifacts. KV holds cache and OAuth data. Do not run
`alchemy destroy` or delete any stage-suffixed resource during an upgrade.

## Local deployment

The private `.env.selfhost` contains the DataForSEO key and authorized email.
It is ignored by Git and should remain mode `0600`. Never commit it or the
Alchemy state directory.

```bash
cd '/Users/ghostmentor/Documents/open-seo-anubis'
CI=true npx -y pnpm@10.30.1 install --frozen-lockfile
npx -y pnpm@10.30.1 exec vite build --mode selfhost
npx -y pnpm@10.30.1 alchemy deploy deploy/alchemy/alchemy.run.ts \
  --env-file .env.selfhost --stage selfhost --yes
```

Before updating, export D1 and copy `.alchemy/state` to a private backup.
Then fetch upstream, merge or rebase the Anubis branch onto the desired
upstream commit, review the deployment patch against upstream changes, build,
and deploy with the same stage and environment file. Confirm that Alchemy's
plan says `noop` for D1, KV, R2, and Access before accepting it. Dashboard
changes to the Access app should be reviewed after redeployment.

```bash
npx wrangler d1 export open-seo-db-selfhost --remote --output /PRIVATE/BACKUP/open-seo-db-selfhost.sql
cp -R .alchemy/state /PRIVATE/BACKUP/alchemy-state
```

The initial private backup is at
`/Users/ghostmentor/Documents/open-seo-anubis-backups/2026-10-22/`.
It contains the D1 export and Alchemy state. Keep this folder private. R2 is
currently empty; after object artifacts exist, include a scoped R2/S3 object
sync in the backup routine. Research-bearing D1 records should be restored
only after taking a fresh export of the current database and inspecting the
target, because importing an older SQL dump can overwrite later research.

For a code rollback, check out the last known working commit, rebuild, and
redeploy using the existing `.alchemy/state` and `selfhost` stage. Leave D1,
KV, and R2 in place. Restore data separately only if it is actually damaged.

## Access and cost controls

Cloudflare Access gates the entire custom hostname. The Worker checks the
Access JWT and its audience on each request. The `workers.dev` URL is disabled.
Managed OAuth belongs in the Access application, not in this source file;
verify its settings after any Access resource change. Codex should use
the `openseo-anubis` MCP registration rather than the official plugin's
hosted `app.openseo.so` endpoint. The twelve public SEO skills were copied
from `plugins/openseo/skills` to `~/.codex/skills`.

The deployment has no scheduled paid research and does not offer
JavaScript-rendered audits. Creating a project with a Domain in the UI starts
a 50-page scan, so initial site URLs are recorded in project context instead.
DataForSEO auto-recharge was off when checked. Cloudflare billing showed
`$0.00` billable usage on 2026-10-22. R2 budget alerts at $5 were added.
Billing alerts notify; they do not cap
spending. Before enabling schedules or running paid research, verify the
provider's current price and remaining project budget. Record the date,
source, request parameters, and returned cost in the project's research log.

Free verification commands:

```bash
npx wrangler d1 execute open-seo-db-selfhost --remote --command \
  'SELECT COUNT(*) FROM projects;'
curl -I https://oseo.anubistechholdings.com/
codex mcp get openseo-anubis
```

An unauthenticated `curl` should redirect to Cloudflare Access. A normal
browser login should show the project dashboard. The first-party company and
client websites are outside this deployment.
