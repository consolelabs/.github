# macOS app CI standard

Every macOS app follows this. The apps live in consolelabs (dwarvesf apps move here over time), so this repo holds the source of truth: the reusable workflow `.github/workflows/macos-app-ci.yml` and the org template "Console Labs macOS app CI". dwarvesf/.github keeps a mirror for dwarvesf repos until they move; change both together.

Three goals: a warm run finishes in about a minute, no CI job ever opens an app on the Mac Mini's own screen, and no CI job runs as the Mini's own user. A fourth follows from the first: a test goes to the cheapest runner that meets its needs, so the one screen runner is kept for the tests that open an app.

## Where each job runs

A test goes where its needs put it. Only a test that opens the app needs the screen.

```
GitHub Actions (routes by runs-on label)
   │                                        │
   ▼ [self-hosted, macos-ui]                ▼ [self-hosted, macos-build]
┌──────────────────── macOS VM on the Mac Mini ───────────────────────┐
│ screen runner                       build runners                   │
│ app-hosted and render tests         logic tests, iOS Simulator      │
│ (the app opens here)                tests, build-for-testing,       │
│                                     path detection, Release build   │
└─────────────────────────────────────────────────────────────────────┘
Mac Mini host: runs no macOS CI job
```

| Tier | Runner | Input | Why |
|---|---|---|---|
| Logic: tests with no `TEST_HOST` | `[self-hosted, macos-build]` | `logic_scheme` | Nothing opens an app, so it needs no screen. Runs on every call. |
| iOS Simulator: `destination` contains `iOS Simulator`, `test_action: test` | `[self-hosted, macos-build]` | `screen_scheme` (or `logic_scheme`) | The app runs inside the Simulator, not on the VM desktop, so it needs no screen runner. |
| App-hosted: a test target with `TEST_HOST` launches the real app | `[self-hosted, macos-ui]` | `screen_scheme` | In the VM it cannot steal the operator's screen, menu bar, login items or installed copy. |
| Render and UI: tests that draw views, size UI to the screen, or take snapshots | `[self-hosted, macos-ui]` | `screen_scheme` | Same screen runner. Gate it with `ui_paths` so it runs only when the UI changed. |
| Build: `test_action: build-for-testing`, and the Release build | `[self-hosted, macos-build]` | `test_action`, `release_scheme` | No Simulator runtime or screen involved. A host runner runs as the Mini's own user, and any write collaborator can point a workflow at it; in the VM, a workflow reaches only the throwaway guest. |
| Agent UI checks (clicks, screenshots, permission sheets) | the same VM, through ops-toolkit `tools/mac-ui-vm` | none | Real input goes to the VM's own screen only. |
| Public repos | GitHub-hosted `macos-*` is allowed | none | Minutes are free on public repos; private repos never use them (the org budget is $0). |

Which input picks which label:

| Input | Default | Used by |
|---|---|---|
| `build_runner` | `["self-hosted", "macos-build"]` | the logic job, the screen job when it only builds or uses the iOS Simulator, the `ui_paths` check, the Release build |
| `test_runner` | `["self-hosted", "macos-ui"]` | the screen job for an app-hosted or render test |

The defaults are soft: a caller may pass its own label list. A screen job runs on `build_runner` when `test_action` is `build-for-testing` or `destination` contains `iOS Simulator`; otherwise on `test_runner`. `destination` and `test_action` apply to both test jobs, so an app with iOS logic tests and macOS screen tests makes two calls.

### When the screen job runs

| Event | `ui_paths` empty | `ui_paths` set |
|---|---|---|
| Pull request | always | only if the PR diff touches a listed path; otherwise the job shows as skipped |
| Push (the default branch or any other) | always | always |
| Manual run (`workflow_dispatch`) | always | always |

The `ui_paths` check is one small job on the build runner. It reads the PR's own changes (the merge commit against its base parent) with `git`, no third-party action. It fails open: if the diff cannot be read, the screen job runs. The logic job never waits on it.

`ui_paths` is a newline or comma list of globs, matched with bash rules: `*` also matches `/`, so `App/Views/**` and `App/Views/*` are the same; a trailing `/` means the whole folder; `**/*.xib` matches a `.xib` at any depth, the top level included.

### Caller example

```yaml
name: CI
on:
  pull_request:
    paths-ignore: ["**/*.md", "docs/**", "LICENSE"]
  push:
    branches: [main]
    paths-ignore: ["**/*.md", "docs/**", "LICENSE"]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  ci:
    uses: consolelabs/.github/.github/workflows/macos-app-ci.yml@main
    with:
      project: MyApp/MyApp.xcodeproj
      logic_scheme: MyAppLogic        # no TEST_HOST; build runner, every run
      screen_scheme: MyApp            # app-hosted and render tests; screen runner
      ui_paths: |
        MyApp/Views/**
        MyApp/Resources/**
        **/*.xib
      release_scheme: MyAppDirect     # default-branch pushes and manual runs
```

