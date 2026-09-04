# Console Labs CI: run it on the Mini

Console Labs CI runs on our own hardware. A new workflow in any `consolelabs`
repo starts from `ci-self-hosted.yml`, not from a GitHub-hosted template.

## Pick the pool by what the job holds

| Pool | Label | Holds a credential | Accepts `pull_request` |
|---|---|---|---|
| PR | `pr-shared` | no | yes |
| Trusted | `cl-trusted` | yes | **never** |

```yaml
runs-on: [self-hosted, pr-shared]    # tests, lint, build checks
runs-on: [self-hosted, cl-trusted]   # deploys, anything reading a secret
```

Both are org-scoped runners on the Mac Mini in Da Nang. `cl-trusted` sits in a
runner group with visibility `selected`, so a repo has to be added to it before
it can use that label.

**The split is the whole point.** A `pull_request` job runs code the author of
the PR controls, including dependency lifecycle scripts. If that job lands on a
machine holding the Cloudflare token, a PR can read the token. So the pool that
can deploy never accepts a `pull_request` trigger, and `pr-shared` carries no
secret to leak.

## Two things the runner image does not give you

It is a plain Debian slim, not a GitHub-hosted image.

- **No package manager is preinstalled.** Declare `pnpm/action-setup` or install
  yarn yourself. Skipping this fails at exit 127, `command not found`, *after*
  any credential has already resolved, which reads like a credential fault and
  is not one.
- **No language toolchain beyond what your actions install.** An app needing
  hugo, go, or similar installs it as a step; `console-apps` does this with
  `apps/log/scripts/ensure-toolchain.sh`, which caches into the runner's
  persistent volume.

## Always set these

```yaml
permissions:
  contents: read          # do not inherit the org default

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true   # false on deploy jobs: a half-published Worker is worse than waiting

steps:
  - uses: actions/checkout@v5
    with:
      persist-credentials: false   # the token is readable by any postinstall in the job
```

## Adding a repo to the trusted pool

The runner group gates it, not the label. Add the repo to the `cl-trusted`
group in org settings, or the job queues forever against a label it cannot see.
