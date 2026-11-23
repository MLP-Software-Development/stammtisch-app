# Which parts of the Busflow and Satzart setup this repo copies, and what changes with Nhost

Research for [#11](https://github.com/MLP-Software-Development/stammtisch-app/issues/11). Written 2026-11-23 by an
agent; nothing here is decided until a grilling ticket (mainly
[#19](https://github.com/MLP-Software-Development/stammtisch-app/issues/19)) accepts it.

**Sources.** A Busflow lessons-learned playbook kept outside this repo (the starting point); the Busflow monorepo
and the Satzart Capacitor app, both read-only; the Nhost, Capacitor and GitHub docs linked inline. Paths are given as
`busflow: <path>` or `satzart: <path>`, relative to each repo's root. Both repos are private, so the paths are there
for maintainers, and the claims are summarised here so the doc reads without them. Identifiers, hostnames and
customer names from those repos are left out on purpose.

**Scope.** What Nhost offers for auth, storage, push and jobs is [#7](https://github.com/MLP-Software-Development/stammtisch-app/issues/7).
Store and legal rules are [#12](https://github.com/MLP-Software-Development/stammtisch-app/issues/12). The stack
choices are [#19](https://github.com/MLP-Software-Development/stammtisch-app/issues/19). This doc covers repo
setup, CI, the release pipeline and secret hygiene.

## TL;DR

1. **Copy the skeleton.** Copy Busflow's skeleton: one required `CI gate` check, gitleaks in hooks and CI, small
   agent entry files, five agent lanes, a reviewer brief, three skills and CODEOWNERS on the paths that act at
   once. Skip the machinery around it: 41 hooks, about 100k lines of scripts, waves, the inbox and the ledger.
2. **Nhost replaces most of Busflow's backend ops.** You push to a branch and Nhost applies `nhost.toml`, the
   migrations, the metadata and the functions. There is no compose deploy, Traefik, host, backup cron or Terraform
   on day one. The Hasura guards still apply in full: claim pins in permissions, timestamped immutable migrations
   and a schema snapshot.
3. **Mobile: take Satzart's fastlane shape, drop its white-label machinery.** Keep two lanes per platform, `match`
   plus an App Store Connect API key, build numbers read from the store that fail closed on deploy, TestFlight and
   the Play internal track with manual promotion, and an artifact check. Drop the per-customer folders, the native
   files rewritten by regex, CocoaPods, Appflow and GitLab CI.
4. **Frontend: follow Busflow's native shells, not its office app.** That means Nuxt 4 with `ssr: false`, Tailwind
   4, headless primitives, `@nhost/nhost-js` and Capacitor 8 with SPM, one codebase for the web and both stores.
   The office app uses Quasar.
5. **Public repo:** fork PRs get no secrets, so CI must pass without any. Anything that signs, uploads or deploys
   runs only from `main` or `workflow_dispatch`, inside a protected GitHub environment. The real backend's
   address, signing material, store keys and the review demo password are never committed.

## 1. Busflow pieces: day one, later, never

Busflow is a self-hosted Hasura + NestJS + Nhost Auth/Storage stack on Docker Compose
(`busflow: infra/docker/docker-compose.local.yml`, `infra/docker/docker-compose.production.yml`). It has about 55
PRs a day, 60 workflows and 41 hook modules (playbook §1). This repo is small and public, its backend is managed by
Nhost cloud, and one founder drives it with agents.

### Day one

| Piece | What to copy | Busflow source | Change for this repo |
|:--|:--|:--|:--|
| CI gate | One aggregate job named `CI gate` that `needs:` every job and runs `if: always()`. `permissions: {}` at the top, and `concurrency` that cancels superseded PR runs. | `busflow: .github/workflows/ci.yml` (lines 27–33, job `CI gate` ~line 322) | Drop the `changes` path classifier, the draft rules, the merge-queue and stack-journey jobs, and the Ubicloud runners. Use `ubuntu-latest`, which is free for public repos ([GitHub billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)). |
| Ruleset on `main` | Require a PR and `CI gate`, linear history, no force push, no deletion. | playbook §3.1 | Same. |
| Secret scan, three layers | gitleaks in `pre-commit` and `pre-push`, and again in CI. Run the CLI, not `gitleaks-action`, which needs an organisation licence. | `busflow: .githooks/pre-commit`, `.githooks/lib/secret-scan.sh`, `.github/workflows/pr-checks.yml` (line ~63 comment) | Add `"prepare": "git config core.hooksPath .githooks"`. Busflow has no install step that sets it, so fresh clones run without hooks. Also turn on GitHub secret scanning with push protection, which is free on public repos. |
| `commit-msg` hook | Conventional Commits form, plus the attribution policy. | `busflow: .githooks/commit-msg` | **Inverted policy.** Busflow blocks `Co-Authored-By: Claude`, but this repo's history carries that trailer on every commit. Here the hook checks only the Conventional Commits form, or requires the trailer. |
| Agent entry files | `AGENTS.md` points to the rest; `CLAUDE.md` is `@AGENTS.md` plus a few rules. | `busflow: AGENTS.md`, `CLAUDE.md` (66 lines, mixed German and English) | Already in place here and in English. Keep `CLAUDE.md` short. |
| `.claude/settings.json` deny list | Deny reading `.env*`, `~/.ssh`, `~/.aws`, `~/.gnupg` and `~/.config/gh`; deny `git push --force` and `-f`. | `busflow: .claude/settings.json` `permissions.deny` | Add `.secrets`, Nhost's local secrets file. Use `~`-relative paths, not machine paths. Drop the `pushInsteadOf` HTTPS rewrite and all the hook wiring. |
| Lanes | `groundwork`, `plan-writer`, `implementer`, `errand`, `final-reviewer`. | `busflow: .claude/agents/*.md` (34–93 lines each) | Remove the Busflow terms. Skip `po-worker`, `proxy`, `teamlead` and `figma-audit`. |
| Reviewer brief | `REVIEW.md`: where failures happen in this stack, the form of a finding, the nit cap. | `busflow: REVIEW.md` (37 lines) | Swap tenant pins for group/member pins and `apps/hasura/` for `nhost/`. Add a public-repo item: no secret, hostname or real name in diffs. |
| Skills | `frontend`, `security` and `docs-authoring` in `.agents/skills/`, symlinked into `.claude/skills/`. | `busflow: AGENTS.md`, `.agents/skills/frontend/SKILL.md` | Write the frontend skill for Nuxt + Capacitor only. Busflow's has Quasar boundary rules, Histoire and analytics that do not apply. This repo also uses the mattpocock skills (`research`, `grilling`, `domain-modeling`) for the map. |
| CODEOWNERS | The founder owns `.claude/`, `.agents/`, `CLAUDE.md`, `AGENTS.md`, `.github/` and `nhost/nhost.toml`. | `busflow: .github/CODEOWNERS` (42 lines) | Swap `infra/` for `nhost/nhost.toml`, because it deploys config on merge. Add `fastlane/` and `ios/`/`android/` signing config once they exist. |
| PR template | Only `## Commit` for small PRs; Testing, Risk and Revert when needed. | `busflow: .github/pull_request_template.md` | Same. |
| Renovate | Weekly, minor and patch grouped, actions pinned by digest, `minimumReleaseAge` 1 day. | `busflow: renovate.json` (223 lines), `pnpm-workspace.yaml` (`minimumReleaseAge: 1440`) | Start from about 30 lines. |
| Feature path | idea → concept → Figma → build plan → build. | `busflow: CLAUDE.md` §Der Weg eines Features | Already in this repo's `AGENTS.md`. |
| Merge on green | The session that builds a change merges it when `CI gate` is green, unless a CODEOWNERS path blocks it. | `busflow: CLAUDE.md` §Die Sitzung merged ihre Arbeit selbst | Same, without `pnpm founder-ping` (Slack). A blocked PR gets a `founder:decide` issue or comment. |

### Week one, with the first code

| Piece | Busflow source | Change for this repo |
|:--|:--|:--|
| Local stack in one command, seeds per screen state | `busflow: package.json` `dev:stack`, `infra/docker/docker-compose.local.yml` (Hasura, `nhost/auth`, `nhost/storage`, Mailhog, SeaweedFS) | `nhost up` replaces the whole compose file. It runs Postgres, Hasura, Auth, Storage, Functions and Mailhog ([Nhost local development](https://docs.nhost.io/platform/cli/local-development)). Seeds use invented names only (`AGENTS.md` §Privacy). |
| Permission claim pin test | `busflow: scripts/permission-tenant-pin.mjs`, `REVIEW.md` §Hasura permissions | Every role filter pins `x-hasura-user-id`, or reaches the group through a membership. Only an explicit list of public tables is exempt. This is the most important guard for a multi-group app. |
| Timestamped, immutable, linted migrations | `busflow: scripts/migration-new.mjs`, `scripts/migration-immutable.mjs`, squawk | Nhost writes `nhost/migrations/default/<timestamp>_<name>/up.sql` ([local development](https://docs.nhost.io/platform/cli/local-development)), so timestamps come for free. Keep the CI checks that fail on an edited migration from `main`, and squawk. |
| Committed GraphQL schema snapshot and codegen drift check | `busflow: codegen.ts`, `scripts/schema-snapshot.mjs`, `package.json` `codegen:check` | Same, run against the `nhost up` stack in CI. |
| Native-origin CORS test | `busflow: scripts/cors-native-shell-origin.test.mjs` | Pin `capacitor://localhost` and `https://localhost` in `nhost.toml`. See §3. |
| Agent GitHub App | playbook §3.9 | Optional. This repo's agents push over the founder's SSH and sign with a trailer, so identity is less urgent than in Busflow. |

### Later, on a trigger

Merge queue (agent PRs start to collide on `main`); preview deploys (someone has to see a change before merge);
dependency and licence gates (before the first store upload); a restore test of backups (once production holds
data the seed cannot rebuild); a smoke check and heartbeat (real users); knip and file-size checks (code passes
about 20k lines). The triggers and Busflow sources are in playbook §4.

### Never, for this repo

Sprint waves, the drain pass, the release issue; the proxy/teamlead/po office organisation; session-log export, the
sprint-end ledger and a 4,000-line decision ledger (this repo records decisions on wayfinder tickets); the 41-module
hook suite and its shell parser; the provenance `Stop` hook; risk routing through a classifier; noise numbers; R&D
hour logging; Terraform, Traefik, self-hosted runners and the office host (Nhost cloud replaces them); marketing,
pitch and docs-hub deploys inside the product repo; and everything Busflow built and retired (semantic-release,
the `release` branch, `ready-to-merge`, the PR size cap, the open-PR limit hook). Sources: playbook §5,
`busflow: .github/workflows/` (60 files), `busflow: .claude/hooks/`.

### Docs layout: a choice for #19

The playbook recommends Busflow's PARA tree (`docs/1-projects/` … `docs/4-archive/`) in every repo, and the founder
has seen that weighting. This repo already follows the mattpocock skill layout: `GLOSSARY.md` at the root, ADRs in
`docs/adr/` (`docs/agents/domain.md`) and research on `research/<name>` branches. The two fit together if ADRs stay
in `docs/adr/`, where the domain-modeling skill reads them, and the PARA folders arrive with the first build work.
**Recommendation:** keep `docs/adr/` and the root `GLOSSARY.md`. Add `docs/1-projects/` and `docs/4-archive/` when
the first feature is built. Leave out `2-areas/` and `3-resources/` until a runbook exists, starting with the store
release runbook.

## 2. What changes with Nhost

| Concern | Busflow (self-hosted) | This repo on Nhost cloud |
|:--|:--|:--|
| Backend folder | `busflow: apps/hasura/` (`config.yaml`, `migrations/`, `metadata/`, `seeds/`, a pinned `bin/hasura`) | `nhost/` with `nhost.toml`, `migrations/`, `metadata/`, plus a `functions/` folder. `nhost init` creates them ([Nhost CLI](https://docs.nhost.io/reference/cli/commands)). |
| Deploy | A GitHub workflow running `docker compose up --wait` on its own host, with a database snapshot before migrations (`busflow: .github/workflows/deploy-production.yml`) | Nhost's GitHub integration. Every push to the deployment branch deploys `nhost.toml`, new migrations, metadata and functions ([Nhost deployments](https://docs.nhost.io/platform/cloud/deployments)). "Database migrations are not rolled back", but metadata, settings and functions are replaced by the deployed ref's. So a bad migration is fixed forward, never reverted. |
| Environments | Previews on the office host, then a release cut to production | One Nhost project per environment, each linked to a branch through the deployment-branch setting ([Nhost deployments](https://docs.nhost.io/platform/cloud/deployments)). Nhost has no documented per-PR backend previews. Start with one production project deploying from `main`. Add a staging project on its own branch only when a change needs a dry run against cloud config. |
| Gate before deploy | Production waits for a GitHub environment reviewer | Nhost deploys on push, so the gate before a deploy is the `main` ruleset plus CODEOWNERS on `nhost/nhost.toml`. Without the GitHub integration, a manual deploy is possible: `nhost deployments new --follow` from a workflow ([CLI reference](https://docs.nhost.io/reference/cli/commands), marked experimental). |
| Secrets | `.env` files on the host, GitHub environment secrets (`busflow: infra/docker/env.production.example`) | Cloud secrets live in the Nhost dashboard. `nhost.toml` references them as `{{ secrets.NAME }}`. Locally, the CLI reads a `.secrets` file at the project root ([Nhost secrets](https://docs.nhost.io/platform/cloud/secrets)), and the docs say secrets must be set in both places. **Add `.secrets` to `.gitignore` by hand.** The docs only mention that step for the config-migration flow, and they warn to review `nhost config pull` output before committing ([migrate config](https://docs.nhost.io/platform/cli/migrate-config)). |
| Config drift | Compose files in git | Nhost warns that "changes made directly in the cloud console will not be tracked locally and may be lost" ([cloud development](https://docs.nhost.io/platform/cli/cloud-development)). Rule: every setting goes through `nhost.toml` in a PR, and CI runs `nhost config validate`. |
| CORS and redirects | Compose env vars, pinned by a test (`busflow: scripts/cors-native-shell-origin.test.mjs`) | `[hasura.settings] corsDomain` (defaults to `*`) and `auth.redirections.clientUrl` / `allowedUrls` in `nhost.toml` ([configuration reference](https://docs.nhost.io/reference/configuration)). Port Busflow's test to read `nhost.toml`. |
| Token claims | A custom "mint" service swaps the Nhost token for tenant claims (`busflow: apps/mint/README.md`) | None. A person can belong to many groups, and permissions reach the group through a membership table checked against `x-hasura-user-id`. That needs no custom claim service. Revisit only if row filters get slow. |
| Backups, observability | Own backup cron with a restore test, Grafana/Loki/Tempo (`busflow: .github/workflows/backup-verify.yml`, `infra/docker/observability/`) | Nhost's platform backups and logs. Depending on the plan tier, which is [#7](https://github.com/MLP-Software-Development/stammtisch-app/issues/7)'s question, add a restore test before real data lands. |

## 3. Satzart: Capacitor packaging and store release

Satzart is a Quasar 2 + Capacitor 7 white-label app built for many customers. Its Capacitor project sits in
`apps/quasar-capacitor-app/src-capacitor/` and is released with fastlane (`satzart:
apps/quasar-capacitor-app/fastlane/Fastfile`). Most releases ran from a laptop through a wrapper script. Its CI is
GitLab, with manual mobile jobs on a macOS-tagged runner (`satzart: .gitlab-ci.yml`, jobs
`mobile-signing-preflight`, `mobile-deploy-matrix`).

### Copy

- **Pinned toolchain.** A `Gemfile` pins fastlane (`satzart: apps/quasar-capacitor-app/Gemfile`). Drop the
  cocoapods and `ffi` pins: they exist only for CocoaPods.
- **Two lanes per platform.** `build_only` does everything except the upload, and `deploy` uploads
  (`satzart: fastlane/Fastfile`, `platform :ios` and `platform :android`).
- **iOS signing.** `match` with type `appstore`, `readonly: is_ci` (CI never creates certificates), and storage in
  a private repo (`satzart: fastlane/Matchfile`). Authentication through an App Store Connect API key
  (`ASC_KEY_ID`, `ASC_ISSUER_ID`, the `.p8` content from a secret) rather than an Apple ID session
  (`load_asc_api_key` in the Fastfile). Use the API key path only; do not keep Satzart's `FASTLANE_SESSION`
  fallback.
- **Android signing.** An upload keystore with Play App Signing, injected at build time and never committed
  (`satzart: scripts/mobile-release.sh`). A Play service-account JSON for uploads.
- **Versioning.** One marketing version for both platforms. The build number comes from the store: the latest
  TestFlight build + 1 on iOS, and the highest `versionCode` across Play tracks + 1 on Android. The `deploy` lanes
  call the bump with `fail_on_store_lookup_error: true`, so a failed store lookup stops the deploy
  (`satzart: fastlane/Fastfile` lines ~324–410, ~452, ~588). **A correction to the playbook:** only `deploy` fails
  closed. The standalone bump lanes still fall back to a local increment. Here, make fail-closed the only
  behaviour.
- **Upload targets and review submission.** iOS uploads through `deliver` with `submit_for_review: false` and
  `force: true`; without `force`, the HTML preview prompt aborts non-interactive runs. Android uploads to the
  `internal` track. Promotion to production and the submission for review are manual steps in the store consoles
  (`satzart: fastlane/Fastfile` ~line 491, ~line 612). Keep them manual until a few releases have gone through
  cleanly.
- **Store metadata in the repo** in `deliver`/`supply` format. Metadata and screenshots are uploaded only from a
  folder that exists and is not empty (`satzart: fastlane/Fastfile` ~lines 469–508).
- **Artifact check before upload.** Check the bundle ID, the version and the display name
  (`satzart: fastlane/verify_artifact.rb`, with specs that CI runs in job `test:fastlane-gate`).
- **Release wrapper.** It refuses a dirty tree, builds in a temporary worktree, takes a lock per platform, needs a
  typed `DEPLOY …` confirmation for uploads, and writes a run manifest and a redacted log
  (`satzart: scripts/mobile-release.sh`, 1,115 lines; `scripts/redact-mobile-release-log.mjs`). Copy the ideas in
  about 150 lines. Most of the length handles customer selection.
- **Screenshots from the web build** with Playwright, signed in as the seeded review account
  (`satzart: scripts/mobile-screenshots.mjs`).
- **Signing preflight** that prints the exact `match` command for whatever is missing
  (`satzart: scripts/mobile-signing-preflight.sh`).

### Do not copy

- **The white-label machinery:** `customers/<name>/` env files, Firebase configs and store metadata, customer
  selection, the deploy matrix, foreign-tenant checks, and rewriting `project.pbxproj`, Gradle and `strings.xml` at
  build time. This repo ships one app, so the committed native projects carry the final identity.
- **CocoaPods.** Capacitor 8 creates SPM iOS projects by default
  ([Capacitor 8 update guide](https://capacitorjs.com/docs/updating/8-0)). With pnpm, set
  `experimental.ios.spm.packageOptions.<plugin>.symlink: true` for each plugin. Without it, `cap sync` writes pnpm
  store paths with version suffixes into the SPM manifest, and the build breaks on every Capacitor bump
  (`busflow: apps/driver/capacitor.config.ts`).
- **Appflow** (`satzart: appflow.config.json` is still tracked after Appflow was dropped) and live (OTA) updates.
  A stale OTA bundle once bricked a Satzart customer's app (playbook §8.5). Ship only store binaries until there is
  a reason not to.
- **Committed review demo password and Firebase config files.** Satzart tracks
  `customers/*/appstore/metadata/review_information/demo_password.txt`, `google-services.json` and
  `GoogleService-Info.plist`. That is acceptable in a private repo and wrong in this one: inject them in CI from
  secrets.
- **Split local and CI paths.** Satzart's CI and local runs build differently (playbook §8.3). Here one wrapper
  runs in both places.

### CI shape for this repo

One `workflow_dispatch` workflow, `mobile-release.yml`, with inputs `platform` (`ios`/`android`/`both`) and `lane`
(`build_only`/`deploy`). It runs on `macos-latest` for iOS and `ubuntu-latest` for Android. Both are standard
GitHub-hosted runners and free on a public repo
([GitHub billing](https://docs.github.com/en/billing/concepts/product-billing/github-actions)). The workflow calls
the same wrapper as a local run. The `deploy` lane runs inside a GitHub environment `store` with the founder as
required reviewer, so the signing secrets reach the job only after approval. A `build_only` run with no secrets
(unsigned Android debug build, iOS simulator build) can run on PRs that touch `ios/`, `android/` or `fastlane/`.

### Native settings to get right on day one

- The bundle ID / application ID is permanent after the first upload (`busflow: apps/driver/capacitor.config.ts`
  comment above `appId`). Decide it in #19 and record it in an ADR.
- Keep the Apple team out of `project.pbxproj`: a gitignored `ios/local.xcconfig` holds `DEVELOPMENT_TEAM`
  (`busflow: apps/driver/README.md` §Auf dem iPhone testen). CI writes the same file from a secret.
- Set `ITSAppUsesNonExemptEncryption = false` in `Info.plist`. Without it, every TestFlight build waits on
  "Missing Compliance" (playbook §8.2, Satzart history).
- No `NSAllowsArbitraryLoads`. Busflow allows cleartext only to local network addresses, for live reload
  (`NSAllowsLocalNetworking`, `busflow: apps/driver/README.md`).
- Refuse a native build that bakes in `localhost` or a live-reload URL
  (`busflow: apps/driver/scripts/native-build-env.mjs`, `scripts/live-reload-leftover.sh`).
- The WebView origins are `capacitor://localhost` on iOS and `https://localhost` on Android
  ([Capacitor config](https://capacitorjs.com/docs/config): `iosScheme` `capacitor`, `androidScheme` `https`,
  `hostname` `localhost`). Both go into Hasura's `corsDomain` and Nhost Auth's allowed redirect URLs.
- Magic-link and OAuth sign-in come back through universal links / App Links. That needs
  `apple-app-site-association` and `assetlinks.json` under `/.well-known/` on the web domain, plus an `appUrlOpen`
  listener that routes inside the app ([Capacitor deep links](https://capacitorjs.com/docs/guides/deep-links)).
  The web host must therefore serve those two files. What Nhost Auth does inside a Capacitor shell is
  [#7](https://github.com/MLP-Software-Development/stammtisch-app/issues/7)'s question.
- Toolchain floor for Capacitor 8: Node 22+, Xcode 26+, iOS 15 deployment target, Android Studio Otter, minSdk 24,
  compile/target SDK 36, AGP 8.13, Gradle 8.14.3 ([Capacitor 8 update guide](https://capacitorjs.com/docs/updating/8-0)).
  Satzart's wrapper also has to juggle JDK 17 and 21 (`satzart: scripts/mobile-release.sh`,
  `check_android_java_available`). Pin one JDK (21) in CI with `actions/setup-java`.

## 4. Frontend: does the web app follow Busflow's Vue setup?

Busflow has two frontend patterns (`busflow: apps/*/package.json`):

| Pattern | Busflow apps | Stack |
|:--|:--|:--|
| Office app | `workspace` | Vue 3 + Vite + **Quasar 2** + Pinia + Apollo + Tailwind 4 |
| Native shells | `driver`, `passenger` | **Nuxt 4 with `ssr: false`** + Tailwind 4 + `@nhost/nhost-js` + Capacitor 8.6 (iOS only so far) |

Busflow's shared UI packages are headless (`reka-ui` + Tailwind) and must not depend on Quasar
(`busflow: .agents/skills/frontend/SKILL.md` §Quasar Boundary Rules). Satzart is Quasar 2 in Capacitor mode on
Capacitor 7, with Apollo and `@nhost/nhost-js` 3.

**Recommendation for [#19](https://github.com/MLP-Software-Development/stammtisch-app/issues/19):** follow the
native-shell pattern. One Nuxt 4 app with `ssr: false` serves both the web and the two stores. What that means for
Capacitor:

- `nuxt generate` writes a static SPA to `.output/public`, which becomes Capacitor's `webDir`
  (`busflow: apps/driver/capacitor.config.ts`). The web deploy and the store binary are the same build, made with
  different environment values.
- No server rendering in the app. If public pages ever need SEO (a landing page or a feature-request page), they
  live outside the app or on prerendered routes.
- A PWA service worker is fine on the web but is turned off in the native build. WKWebView serves from
  `capacitor://`, where a service worker cannot register (`busflow: apps/driver/nuxt.config.ts` `pwa.disable`).
- Set the Vite build target to the shell's floor (`safari15`, `ios15`) (`busflow: apps/driver/nuxt.config.ts`).
- Avoid Quasar. Busflow spends a whole skill section keeping Quasar's styles from fighting Tailwind. Use Tailwind 4
  plus a headless kit. Which kit is #19's call.
- Use one data client. Busflow mixes Apollo and plain fetch across apps. Pick one GraphQL client and one codegen in
  #19.
- Live reload on a device runs through a `CAP_SERVER_URL` that must never reach a release build
  (`busflow: apps/driver/README.md` §Live-Reload; the `live-reload-leftover` build phase).

## 5. What stays out of a public repo, and how CI works with it

### Never committed

| Item | Where it lives instead |
|:--|:--|
| Nhost admin secret, JWT secret, webhook secret, OAuth client secrets (Apple, Google), SMTP password | Nhost dashboard secrets, referenced from `nhost.toml` as `{{ secrets.NAME }}`; locally in gitignored `.secrets` |
| Postgres connection string with password (needed for `nhost up cloud`) | Local shell or password manager only |
| App Store Connect API key (`.p8`), key ID, issuer ID | GitHub environment `store` secrets + password manager |
| `match` password and the URL of the private certificates repo | GitHub environment `store` secrets |
| Android upload keystore, its passwords, the Play service-account JSON | GitHub environment `store` secrets (keystore base64-encoded) + password manager |
| APNs key, FCM server credentials | Per #7's answer: Nhost secrets or the push provider's dashboard |
| Review demo account password | A secret injected into `deliver`'s review information at upload time |
| Apple team ID | Gitignored `ios/local.xcconfig`; a CI secret |
| Agent or bot tokens, Slack or webhook URLs | Never in the repo; GitHub secrets if a workflow needs one |
| Nhost project subdomain and region | Not secret: they end up in the shipped JS bundle anyway. Still keep them in CI variables and `.env`, not in committed config, so forks and local runs never point at production by default. |
| Real names, the real group, places, bank details (including seeds and screenshots) | Nowhere. Seeds and screenshots use invented data (`AGENTS.md` §Privacy). |

`.env.example` and a `.secrets.example` list the names, never the values.

### How CI works with that

- **Fork PRs get no secrets.** GitHub does not pass secrets to workflows triggered from a fork; only `GITHUB_TOKEN`
  is passed ([GitHub secrets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)).
  So `CI gate` must be fully green with no secrets at all: lint, typecheck, unit tests, `nhost up` with dummy local
  secrets, SQL and permission tests, codegen drift, gitleaks and actionlint.
- **Never use `pull_request_target`** to check out and run PR code; it runs with the base repo's secrets. Busflow's
  rule: pass `${{ github.event.* }}` through `env:`, never inline in `run:` (`busflow: REVIEW.md` §CI workflows).
- **Anything that deploys or signs** runs only on `push` to `main` or on `workflow_dispatch`, inside a GitHub
  environment: `store` for store uploads (required reviewer: the founder) and `web` for the web deploy. The backend
  needs no GitHub secret at all when Nhost's GitHub integration deploys it.
- `permissions: {}` at the workflow level, with per-job grants (`busflow: .github/workflows/ci.yml` line 27).
  Third-party actions pinned by commit SHA.
- gitleaks runs over the PR's commits, not only the tree (`busflow: .github/workflows/pr-checks.yml`). GitHub
  push protection is the backstop.
- Logs: mask anything sensitive that is not a GitHub secret with `::add-mask::`
  ([GitHub secrets](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)).
  fastlane logs go through a redaction step before upload as artifacts (`satzart: scripts/redact-mobile-release-log.mjs`).

## Day-one bootstrap list for this repo

Steps marked **[founder]** need an account or a repo setting. The rest an agent can do. Each ends with a proof.

1. **[founder] Decisions, before any code (#19):** frontend framework and UI kit, web host, bundle ID / application
   ID, whether Apple and Google developer accounts are company-owned, and one Nhost project (EU region per #7) or
   production plus staging. Record them in `docs/adr/0001-…`.
2. **Repo shell:** `package.json` with `packageManager: pnpm@<exact>`, `engines.node >=22`, and
   `"prepare": "git config core.hooksPath .githooks"`; `.nvmrc`; `.editorconfig`; `pnpm-workspace.yaml`
   (`minimumReleaseAge: 1440`). Extend `.gitignore` with `.secrets`, `ios/local.xcconfig`, `*.keystore`, `*.jks`,
   `*.p8`, `*.mobileprovision`, `ios/App/build`, `android/.gradle`, `fastlane/report.xml`, `.output/`.
   *Proof:* `pnpm install && git config core.hooksPath` prints `.githooks`.
3. **Hooks:** `.githooks/pre-commit` and `pre-push` run gitleaks, and `commit-msg` checks the Conventional Commits
   form and keeps this repo's trailer policy. Each failure names its fix. *Proof:* staging a fake AWS key fails the
   commit.
4. **Agent files:** keep `AGENTS.md` and `CLAUDE.md` short. Add `.claude/settings.json` with the deny list
   (`.env*`, `.secrets`, `~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.config/gh`, force push). Add the five lanes in
   `.claude/agents/`, `REVIEW.md`, and `.agents/skills/{frontend,security,docs-authoring}` symlinked into
   `.claude/skills/`. *Proof:* a fresh session lists the lanes.
5. **CI:** `ci.yml` with `permissions: {}`, PR concurrency, and jobs `lint`, `typecheck`, `test`, `build`,
   `secrets` (gitleaks CLI) and `actionlint`, all feeding `CI gate`. Add the PR template, CODEOWNERS (`.claude/`,
   `.agents/`, `CLAUDE.md`, `AGENTS.md`, `.github/`, `nhost/nhost.toml`, `fastlane/`) and a short `renovate.json`.
   *Proof:* a PR shows `CI gate`, and a broken test turns it red.
6. **[founder] GitHub settings:** a ruleset on `main` (PR, `CI gate`, linear history, no force push or deletion,
   code-owner review); secret scanning with push protection, Dependabot alerts and CodeQL default setup; delete
   branches on merge; environments `web` and `store` (reviewer: founder). *Proof:* a direct push to `main` is
   refused.
7. **Backend:** `nhost init` creates `nhost/`. Check that `.secrets` is gitignored, commit `.secrets.example`, and
   add `pnpm dev:stack` wrapping `nhost up` plus the seeds. Set `corsDomain` and the auth redirect URLs to the web
   origin plus `capacitor://localhost` and `https://localhost`, with a test that pins them. *Proof:* `nhost up`
   starts and `nhost config validate` passes.
8. **Hasura guards with the first migration:** a CI check that a migration already on `main` never changes; squawk
   on new `up.sql`; a permission test that every role filter pins the member; a committed schema snapshot with a
   `codegen && git diff --exit-code` check. *Proof:* removing the pin from one permission turns CI red.
9. **[founder] Nhost GitHub integration:** link the repo with base directory `/` and deployment branch `main`;
   add the cloud secrets in the dashboard. *Proof:* a push to `main` shows a green deployment in Nhost's
   Deployments tab.
10. **Web deploy:** build once and deploy the static `.output/public` from `main`, inside the `web` environment.
    Serve `/.well-known/apple-app-site-association` and `assetlinks.json` from it. *Proof:* the URL answers 200.

**Before the first store upload** (playbook §8.7, adapted): **[founder]** create the App Store Connect record, the
Play Console app, the ASC API key, the Play service account, the private `match` repo and the upload keystore, and
store them in `store` secrets and the password manager. Then `npx cap add ios` / `android` (SPM, final identity,
`local.xcconfig`); the `Gemfile` and one `fastlane/Fastfile` with `build_only` and `deploy`, fail-closed store
build numbers, `force: true`, `submit_for_review: false`, the internal track and `verify_artifact`; the wrapper
script; `mobile-release.yml` on `workflow_dispatch`; metadata and screenshots from the seeded demo group;
`docs/runbooks/store-release.md`.

## Differs from Busflow

- **Backend:** Nhost cloud deploys `nhost/` on push. Busflow deploys self-hosted Hasura, NestJS, auth and storage
  with Docker Compose on its own host. There is no `infra/`, Terraform, Traefik, backup cron, observability stack
  or self-hosted runner.
- **No token mint.** Group membership is checked in permission row filters against `x-hasura-user-id`. Busflow
  swaps Nhost tokens for tenant claims in a custom service.
- **Migrations:** the Nhost CLI writes timestamped migrations from the start, and a bad migration is fixed forward,
  since Nhost does not roll migrations back. Busflow needed a generator and later a baseline rebuild.
- **Environments:** one Nhost project per branch-mapped environment, starting with production only. Busflow runs
  per-PR previews on an office host and a release cut with `rc/*` tags.
- **Public repo:** CI must be green without secrets, fork PRs get none, deploys and signing run only from `main` or
  `workflow_dispatch` inside environments, and seeds and screenshots carry invented data. Busflow is private and
  hard-codes machine paths and internal hosts in its agent config.
- **Commit attribution:** this repo keeps the `Co-Authored-By: Claude` trailer. Busflow forbids it with a
  `commit-msg` hook, a `pre-push` scan and an HTTPS push rewrite.
- **Language:** everything except UI copy is English from the first commit. Busflow's `CLAUDE.md` and hook
  messages are partly German.
- **Frontend:** one Nuxt 4 SPA app (`ssr: false`) for the web and both stores. Busflow has nine apps across Quasar,
  Nuxt and others, with shared UI packages and a Histoire hub.
- **Mobile:** Android ships alongside iOS, with a fastlane pipeline from day one of the store work. Busflow's
  shells are iOS-only with no store pipeline yet. The pipeline comes from Satzart, minus its white-label machinery.
- **Tracking work:** this repo tracks work on a wayfinder map, with grilling, research and prototype tickets and
  research on `research/<name>` branches. Busflow uses sprints, waves, a founder inbox, session logs and a
  4,000-line decision ledger.
- **Docs:** this repo keeps `GLOSSARY.md` and `docs/adr/` and adds PARA folders only when needed. Busflow has the
  full PARA tree with 2,500 docs.
- **Agent tooling:** settings deny rules and at most one small guard hook. Busflow has 41 hook modules, a lane
  guard and a provenance `Stop` hook.
- **CI:** one `ci.yml` plus `mobile-release.yml`, on GitHub-hosted runners that are free for public repos.
  Busflow has 60 workflows on Ubicloud and self-hosted runners.
- **Merge on green without Slack.** Busflow uses the same rule, but a blocked session pings the founder through
  `pnpm founder-ping` on Slack. Here a blocked PR becomes an issue comment or a `founder:decide` issue.
