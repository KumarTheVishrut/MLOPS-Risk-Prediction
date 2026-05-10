# CareerShield MLOps — Jenkins CI/CD Guide

> **Project:** Layoff Risk Prediction API  
> **Stack:** TensorFlow + FastAPI (backend) | React + Vite (frontend) | Docker + Jenkins (CI/CD)  
> **Author:** Vishrut  
> **Date:** May 2026

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Jenkins Installation](#2-jenkins-installation)
3. [Plugin Setup](#3-plugin-setup)
4. [Docker Configuration](#4-docker-configuration)
5. [Credential Setup](#5-credential-setup)
6. [Pipeline Job Configuration](#6-pipeline-job-configuration)
7. [Jenkinsfile Walkthrough](#7-jenkinsfile-walkthrough)
8. [Troubleshooting](#8-troubleshooting)
9. [Maintenance](#9-maintenance)

---

## 1. Prerequisites

### System Requirements

| Component | Minimum | Recommended |
|-----------|---------|-------------|
| CPU | 4 cores | 8+ cores (model training) |
| RAM | 8 GB | 16+ GB |
| Disk | 50 GB | 100+ GB (Docker images) |
| OS | Ubuntu 20.04/22.04 | Ubuntu 22.04 LTS |
| Docker | 24.0+ | Latest stable |
| Jenkins | 2.426+ | Latest LTS |

### Required Accounts

- **Docker Hub:** [hub.docker.com](https://hub.docker.com) — for pushing images
- **GitHub/GitLab:** Repository access for webhook triggers

### Network Requirements

- Jenkins server must reach Docker Hub (`registry-1.docker.io`)
- Jenkins server must reach your Git provider for SCM polling
- Port `8080` open for Jenkins UI
- Port `50000` open for Jenkins agent communication (if using agents)

---

## 2. Jenkins Installation

### Option A: Native Installation (Recommended)

```bash
# Install Java (Jenkins dependency)
sudo apt update
sudo apt install -y openjdk-17-jre-headless

# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee   /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]   https://pkg.jenkins.io/debian-stable binary/ | sudo tee   /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install -y jenkins

# Start and enable Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check status
sudo systemctl status jenkins
```

### Option B: Docker Installation (Alternative)

```bash
# Run Jenkins in Docker (with Docker socket access for building images)
docker run -d   --name jenkins   --restart unless-stopped   -p 8080:8080   -p 50000:50000   -v jenkins_home:/var/jenkins_home   -v /var/run/docker.sock:/var/run/docker.sock   -v $(which docker):/usr/bin/docker   jenkins/jenkins:lts
```

### Initial Setup

1. Open `http://<your-server-ip>:8080`
2. Retrieve initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```
3. Install **suggested plugins**
4. Create your admin user

---

## 3. Plugin Setup

### Required Plugins

Navigate to **Manage Jenkins → Plugins → Available plugins** and install:

| Plugin | Purpose |
|--------|---------|
| **Pipeline** | Core pipeline support |
| **Docker Pipeline** | Docker build/push steps |
| **Credentials Binding** | Secure credential injection |
| **Git** | SCM integration |
| **GitHub Branch Source** | Multi-branch pipeline support |
| **Blue Ocean** (optional) | Better pipeline visualization |
| **Timestamper** | Add timestamps to console output |
| **Workspace Cleanup** | Clean workspace between builds |

### Post-Installation

Restart Jenkins after plugin installation:
```bash
sudo systemctl restart jenkins
```

---

## 4. Docker Configuration

### Add Jenkins User to Docker Group

Jenkins needs permission to run Docker commands:

```bash
# Add jenkins user to docker group
sudo usermod -aG docker jenkins

# Verify
groups jenkins

# Restart Jenkins to apply group changes
sudo systemctl restart jenkins
```

### Verify Docker Access

Test as the Jenkins user:
```bash
sudo -u jenkins docker ps
sudo -u jenkins docker run hello-world
```

If you get permission denied, log out and back in, or reboot the server.

---

## 5. Credential Setup

This is the **most critical step** — your pipeline will fail without this.

### 5.1 Docker Hub Access Token

1. Log in to [hub.docker.com](https://hub.docker.com) as `vishruthere`
2. Go to **Account Settings → Security**
3. Click **New Access Token**
4. Configure:
   - **Token name:** `jenkins-ci`
   - **Access permissions:** `Read, Write, Delete`
5. Click **Generate** and **copy the token immediately** (shown only once)

### 5.2 Add Credential to Jenkins

1. Go to **Manage Jenkins → Credentials → System → Global credentials (unrestricted)**
2. Click **+ Add Credentials**
3. Fill the form:

   | Field | Value |
   |-------|-------|
   | **Kind** | `Username with password` |
   | **Scope** | `Global` |
   | **Username** | `vishruthere` |
   | **Password** | *Paste your Docker Hub access token here* |
   | **ID** | `docker-hub-credentials` |
   | **Description** | `Docker Hub push access for CareerShield` |

4. Click **OK**

### 5.3 Verify Credential

Go to **Manage Jenkins → Script Console** and run:
```groovy
def creds = com.cloudbees.plugins.credentials.CredentialsProvider.lookupCredentials(
    com.cloudbees.plugins.credentials.common.StandardUsernameCredentials.class,
    Jenkins.instance,
    null,
    null
)
creds.each { println "ID: ${it.id}, User: ${it.username}" }
```

You should see: `ID: docker-hub-credentials, User: vishruthere`

---

## 6. Pipeline Job Configuration

### 6.1 Create the Pipeline Job

1. Go to **Jenkins Dashboard → New Item**
2. Enter name: `careershield-mlops`
3. Select **Pipeline**
4. Click **OK**

### 6.2 Configure Build Triggers (Optional)

In the job configuration:
- **Build Triggers → GitHub hook trigger for GITScm polling** (for webhooks)
- Or **Poll SCM** with schedule: `H/5 * * * *` (every 5 minutes)

### 6.3 Pipeline Definition

Under **Pipeline** section, select:

**Definition:** `Pipeline script from SCM`

**SCM:** `Git`

**Repository URL:** `https://github.com/vishruthere/careershield.git` (your repo)

**Credentials:** (your GitHub credential if private repo)

**Branch Specifier:** `*/main` (or `*/vishrut-feat-branch` for testing)

**Script Path:** `layoff-risk-prediction/Jenkinsfile`

> **Note:** Ensure your `Jenkinsfile` is committed to the repository at the specified path.

### 6.4 Save and Run

Click **Save**, then **Build Now** to test.

---

## 7. Jenkinsfile Walkthrough

Your pipeline has 12 stages:

```
┌─────────────────────────────────────────────────────────────┐
│  Stage 1: Checkout Code                                     │
│  → Pulls latest code from Git                               │
├─────────────────────────────────────────────────────────────┤
│  Stage 2: Setup Environment                                 │
│  → Creates Python venv, installs dependencies               │
├─────────────────────────────────────────────────────────────┤
│  Stage 3: Retrain Model                                     │
│  → Runs TensorFlow retraining script                        │
│  → Generates new model artifacts in models/                 │
├─────────────────────────────────────────────────────────────┤
│  Stage 4: Validate Metrics                                  │
│  → Checks model_schema.json exists                         │
├─────────────────────────────────────────────────────────────┤
│  Stage 5: Run Unit Tests                                    │
│  → Validates Python environment                            │
├─────────────────────────────────────────────────────────────┤
│  Stage 6: Build Docker Images                               │
│  → Builds backend (Python/TensorFlow)                       │
│  → Builds frontend (Node/Vite/Nginx)                        │
├─────────────────────────────────────────────────────────────┤
│  Stage 7: Push Docker Images                                │
│  → Logs in to Docker Hub                                   │
│  → Pushes both images with version tag + latest            │
├─────────────────────────────────────────────────────────────┤
│  Stage 8: Deploy Application                                │
│  → Runs docker-compose up -d                                │
├─────────────────────────────────────────────────────────────┤
│  Stage 9: Health Check                                      │
│  → Polls http://localhost:8000/health                       │
│  → Retries for 60 seconds                                   │
├─────────────────────────────────────────────────────────────┤
│  Stage 10: Smoke Test                                       │
│  → Sends test prediction request                            │
│  → Validates risk_probability in response                  │
├─────────────────────────────────────────────────────────────┤
│  Stage 11: Cleanup                                          │
│  → Removes dangling Docker images                          │
├─────────────────────────────────────────────────────────────┤
│  Stage 12: Final Status                                     │
│  → Confirms deployment success                             │
└─────────────────────────────────────────────────────────────┘
```

### Key Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `DOCKER_USER` | `vishruthere` | Docker Hub username |
| `VERSION` | `${BUILD_NUMBER}` | Auto-incrementing version tag |
| `PROJECT_DIR` | `layoff-risk-prediction` | Working directory |

---

## 8. Troubleshooting

### 8.1 Common Errors

#### Error: `Could not find credentials entry with ID 'docker-hub-credentials'`

**Cause:** Credential not created or ID mismatch.

**Fix:**
1. Go to **Manage Jenkins → Credentials**
2. Verify credential ID is exactly `docker-hub-credentials`
3. Check for trailing spaces or hidden characters
4. Ensure credential is in **Global** scope, not folder-scoped

---

#### Error: `denied: requested access to the resource is denied`

**Cause:** Wrong password type or insufficient permissions.

**Fix:**
1. Ensure you're using a **Docker Hub Access Token**, not your account password
2. Regenerate token with `Read, Write, Delete` permissions
3. Update credential in Jenkins with new token

---

#### Error: `failed to calculate checksum: "/models": not found`

**Cause:** Docker build context doesn't include `models/` directory.

**Fix:**
- Ensure `Dockerfile` is at project root
- Or build from parent directory: `docker build -f backend/Dockerfile .`
- Verify `models/` exists in repository

---

#### Error: `libgomp1` or TensorFlow runtime crash

**Cause:** Missing system libraries in Docker runtime stage.

**Fix:** Add to Dockerfile runtime stage:
```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends     libgomp1 libstdc++6     && rm -rf /var/lib/apt/lists/*
```

---

#### Error: `vv13` double version prefix

**Cause:** `VERSION = "v${BUILD_NUMBER}"` combined with `v${VERSION}` in echo.

**Fix:** In Jenkinsfile:
```groovy
// Change this:
VERSION = "v${BUILD_NUMBER}"
// To this:
VERSION = "${BUILD_NUMBER}"
```

---

#### Error: Health check timeout

**Cause:** Model loading takes longer than 10 seconds.

**Fix:** Increase start period in Dockerfile:
```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s --retries=3     CMD curl -f http://localhost:8000/health || exit 1
```

---

### 8.2 Debug Commands

```bash
# Check Jenkins logs
sudo journalctl -u jenkins -f

# Check Docker access as Jenkins user
sudo -u jenkins docker ps

# Inspect built image
docker run --rm -it vishruthere/careershield-backend:v13 /bin/bash

# Test health endpoint manually
curl http://localhost:8000/health

# Check running containers
docker-compose ps

# View container logs
docker logs careershield-backend
```

---

## 9. Maintenance

### Regular Tasks

| Task | Frequency | Command/Action |
|------|-----------|----------------|
| Update Jenkins plugins | Monthly | Manage Jenkins → Plugins → Updates |
| Rotate Docker Hub token | Quarterly | Regenerate in Docker Hub, update Jenkins |
| Clean old images | Weekly | `docker image prune -af` |
| Backup Jenkins home | Daily | `sudo tar -czf jenkins-backup.tar.gz /var/lib/jenkins` |
| Review build logs | Weekly | Check for warnings or deprecations |

### Performance Tuning

For faster builds, add these to your Jenkins server:

```bash
# Enable Docker BuildKit
export DOCKER_BUILDKIT=1

# Use BuildKit cache mounts in Dockerfile
RUN --mount=type=cache,target=/root/.cache/pip     pip install -r requirements.txt
```

### Security Best Practices

1. **Never commit credentials** to Git — always use Jenkins Credentials
2. **Use access tokens** instead of passwords for Docker Hub
3. **Enable CSRF protection** in Jenkins security settings
4. **Restrict job execution** to specific nodes/agents
5. **Regularly audit** who has Jenkins admin access

---

## Quick Reference Card

```
Jenkins URL:      http://<server>:8080
Credentials ID:   docker-hub-credentials
Docker Hub User:  vishruthere
Backend Image:    vishruthere/careershield-backend:v<BUILD_NUMBER>
Frontend Image:   vishruthere/careershield-frontend:v<BUILD_NUMBER>
Health Endpoint:  http://localhost:8000/health
Predict Endpoint: http://localhost:8000/predict
```

---

## Support

If issues persist:
1. Check **Jenkins console output** for exact error messages
2. Verify **Docker Hub token** is valid and not expired
3. Ensure **Jenkins user** has Docker permissions
4. Review **docker-compose.yml** for correct service names

---

*End of Guide — CareerShield MLOps Pipeline*
