Run a full security scan on the current project. Arguments: $ARGUMENTS

## Step 1 — Parse arguments and resolve the target directory

Parse `$ARGUMENTS` for the following optional flags:

- `--dir <path>` — scan this subdirectory instead of the project root (relative to the resolved target)
- `--severity CRITICAL` — filter the final report to CRITICAL findings only (default: HIGH and CRITICAL)
- `--fix` — after scanning, attempt auto-fixes where available (npm audit fix, Trivy patch suggestions)

After extracting flags, the remaining text (if any) is treated as a project name argument.

**Resolve the target:**
- If a project name was provided (e.g., `/security-scan my-project`), resolve the target as `$HOME/projects/<project-name>` or the nearest parent directory containing a project with that name
- If `--dir` was provided, append that path to the resolved target
- If no project name and no `--dir`, the target is the current working directory (`$PWD`)

Print:
```
Target directory : <resolved path>
Severity filter  : HIGH,CRITICAL  (or CRITICAL if --severity CRITICAL was passed)
Auto-fix mode    : on / off
```

## Step 2 — Verify the target exists

If the target directory does not exist, print an error and stop:
```
ERROR: Directory not found: <path>
Usage: /security-scan [project-name]
```

## Step 3 — Check prerequisites

For each tool, run the version check command. Mark it as AVAILABLE or MISSING:

| Tool | Check command |
|------|---------------|
| gitleaks | `gitleaks version` |
| npm | `npm --version` |
| trivy | `trivy --version` |
| semgrep | `semgrep --version` |
| tfsec | `tfsec --version` |
| checkov | `checkov --version` |
| bandit | `bandit --version` |
| gosec | `gosec --version` |
| shellcheck | `shellcheck --version` |
| cargo | `cargo --version` |
| cppcheck | `cppcheck --version` |

Print a prerequisite summary table. Missing tools will be skipped (not a fatal error).
At the end, print install instructions for any missing tools.

## Step 4 — Detect the stack

Inspect the target directory and set flags for what is present:

- `HAS_NPM` — `package.json` exists anywhere under the target (search recursively, max depth 3)
- `HAS_TERRAFORM` — any `*.tf` file exists under the target
- `HAS_OPENTOFU` — any `*.tf` file exists under the target (same HCL syntax as Terraform)
- `HAS_PULUMI` — `Pulumi.yaml` exists under the target
- `HAS_CDK` — `cdk.json` exists under the target
- `HAS_CFN` — any `*.yaml` or `*.json` file contains `AWSTemplateFormatVersion` or `Resources:` with CloudFormation resource types
- `HAS_BICEP` — any `*.bicep` file exists under the target
- `HAS_ARM` — any `*.json` file contains `"$schema"` referencing `deploymentTemplate` or `subscriptionDeploymentTemplate`
- `HAS_CROSSPLANE` — any `*.yaml` file contains `apiVersion` with `crossplane.io` or `upbound.io`
- `HAS_ANSIBLE` — `roles/` directory exists OR any `*.yml` file contains both `hosts:` and `tasks:`
- `HAS_CHEF` — `Berksfile` exists OR `cookbooks/` directory exists
- `HAS_PUPPET` — any `*.pp` file exists OR `manifests/` directory exists
- `HAS_SALTSTACK` — any `*.sls` file exists OR `salt/` directory exists
- `HAS_DOCKER` — any `Dockerfile` exists under the target (excluding node_modules)
- `HAS_GO` — `go.mod` exists OR any `*.go` file exists under the target (excluding node_modules)
- `HAS_PYTHON` — `requirements.txt`, `pyproject.toml`, or any `*.py` file exists (excluding node_modules)
- `HAS_JAVA` — any `*.java` file exists OR `pom.xml` OR `build.gradle` exists
- `HAS_CPP` — any `*.cpp`, `*.cc`, or `*.cxx` file exists
- `HAS_RUST` — `Cargo.toml` exists under the target
- `HAS_TYPESCRIPT` — `tsconfig.json` exists OR any `*.ts` file exists (excluding node_modules)
- `HAS_BASH` — any `*.sh` file exists under the target
- `HAS_K8S` — a `k8s/` directory exists OR any `*.yaml` file contains `kind:` at the root level
- `HAS_GIT` — `.git/` directory exists under the target (always true for initialized repos)

Print the detected stack flags.

## Step 5 — Run the scans

Run each applicable scan below. Capture stdout + stderr and the exit code for each tool.
A non-zero exit code means FAIL for that tool. Capture all output — do not stop on first failure.

---

### gitleaks — Secrets scan (runs if HAS_GIT or always)

```bash
cd <target> && gitleaks detect --source . --verbose --redact --no-git 2>&1
cd <target> && gitleaks detect --source . --verbose --redact 2>&1
```

Status: PASS if exit code 0, FAIL if any secrets found.

---

### npm audit — Dependency CVEs (runs if HAS_NPM and npm is AVAILABLE)

Find all `package.json` files (excluding `node_modules`). For each one, run:

```bash
cd <package.json directory> && npm audit --audit-level=high --omit=dev --json 2>&1
```