`concurrency` lives in the caller, not in the reusable workflow: inside a reusable workflow `github.workflow` is the caller's workflow name, so a group set there would collide with the caller's own. Cancel only PR runs, never a default-branch run.

An existing caller that passes only `test_scheme` needs no change: `test_scheme` is the old name of `screen_scheme`, and the job keeps its id, so the check name stays `<caller job> / test`. `screen_scheme` wins when both are set. The one placement difference: a `test_scheme` call with `build-for-testing` or an iOS Simulator destination now runs on the build runner instead of the screen runner.

## Speed rules

Measured on hacker-bar, hidden and blurred (Nov 2026): the queue wait was a few seconds; nearly all the time was cold builds and runs that did not need to happen.

| Rule | What it fixes (measured) |
|---|---|
| DerivedData and SwiftPM checkouts live on the runner's disk, never in `$RUNNER_TEMP` | hacker-bar rebuilt from scratch every run: 80 s of its 106 s job. blurred keeps DerivedData and drops from 30 s to 9 s when nothing changed. |
| Each job has its own cache folder (`logic`, `test`, `release`, each with an `-spm` twin) | The jobs run at the same time; a shared folder would let two `xcodebuild` processes write the same files. |
| No `actions/cache` on self-hosted runners | The local disk already persists. hidden spends about 10 s per job restoring and saving its cache. |
| Release build only on pushes to the default branch | hacker-bar built Release on every PR and again on the merge push. |
| `paths-ignore` for docs, plus `concurrency` in the caller that cancels superseded PR runs (never a default-branch run) | hacker-bar ran 66 times in 24 h, docs-only commits included. |
| Pure-logic code gets a test target without `TEST_HOST`, run as `logic_scheme` | An app-hosted test pays about 15 to 20 s of app startup; hacker-bar's tests themselves run in about 2 s. |
| `xcodebuild -quiet` on the Release build only, never on tests | `-quiet` hides a failing test's assertion text. |
| `parallel_testing: true` when the tests split across several test classes | foundations, 340 tests on the VM's 4 cores: a warm test job went from 4 min 4 s to 2 min 56 s. Parallel testing splits by class, so the longest class bounds the run (there, one 158 s render class). |

Measured after the move: blurred went from a 51 s median on the host to 34 s per master push; hacker-bar from a 109 s median to about 60 s.

## Optional inputs

Each is off by default, so a caller that does not set it sees no change.

| Input | What it does | When |
|---|---|---|
| `logic_scheme` | Runs a scheme with no `TEST_HOST` on the build runner on every call | Pure-logic code in its own test target |
| `screen_scheme` | Runs the app-hosted and render scheme; same as the old `test_scheme` | Tests that open the app |
| `ui_paths` | A PR runs the screen job only if its diff touches one of these globs | The screen tests are slow and only UI changes need them |
| `parallel_testing` | Adds `-parallel-testing-enabled YES` to the test run | Tests spread over several classes; measure before and after |
| `test_action` | `test` (default) or `build-for-testing`; `build-for-testing` runs on the build runner | A destination with no runtime to run on: the VM has the iOS Simulator SDK, so a caller without a Simulator runtime passes `build-for-testing` with `destination: generic/platform=iOS Simulator` |
| `pre_test_command` | Runs before `xcodebuild` in the logic job and again in the screen job | Cheap scripted checks, or generating the project (each job has its own checkout) |
| `pre_build_command` | Runs before the Release build, like `pre_test_command` before the tests | A generated project (XcodeGen) has to exist before the Release build too |

A second call of the workflow with its own schemes and `destination` adds a second platform. The screen jobs of both calls queue on the screen runner.

## Parallel work

- Within one run, the logic job, the screen job and the Release build run at the same time, on whichever runners are free.
- Across repos of one org, screen jobs queue on the screen runner. That is on purpose: two app-hosted test runs in one desktop session can fight over the menu bar and the focused window. Build-runner jobs do not need the screen, so they run in parallel up to the number of build runners.
- More screen capacity means a second VM (Apple's licence allows two macOS VMs per Mac), not a second screen runner in the same VM. Add it when queue waits show up in run timings.

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

1. Create the repo in consolelabs. Add the workflow from the org template "Console Labs macOS app CI" and set `project`, `logic_scheme` and/or `screen_scheme`, `release_scheme`.
2. Give the test and Debug configurations their own bundle id.
3. Put pure-logic tests in a target without a test host and pass it as `logic_scheme`; pass the app-hosted and render target as `screen_scheme`, with `ui_paths` listing the UI folders.
4. Open the first PR and check the logic job ran on `macos-build`, the screen job on `macos-ui` (or skipped, when the PR touches no `ui_paths`), and both finished in about a minute on the second run; after the merge, check the release job ran on `macos-build`.
