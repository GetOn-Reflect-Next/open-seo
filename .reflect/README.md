# .reflect

Reflect Media's deployment layer for this fork. Everything Reflect owns lives in
**new paths only** — this directory, `.github/workflows/reflect-*.yml`, and
`scripts/reflect-apply-resources.mjs` — so an upstream merge can never conflict
with it. No tracked upstream file is modified.

Upstream is [`every-app/open-seo`](https://github.com/every-app/open-seo) (MIT).
The private companion repo is `GetOn-Reflect-Next/reflect-openseo`, which holds
the resource IDs, the runbooks and the cost log.

## The rule

**This fork is public.** Forks of public repositories always are, and that cannot
be changed. So Reflect's Cloudflare resource IDs — KV and OAuth KV namespaces, the
D1 database, the R2 bucket — **never go in git here.** They arrive as repo secrets
and get written into `wrangler.jsonc` at deploy time by `apply-resources.mjs`.

`wrangler.jsonc` is **never modified in git on this fork.** If you find a commit
that edits it, something has gone wrong.

## Files

| Path                                      | What                                                                                                    |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `scripts/reflect-apply-resources.mjs`     | Writes the six Reflect values into `wrangler.jsonc`. Hard-fails rather than deploy an unverified config |
| `scripts/reflect-classify-migrations.mjs` | Decides whether a release may ship unattended. `--self-test` runs its own tests                         |
| `.reflect/upstream.lock`                  | The upstream release this fork is synced to. Bumped only by `reflect-upstream-watch`, never by hand     |

### Why the script is in `scripts/` and not here

Upstream's `ci:check` runs `oxlint . --type-aware` over the whole tree. A plain
`.mjs` file outside `tsconfig` resolves to `any`, so the `typescript/no-unsafe-*`
rules fire on every line — **81 errors** for this one file. Type-aware rules run in
a separate engine (`tsgolint`) that **ignores `// oxlint-disable` comments**, so
suppressing them in-file is not possible.

`scripts/` is already in `ignorePatterns` in `.oxlintrc.json`, which is how upstream
solved the identical problem for its own `.mjs` scripts. Putting our script there
costs nothing and avoids editing a tracked upstream file — adding `.reflect` to
`ignorePatterns` would have put a permanent conflict in `.oxlintrc.json` on every
future merge.

The `reflect-` filename prefix keeps ownership obvious and makes a collision with an
upstream file effectively impossible. Adding a _new_ file to an existing directory
cannot conflict on merge; only two edits to the same path can.

## Workflows

| Workflow                 | When                            | What                                                                                                       |
| ------------------------ | ------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `reflect-upstream-watch` | Mondays 04:00 PT, or manual     | New upstream release → merge the tag to a branch, bump the lock, classify the risk, open a PR              |
| `reflect-sync-nudge`     | Mondays 09:00 PT, or manual     | Posts waiting sync PRs and their risk to the Reflect Chat space. Silent when there is nothing to do        |
| `reflect-migration-gate` | PRs touching `drizzle/**/*.sql` | Blocks merge if a new migration DROPs a table or column, until someone adds the `migration-reviewed` label |
| `reflect-auto-ship`      | CI settles on a sync PR         | Merges a PR labelled `auto-ship` once it is CLEAN. `--merge` only, never a squash. **Off** — see below     |
| `reflect-deploy`         | push to `main`, or manual       | Apply config, capture a D1 restore point, export, deploy, smoke test, roll the code back if a check fails  |

Upstream's own `ci.yml` runs on the sync PR and gates the merge, which is why the
watch workflow pushes with a PAT rather than `GITHUB_TOKEN` — events raised by
`GITHUB_TOKEN` do not start new workflow runs.

## What ships without asking

Half of upstream's releases change nothing about the database. Those do not need a
human, because a code-only rollback puts everything back exactly as it was.
`scripts/reflect-classify-migrations.mjs` sorts each release into one of three
buckets and the pipeline follows it:

| Class      | Means                                                              | Merge               | Deploy                          |
| ---------- | ------------------------------------------------------------------ | ------------------- | ------------------------------- |
| `none`     | no new migrations                                                  | `reflect-auto-ship` | `production-auto`, no reviewer  |
| `additive` | every statement is `CREATE TABLE` / `CREATE INDEX`                 | `reflect-auto-ship` | `production-auto`, no reviewer  |
| `risky`    | `ALTER`, `DROP`, `INSERT`/`UPDATE`/`DELETE`, `PRAGMA`, unparseable | a human, on purpose | `production`, required reviewer |

Measured over the 18 releases from v0.0.18 to v0.1.6: 7 `none`, 2 `additive`,
9 `risky`. So roughly half ship unattended and the half that can actually hurt
still stops for a person.

**Why the line is drawn there.** `wrangler rollback` restores the Worker, but
drizzle migrations are forward-only — upstream ships no down migrations. Once a
release has migrated production D1, reverting the code leaves the schema ahead of
it. The only route back for data is `wrangler d1 time-travel restore`, which
rewinds the entire database to a bookmark and discards every write since,
including writes that had nothing to do with the bad release. That is an outage
with data loss, not a rollback. So "we can roll this back" is only honestly true
for releases that leave existing tables alone, and those are exactly the ones
allowed through.

Anything the classifier cannot read — an unresolvable diff, an empty migration, a
statement it does not recognise — is treated as `risky`. A wrong guess here would
auto-apply a migration that rewrites production data, so ambiguity always resolves
toward the human.

To stop a release that was cleared, remove its `auto-ship` label before CI
finishes; `reflect-auto-ship` re-classifies at merge time and strips the label
itself if the two ever disagree.

### It is off, on purpose

`REFLECT_AUTO_SHIP` is unset and the `production-auto` environment has been
deleted. Every release routes to `production` and waits for a reviewer. The
classification still runs — it drives the `destructive-migration` label and the
risk line in the PR body — but nothing acts on it automatically.

That was a deliberate call on 2026-08-20, not an unfinished rollout. The reasoning:
auto-shipping only helps at per-release granularity, and reviewing weekly means a
typical PR carries ~1.7 releases, so if any one of them has a risky migration the
whole batch needs a human anyway. At a 50% per-release risky rate that is about
two weeks in three. The unattended path would have fired roughly monthly, in
exchange for a workflow that can itself break — and a broken control plane is what
cost a morning on the day this was written. Staleness costs nothing here: this is
an internal tool behind Access, so being a few days behind upstream is free.

**Do not turn it on without redoing the setup below.** With `production-auto`
deleted, GitHub recreates it on first reference with no reviewers and no secrets,
so a cleared release would fail its preflight check rather than deploy. That
failure is the safety net working, not a bug.

Before setting the variable:

1. **Create a `production-auto` environment** with no required reviewers, and add
   the same seven secrets as `production`. They are environment-scoped, so the new
   environment starts with none of them and the deploy fails its precondition
   check until they are there. Do not move them to repo-level secrets to avoid the
   duplication: repo secrets are readable by every workflow in the repo, including
   upstream's `ci.yml`, whose contents upstream controls.
2. **Set `CF_ACCESS_CLIENT_ID` / `CF_ACCESS_CLIENT_SECRET`** on both environments.
   Unattended deploys refuse to run without them. The anonymous smoke test only
   proves Access is in front of the Worker; the authenticated one is the only check
   that proves the app still serves, and without it an auto-deploy could ship a
   broken release and report success.
3. The **`auto-ship` label** already exists; nothing applies it while the
   variable is unset.

Worth doing regardless: **turn off squash and rebase merging** on the fork, leaving
only merge commits. A squash-merged sync PR drops the ancestry link to the upstream
tag, and the next release then conflicts against every file the two releases share
— that is what broke the v0.1.6 sync. `reflect-upstream-watch` now detects and
repairs it with a `-s ours` merge, but not allowing it in the first place is
better.

To switch it back off, unset `REFLECT_AUTO_SHIP` — releases go straight back to
review. Deleting the `production-auto` environment is not the way to do it: GitHub
recreates a referenced environment on demand, with no reviewers and no secrets, so
cleared releases would then fail the precondition check instead of deploying at
all. That failure is the safety net, not the off switch. It is also the reason the
seven secrets stay environment-scoped: an auto-created environment inherits
nothing, so a half-finished setup cannot deploy unreviewed.

## Deploying by hand

`apply-resources.mjs` must run **before** `pnpm run deploy`. That script is
`db:migrate:prod && build && wrangler deploy`, and the migrate step reads
`database_id` and `migrations_dir` out of the same config file. Skipping it
migrates upstream's D1, not ours.

```bash
export REFLECT_KV_ID=... REFLECT_OAUTH_KV_ID=... \
       REFLECT_D1_NAME=... REFLECT_D1_ID=... REFLECT_R2_BUCKET=...
node scripts/reflect-apply-resources.mjs && pnpm run deploy
```

`node scripts/reflect-apply-resources.mjs --check` verifies the file as it stands and
writes nothing. It reports failure on an untouched upstream config, which is the
whole point — use it to confirm what you are about to deploy.

## Required repo secrets

`reflect-deploy` needs: `CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`,
`REFLECT_KV_ID`, `REFLECT_OAUTH_KV_ID`, `REFLECT_D1_NAME`, `REFLECT_D1_ID`,
`REFLECT_R2_BUCKET`, and optionally `CF_ACCESS_CLIENT_ID` / `CF_ACCESS_CLIENT_SECRET`
for the authenticated smoke test.

`reflect-upstream-watch` needs `REFLECT_SYNC_TOKEN`, a fine-grained PAT scoped to
this fork with contents and pull-requests write.

`reflect-sync-nudge` needs `CHAT_WEBHOOK_URL` — the **same** Google Chat incoming
webhook the dashboard ETL posts through, so alerts land in one space and there is
one credential to rotate. It lives in Secret Manager as `chat-webhook-url` and
points at space `AAQAVsr-kPw`. Copy it without it touching a clipboard:

```bash
gcloud secrets versions access latest --secret=chat-webhook-url \
  --account admin@reflectmedia.ca --project reflect-ai-workspace \
  | gh secret set CHAT_WEBHOOK_URL --repo GetOn-Reflect-Next/open-seo
```

Treat that URL as a credential: its `key` and `token` query parameters are the
whole authentication, so never echo it or a Chat error body into a log.

The Cloudflare token needs **D1 Edit**. The dashboard's "Edit Cloudflare Workers"
template does not include it, and a token without it fails about 30 seconds in
with an unhelpful message.

## Preview URLs stay off

`apply-resources.mjs` forces `preview_urls: false`. Our Access applications match
the production hostname exactly, with no wildcard, so a version preview URL is a
different — unprotected — hostname running the full app against production data.
One was found live and executing on 2026-07-29.

Do not enable previews, and do not use `wrangler versions upload`, until an Access
application covers the preview hostnames.
