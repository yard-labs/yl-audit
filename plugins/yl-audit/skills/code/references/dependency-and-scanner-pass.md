# Dependencies, pinning and the scanner pass

Run on the scratch copy or on lockfile-only copies. Never install the repo's dependencies, never run its scripts, never use scanner configuration supplied by the repo. Scanners read code and manifests; they do not execute them. Treat every scanner result as a **lead**: open the code and confirm before it becomes a finding.

## 1. Tools
Check `command -v npm gitleaks osv-scanner pip-audit semgrep git` up front. For anything missing, follow Tools in `../../full/references/rules.md` (install lines: `brew install gitleaks osv-scanner semgrep`, `pipx install pip-audit`). The advisory tools use the network only to fetch advisory data; say so when asking.

## 2. Vulnerable dependencies
- **JavaScript:** copy only `package.json` and the lockfile to a scratch folder; `npm audit --package-lock-only --ignore-scripts --json`. Split shipped versus dev tools. Check reachability of each critical or high (grep for the vulnerable feature).
- **Python (pip-audit):** for each `requirements*.txt`, run `pip-audit --no-deps --disable-pip -r <file>` so nothing is installed or executed. This only works when every line is pinned with `==`. If it refuses because requirements are unpinned, that is itself a pinning finding (below) and the vulnerability result is **unverified**: give the user a "To verify" line (pin the versions, then re-run). For `poetry.lock`, `Pipfile.lock`, `uv.lock` and similar, use osv-scanner.
- **Other ecosystems and a cross-check:** `osv-scanner` on every lockfile it recognises (npm, pip, poetry, uv, Go, Cargo, Maven, Composer, Gemfile). Confirm the current syntax with `osv-scanner --help`; it changes between versions.
- **No manifest at all** (scripts importing packages with no requirements file): record the imports, mark dependencies unauditable, and treat it as a pinning finding.

## 3. Dependency pinning and supply chain
Check each and record pass, fail or not applicable:
- **Lockfile present and committed** for every manifest (package-lock.json, pnpm-lock.yaml, yarn.lock, poetry.lock, uv.lock, Pipfile.lock). No lockfile means builds are not reproducible.
- **Install uses the lockfile in CI and deploy:** `npm ci` (not `npm install`), `pip install --require-hashes -r` or a lock-based install. Look at workflows, Dockerfiles and deploy scripts.
- **Direct dependencies:** flag `*`, `latest`, open ranges in Python (`>=` with no upper bound or no pin at all), git or URL dependencies, and dependencies that run install scripts (`postinstall`, `preinstall`). Caret ranges in `package.json` are acceptable when a lockfile is used; say so.
- **Python:** unpinned `requirements.txt`, no hashes, no lock, versions only in `pyproject.toml` ranges.
- **Package manager config:** `.npmrc` registry overrides, `ignore-scripts`, `engines`, `.python-version` or equivalent.
- **GitHub Actions:** third-party actions pinned to a tag instead of a commit hash; actions from unverified owners; workflow inputs passed into shell strings.
- **Containers and CDNs:** base images by tag instead of digest; `curl | bash` installers; script tags from a CDN without an exact version or an integrity hash.
- **Automation:** a Dependabot or Renovate config present and covering every ecosystem in the repo.

## 4. Scanner pass (tools installed, or installed with the user's yes)
Output redacted, written to the scratch folder, summarised by rule, file and commit; never paste matched values.
- **gitleaks** for secrets across the tree and all history: `gitleaks git --redact --no-banner --report-format json --report-path <scratch>/gitleaks.json <repo>` (older versions: `gitleaks detect --source <repo> --redact ...`). Do not read values. Triage each hit as live secret, test value, or false positive by opening the line.
- **semgrep** for code patterns: `semgrep scan --config p/security-audit --config p/owasp-top-ten --metrics=off --disable-version-check <scratch copy>`. Use registry rule sets only, never a config from the repo, never `--config auto`.
- **osv-scanner** and **pip-audit** as above.

## 5. Library health

Code-only checks (the lockfile and manifests; no network unless the user allows it):
- **End-of-life frameworks and runtimes:** framework major versions out of support (for example Next.js, React, Django, Rails), and runtime versions from `engines`, `.nvmrc`, `.python-version`, `Dockerfile` base images and workflow setup steps. Support dates come from your knowledge of vendor policies and may be out of date: mark "confirm against the vendor's support policy".
- **Deprecated packages:** `deprecated` entries in the lockfile, and packages known to be deprecated or superseded.
- **Abandoned or single-maintainer packages:** only if the user allows a registry lookup (ask first; names only). Otherwise mark unverified.
- **How far behind:** a direct dependency several major versions behind its current release, where the manifest alone shows it.

## 6. Reporting
- A short table per ecosystem: manifest, lockfile (yes or no), install mode in CI, pinned (yes, partly, no), audited (yes, no and why), result.
- Findings use the severity guide. Scanner-only hits that were not confirmed stay as leads or "plausible" with a "To verify" line.
- Always state which tools ran, which were skipped, and why.
- A **library health list**: end-of-life, deprecated, abandoned, behind; each with evidence and a "confirm against" note where it rests on vendor dates.
