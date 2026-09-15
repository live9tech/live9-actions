# live9-actions

Shared GitHub Actions workflows for the Live9 estate. **This repo is meant to be
public**: GitHub only lets a *private* reusable workflow be called from inside
its own organisation, and the products live in three (`rowbotapp`, `evercoach`,
`live9tech`). It contains no secrets; callers pass credentials in explicitly.

**Don't use `secrets: inherit`.** It only forwards secrets to reusable workflows
in the *same* organisation, so from `rowbotapp` or `evercoach` the call fails
before any step runs: "Secret CLOUDFLARE_API_TOKEN is required, but not provided
while calling".

## `static-site.yml`

Builds a static site and deploys it as a Cloudflare Worker with static assets
(no Worker code). Architecture, the DNS ownership rules and how to add a site
are in `live9-infra/edge/README.md`.

```yaml
jobs:
  docs:
    uses: live9tech/live9-actions/.github/workflows/static-site.yml@v1
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
    with:
      site-dir: apps/docs                     # where wrangler.jsonc is
      environment: staging                    # staging | production
      mode: deploy                            # deploy | preview
      install-command: pnpm install --frozen-lockfile --filter @rowbot/docs...
      build-command: pnpm --filter @rowbot/docs run build
      build-env: |
        NODE_ENV=production
```

The caller needs:

- a **repo** secret `CLOUDFLARE_API_TOKEN`: an account-owned token in the Live9
  Technologies Cloudflare account with *Account › Workers Scripts: Edit* plus *Zone › Zone:
  Read, Workers Routes: Edit, DNS: Edit* on the site zones only. It has to be
  set per repo: the orgs are on GitHub Free, which doesn't expose org secrets or
  variables to private repos. To put required reviewers on production deploys,
  set it as a `production` environment secret instead.
- `account_id` pinned in the site's `wrangler.jsonc`
  (Live9 Technologies, `13c30ad8438b49857e4c34127e4c363f`). The workflow checks for it, and it
  replaces a `CLOUDFLARE_ACCOUNT_ID` variable.

## `site-build.yml` + `site-deploy.yml` (v2)

Build once, deploy to staging, promote the **same artifact** to production. Used
by the brand websites (live9tech/website, ticketlayer/website, rowbotapp/website,
evercoach/website).

```yaml
on:
  push: { branches: [main] }
  workflow_dispatch:
    inputs:
      promote_run_id: { description: Run id of a green staging deploy, required: true, type: string }

permissions: { contents: read, actions: read }

jobs:
  build:
    if: github.event_name == 'push'
    uses: live9tech/live9-actions/.github/workflows/site-build.yml@v2
    with:
      site-dir: site          # where wrangler.jsonc and package.json are (default .)
      assets-dir: out         # relative to site-dir
      build-command: pnpm build
      build-env: |
        NEXT_PUBLIC_TURNSTILE_SITE_KEY=${{ vars.TURNSTILE_SITE_KEY }}

  staging:
    needs: build
    uses: live9tech/live9-actions/.github/workflows/site-deploy.yml@v2
    secrets: { CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }} }
    with: { environment: staging, artifact: ${{ needs.build.outputs.artifact }} }

  production:
    if: github.event_name == 'workflow_dispatch'
    uses: live9tech/live9-actions/.github/workflows/site-deploy.yml@v2
    secrets: { CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }} }
    with: { environment: production, source-run-id: ${{ inputs.promote_run_id }} }
```

- `site-build` installs with the lockfile it finds in `site-dir` (pnpm, bun or
  npm), builds, writes `BUILD.json` and uploads `wrangler.jsonc`, `BUILD.json`
  and the assets as one artifact (kept 90 days).
- `site-deploy` runs `wrangler deploy --env <environment>` on that artifact. A
  promote refuses a source run that isn't a successful run on the default
  branch, so production only gets builds that went to staging first.
- `wrangler.jsonc` needs `account_id` pinned and a `staging` and `production`
  env, each with its own `name`, `assets` and `routes`.
- Same repo secret as above; `actions: read` is needed to fetch the artifact.

## Versions

Pin callers to a tag (`@v1`, `@v2`), not `@main`. A change here then only
reaches a product once it moves to the new tag.

- `v1`: `static-site.yml` (build and deploy in one job).
- `v2`: adds `site-build.yml` and `site-deploy.yml`. `static-site.yml` is unchanged.
