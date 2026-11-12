---
name: ci
description: Read-only audit of an application's CI/CD pipeline from its repo config (GitHub Actions, Dockerfiles, hosting config, deploy and migration scripts, git hooks). Checks local git hooks, quality gates (lint, format, typecheck, tests, dependency audit, build), secret scanning and static analysis, vulnerability handling (Dependabot coverage, security fix PRs, exact pins), runtime version drift, PR practice and automated PR review, build-once artifacts promoted through environments, a pre-production environment, manual production trigger, pre-deploy migrations, feature flags, environment variables, short-lived branches and pipeline supply-chain hygiene. Works on one repo or compares several side by side. Scores gates, security and delivery 0 to 5 each against what is right for the team's stage, with the CI level set by the weakest, and gives ordered steps plus "to verify" checks for the settings the repo cannot show. Use when reviewing how an app is built, tested and shipped.
argument-hint: [path to repo, or several] [optional: team size and stage, e.g. "solo, early" or "5 people, with a test environment"]
---

# CI audit

Review how code gets from a commit to production. The goal is a pipeline that is safe, repeatable and proportionate: more is expected of a team than of one person, and the report says which is which.

## Rules

Follow `../full/references/rules.md`. On top of it: never trigger a workflow. Platform settings (GitHub, the host, the registry) are not visible from the repo: mark them unverified with a To verify line each from `references/settings-checks.md`.

## What good looks like

Each item is expected by default; the stage table in `references/practices.md` says when it is optional.

1. **Quality gates on every push to main, and on PRs if PRs are used:** lint, format check, typecheck (every language and package), tests, dependency audit, secret scanning over the full history, and a build. Static analysis (Semgrep or CodeQL) where the stage calls for it.
2. **PRs.** Optional for a one- or two-person team who are happy with their coverage. If PRs are used: the same gates required before merge, branches short-lived, and an automated review on every PR.
3. **Build once, promote the artifact.** CI builds a versioned, immutable package (for example a container image tagged with the commit) and pushes it to a registry. The same artifact moves through environments. Apps built and deployed by the hosting provider from git are flagged: each environment builds differently, builds are unreviewed, and the pipeline does not control what runs.
4. **A pre-production environment.** Staging and test mean the same thing here: a place to try a change before production, ideally with its own database and its own secrets. Call it whatever the client does. What matters is that one exists, or that it is added soon if it does not. Two environments (production plus one pre-production) is the right number for most teams at this stage; three is almost always too many to keep in step, so do not recommend adding one. On push or merge to main, deploy to it automatically.
5. **Production by a deliberate trigger.** A manual button or dispatch, automated in the pipeline (for example GitHub Actions with an environment approval), promoting the artifact already proven in the pre-production environment, with a rollback path.
6. **Migrations applied automatically before the deploy,** in the pipeline, with a dedicated migration credential scoped to the environment, applied-state tracked, backward-compatible with the code still running (expand then contract), and tested on a disposable database in CI.
7. **Feature flags** to separate deploy from release. A win, not required; depends on stage and app.
8. **Environment variables for configuration.** No hard-coded config or secrets; per-environment values; secrets in environment-scoped secrets, not the repo; an example file listing names; the app fails fast if a required value is missing.
9. **No long-lived branches.** Short-lived branches for PRs are fine.
10. **Pipeline hygiene.** Third-party Actions pinned to commit hashes (version tags on GitHub-owned or verified actions are accepted, not a gap), least-privilege tokens (read-only by default, widened per job), no secrets reachable by untrusted code, short artifact retention, concurrency and timeouts set on every job, one runtime version across CI, deploy and Dockerfiles.
11. **Vulnerability handling.** Dependabot alerts and security fix PRs on, version updates for every ecosystem (packages, Docker base images, Actions), exact dependency pins, and a free secret scanner (gitleaks) where GitHub's paid secret scanning is off.
12. **Local git hooks** that install themselves and run a secret scan, format, lint and typecheck before code reaches CI. CI stays the gate; hooks catch problems earlier.

## Levels

Score three pillars 0 to 5. Each level needs everything below it. The **CI level is the lowest of the three**, so one weak pillar caps it. Report all three so the reader sees which one sets the cap.

**Gates** (hooks and CI together: what stops bad code)

| Level | Gates |
|---|---|
| 0 | No gates |
| 1 | Some checks run, but they do not fail the job, or not on every push |
| 2 | Core gates fail the job on every push to main (and every PR if used): format, lint, typecheck, tests, build. Some languages or packages may be missing |
| 3 | Core gates cover every language and package; every job has a timeout; hooks install themselves and run format, lint and typecheck |
| 4 | Plus **migrations** tested on a fresh database (and a schema contract where there is one), and the **build output audited** for leaked secrets. Not applicable where there is no database or client bundle |
| 5 | Plus an **automated review on every PR** (n/a if no PRs), required before merge, and nothing weakened: no `continue-on-error`, skipped or deselected tests, or rules switched off repo-wide without a reason |

**Security** (what finds and fixes leaked secrets and known holes)

