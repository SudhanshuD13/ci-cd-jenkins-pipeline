# 🚀 CI/CD Pipeline with Jenkins + Security Scanning

A production-grade Jenkins pipeline for a Flask application with integrated
security scanning at every stage.

## 🔧 Pipeline Stages

| Stage | Tool | Purpose |
|---|---|---|
| Checkout | Jenkins SCM | Pull latest code |
| Secret Scan | Gitleaks | Detect hardcoded secrets |
| Code Analysis | SonarQube | Code quality & vulnerabilities |
| Docker Build | Docker | Multi-stage optimized image |
| Image Scan | Trivy | CVE scan (blocks HIGH/CRITICAL) |
| Docker Push | Docker Hub | Publish versioned image |
| Deploy | Docker | Run latest container |

## 📦 Stack

- **Jenkins** — Pipeline orchestration
- **SonarQube** — Static code analysis
- **Gitleaks** — Secret detection
- **Trivy** — Container vulnerability scanning
- **Docker** — Build & deployment

## 🚀 Setup

### 1. Start Jenkins
```bash
cd jenkins
docker-compose up -d
```

### 2. Start SonarQube
```bash
cd sonarqube
docker-compose up -d
# Access at http://localhost:9000 (admin/admin)
```

### 3. Configure Jenkins Credentials
Add these in Jenkins → Manage Jenkins → Credentials:
- `dockerhub-creds` — Docker Hub username/password
- `sonar-token` — SonarQube token

### 4. Create Pipeline Job
- New Item → Pipeline
- Pipeline script from SCM → point to this repo
- Branch: main

## 🔐 Security Design

- Secrets scanned **before** build (Gitleaks runs first)
- Image scanned **before** push (Trivy blocks on HIGH/CRITICAL)
- Docker credentials handled via Jenkins credential store — never in code
- Reports archived as build artifacts for audit trail

## 📊 Reports

Both `gitleaks-report.json` and `trivy-report.json` are archived 
as Jenkins build artifacts after every run.
