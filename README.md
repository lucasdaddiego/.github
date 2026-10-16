# lucasdaddiego/.github

Reusable GitHub Actions workflows for my repos. A repo that deploys to Cloudflare keeps its own
triggers, tests and gates in its `deploy.yml` and calls `cloudflare-deploy.yml` for the deploy itself.

## cloudflare-deploy.yml

One job: checkout → Node → install → build → check → (artifact download) → wrangler → post-deploy.
Every step except the checkout and the wrangler call is optional. The run steps execute in
`working-directory` with bash (`-e -o pipefail`), so a command input can be `a && b` or a block
scalar with one command per line.

```yaml
jobs:
  test:
    # the repo's own tests
  deploy:
    needs: test
    if: github.event_name != 'pull_request'
    # @main on purpose: my own public repo, and the actions are SHA-pinned there.
    uses: lucasdaddiego/.github/.github/workflows/cloudflare-deploy.yml@main
    with:
      build: npm run build
      wrangler-command: pages deploy dist --project-name=my-project
      account-id: ${{ vars.CLOUDFLARE_ACCOUNT_ID }}
    secrets:
      CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

### Inputs (all strings)

| Input | Default | Meaning |
| --- | --- | --- |
| `wrangler-command` | required | The wrangler command without the leading `wrangler`: `deploy`, `pages deploy dist --project-name=x`, … |
| `account-id` | `""` | Cloudflare account id, usually `${{ vars.CLOUDFLARE_ACCOUNT_ID }}`. Leave empty when the repo stores it as a secret and pass the `CLOUDFLARE_ACCOUNT_ID` secret instead (`secrets` is not available in `with`). One of the two is required. |
| `node-version` | `26` | Node.js version for setup-node. |
| `working-directory` | `.` | Directory the install, build, check, wrangler and post-deploy commands run in. |
| `install` | `npm ci` | Install command. `""` skips it. setup-node caches `~/.npm` only when a `package-lock.json` exists there and the install runs. |
| `build` | `""` | Build command, run after the install. |
| `check` | `""` | Shell command run after the build, e.g. `node scripts/check-dist.mjs`. |
| `artifact` | `""` | Name of an artifact an earlier caller job uploaded (`actions/upload-artifact`). Downloaded before the deploy. |
| `artifact-path` | `dist` | Directory, relative to `working-directory`, the artifact is extracted into. |
| `wrangler-version` | `4.151.0` | The wrangler version `cloudflare/wrangler-action` installs. |
| `post-deploy` | `""` | Shell command run after the deploy, e.g. an assertion against the live site. |

### Secrets

| Secret | Required | Meaning |
| --- | --- | --- |
| `CLOUDFLARE_API_TOKEN` | yes | Token with the permissions the wrangler command needs. |
| `CLOUDFLARE_ACCOUNT_ID` | no | The account id, for repos that store it as a secret. The `account-id` input wins when both are set. |

### Outputs

| Output | Meaning |
| --- | --- |
| `deployment-url` | The URL wrangler reported (Pages deployment URL, Workers URL), empty when it printed none. Also written to the step summary. |

### What stays in the caller

- `on:`, `needs:`, `if:` gates (default branch only, fork PRs, …), `concurrency`, `permissions` (the called job runs with `contents: read`).
- Anything that needs the git history: the checkout is shallow (`fetch-depth: 1`).
- Builds that must run on pull requests without a deploy: a caller job, or an artifact handed over with `artifact`.
