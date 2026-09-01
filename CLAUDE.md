# base-ci

Reusable GitHub Actions for Bun/TypeScript repos (6DQ jobs + SSH deploy action).
Profile: docs-config
Direction: [README.md](README.md). No `docs/` tree. Frameworks must not rewrite this file.

## Sources of Truth

This file is the **contract**. Workflows and self-test CI are **enforcement**. If they disagree, that is a failure — raise enforcement; never lower this file.

| Fact | Where |
|---|---|
| Agent handbook | this file |
| Human docs | README.md, `.github/actions/ssh-deploy/README.md` |
| Version | git tags `v2026`, `v2026.N` (latest `v2026.6`) |
| Enforcement | `.github/workflows/self-test.yml`, `self-test-ssh-deploy.yml` |
| Machine rules | global `AGENTS.md`, `rules/git-commit.md` |
| Accidents | [Retrospective.md](Retrospective.md) |
| Env files | omit |

## Project Invariants

- Consumers call `nocoo/base-ci/.github/workflows/bun-quality.yml@v2026` (or a pinned `v2026.N`). Do not copy-paste job YAML into callers.
- This repo must not store caller secrets. `secrets: inherit` stays on the caller.
- `ignore-scripts` / `trusted-native-deps` are supply-chain contract: keep them consistent with the caller’s `package.json#trustedDependencies`.
- Changing `bun-quality.yml` inputs or job names is a breaking change; bump a `v2026.N` tag and keep `v2026` moving only when intended.
- Do not weaken self-test assertions to land a workflow edit.

## Stack / Layout

| Component | Choice |
|---|---|
| Language | YAML + composite actions (bash) |
| Package manager | omit (fixture Bun locks only under `.github/fixtures/`) |
| Runtime | GitHub Actions |
| Lint | YAML parse in `self-test.yml` |
| Tests | `self-test.yml` + `self-test-ssh-deploy.yml` |
| Data | none |

```
.github/workflows/bun-quality.yml     reusable 6DQ workflow
.github/workflows/self-test.yml       syntax + fixture run
.github/actions/ssh-deploy/           composite SSH deploy
.github/fixtures/self-test/           dummy Bun app for the reusable workflow
```

## Commands

No root package scripts. Validate by pushing to `main` (self-test) or opening a PR.

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/bun-quality.yml'))"
```

## Verification

Status: `enforced` | `planned` | `manual` | `N/A`. `enforced` Evidence = hook/CI/config/script. `planned` has no Evidence.

docs-config: omit product L1/L2/L3/G2/build/release rows. This repo’s bar is self-test CI.

| Change | Proof | Status | Evidence |
|---|---|---|---|
| Workflow syntax + inputs | YAML load + structure asserts | enforced | `.github/workflows/self-test.yml` `validate` job |
| Reusable workflow behavior | fixture app through `bun-quality.yml` | enforced | `self-test.yml` remaining jobs |
| SSH deploy action | self-test workflow | enforced | `.github/workflows/self-test-ssh-deploy.yml` |
| Types / lint / coverage | n/a product suite | N/A | — |
| Docs | README matches new inputs | manual | human review |
| Release | annotated tag `v2026.N` and moving `v2026` | manual | operator `git tag` |

No local husky. `--no-verify` still forbidden if hooks appear later.

## Retrospective

| Kind | Where |
|---|---|
| Accident narrative | [Retrospective.md](Retrospective.md) |
| Recurring project rule | one line here (cap ~10) |
| Cross-project | nmem / global rules |
| Checkable rule | workflow assert |

- (none yet)
