# Building a Security Code Scan Skill with Claude Code

A Claude Code skill that automatically detects your project stack and runs **11 specialized security tools** — covering secrets, dependency CVEs, container images, IaC misconfigurations, and SAST across 8 languages and 7 IaC frameworks.

One command. Any project. Zero configuration.

---

## What It Does

```
/security-scan
```

Claude detects your stack, runs every applicable tool, and produces a structured Markdown report with findings sorted by severity and actionable remediation steps.

```
Status: PASS   — safe to push
Status: FAIL   — fix findings before pushing
Status: PARTIAL — some tools missing; results incomplete
```

---

## Tools Covered

| # | Tool | Threat Category | Runs When |
|---|------|-----------------|-----------|
| 1 | **gitleaks** | Secrets & credentials | Always |
| 2 | **npm audit** | Node.js dependency CVEs | `package.json` found |
| 3 | **tfsec** | Terraform misconfigurations | `*.tf` files found |
| 4 | **Trivy (config)** | IaC / K8s misconfigs | Terraform or K8s detected |
| 5 | **Trivy (image)** | Container image CVEs | `Dockerfile` found |
| 6 | **Checkov** | Multi-framework IaC | CDK, CFN, Pulumi, Bicep, ARM, Ansible… |
| 7 | **Semgrep** | Polyglot SAST | JS, TS, Python, Go, Java, C/C++, Rust |
| 8 | **Bandit** | Python SAST | `*.py` files found |
| 9 | **gosec** | Go SAST | `go.mod` or `*.go` found |
| 10 | **ShellCheck** | Bash/shell scripts | `*.sh` files found |
| 11 | **cargo audit** | Rust dependency CVEs | `Cargo.toml` found |
| 12 | **cppcheck** | C/C++ static analysis | `*.cpp` / `*.cc` / `*.cxx` found |

All tools are **free and open-source**.

---

## Stack Detection

The skill inspects the target directory for 20+ signals and only runs tools relevant to what it finds. A missing tool is skipped gracefully — it does not fail the scan.

Supported stacks: Node.js, Terraform, OpenTofu, AWS CDK, CloudFormation, Pulumi, Azure Bicep, ARM, Crossplane, Ansible, Chef, Puppet, SaltStack, Docker, Kubernetes, Go, Python, Java, Rust, TypeScript, C/C++, Bash/Shell.

---

## Installation

### Step 1 — Install the skill

**macOS / Linux:**
```bash
mkdir -p ~/.claude/commands

curl -sL https://raw.githubusercontent.com/jbaez22/security-scan/main/commands/security-scan.md \
  -o ~/.claude/commands/security-scan.md
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\commands" | Out-Null

Invoke-WebRequest -Uri https://raw.githubusercontent.com/jbaez22/security-scan/main/commands/security-scan.md `
  -OutFile "$env:USERPROFILE\.claude\commands\security-scan.md"
```

That is the entire install. The skill is now available globally in every project you open in Claude Code.

### Step 2 — Install the tools

Install whichever tools apply to your projects. Missing tools are skipped automatically — you do not need all of them for the skill to run.

<details>
<summary><strong>macOS (Homebrew)</strong></summary>

```bash
# Homebrew — https://brew.sh
brew install gitleaks       # secrets — always recommended
brew install trivy          # containers + IaC
brew install tfsec          # Terraform
brew install semgrep        # polyglot SAST
brew install gosec          # Go SAST
brew install shellcheck     # shell scripts
brew install cppcheck       # C/C++

# Python (pip)
pip install bandit          # Python SAST
pip install checkov         # multi-framework IaC (or: brew install checkov)

# Rust (cargo — install via https://rustup.rs)
cargo install cargo-audit   # Rust dependency CVEs
```

</details>

<details>
<summary><strong>Linux (Debian/Ubuntu)</strong></summary>

```bash
# gitleaks
curl -sSL https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_linux_x64.tar.gz \
  | tar -xz && sudo mv gitleaks /usr/local/bin/

# Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sudo sh -s -- -b /usr/local/bin