| Level | Security |
|---|---|
| 0 | Nothing |
| 1 | Vulnerability alerts on |
| 2 | Gitleaks (or equivalent) in CI over the full history, failing the job. GitHub secret scanning with push protection counts where it is on (To verify) |
| 3 | Plus a gitleaks hook that fails the commit (partial if it skips when gitleaks is missing), a dependency audit failing CI for every ecosystem, and security fix PRs on |
| 4 | Plus Dependabot or Renovate for every ecosystem (packages, Docker base images, Actions), exact pins and a committed lockfile |
| 5 | Plus static analysis (Semgrep or CodeQL) in CI, and no High alert open longer than 30 days (To verify with `gh api` if alerts are not readable) |

**Delivery** (how code reaches production)

| Level | Delivery |
|---|---|
| 0 | Built and deployed by hand |
| 1 | The host builds and deploys from git; the pipeline does not control what runs |
| 2 | The pipeline deploys; migrations run automatically before the deploy; config comes from the environment |
| 3 | A pre-production environment deploys automatically on merge to main |
| 4 | Built once into a registry, the same artifact promoted; production by a manual trigger with approval |
| 5 | Plus a rollback path, feature flags where they fit, short-lived branches, and pinned, least-privilege pipeline permissions |

**Targets by stage** (stages from `../full/references/rules.md`):

| Stage | Gates | Security | Delivery |
|---|---|---|---|
| S1 | 3 | 2 | 3 |
| S2 | 4 | 3 | 4 |
| S3 | 4 | 4 | 5 |

A pillar below its target is a finding. A solo early-stage app at gates 3, security 2, delivery 3 is in a good place; a five-person team there is not.

Report it as a table, then the CI level:

| Pillar | Level | Target | Capped by |
|---|---|---|---|
| Gates | n | t | the first unmet rung, with check IDs |
| Security | n | t | ... |
| Delivery | n | t | ... |

**CI level: n / 5** (lowest pillar, named). Score main as it is; where an open PR would lift a pillar, add "(m after PR n)".

## Procedure

1. **Intake.** Ask once: team size and stage, whether PRs are used, hosting, database, whether containers are intended, known findings. Use the answer to set the stage and the three pillar targets.
2. **Inventory.** `.github/workflows/*`, Dockerfiles and compose files, hosting config (`vercel.json`, `netlify.toml`, `fly.toml`, `render.yaml`, `Procfile`), deploy and migration scripts, package scripts (`prepare`, `postinstall`), `.env*.example`, Dependabot or Renovate config, hook config (`lefthook.yml`, `.husky/`, `.pre-commit-config.yaml`), scanner config (`.gitleaks.toml`, Semgrep, CodeQL), runtime files (`.nvmrc`, `.python-version`, `engines`), `.gitignore`, PR templates, CODEOWNERS, feature-flag libraries. If none of these exist, the repo has no CI/CD: report **level 0** on every pillar against the stage targets and treat "no pipeline" as the finding, rather than reporting nothing.
3. **Map the pipeline.** Draw a plain-text diagram from commit to production: triggers, jobs, artifacts, environments, who or what can trigger production, and where migrations run. Mark each element Code, Config or Docs.
4. **Evaluate** each check in `references/practices.md` (C1 to C31, then hooks H1 to H5) with `file:line` evidence: pass, partial, fail, not applicable for the stage, or unverified. Assess main as it is. If an open PR fixes a gap (read-only: `gh pr list`, `gh pr diff`), still rate the gap on main and mark it with the PR number; do not count it as fixed.
5. **Branches and PRs** via read-only git: branch list and last-commit ages, branches far ahead of or behind main, stale or unmerged branches, the commit pattern (direct to main versus merges).
6. **Levels.** Score each pillar against its rungs and target, using the check results from step 4, and give the pillar table and CI level from the Levels section.
7. **Settings to verify.** Produce the table from `references/settings-checks.md` for everything the repo cannot show. Each row: **To verify:** do this (where). **Expect:** result. **If not:** meaning.
8. **Several repos.** When given more than one repo, run steps 2 to 7 for each, then add a comparison before the findings:
   - a one-line verdict per repo (where it stands, its biggest gap);
   - a levels table with repos as rows: gates, security, delivery and CI level, each with its target, and what caps it;
   - three matrices with repos as columns, rated In place, Partial, Gap or Not applicable: **Git hooks** (tool, installs itself, gitleaks, format and lint, typecheck, tests), **CI checks** (format and lint, typecheck per language, tests, build, database, dependency audit and threshold, gitleaks, static analysis, automated PR review, job timeouts) and **Vulnerability handling** (alerts, security fix PRs, Dependabot ecosystems, Docker base images, exact pins, GitHub secret scanning);
   - gaps by repo, most important first, with the riskiest repo first and its reason (for example the repo holding the most third-party keys).
   Keep each cell short: what is there, or what is missing, with the PR number if a fix is pending.
9. **Report** in the shared findings format (`../full/references/report-template.md`): overview, the pillar table and CI level, pipeline diagram, check table, findings (ID, name, finding, criticality, risk, remediation) High to Low, ordered recommendations with effort tags and a suggested order, what is working well, the To verify list, and coverage limits.
