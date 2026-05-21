# Spinel-AI/interlink_semgrep_scan

Shared Semgrep SAST configuration for CI/CD pipelines across all projects in the organization.

---

## What is this?

[Semgrep](https://semgrep.dev) is a static analysis (SAST) tool that scans source code for security vulnerabilities, malware patterns, hardcoded secrets, and dangerous code — **before the build runs**.

This repo contains:
- A **reusable GitHub Actions workflow** that any project in the org can call with 3 lines of YAML
- Custom rules for TypeScript/JavaScript and Dockerfile
- Telegram alerting for findings and scan errors

---

## Why scan before build?

- **Fail fast**: Catch vulnerabilities at push time, not after deployment
- **No wasted CI time**: Stop the pipeline before build if critical issues are found
- **Immediate developer feedback**: Findings appear directly in the PR/push run

---

## Repo Structure

```
interlink_semgrep_scan/
├── README.md
├── .semgrepignore
├── .github/
│   └── workflows/
│       └── semgrep.yml              # Reusable GitHub Actions workflow
├── templates/
│   └── gitlab-semgrep.yml           # GitLab CI template
└── rules/
    ├── typescript/
    │   ├── nodejs-security.yml      # Core security rules (eval, injection, crypto, SSRF)
    │   ├── malware-detection.yml    # Malware, obfuscation, reverse shell, exfiltration
    │   └── secrets.yml              # Hardcoded secrets and credentials
    └── docker/
        └── dockerfile-security.yml  # Dockerfile security rules
```

---

## Integrating into a GitHub Project

### Step 1 — Add `.github/workflows/ci.yml` to your project

```yaml
name: CI

on:
  push:
  pull_request:

jobs:
  security-scan:
    name: Security Scan (Semgrep)
    permissions:
      contents: read
      security-events: write
    uses: Spinel-AI/interlink_semgrep_scan/.github/workflows/semgrep.yml@main
    with:
      fail-on-findings: false   # set to true when ready to enforce
    secrets: inherit

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: security-scan
    steps:
      - uses: actions/checkout@v4
      - run: npm run build
```

> `permissions: security-events: write` is required on the calling job for SARIF upload to the GitHub Security tab.

### Step 2 — Allow access to this rules repo (private repo only)

If `interlink_semgrep_scan` is private, go to:  
`Settings → Actions → General → Access → Allow access from other repos in the organization`

Or create a PAT and store it as secret `SEMGREP_RULES_TOKEN` in each project.

---

## Workflow Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `fail-on-findings` | boolean | `true` | Fail workflow if Semgrep finds any issues |
| `rules-ref` | string | `main` | Branch or tag of this rules repo to use |

---

## Workflow Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `SEMGREP_RULES_TOKEN` | No | PAT to clone this repo if private (falls back to `GITHUB_TOKEN`) |
| `TELEGRAM_BOT_TOKEN` | No | Telegram bot token for notifications |
| `TELEGRAM_CHAT_ID` | No | Telegram chat or group ID |
| `TELEGRAM_MESSAGE_THREAD_ID` | No | Telegram topic thread ID (for groups with topics) |

---

## Telegram Notifications

When configured, the workflow sends a Telegram message on:
- **Findings detected** (malware, secrets, errors, warnings)
- **Scan error** (Semgrep failed to run)
- **Silent on clean scans** — no message when 0 findings

Message format:
```
🚨 Semgrep SAST FAILED — Security issues detected
⚠️ MALWARE / OBFUSCATED CODE DETECTED: 8 finding(s)
🔑 HARDCODED SECRETS DETECTED: 3 finding(s)

Repository: org/project
Branch: main
Commit: a1b2c3d
Triggered by: dev-username

Findings: 11 total | 🔴 9 errors | 🟡 2 warnings

Top findings:
  ▸ ts.malware.global-require-hijack
    next.config.ts:3
  ▸ ts.secrets.aws-access-key-id
    src/lib/config.ts:12

View GitHub Actions run →
```

---

## Outputs

| Output | Description |
|--------|-------------|
| SARIF artifact | Downloaded from the Actions run (`semgrep-sarif-<sha>`) |
| GitHub Security tab | Available on public repos or private repos with GitHub Advanced Security |

> For private repos on free/Team plans, the Security tab upload will show a warning but CI will not fail (`continue-on-error: true`). The SARIF file is still available as an artifact.

---

## Fail vs Warn Mode

**Fail on findings — recommended for production branches:**

```yaml
jobs:
  security-scan:
    permissions:
      contents: read
      security-events: write
    uses: Spinel-AI/interlink_semgrep_scan/.github/workflows/semgrep.yml@main
    with:
      fail-on-findings: true
    secrets: inherit
```

**Warn only — use when onboarding a new project:**

```yaml
jobs:
  security-scan:
    permissions:
      contents: read
      security-events: write
    uses: Spinel-AI/interlink_semgrep_scan/.github/workflows/semgrep.yml@main
    with:
      fail-on-findings: false
    secrets: inherit
```

---

## Running Locally

**Scan with Semgrep default ruleset only:**

```bash
docker run --rm -v "${PWD}:/src" semgrep/semgrep:1.162.0 semgrep scan --config auto /src
```

**Scan with default + all custom rules from this repo:**

```bash
# Clone rules
git clone git@github.com:Spinel-AI/interlink_semgrep_scan.git /tmp/semgrep-rules

# Run scan
docker run --rm \
  -v "${PWD}:/src" \
  -v "/tmp/semgrep-rules/rules:/rules" \
  semgrep/semgrep:1.162.0 \
  semgrep scan --config auto --config /rules /src
```

**If you are inside this repo:**

```bash
docker run --rm -v "${PWD}:/src" semgrep/semgrep:1.162.0 semgrep scan --config auto --config /src/rules /src
```

---

## Rule Coverage

### TypeScript / Node.js — Core Security (`nodejs-security.yml`)

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `ts.security.no-eval` | ERROR | `eval()` — arbitrary code execution |
| `ts.security.no-new-function` | ERROR | `new Function()` — equivalent to eval |
| `ts.security.vm-execution` | ERROR | `vm.runInNewContext/ThisContext/Context` — arbitrary execution |
| `ts.security.settimeout-string-eval` | WARNING | `setTimeout/setInterval` with string argument |
| `ts.security.sql-injection-template` | ERROR | SQL built with template literals from user input |
| `ts.security.sql-injection-concat` | ERROR | SQL built with string concatenation |
| `ts.security.nosql-injection` | ERROR | MongoDB query with unvalidated request input |
| `ts.security.child-process-exec` | WARNING | `exec()` — shell injection risk |
| `ts.security.path-traversal-params` | ERROR | `fs` operations using request params/query/body as path |
| `ts.security.open-redirect` | ERROR | `res.redirect()` with unvalidated user-supplied URL |
| `ts.security.ssrf-fetch` | ERROR | `fetch()` with URL from user input — SSRF |
| `ts.security.ssrf-axios` | ERROR | `axios.get/post` with URL from user input — SSRF |
| `ts.security.prototype-pollution` | ERROR | Assignment to `__proto__` or `Object.prototype` |
| `ts.security.insecure-deserialize` | ERROR | `node-serialize` / unsafe deserialization |
| `ts.security.jwt-hardcoded-secret` | ERROR | JWT signed with a hardcoded string literal |
| `ts.security.weak-hash` | WARNING | MD5 or SHA1 used for cryptographic hashing |
| `ts.security.weak-cipher` | ERROR | `crypto.createCipher` — deprecated, no IV |
| `ts.security.insecure-random` | WARNING | `Math.random()` in security-sensitive context |
| `ts.security.direct-process-env` | WARNING | `process.env` read directly (use `ConfigService`) |
| `ts.security.no-console-log` | WARNING | `console.log` in production code |

### TypeScript / Node.js — Malware & Obfuscation (`malware-detection.yml`)

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `ts.malware.eval-obfuscated-fromcharcode` | ERROR | `eval(String.fromCharCode(...))` — obfuscated payload |
| `ts.malware.eval-obfuscated-atob` | ERROR | `eval(atob(...))` — base64-encoded execution |
| `ts.malware.eval-obfuscated-buffer-b64` | ERROR | `eval(Buffer.from(..., 'base64').toString())` |
| `ts.malware.base64-decode-exec` | ERROR | Decode base64 to variable then `eval(variable)` |
| `ts.malware.eval-dynamic-concat` | ERROR | `eval(a + b)` — dynamically constructed eval |
| `ts.malware.dynamic-require` | ERROR | `require(variable)` — runtime module loading |
| `ts.malware.env-exfiltration-fetch` | ERROR | `process.env` sent externally via `fetch` |
| `ts.malware.env-exfiltration-axios` | ERROR | `process.env` sent externally via `axios` |
| `ts.malware.env-exfiltration-http` | ERROR | `process.env` written to outbound HTTP request |
| `ts.malware.sensitive-file-access` | ERROR | Read `/etc/passwd`, `/etc/shadow`, `~/.ssh/id_rsa` |
| `ts.malware.suspicious-network-spawn` | ERROR | Spawn `curl`/`wget`/`nc`/`bash -c` — reverse shell pattern |
| `ts.malware.exec-dynamic-input` | ERROR | `exec()` with dynamic/concatenated command string |
| `ts.malware.write-to-cron` | ERROR | Write to `/etc/cron*` or `/var/spool/cron` — persistence |
| `ts.malware.write-to-ssh-authorized-keys` | ERROR | Write to `~/.ssh/authorized_keys` — SSH backdoor |
| `ts.malware.suspicious-hex-string-exec` | WARNING | Hex-escaped string in `eval` or `new Function` |
| `ts.malware.postinstall-network-call` | WARNING | Network call in npm lifecycle scripts |
| `ts.malware.cors-wildcard-with-credentials` | ERROR | CORS `origin: "*"` with `credentials: true` |
| `ts.malware.global-require-hijack` | ERROR | `global[key] = require` — supply-chain obfuscation |
| `ts.malware.global-module-hijack` | ERROR | `global[key] = module` — module system hijack |
| `ts.malware.iife-string-fromcharcode-obfuscation` | ERROR | IIFE + `String.fromCharCode` obfuscation wrapper |
| `ts.malware.global-sentinel-variable` | ERROR | `global.x = "N-NNNN"` — infection marker pattern |
| `ts.malware.obfuscated-array-var` | ERROR | `var _$_XXXX` naming — generated by JS obfuscators |

### TypeScript / Node.js — Hardcoded Secrets (`secrets.yml`)

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `ts.secrets.private-key` | ERROR | RSA/EC/DSA private key embedded in source |
| `ts.secrets.aws-access-key-id` | ERROR | AWS Access Key ID (`AKIA...`) |
| `ts.secrets.aws-secret-key-assignment` | ERROR | AWS Secret Access Key hardcoded |
| `ts.secrets.credentials-in-url` | ERROR | Password in database connection URL |
| `ts.secrets.github-token` | ERROR | GitHub token (`ghp_...`, `ghs_...`) |
| `ts.secrets.gitlab-token` | ERROR | GitLab token (`glpat-...`) |
| `ts.secrets.google-api-key` | ERROR | Google API key (`AIza...`) |
| `ts.secrets.firebase-service-account` | ERROR | Firebase service account JSON |
| `ts.secrets.slack-token` | ERROR | Slack token (`xox[baprs]-...`) |
| `ts.secrets.stripe-secret-key` | ERROR | Stripe secret key (`sk_live_...`) |
| `ts.secrets.sendgrid-api-key` | ERROR | SendGrid API key (`SG....`) |
| `ts.secrets.generic-api-key` | WARNING | `apiKey`/`api_key`/`secret` assigned a string literal |
| `ts.secrets.hardcoded-password` | WARNING | `password`/`passwd` assigned a non-empty string |

### Dockerfile (`dockerfile-security.yml`)

| Rule ID | Severity | Description |
|---------|----------|-------------|
| `dockerfile.no-latest-tag` | WARNING | `FROM image:latest` — non-reproducible builds |
| `dockerfile.no-version-tag` | WARNING | `FROM image` with no tag at all |
| `dockerfile.pin-by-digest` | INFO | Recommend pinning by digest for critical images |
| `dockerfile.secrets-in-env` | ERROR | `ENV` instruction with secrets or tokens |
| `dockerfile.secrets-in-arg` | WARNING | `ARG` used to pass secrets — visible in image history |
| `dockerfile.no-root-user` | WARNING | `USER root` — container runs as root |
| `dockerfile.no-user-directive` | WARNING | No `USER` directive — defaults to root |
| `dockerfile.curl-pipe-sh` | ERROR | `curl \| sh` or `wget \| sh` — remote code execution |
| `dockerfile.apt-no-version-pin` | INFO | `apt-get install` without pinned package versions |
| `dockerfile.prefer-copy-over-add` | INFO | `ADD` used where `COPY` is safer |
| `dockerfile.copy-all-context` | WARNING | `COPY . .` copies entire context including sensitive files |
| `dockerfile.expose-sensitive-port` | WARNING | Exposing ports 22, 3306, 5432, 6379, 27017, etc. |
| `dockerfile.missing-healthcheck` | INFO | No `HEALTHCHECK` directive |

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

> Only use `nosemgrep` after confirming this is a genuine false positive. Add a comment explaining why.

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
    severity: ERROR   # ERROR | WARNING | INFO
    metadata:
      category: security
      cwe: CWE-XXX
```

3. Test locally before opening a PR:

```bash
docker run --rm \
  -v "${PWD}:/src" \
  semgrep/semgrep:1.162.0 \
  semgrep scan --config /src/rules/typescript/your-new-rule.yml /src/path/to/test/file.ts
```

---

## Requirements

- GitHub Actions with access to pull `semgrep/semgrep:1.162.0` from Docker Hub
- `permissions: security-events: write` on the calling job (for SARIF upload)
- For private repos: allow Actions access from other repos in org settings, or set `SEMGREP_RULES_TOKEN`
- Telegram secrets optional — scan runs normally without them