# tfsec
curl -sSL https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-linux-amd64 \
  -o /usr/local/bin/tfsec && chmod +x /usr/local/bin/tfsec

# Semgrep
pip install semgrep

# Checkov
pip install checkov

# Bandit
pip install bandit

# gosec
curl -sSL https://raw.githubusercontent.com/securego/gosec/master/install.sh \
  | sudo sh -s -- -b /usr/local/bin

# ShellCheck
sudo apt-get install -y shellcheck

# cppcheck
sudo apt-get install -y cppcheck

# cargo audit (requires Rust — https://rustup.rs)
cargo install cargo-audit
```

> For RPM-based distros (RHEL, Fedora, Amazon Linux), replace `apt-get` with `dnf` or `yum` and use the equivalent package names.

</details>

<details>
<summary><strong>Windows (PowerShell / winget)</strong></summary>

> **Prerequisite:** Install [Git for Windows](https://git-scm.com/download/win) and [Python](https://python.org/downloads) before running the commands below. Claude Code itself requires WSL2 or Git Bash on Windows.

```powershell
# winget (Windows Package Manager — built into Windows 11)
winget install gitleaks         # secrets

# Trivy
winget install AquaSecurity.Trivy

# tfsec — download the Windows binary directly
Invoke-WebRequest -Uri https://github.com/aquasecurity/tfsec/releases/latest/download/tfsec-windows-amd64.exe `
  -OutFile "$env:LOCALAPPDATA\Microsoft\WindowsApps\tfsec.exe"

# Semgrep, Checkov, Bandit (pip — requires Python)
pip install semgrep checkov bandit

# gosec — download the Windows binary
Invoke-WebRequest -Uri https://github.com/securego/gosec/releases/latest/download/gosec_windows_amd64.zip `
  -OutFile gosec.zip
Expand-Archive gosec.zip -DestinationPath "$env:LOCALAPPDATA\Microsoft\WindowsApps\"
Remove-Item gosec.zip

# ShellCheck — via Scoop (https://scoop.sh) or winget
scoop install shellcheck
# or: winget install koalaman.shellcheck

# cppcheck — via Scoop or winget
scoop install cppcheck
# or: winget install Cppcheck.Cppcheck

# cargo audit (requires Rust — https://rustup.rs)
cargo install cargo-audit
```

> **Tip:** [Scoop](https://scoop.sh) (`iwr -useb get.scoop.sh | iex`) simplifies CLI tool management on Windows and is the recommended package manager for developer tools.

</details>

---

## Usage

### Basic scan (current directory)

```bash
cd /path/to/your-project
/security-scan
```

### Scan a named project

```bash
/security-scan my-project-name
```

### Scan a subdirectory only

```bash
/security-scan --dir ./website
```

### Report only CRITICAL findings

```bash
/security-scan --severity CRITICAL
```

### Scan and attempt auto-fixes

```bash
/security-scan --fix
```

---

## Report Output

Every run saves a `security-scan-report-YYYY-MM-DD.md` file to the project root. The report includes:

- **Summary table** — PASS / FAIL / SKIPPED per tool with finding counts
- **Findings detail** — file, line, rule ID, severity, description for every finding
- **Remediation steps** — numbered action list, highest severity first
- **Skipped checks** — tools not run and why (not installed, stack not detected)
- **Next steps** — push-readiness verdict

---

## Updating

Re-run the install command for your platform to pull the latest version:

**macOS / Linux:**
```bash
curl -sL https://raw.githubusercontent.com/jbaez22/security-scan/main/commands/security-scan.md \
  -o ~/.claude/commands/security-scan.md
```

**Windows (PowerShell):**
```powershell
Invoke-WebRequest -Uri https://raw.githubusercontent.com/jbaez22/security-scan/main/commands/security-scan.md `
  -OutFile "$env:USERPROFILE\.claude\commands\security-scan.md"
```

---

## Related

- Blog article: [Building a /security-scan Skill with Claude Code](https://www.craftingnewtech.com/blog/post-18)
- [Claude Code documentation](https://docs.anthropic.com/claude-code)

---

## License

MIT — free to use, modify, and distribute.