Parse the JSON output. Status: PASS if no HIGH or CRITICAL vulns, FAIL otherwise.
Report: package name, vulnerability, severity, affected version, fix version.

---

### tfsec — Terraform IaC misconfigurations (runs if HAS_TERRAFORM and tfsec is AVAILABLE)

```bash
cd <target> && tfsec . --minimum-severity HIGH --format json 2>&1
```

Status: PASS if no HIGH/CRITICAL findings not in suppression list, FAIL otherwise.
Report: rule ID, severity, description, file, line number.

---

### Trivy config — IaC/filesystem misconfigs (runs if HAS_TERRAFORM or HAS_K8S, and trivy is AVAILABLE)

```bash
trivy config <target> --severity HIGH,CRITICAL --exit-code 1 2>&1
```

Status: PASS if exit code 0, FAIL otherwise.
Report: resource, misconfiguration ID, severity, description.

---

### Trivy image — Container image scan (runs if HAS_DOCKER and trivy is AVAILABLE)

For each Dockerfile found:
1. Determine a tag: `security-scan-<dirname>:latest`
2. Build the image: `docker build -t <tag> -f <Dockerfile path> <context dir>`
3. Scan: `trivy image <tag> --severity HIGH,CRITICAL --ignore-unfixed --exit-code 1 2>&1`
4. Remove the image after scan: `docker rmi <tag>`

Status: PASS if no HIGH/CRITICAL unfixed CVEs, FAIL otherwise.

---

### Semgrep — SAST code patterns (runs if HAS_NPM or HAS_GO or HAS_PYTHON or HAS_JAVA or HAS_CPP or HAS_RUST or HAS_TYPESCRIPT, and semgrep is AVAILABLE)

```bash
cd <target> && semgrep scan --config=auto --error --json . 2>&1
```

Status: PASS if no ERROR-level findings, FAIL otherwise.
Report: rule ID, severity, file, line, message.

---

### Checkov — Multi-framework IaC scan (runs if checkov is AVAILABLE and any IaC stack is detected)

Runs when any of these flags are set: HAS_TERRAFORM, HAS_OPENTOFU, HAS_PULUMI, HAS_CDK, HAS_CFN, HAS_BICEP, HAS_ARM, HAS_CROSSPLANE, HAS_ANSIBLE, HAS_CHEF, HAS_PUPPET, HAS_SALTSTACK, HAS_K8S, HAS_DOCKER.

Checkov covers IaC tools that Trivy and tfsec do not fully support:

| Framework flag | Checkov framework arg |
|----------------|-----------------------|
| HAS_TERRAFORM / HAS_OPENTOFU | `terraform` |
| HAS_PULUMI | `pulumi` |
| HAS_CDK | `cloudformation` (CDK synth output) |
| HAS_CFN | `cloudformation` |
| HAS_BICEP | `bicep` |
| HAS_ARM | `arm` |
| HAS_CROSSPLANE | `kubernetes` |
| HAS_K8S | `kubernetes` |
| HAS_DOCKER | `dockerfile` |
| HAS_ANSIBLE | `ansible` |
| HAS_CHEF | `secrets` (limited Chef support) |
| HAS_PUPPET | `secrets` (limited Puppet support) |
| HAS_SALTSTACK | `secrets` (limited SaltStack support) |

Build the `--framework` list from detected flags, then run:

```bash
checkov -d <target> --framework <comma-separated-list> --quiet --compact \
  --skip-check CKV_AWS_18,CKV_AWS_86 \
  --severity HIGH 2>&1
```

Notes:
- For AWS CDK projects, run `cdk synth` first to generate CloudFormation templates, then point checkov at the `cdk.out/` directory.
- `--quiet --compact` reduces noise; full output is still captured.
- `--severity HIGH` filters to HIGH and CRITICAL only (consistent with other tools).

Status: PASS if exit code 0, FAIL otherwise.
Report: check ID, severity, resource, file, line, description.

---

### Bandit — Python SAST (runs if HAS_PYTHON and bandit is AVAILABLE)

```bash
bandit -r <target> -ll -ii --format json 2>&1
```

Flags: `-ll` = LOW+ severity, `-ii` = MEDIUM+ confidence. Catches SQL injection, shell injection, hardcoded passwords, insecure use of subprocess, pickle, eval, etc.

Status: PASS if no MEDIUM/HIGH severity + MEDIUM/HIGH confidence findings, FAIL otherwise.
Report: test ID, severity, confidence, file, line, issue text.

---

### gosec — Go SAST (runs if HAS_GO and gosec is AVAILABLE)

```bash
gosec -fmt json -severity medium -confidence medium ./... 2>&1
```

Runs from the directory containing `go.mod`. Catches SQL injection, command injection, insecure crypto, hardcoded credentials, race conditions, etc.

Status: PASS if no MEDIUM or HIGH findings, FAIL otherwise.
Report: rule ID, severity, confidence, file, line, description.

---

### ShellCheck — Bash/shell script analysis (runs if HAS_BASH and shellcheck is AVAILABLE)

Find all `*.sh` files (excluding node_modules), then run:

