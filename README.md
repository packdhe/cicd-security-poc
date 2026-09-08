# CICD Security POC

A production-grade CI/CD pipeline with **6 mandatory security gates**.
Deployment is blocked unless every gate passes.

```
Checkout → Gitleaks → Semgrep → SonarQube → Dependency-Check → Build Image → Trivy → OWASP ZAP → Deploy
```

---

## Security Gates

| # | Tool | Type | Blocks deploy on |
|---|------|------|-----------------|
| 1 | **Gitleaks** | Secret scanning | Any leaked credential/key |
| 2 | **Semgrep** | SAST | OWASP Top-10 code issues |
| 3 | **SonarQube** | SAST + Quality Gate | Blocker/Critical issues |
| 4 | **OWASP Dependency-Check** | SCA | CVSSv3 ≥ 7.0 in dependencies |
| 5 | **Trivy** | Container scan | HIGH/CRITICAL CVEs in image |
| 6 | **OWASP ZAP** | DAST | FAIL-level alerts (see zap-baseline.conf) |

---

## Architecture

```
GitHub Actions (cloud runner)     VPS - Ubuntu 22.04 (4 CPU / 8 GB)
├── Gitleaks                       └── docker-compose.yml
├── Semgrep                            ├── SonarQube (port 9000)
├── SonarQube scan ──────────────────► └── PostgreSQL
├── Dependency-Check
├── Build Docker image
├── Trivy
└── OWASP ZAP
```

---

## Prerequisites

- GitHub repository
- VPS Ubuntu 22.04 (4 CPU / 8 GB) — for SonarQube only
- Docker + Docker Compose v2 on VPS
- Port `9000` open on VPS (SonarQube)

---

## Quick Start

### 1. Start SonarQube on your VPS

```bash
# Required by SonarQube
sudo sysctl -w vm.max_map_count=524288
sudo sysctl -w fs.file-max=131072

# Make persistent
echo "vm.max_map_count=524288" | sudo tee -a /etc/sysctl.conf
echo "fs.file-max=131072" | sudo tee -a /etc/sysctl.conf

# Install Docker
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Start SonarQube
docker compose up -d
```

Wait ~2 minutes:
```bash
docker compose logs -f sonarqube   # wait for "SonarQube is operational"
```

### 2. Configure SonarQube

1. Open `http://<vps-ip>:9000` → login `admin / admin` → change password
2. **Administration → Security → Generate Token** → copy the token
3. Create project with key `cicd-security-poc`
4. Ensure **"Sonar way"** Quality Gate is set as default

### 3. Configure GitHub Secrets

Go to your repo → **Settings → Secrets and variables → Actions** → add:

| Secret | Value |
|--------|-------|
| `SONAR_TOKEN` | Token from step 2 |
| `SONAR_HOST_URL` | `http://<vps-ip>:9000` |

### 4. Push & trigger the pipeline

```bash
git add .
git commit -m "feat: initial commit"
git push origin main
```

GitHub Actions triggers automatically on every push to `main`.

---

## Branching Strategy

```
feature/* → PR → dev → PR → uat → PR → main (production)
```

| Branch | Security Gates | Deploy |
|--------|---------------|--------|
| `dev` | Gitleaks + Semgrep | Auto |
| `uat` | All 6 gates | Auto |
| `main` | All 6 gates | Manual approval |

### 1. Create branches

```bash
git checkout -b uat && git push origin uat
git checkout -b dev && git push origin dev
git checkout main
```

### 2. Create GitHub Environments

Go to: **Settings → Environments**

- Click **New environment** → name: `uat` → click **Configure environment** → Save
- Click **New environment** → name: `production` → click **Configure environment**
  - Enable **Required reviewers** → add your GitHub username → Save

> Note: Required reviewers on `production` is only available on public repos or GitHub Team plan.

### 3. Setup Branch Protection Rules

Go to: **Settings → Branches → Add branch ruleset**

**protect-main** (target: `main`)

| Setting | Value |
|---------|-------|
| Require a pull request before merging | ✅ Enable |
| Required approvals | `1` |
| Require status checks to pass | ✅ Enable |
| Required status checks | `Gitleaks`, `Semgrep`, `SonarQube`, `Dependency Check`, `Build + Trivy + ZAP` |
| Block force pushes | ✅ Enable |
| Restrict deletions | ✅ Enable |

**protect-uat** (target: `uat`)

| Setting | Value |
|---------|-------|
| Require a pull request before merging | ✅ Enable |
| Required approvals | `1` |
| Require status checks to pass | ✅ Enable |
| Required status checks | `Gitleaks`, `Semgrep`, `SonarQube`, `Dependency Check`, `Build + Trivy + ZAP` |
| Block force pushes | ✅ Enable |
| Restrict deletions | ✅ Enable |

**protect-dev** (target: `dev`)

| Setting | Value |
|---------|-------|
| Require a pull request before merging | ✅ Enable |
| Required approvals | `1` |
| Require status checks to pass | ✅ Enable |
| Required status checks | `Gitleaks`, `Semgrep` |
| Block force pushes | ✅ Enable |
| Restrict deletions | ✅ Enable |

### 4. Normal development flow

```bash
# Create feature branch from dev
git checkout dev
git checkout -b feature/my-feature

# Make changes, commit, push
git add .
git commit -m "feat: my feature"
git push origin feature/my-feature

# Open PR: feature/* → dev (triggers Gitleaks + Semgrep)
# Open PR: dev → uat (triggers all 6 gates)
# Open PR: uat → main (triggers all 6 gates + approval)
```

---

## Customising Security Thresholds

| File | What to change |
|------|---------------|
| `.gitleaks.toml` | Add/remove secret patterns |
| `.semgrep.yml` | Add custom SAST rules or registry packs |
| `sonarqube/sonar-project.properties` | Exclusions, language settings |
| `.github/workflows/security-pipeline.yml` `--failOnCVSS` | Raise/lower CVE severity threshold |
| `.github/workflows/security-pipeline.yml` `severity` (Trivy) | Change from HIGH,CRITICAL to MEDIUM |
| `owasp-zap/zap-baseline.conf` | Change WARN → FAIL for specific ZAP rules |

---

## Directory Structure

```
cicd-security-poc/
├── .github/
│   └── workflows/
│       └── security-pipeline.yml   # Full pipeline definition
├── owasp-zap/
│   └── zap-baseline.conf
├── sample-app/                     # Demo Flask app
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
├── sonarqube/
│   └── sonar-project.properties
├── docker-compose.yml              # SonarQube + Postgres only
├── .gitleaks.toml
└── .semgrep.yml
```

---

## Troubleshooting

**SonarQube won't start**
```bash
sudo sysctl -w vm.max_map_count=524288
docker compose restart sonarqube
```

**GitHub Actions can't reach SonarQube**
Ensure port `9000` is open on your VPS firewall:
```bash
sudo ufw allow 9000
```

**Dependency-Check is slow on first run**
It downloads the NVD database (~500 MB). Subsequent runs use GitHub Actions cache.

**ZAP can't reach the app**
ZAP runs on the same GitHub Actions runner as the app container, so it uses `localhost:5000`. Verify the app started correctly in the logs.
