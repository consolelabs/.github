# macOS app CI standard

Every macOS app follows this. The apps live in consolelabs (dwarvesf apps move here over time), so this repo holds the source of truth: the reusable workflow `.github/workflows/macos-app-ci.yml` and the org template "Console Labs macOS app CI". dwarvesf/.github keeps a mirror for dwarvesf repos until they move; change both together.

Three goals: a warm run finishes in about a minute, no CI job ever opens an app on the Mac Mini's own screen, and no CI job runs as the Mini's own user.

## Where each job runs

```
GitHub Actions (routes by runs-on label)
   │                                │
   ▼ [self-hosted, macos-ui]         ▼ [self-hosted, macos-build]
┌──────────────── mac-ui VM on the Mac Mini ─────────────────┐
│ test runner                       build runner             │
│ xcodebuild test (app opens here)  Release build, pushes    │
│                                   to the default branch    │
│ the two run side by side; one runner per label per org     │
└────────────────────────────────────────────────────────────┘
Mac Mini host: runs no macOS CI job
```

| Job | Runner | Why |
|---|---|---|
| Every test job of a private macOS app | `[self-hosted, macos-ui]` | One rule, whatever the test type. A test target with `TEST_HOST` launches the real app; in the VM it cannot steal the operator's screen, menu bar, login items or installed copy. |
| Release build | `[self-hosted, macos-build]` | A host runner runs as the Mini's own user, and any write collaborator can point a workflow at it. In the VM, a workflow reaches only the throwaway guest. |
| Agent UI checks (clicks, screenshots, permission sheets) | the same VM, through ops-toolkit `tools/mac-ui-vm` | Real input goes to the VM's own screen only. |
| Public repos | GitHub-hosted `macos-*` is allowed | Minutes are free on public repos; private repos never use them (the org budget is $0). |

## Speed rules

Measured on hacker-bar, hidden and blurred (Nov 2026): the queue wait was a few seconds; nearly all the time was cold builds and runs that did not need to happen.

| Rule | What it fixes (measured) |
|---|---|
| DerivedData and SwiftPM checkouts live on the runner's disk, never in `$RUNNER_TEMP` | hacker-bar rebuilt from scratch every run: 80 s of its 106 s job. blurred keeps DerivedData and drops from 30 s to 9 s when nothing changed. |
| Each job has its own cache folder (`test`, `test-spm`, `release`, `release-spm`) | The test and build jobs run at the same time; a shared folder would let two `xcodebuild` processes write the same files. |
| No `actions/cache` on self-hosted runners | The local disk already persists. hidden spends about 10 s per job restoring and saving its cache. |
| Release build only on pushes to the default branch | hacker-bar built Release on every PR and again on the merge push. |
| `paths-ignore` for docs, plus `concurrency` that cancels superseded PR runs (never a default-branch run) | hacker-bar ran 66 times in 24 h, docs-only commits included. |
| Pure-logic code gets a test target without `TEST_HOST` | An app-hosted test pays about 15 to 20 s of app startup; hacker-bar's tests themselves run in about 2 s. |
| `xcodebuild -quiet` on the Release build only, never on tests | `-quiet` hides a failing test's assertion text. |

Measured after the move: blurred went from a 51 s median on the host to 34 s per master push; hacker-bar from a 109 s median to about 60 s.

## Parallel work

- Within one run, the test job and the Release build run at the same time on their two runners.
- Across repos of one org, test jobs queue on the one `macos-ui` runner. That is on purpose: two app-hosted test runs in one desktop session can fight over the menu bar and the focused window.
- More test capacity means a second VM (Apple's licence allows two macOS VMs per Mac), not a second test runner in the same VM. Add it when queue waits show up in run timings.

## The test VM

The runners live in the `mac-ui` VM (ops-toolkit `tools/mac-ui-vm`, command `mac-ui-runner-register <org> <dir> <name> [macos-ui|macos-build]`), with Xcode copied from the Mini. The VM screen is 2560x1440, the Mini's point size, because some tests size UI to the screen: at 1920x1200 hacker-bar's menu showed 11 rows instead of 12 and two render tests failed.

Each repo's caches sit under `~/ci-cache/<owner>_<repo>/` in the guest. Reset a bad cache by moving that folder away; the next run rebuilds cold. The VM disk is fixed size and nothing prunes these folders yet, so check `df -h` in the guest when a new app joins.

## Safety rules

- No macOS CI job runs on the Mini host. A host runner runs as the operator's user, and a private Free-plan repo has no branch protection, so any write collaborator could run code there.
- Test and Debug builds use a bundle id different from the shipping one (for example `.dev`), so a test run never touches the installed app or its login item.
- CI signs ad hoc. Developer ID signing and notarization run on a release machine, never on a runner.
- Test hosts are stopped by their exact DerivedData path, never by a broad `pkill` pattern.
- `timeout-minutes: 20` on every job.
- Callers use `@main`. A change to the reusable workflow reaches every app at once, so it goes through review and a pilot run first.

## New app checklist

1. Create the repo in consolelabs. Add the workflow from the org template "Console Labs macOS app CI" and set `project`, `test_scheme`, `release_scheme`.
2. Give the test and Debug configurations their own bundle id.
3. Put pure-logic tests in a target without a test host.
4. Open the first PR and check the test job ran on `macos-ui` and finished in about a minute on its second run; after the merge, check the release job ran on `macos-build`.
