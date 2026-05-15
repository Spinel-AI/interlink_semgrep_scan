# Prompt cho Claude: Tạo repo Semgrep CI/CD dùng chung và tích hợp vào các project

## Bối cảnh

Tôi muốn tạo một repo riêng để quản lý cấu hình Semgrep dùng chung cho nhiều project. Sau đó, trong CI/CD của từng project, tôi muốn thêm bước scan code bằng Semgrep trước bước build.

Mục tiêu chính:

- Scan code sớm trong pipeline, trước khi build/package/deploy.
- Dùng chung một bộ template CI/CD và custom rules Semgrep cho nhiều project.
- Ưu tiên GitLab CI/CD.
- Có thể dùng cho các project Node.js / TypeScript / NestJS / Next.js / Docker.
- Nếu phát hiện lỗi nghiêm trọng thì pipeline phải fail sớm.
- Có report dạng GitLab SAST artifact nếu dùng GitLab.

---

## Yêu cầu Claude cần tạo

Hãy tạo đầy đủ nội dung cho một repo tên:

```txt
security/semgrep-cicd
```

Repo này cần có cấu trúc như sau:

```txt
security/semgrep-cicd
├── README.md
├── .semgrepignore
├── templates
│   └── gitlab-semgrep.yml
└── rules
    ├── typescript
    │   └── nodejs-security.yml
    └── docker
        └── dockerfile-security.yml
```

---

## 1. File `README.md`

README cần giải thích rõ:

- Repo này dùng để làm gì.
- Semgrep là gì trong CI/CD.
- Vì sao cần chạy Semgrep trước build.
- Cách include template này vào từng project GitLab.
- Cách chạy local.
- Cách thêm custom rules.
- Cách xử lý false positive.
- Cách bật/tắt fail pipeline.
- Có ví dụ tích hợp vào `.gitlab-ci.yml`.

README nên viết bằng tiếng Việt, rõ ràng, thực tế, dành cho dev đọc và làm theo được ngay.

---

## 2. File `.semgrepignore`

Tạo `.semgrepignore` để bỏ qua các thư mục không nên scan:

```gitignore
node_modules/
dist/
build/
coverage/
.next/
.turbo/
generated/
prisma/migrations/
**/*.min.js
**/*.map
```

Có thể bổ sung thêm nếu thấy cần.

---

## 3. File `templates/gitlab-semgrep.yml`

Tạo GitLab CI template có job `semgrep_scan`.

Yêu cầu:

- Job nằm ở stage `scan`.
- Dùng image `semgrep/semgrep:latest`.
- Chạy trước `build`.
- Scan bằng cả rule mặc định và custom rules trong repo `security/semgrep-cicd`.
- Có thể clone repo rules về trong job trước khi scan.
- Exclude các thư mục như `node_modules`, `dist`, `build`, `.next`, `coverage`.
- Xuất report `gl-sast-report.json`.
- Upload artifact dạng GitLab SAST report.
- Chạy khi:
  - Có Merge Request.
  - Hoặc push vào default branch.
- Có biến để cấu hình:
  - `SEMGREP_RULES_REPO`
  - `SEMGREP_RULES_REF`
  - `SEMGREP_FAIL_ON_FINDINGS`
- Nếu `SEMGREP_FAIL_ON_FINDINGS=true` thì pipeline fail khi có finding.
- Nếu `SEMGREP_FAIL_ON_FINDINGS=false` thì chỉ warning, không fail pipeline.

Ví dụ logic script mong muốn:

```bash
if [ "$SEMGREP_FAIL_ON_FINDINGS" = "true" ]; then
  semgrep scan --error ...
else
  semgrep scan ...
fi
```

Lưu ý: Nếu dùng GitLab private repo để clone rules thì có thể dùng `CI_JOB_TOKEN`.

---

## 4. File `rules/typescript/nodejs-security.yml`

Tạo một bộ Semgrep rules cơ bản cho Node.js / TypeScript.

Nên có các rule sau:

### Rule 1: Không dùng `eval`

- ID: `typescript.no-eval`
- Severity: `ERROR`
- Detect: `eval(...)`

### Rule 2: Không dùng `new Function`

- ID: `typescript.no-new-function`
- Severity: `ERROR`
- Detect: `new Function(...)`

### Rule 3: Không hardcode JWT secret

