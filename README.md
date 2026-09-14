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

Pin callers to a tag (`@v1`), not `@main`. A change here then only reaches a
product once it moves to the new tag.
