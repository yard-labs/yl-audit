---
name: ci
description: Read-only audit of an application's CI/CD pipeline from its repo config (GitHub Actions, Dockerfiles, hosting config, deploy and migration scripts, git hooks). Checks local git hooks, quality gates (lint, format, typecheck, tests, dependency audit, build), secret scanning and static analysis, vulnerability handling (Dependabot coverage, security fix PRs, exact pins), runtime version drift, PR practice and automated PR review, build-once artifacts promoted through environments, a pre-production environment, manual production trigger, pre-deploy migrations, feature flags, environment variables, short-lived branches and pipeline supply-chain hygiene. Works on one repo or compares several side by side. Scores gates, security and delivery 0 to 5 each against what is right for the team's stage, with the CI level set by the weakest, reads the repo's GitHub settings with read-only `gh api` calls when allowed, and gives ordered steps plus "to verify" checks for the settings it cannot read. Use when reviewing how an app is built, tested and shipped.
argument-hint: [path to repo, or several] [optional: team size and stage, e.g. "solo, early" or "5 people, with a test environment"]
---

# CI audit

Review how code gets from a commit to production. The goal is a pipeline that is safe, repeatable and proportionate: more is expected of a team than of one person, and the report says which is which.

## Rules

Follow `../full/references/rules.md`. On top of it: never trigger a workflow. GitHub settings are read with the read-only `gh api` calls in `references/settings-checks.md` when the user agreed at intake. Settings that cannot be read that way (the host, the registry, cloud IAM, or GitHub without access) get a To verify line each from the same file.

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
| 3 | Core gates cover every language and package that holds application code (the app, its jobs, shared libraries); every job has a timeout; hooks install themselves and run format, lint and typecheck |
| 4 | Plus **migrations** tested on a fresh database, and the **build output audited** for leaked secrets. A schema drift check or contract is a plus, not required. Migrations are n/a with no database. The build output audit is n/a with no client bundle, and met with no scan when nothing environment-specific is baked into the build (config served at runtime from an allowlist of public values; see C30) |
| 5 | Plus an **automated review on every PR** (n/a if no PRs), required before merge, and nothing weakened: no `continue-on-error`, skipped or deselected tests, or rules switched off repo-wide without a reason |

A small tooling or config package (infra-as-code, scripts) that has its own tests but misses a gate is a Low finding, not a cap.

**Security** (what finds and fixes leaked secrets and known holes)

| Level | Security |
|---|---|
| 0 | Nothing |
| 1 | Vulnerability alerts on |
| 2 | Gitleaks (or equivalent) in CI over the full history, failing the job. GitHub secret scanning with push protection counts where it is on |
| 3 | Plus a gitleaks hook that fails the commit (partial if it skips when gitleaks is missing), a dependency audit failing CI for every ecosystem, and security fix PRs on |
| 4 | Plus Dependabot or Renovate for every ecosystem (packages, Docker base images, Actions), exact pins and a committed lockfile |
| 5 | Plus static analysis (Semgrep or CodeQL) in CI, and no High alert open longer than 30 days |

Security fix PRs and open alerts are repo settings, not `dependabot.yml`: read them with `gh api` (see `references/settings-checks.md`). A push's `remote:` output naming open vulnerabilities by severity is also evidence.

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

| Pillar | Level | Target | Why not higher |
|---|---|---|---|
| Gates | n | t | one plain sentence naming what is missing, with a finding ID |
| Security | n | t | ... |
| Delivery | n | t | ... |

**CI level: n / 5** (lowest pillar, named). Score main as it is; where an open PR would lift a pillar, add "(m after PR n)".

Write for a reader who has not seen the rubric. Say what each pillar means in a few words (what stops bad code, what finds holes, how code reaches production), say plainly whether the target is met, and name gaps, not rung numbers. A gap that is a deliberate, documented choice (an advisory reviewer kept out of required checks) is stated as such, not as a failure.

## Procedure

1. **Intake.** Ask once: team size and stage, whether PRs are used, hosting, database, whether containers are intended, known findings, and whether you may read the repo's GitHub settings with their `gh` login (read-only GET calls). Use the answer to set the stage and the three pillar targets.
2. **Inventory.** `.github/workflows/*`, Dockerfiles and compose files, hosting config (`vercel.json`, `netlify.toml`, `fly.toml`, `render.yaml`, `Procfile`), deploy and migration scripts, package scripts (`prepare`, `postinstall`), `.env*.example`, Dependabot or Renovate config, hook config (`lefthook.yml`, `.husky/`, `.pre-commit-config.yaml`), scanner config (`.gitleaks.toml`, Semgrep, CodeQL), runtime files (`.nvmrc`, `.python-version`, `engines`), `.gitignore`, PR templates, CODEOWNERS, feature-flag libraries. If none of these exist, the repo has no CI/CD: report **level 0** on every pillar against the stage targets and treat "no pipeline" as the finding, rather than reporting nothing.
3. **Map the pipeline.** Draw a plain-text diagram from commit to production: triggers, jobs, artifacts, environments, who or what can trigger production, and where migrations run. Mark each element Code, Config or Docs.
4. **Evaluate** each check in `references/practices.md` (C1 to C31, then hooks H1 to H5) with `file:line` evidence: pass, partial, fail, not applicable for the stage, or unverified. Assess main as it is. If an open PR fixes a gap (read-only: `gh pr list`, `gh pr diff`), still rate the gap on main and mark it with the PR number; do not count it as fixed.
5. **Branches and PRs** via read-only git: branch list and last-commit ages, branches far ahead of or behind main, stale or unmerged branches, the commit pattern (direct to main versus merges).
6. **Settings.** Read every GitHub row of `references/settings-checks.md` with its `gh api` call, if the user agreed, and rate the related checks from what comes back, before scoring. Only rows that could not be read go in the report's To verify list. Each: **To verify:** do this (where). **Expect:** result. **If not:** meaning.
7. **Levels.** Score each pillar against its rungs and target, using the check results from steps 4 and 6, and give the pillar table and CI level from the Levels section.
8. **Several repos.** When given more than one repo, run steps 2 to 7 for each, then add a comparison before the findings:
   - a one-line verdict per repo (where it stands, its biggest gap);
   - a levels table with repos as rows: gates, security, delivery and CI level, each with its target, and what caps it;
   - three matrices with repos as columns, rated In place, Partial, Gap or Not applicable: **Git hooks** (tool, installs itself, gitleaks, format and lint, typecheck, tests), **CI checks** (format and lint, typecheck per language, tests, build, database, dependency audit and threshold, gitleaks, static analysis, automated PR review, job timeouts) and **Vulnerability handling** (alerts, security fix PRs, Dependabot ecosystems, Docker base images, exact pins, GitHub secret scanning);
   - gaps by repo, most important first, with the riskiest repo first and its reason (for example the repo holding the most third-party keys).
   Keep each cell short: what is there, or what is missing, with the PR number if a fix is pending.
9. **Report** in the shared findings format (`../full/references/report-template.md`), ordered so the reader gets the answer first:
   1. a verdict of two or three sentences: where it stands for the stage, and what needs fixing;
   2. the pillar table and CI level;
   3. findings rated Medium or above, each with what, why it matters, the fix and effort;
   4. Low findings as one table of smaller improvements (improvement, why, effort), roughly by value;
   5. what is working well;
   6. the pipeline diagram;
   7. the check table (C1 to C31, H1 to H5), collapsed or last, since it is reference;
   8. the To verify list, only for what could not be read;
   9. coverage limits.
   Keep the reader-facing text free of rung numbers and audit jargon; check IDs belong in the check table and evidence.
