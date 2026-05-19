# security/semgrep-cicd

Shared Semgrep configuration for CI/CD pipelines across all projects in the organization.

---

## What is Semgrep in CI/CD?

[Semgrep](https://semgrep.dev) is a static analysis (SAST) tool that scans source code for security vulnerabilities, dangerous patterns, and policy violations — **before code is built or deployed**.

In CI/CD, Semgrep acts as a **security gate** at the very start of the pipeline.

---

## Why scan before build?

- **Fail fast, fix cheap**: Catch vulnerabilities the moment code is pushed, not after deployment.
- **No wasted build time**: Stop the pipeline at `scan` stage if critical issues are found, before spending time on build/test/deploy.
- **Security in the development loop**: Developers get immediate feedback in their MR, not from a delayed manual security review.

---

## Repo Structure

```
security/semgrep-cicd
├── README.md
├── .semgrepignore
├── .github/
│   └── workflows/
│       └── semgrep.yml                  # Reusable GitHub Actions workflow
├── templates/
│   └── gitlab-semgrep.yml               # GitLab CI template (if using GitLab)
└── rules/
    ├── typescript/
    │   ├── nodejs-security.yml          # Core security rules (eval, JWT, injection)
    │   ├── malware-detection.yml        # Malware, obfuscation, suspicious code
    │   └── secrets.yml                  # Hardcoded secrets and credentials
    └── docker/
        └── dockerfile-security.yml     # Dockerfile security rules
```

---

## Integrating into a GitHub Project

GitHub uses **Reusable Workflows** — other repos call this workflow directly instead of copying it.

### Step 1: Create `.github/workflows/ci.yml` in your project

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  security-scan:
    uses: Spinel-AI/interlink_semgrep_scan/.github/workflows/semgrep.yml@main
    with:
      fail-on-findings: true
    secrets: inherit

  build:
    needs: security-scan      # build only runs after scan passes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```



### Step 2: Allow access to the rules repo

If `interlink_semgrep_scan` is a **private repo**, go to:
`Settings → Actions → General → Access → Allow access from other repos in the org`

Or create a PAT and store it as a secret `SEMGREP_RULES_TOKEN` in each project.

---

## Workflow Inputs

| Input | Default | Description |
|-------|---------|-------------|
| `fail-on-findings` | `true` | `true` = fail workflow on findings; `false` = warn only |
| `rules-ref` | `main` | Branch or tag of this repo to use |

---

## Enable / Disable Pipeline Failure

**Fail on findings (default — recommended for production):**

```yaml
jobs:
  security-scan:
    uses: Spinel-AI/interlink_semgrep_scan/.github/workflows/semgrep.yml@main
    with:
      fail-on-findings: true
```

**Warn only — use when onboarding a new project:**

```yaml
jobs:
  security-scan:
    uses: Spinel-AI/interlink_semgrep_scan/.github/workflows/semgrep.yml@main
    with:
      fail-on-findings: false
```

---

## Running Locally

**Scan with Semgrep default ruleset:**

```bash
docker run --rm -v "${PWD}:/src" semgrep/semgrep:1.162.0 semgrep scan --config auto /src
```

**Scan with both default and custom rules from this repo:**

```bash
# Clone this repo first
git clone git@github.com:Spinel-AI/interlink_semgrep_scan.git /tmp/semgrep-rules

# Run with all rules
semgrep scan --config auto --config /tmp/semgrep-rules/rules .
```

**If you are inside this repo:**

```bash
semgrep scan --config auto --config rules .
```

---

## Rule Coverage

### TypeScript / Node.js — Core Security (`nodejs-security.yml`)

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `ts.security.no-eval` | ERROR | Detects `eval()` usage |
| `ts.security.no-new-function` | ERROR | Detects `new Function()` — equivalent to eval |
| `ts.security.jwt-hardcoded-secret` | ERROR | JWT signed with a hardcoded string literal |
| `ts.security.child-process-exec` | WARNING | `exec()` / `child_process.exec()` — shell injection risk |
| `ts.security.sql-injection-template` | ERROR | SQL queries built with template literals from user input |
| `ts.security.sql-injection-concat` | ERROR | SQL queries built with string concatenation |
| `ts.security.nosql-injection` | ERROR | MongoDB queries with unvalidated user input |
| `ts.security.open-redirect` | ERROR | `res.redirect()` with unvalidated user-supplied URL |
| `ts.security.path-traversal` | ERROR | `fs` operations with user input as file path |
| `ts.security.prototype-pollution` | ERROR | Direct assignment to `__proto__` or `Object.prototype` |
| `ts.security.insecure-deserialize` | ERROR | `node-serialize` / unsafe deserialization |
| `ts.security.weak-hash` | WARNING | MD5 or SHA1 used for cryptographic hashing |
| `ts.security.weak-cipher` | ERROR | Deprecated `crypto.createCipher` (no IV — vulnerable to attacks) |
| `ts.security.insecure-random` | WARNING | `Math.random()` used in security-sensitive context |
| `ts.security.vm-execution` | ERROR | `vm.runInNewContext` / `vm.runInThisContext` — arbitrary code execution |
| `ts.security.settimeout-string-eval` | WARNING | `setTimeout`/`setInterval` called with a string argument |
| `ts.security.ssrf-fetch` | ERROR | `fetch()` called with unvalidated user-supplied URL |
| `ts.security.ssrf-axios` | ERROR | `axios.get/post` called with unvalidated user-supplied URL |
| `ts.security.direct-process-env` | WARNING | `process.env` read directly instead of via `ConfigService` |
| `ts.security.no-console-log` | WARNING | `console.log` in production source code |

### TypeScript / Node.js — Malware & Obfuscation (`malware-detection.yml`)

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `ts.malware.eval-obfuscated-fromcharcode` | ERROR | `eval(String.fromCharCode(...))` — classic obfuscation |
| `ts.malware.eval-obfuscated-atob` | ERROR | `eval(atob(...))` — base64-encoded payload execution |
| `ts.malware.eval-obfuscated-buffer-b64` | ERROR | `eval(Buffer.from(..., 'base64').toString())` — encoded payload |
| `ts.malware.dynamic-require` | ERROR | `require()` called with a variable instead of a string literal |
| `ts.malware.env-exfiltration-fetch` | ERROR | `process.env` sent to external URL via `fetch` |
| `ts.malware.env-exfiltration-axios` | ERROR | `process.env` sent to external URL via `axios` |
| `ts.malware.sensitive-file-access` | ERROR | Read access to `/etc/passwd`, `/etc/shadow`, `~/.ssh` |
| `ts.malware.exec-dynamic-input` | ERROR | `exec()` called with dynamic/concatenated string input |
| `ts.malware.suspicious-network-spawn` | ERROR | Spawning `curl`/`wget`/`nc` — common in reverse shells |
| `ts.malware.write-to-cron` | ERROR | Writing to cron directories (`/etc/cron*`, `/var/spool/cron`) |
| `ts.malware.base64-decode-exec` | ERROR | Decoding base64 and immediately executing the result |
| `ts.malware.suspicious-hex-string-exec` | WARNING | Hex-encoded string used in execution context |
| `ts.malware.postinstall-network-call` | WARNING | Network call detected inside lifecycle scripts |
| `ts.malware.dns-rebinding-wildcard` | WARNING | Wildcard DNS or CORS `*` origin allowing any host |

### TypeScript / Node.js — Hardcoded Secrets (`secrets.yml`)

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `ts.secrets.private-key` | ERROR | RSA/EC/DSA private key embedded in source |
| `ts.secrets.aws-access-key` | ERROR | AWS Access Key ID pattern (`AKIA...`) |
| `ts.secrets.aws-secret-key` | ERROR | AWS Secret Access Key hardcoded |
| `ts.secrets.credentials-in-url` | ERROR | Password in database connection URL |
| `ts.secrets.generic-api-key` | WARNING | Generic `apiKey`/`api_key`/`secret` assigned a string literal |
| `ts.secrets.github-token` | ERROR | GitHub personal access token (`ghp_...`) |
| `ts.secrets.google-api-key` | ERROR | Google API key (`AIza...`) |

### Dockerfile (`dockerfile-security.yml`)

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `dockerfile.no-latest-tag` | WARNING | `FROM image:latest` — non-reproducible builds |
| `dockerfile.no-root-user` | WARNING | `USER root` — container runs as root |
| `dockerfile.prefer-copy-over-add` | INFO | `ADD` used where `COPY` is safer |
| `dockerfile.expose-sensitive-port` | WARNING | Exposing sensitive ports (22, 3306, 5432, 6379, 27017, etc.) |
| `dockerfile.curl-pipe-sh` | ERROR | `curl ... \| sh` or `wget ... \| sh` — remote code execution risk |
| `dockerfile.apt-no-version-pin` | INFO | `apt-get install` without pinned package versions |
| `dockerfile.secrets-in-env` | ERROR | `ENV` instruction with secrets or tokens |
| `dockerfile.secrets-in-arg` | WARNING | `ARG` used to pass secrets — visible in image history |

---

## Adding Custom Rules

1. Create a `.yml` file in the relevant `rules/` subdirectory.

2. Minimal rule structure:

```yaml
rules:
  - id: ts.security.your-rule-id
    pattern: dangerous_function(...)
    message: "Short description of the issue and how to fix it."
    languages: [typescript, javascript]
    severity: ERROR  # ERROR | WARNING | INFO
    metadata:
      category: security
      cwe: CWE-XXX
```

3. Test before opening MR:

```bash
semgrep --config rules/typescript/your-new-rule.yml path/to/test/file.ts
```

---

## Suppressing False Positives

Suppress a single line:

```typescript
const result = eval(safeExpression); // nosemgrep: ts.security.no-eval
```

Suppress a block:

```typescript
// nosemgrep: ts.security.no-eval
const result = eval(safeExpression);
```

> Only use `nosemgrep` after confirming this is a genuine false positive. Leave a comment explaining why.

---

## GitLab SAST Report

The `semgrep_scan` job outputs `gl-sast-report.json` in GitLab SAST format. This artifact is uploaded and displayed in the **Security** tab of any Merge Request on GitLab Ultimate/Gold.

---

## Requirements

- GitLab CI/CD runner with Docker executor
- Runner must be able to pull from `registry.docker.com` (Semgrep image)
- Runner uses `CI_JOB_TOKEN` automatically to clone `security/semgrep-cicd`
# interlink_semgrep_scan
