# live9-actions

Shared GitHub Actions workflows for the Live9 estate. **This repo is meant to be
public**: GitHub only lets a *private* reusable workflow be called from inside
its own organisation, and the products live in three (`rowbotapp`, `evercoach`,
`live9tech`). It contains no secrets; callers pass credentials in with
`secrets: inherit`.

## `static-site.yml`

Builds a static site and deploys it as a Cloudflare Worker with static assets
(no Worker code). Architecture, the DNS ownership rules and how to add a site
are in `live9-infra/edge/README.md`.

```yaml
jobs:
  docs:
    uses: live9tech/live9-actions/.github/workflows/static-site.yml@v1
    secrets: inherit
    with:
      site-dir: apps/docs                     # where wrangler.jsonc is
      environment: staging                    # staging | production
      mode: deploy                            # deploy | preview
      install-command: pnpm install --frozen-lockfile --filter @rowbot/docs...
      build-command: pnpm --filter @rowbot/docs run build
      build-env: |
        NODE_ENV=production
```

The caller's repo (or org) needs:

| Kind | Name | Value |
|---|---|---|
| variable | `CLOUDFLARE_ACCOUNT_ID` | the Live9 Cloudflare account |
| secret | `CLOUDFLARE_API_TOKEN` | *Account › Workers Scripts: Edit* + *Zone › Workers Routes: Edit* + *Zone › DNS: Edit* on the site zones only |

Set both at org level once per org, and put the production values behind
the `production` GitHub environment if you want required reviewers on deploys.

Pin callers to a tag (`@v1`), not `@main`. A change here then only reaches a
product once it moves to the new tag.
