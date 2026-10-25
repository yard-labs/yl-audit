---
name: ci
description: Read-only audit of an application's CI/CD pipeline from its repo config (GitHub Actions, Dockerfiles, hosting config, deploy and migration scripts). Checks quality gates (lint, format, typecheck, tests, dependency audit, build), PR practice and automated PR review, build-once artifacts promoted through environments, a pre-production environment, manual production trigger, pre-deploy migrations, feature flags, environment variables, short-lived branches and pipeline supply-chain hygiene. Levels the pipeline 0 to 5 against what is right for the team's stage, and gives ordered steps plus "to verify" checks for the settings the repo cannot show. Use when reviewing how an app is built, tested and shipped.
argument-hint: [path to repo] [optional: team size and stage, e.g. "solo, early" or "5 people, with a test environment"]
---

# CI audit

Review how code gets from a commit to production. The goal is a pipeline that is safe, repeatable and proportionate: more is expected of a team than of one person, and the report says which is which.

## Rules

Follow `../full/references/rules.md`. On top of it: never trigger a workflow. Platform settings (GitHub, the host, the registry) are not visible from the repo: mark them unverified with a To verify line each from `references/settings-checks.md`.

## What good looks like

Each item is expected by default; the stage table in `references/practices.md` says when it is optional.

1. **Quality gates on every push to main, and on PRs if PRs are used:** lint, format check, typecheck, tests, dependency audit, and a build.
2. **PRs.** Optional for a one- or two-person team who are happy with their coverage. If PRs are used: the same gates required before merge, branches short-lived, and an automated review on every PR.
3. **Build once, promote the artifact.** CI builds a versioned, immutable package (for example a container image tagged with the commit) and pushes it to a registry. The same artifact moves through environments. Apps built and deployed by the hosting provider from git are flagged: each environment builds differently, builds are unreviewed, and the pipeline does not control what runs.
4. **A pre-production environment.** Staging and test mean the same thing here: a place to try a change before production, ideally with its own database and its own secrets. Call it whatever the client does. What matters is that one exists, or that it is added soon if it does not. Two environments (production plus one pre-production) is the right number for most teams at this stage; three is almost always too many to keep in step, so do not recommend adding one. On push or merge to main, deploy to it automatically.
5. **Production by a deliberate trigger.** A manual button or dispatch, automated in the pipeline (for example GitHub Actions with an environment approval), promoting the artifact already proven in the pre-production environment, with a rollback path.
6. **Migrations applied automatically before the deploy,** in the pipeline, with a dedicated migration credential scoped to the environment, applied-state tracked, backward-compatible with the code still running (expand then contract), and tested on a disposable database in CI.
7. **Feature flags** to separate deploy from release. A win, not required; depends on stage and app.
8. **Environment variables for configuration.** No hard-coded config or secrets; per-environment values; secrets in environment-scoped secrets, not the repo; an example file listing names; the app fails fast if a required value is missing.
9. **No long-lived branches.** Short-lived branches for PRs are fine.
10. **Pipeline hygiene.** Actions pinned to commit hashes, least-privilege tokens (read-only by default, widened per job), no secrets reachable by untrusted code, short artifact retention, concurrency and timeouts set, Dependabot or Renovate.

## Levels

| Level | Pipeline |
|---|---|
| 0 | No CI; hand-run builds and deploys |
| 1 | CI exists but only for scheduled or utility jobs; no gates |
| 2 | Gates run on every push to main (and PRs if used) |
| 3 | Gates plus automatic pre-deploy migrations and environment-based config; the host still builds from git |
| 4 | Build-once artifact in a registry, deployed by the pipeline, production by a manual trigger with approval and a rollback path |
| 5 | The pre-production environment deploys automatically on merge, the same artifact is promoted to production, PRs get an automated review, short-lived branches, feature flags where they fit, pinned and scoped everywhere |

Report the level reached and the level that is **right for the stage** (see the stage table). A solo early-stage app at level 4 with one pre-production environment and no PRs is in a good place. A five-person team at level 3 is not.

## Procedure

1. **Intake.** Ask once: team size and stage, whether PRs are used, hosting, database, whether containers are intended, known findings. Use the answer to set the target level.
2. **Inventory.** `.github/workflows/*`, Dockerfiles and compose files, hosting config (`vercel.json`, `netlify.toml`, `fly.toml`, `render.yaml`, `Procfile`), deploy and migration scripts, package scripts, `.env*.example`, Dependabot or Renovate config, PR templates, CODEOWNERS, feature-flag libraries. If none of these exist, the repo has no CI/CD: report **level 0** against the stage target and treat "no pipeline" as the finding, rather than reporting nothing.
3. **Map the pipeline.** Draw a plain-text diagram from commit to production: triggers, jobs, artifacts, environments, who or what can trigger production, and where migrations run. Mark each element Code, Config or Docs.
4. **Evaluate** each check in `references/practices.md` with `file:line` evidence: pass, partial, fail, not applicable for the stage, or unverified.
5. **Branches and PRs** via read-only git: branch list and last-commit ages, branches far ahead of or behind main, stale or unmerged branches, the commit pattern (direct to main versus merges).
6. **Level and target.** Report both.
7. **Settings to verify.** Produce the table from `references/settings-checks.md` for everything the repo cannot show. Each row: **To verify:** do this (where). **Expect:** result. **If not:** meaning.
8. **Report** in the shared findings format (`../full/references/report-template.md`): overview, pipeline diagram, check table, findings (ID, name, finding, criticality, risk, remediation) High to Low, ordered recommendations with effort tags and a suggested order, what is working well, the To verify list, and coverage limits.