```bash
shellcheck --severity=warning --format=json <file> 2>&1
```

Catches command injection via unquoted variables, use of `eval`, dangerous `rm -rf` patterns, missing `set -e` / `set -u`, etc.

Status: PASS if no WARNING or ERROR level findings, FAIL otherwise.
Report: file, line, severity, code, message.

---

### cargo audit — Rust dependency CVEs (runs if HAS_RUST and cargo is AVAILABLE)

```bash
cd <Cargo.toml directory> && cargo audit --json 2>&1
```

Queries the RustSec advisory database for known CVEs in Rust dependencies.

Status: PASS if no vulnerabilities found, FAIL otherwise.
Report: advisory ID, package, version, severity, title, URL.

---

### cppcheck — C/C++ static analysis (runs if HAS_CPP and cppcheck is AVAILABLE)

```bash
cppcheck --enable=warning,style,performance,portability --error-exitcode=1 \
  --suppress=missingIncludeSystem --xml <target> 2>&1
```

Catches buffer overflows, null pointer dereferences, use-after-free, integer overflows, uninitialized variables, etc.

Status: PASS if no ERROR or WARNING findings, FAIL otherwise.
Report: file, line, severity, id, message.

---

## Step 5b — Auto-fix (only if --fix was passed)

If `--fix` was provided, attempt the following fixes **before** generating the report:

**npm audit fix** (if HAS_NPM and npm audit returned findings):
```bash
cd <package.json directory> && npm audit fix 2>&1
```
Re-run `npm audit --audit-level=high --omit=dev` after fixing and update the findings count.

**Trivy patch suggestions** (if Trivy image scan returned findings):
For each vulnerable package found, print the exact upgrade command:
```
Suggested fix: upgrade <package> from <current-version> to <fixed-version>
```
Trivy image fixes cannot be applied automatically — print the suggestions and note that the Dockerfile or base image must be updated manually.

**Semgrep / tfsec / Checkov / Bandit / gosec / ShellCheck / cppcheck / cargo audit:**
These tools do not have auto-fix modes. For each finding, print a remediation hint:
- The exact file and line number
- A one-sentence description of the fix
Note: "Manual fix required — see Remediation Steps section of the report."

---

## Step 6 — Generate the report

Compose the full report as a Markdown string with this structure.
If `--severity CRITICAL` was passed, omit HIGH findings from the Findings Detail and Remediation Steps sections (still count them in the summary table but mark them as filtered).

```
# Security Scan Report

**Project:** <directory name>
**Path:** <full path>
**Date:** <YYYY-MM-DD HH:MM>
**Status:** PASS | FAIL | PARTIAL

---

## Summary

| Tool            | Status  | Findings | Highest Severity |
|-----------------|---------|----------|------------------|
| gitleaks        | PASS/FAIL/SKIPPED | N | — / HIGH / CRITICAL |
| npm audit       | ...     | ...      | ...              |
| tfsec           | ...     | ...      | ...              |
| Trivy (config)  | ...     | ...      | ...              |
| Trivy (image)   | ...     | ...      | ...              |
| Checkov         | ...     | ...      | ...              |
| Semgrep         | ...     | ...      | ...              |
| Bandit          | ...     | ...      | ...              |
| gosec           | ...     | ...      | ...              |
| ShellCheck      | ...     | ...      | ...              |
| cargo audit     | ...     | ...      | ...              |
| cppcheck        | ...     | ...      | ...              |

Overall status rules:
- PASS: all run tools passed
- FAIL: at least one run tool failed
- PARTIAL: one or more tools were skipped due to missing install

---

## Findings Detail

(One subsection per tool that has findings. Skip tools with zero findings.)

### <Tool Name>
<findings in a table or code block>

---

## Remediation Steps

Numbered list, highest severity first. Each item includes:
- What to fix
- Exact command or file change to apply the fix
- Which tool reported it

---

## Skipped Checks

List each tool that was skipped and why:
- Tool not installed: include brew/pip install command
- Stack not detected: explain what signal was missing

---

## Next Steps

- If Status is PASS: "Safe to push. Run /security-scan again after any dependency updates."
- If Status is FAIL: "Fix the findings above before running git push. Re-run /security-scan to confirm."
- If Status is PARTIAL: "Install missing tools for a complete scan, then re-run /security-scan."
```

## Step 7 — Save the report

Determine the report filename:
- Primary: `security-scan-report-<YYYY-MM-DD>.md`
- If that file already exists in the target directory: `security-scan-report-<YYYY-MM-DD>-<HHMM>.md`

Save the report to the **target project root** using the Write tool.

Print: `Report saved: <target>/<filename>`

## Step 8 — Print install instructions for missing tools

If any tools were MISSING, print at the end:

```
---
Install missing tools (macOS):

  brew install gitleaks
  brew install trivy
  brew install tfsec
  brew install semgrep       # or: pip install semgrep
  brew install node@22       # provides npm
  brew install gosec
  brew install shellcheck
  brew install cppcheck
  pip install checkov        # or: brew install checkov
  pip install bandit
  # cargo audit — install via rustup (ships with cargo):
  cargo install cargo-audit
```
