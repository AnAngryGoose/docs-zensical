---
icon: simple/forgejo
title: Deploying from Forgejo to Cloudflare Pages
---

# Deploying from Forgejo to Cloudflare Pages

Cloudflare Pages' built-in Git integration only connects to GitHub and GitLab —
**not** a self-hosted Forgejo/Gitea instance. To deploy from Forgejo, a
**Forgejo Actions** workflow builds the site and pushes it to Pages with
**Wrangler** (Direct Upload).

The workflow lives at [`.forgejo/workflows/deploy.yml`](https://gitea.vanth.internal/goose/docs-zensical/src/branch/main/.forgejo/workflows/deploy.yml)
and runs on every push to `main`.

```mermaid
graph LR
  A[git push main] --> B[Forgejo Actions runner]
  B --> C[zensical build --clean]
  C --> D[wrangler pages deploy site]
  D --> E[Cloudflare Pages]
```

!!! info "No secrets in the repo"

    The API token and account ID are stored as **Forgejo Actions secrets**, never
    committed. The project name is a plain **variable**.

## 1. Register a Forgejo Actions runner

Forgejo Actions needs a runner ([`act_runner`](https://forgejo.org/docs/latest/admin/actions/))
to execute jobs — the instance has none by default. On a host with Docker:

```bash
# Get a registration token from:
#   Repo → Settings → Actions → Runners → Create new runner
# (or a site-wide token under Site Administration → Actions → Runners)

docker run -d --restart=always --name forgejo-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $PWD/runner-data:/data \
  -e FORGEJO_INSTANCE_URL=https://gitea.vanth.internal \
  -e FORGEJO_RUNNER_REGISTRATION_TOKEN=<registration-token> \
  code.forgejo.org/forgejo/runner:latest
```

The runner registers with labels (e.g. `ubuntu-latest`) mapped to Docker images.
`ubuntu-latest` maps to `catthehacker/ubuntu:act-latest`, which already includes
Python, Node, and npm — everything the build needs.

Then enable Actions on the repo: **Settings → Actions → Enable**.

## 2. Create the Cloudflare Pages project

Create a **Direct Upload** project (not Git-connected — Wrangler pushes to it):

```bash
npx wrangler pages project create docs-zensical --production-branch main
```

…or in the dashboard: **Workers & Pages → Create → Pages → Direct Upload**.

!!! warning "Don't mix Git integration with Direct Upload"

    A Pages project connected to a Git repo rejects `wrangler` uploads. If this
    site is *already* deploying from the [GitHub → Pages](zensical-cloudflare.md)
    integration, either point this pipeline at a **separate** Pages project, or
    disconnect the Git integration and let Forgejo be the single source of deploys.

## 3. Create a scoped API token

**Cloudflare dashboard → My Profile → API Tokens → Create Token → Custom token**:

| Permission | Scope |
|---|---|
| Account → Cloudflare Pages → **Edit** | Your account |

Copy the token, and grab your **Account ID** from the Cloudflare dashboard sidebar
(or `wrangler whoami`).

## 4. Add the secrets and variable in Forgejo

**Repo → Settings → Actions**:

| Type | Name | Value |
|---|---|---|
| Secret | `CLOUDFLARE_API_TOKEN` | the token from step 3 |
| Secret | `CLOUDFLARE_ACCOUNT_ID` | your Cloudflare account ID |
| Variable | `CF_PAGES_PROJECT` | the Pages project name (e.g. `docs-zensical`) |

## 5. Push

```bash
git push forgejo main
```

The runner picks up the push, builds with Zensical, and uploads `site/` to Pages.
Watch it under **Repo → Actions**. Once the first deploy succeeds, add your custom
domain on the Pages project (**Custom domains → Set up a custom domain**).

## Notes

- **Mirroring GitHub:** this repo also has a `.github/workflows/docs.yml` that
  deploys to GitHub Pages. The two CI systems ignore each other's workflow
  directories, so keeping both remotes in sync is fine.
- **Action resolution:** Forgejo resolves `uses:` actions from the runner's default
  actions URL. If `actions/checkout` fails to resolve, set the runner's
  `DEFAULT_ACTIONS_URL` to `https://github.com`, or pin the full URL in the workflow.

## Sources

- [Forgejo Actions documentation](https://forgejo.org/docs/latest/user/actions/)
- [Cloudflare Pages — Direct Upload with Wrangler](https://developers.cloudflare.com/pages/get-started/direct-upload/)
- [wrangler pages deploy](https://developers.cloudflare.com/workers/wrangler/commands/#pages-deploy)