- ID: `typescript.jwt-hardcoded-secret`
- Severity: `ERROR`
- Detect các pattern như:

```ts
jwt.sign(payload, "secret")
jwt.sign(payload, 'secret')
JWT.sign(payload, "secret")
JWT.sign(payload, 'secret')
```

### Rule 4: Cảnh báo dùng `child_process.exec`

- ID: `typescript.child-process-exec`
- Severity: `WARNING`
- Detect:

```ts
exec(...)
child_process.exec(...)
```

### Rule 5: Cảnh báo dùng `process.env` trực tiếp trong business logic

- ID: `typescript.direct-process-env`
- Severity: `WARNING`
- Mục tiêu: Trong NestJS nên ưu tiên dùng `ConfigService`, tránh đọc `process.env` rải rác.
- Rule này có thể warning thôi, không fail nghiêm trọng.

### Rule 6: Cảnh báo dùng `console.log` trong production source

- ID: `typescript.no-console-log`
- Severity: `INFO` hoặc `WARNING`
- Detect `console.log(...)`
- Có thể exclude test files nếu cần.

---

## 5. File `rules/docker/dockerfile-security.yml`

Tạo một bộ Semgrep rules cơ bản cho Dockerfile.

Nên có các rule sau:

### Rule 1: Không dùng image tag `latest`

- ID: `dockerfile.no-latest-tag`
- Severity: `WARNING`
- Detect:

```Dockerfile
FROM node:latest
FROM nginx:latest
```

### Rule 2: Không chạy container bằng root

- ID: `dockerfile.no-root-user`
- Severity: `WARNING`
- Detect:

```Dockerfile
USER root
```

### Rule 3: Cảnh báo nếu dùng `ADD` thay vì `COPY`

- ID: `dockerfile.prefer-copy-over-add`
- Severity: `INFO`
- Detect:

```Dockerfile
ADD ...
```

### Rule 4: Cảnh báo nếu expose port quá rộng hoặc không cần thiết

- ID: `dockerfile.expose-sensitive-port`
- Severity: `WARNING`
- Có thể detect các port như `22`, `3306`, `5432`, `6379`, `27017`.

---

## 6. Ví dụ tích hợp vào project GitLab

Claude cần tạo đoạn hướng dẫn để copy vào `.gitlab-ci.yml` của từng project.

Ví dụ project hiện tại:

```yaml
stages:
  - validate
  - build
  - deploy
```

Cần sửa thành:

```yaml
include:
  - project: 'security/semgrep-cicd'
    ref: main
    file: '/templates/gitlab-semgrep.yml'

stages:
  - scan
  - validate
  - build
  - deploy

variables:
  SEMGREP_RULES_REPO: "security/semgrep-cicd"
  SEMGREP_RULES_REF: "main"
  SEMGREP_FAIL_ON_FINDINGS: "true"
```

Nếu build job cần phụ thuộc scan:

```yaml
build:
  stage: build
  needs:
    - job: semgrep_scan
      artifacts: false
  script:
    - npm run build
```

---

## 7. Ví dụ chạy local

Claude cần thêm lệnh chạy local trong README:

```bash
docker run --rm -v "${PWD}:/src" semgrep/semgrep semgrep scan --config auto /src
```

Nếu muốn chạy với custom rules:

```bash
semgrep scan --config auto --config rules .
```

---

## 8. Acceptance Criteria

Kết quả cuối cùng phải đáp ứng:

- Có đầy đủ repo structure.
- Có đầy đủ nội dung từng file.
- GitLab CI template có thể include vào project khác.
- Job `semgrep_scan` chạy trước build.
- Có support GitLab SAST report artifact.
- Có custom rules TypeScript và Dockerfile cơ bản.
- Có biến `SEMGREP_FAIL_ON_FINDINGS` để bật/tắt fail pipeline.
- README đủ rõ để dev khác tự tích hợp.
- Không hardcode thông tin cá nhân hoặc token.
- Không đưa secret vào repo.

---

## 9. Output mong muốn từ Claude

Hãy output đầy đủ theo format:

```txt
# File: README.md
<content>

# File: .semgrepignore
<content>

# File: templates/gitlab-semgrep.yml
<content>

# File: rules/typescript/nodejs-security.yml
<content>

# File: rules/docker/dockerfile-security.yml
<content>
```

Code phải sạch, copy vào repo chạy được ngay.
