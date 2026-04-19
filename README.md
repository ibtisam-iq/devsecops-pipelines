# DevSecOps Pipelines

> **The CI side of a two-repo GitOps architecture.**
> This repository takes real applications as input and runs them through secure, automated CI pipelines — building artifacts, scanning for vulnerabilities, pushing to registries, and triggering deployment in the CD repo.

---

## Tech Stack

![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?logo=docker&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-Code%20Quality-4E9BCD?logo=sonarqube&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-Security%20Scan-1904DA?logo=aquasecurity&logoColor=white)
![Nexus](https://img.shields.io/badge/Nexus-Artifact%20Registry-1B9640?logo=sonatype&logoColor=white)
![AWS ECR](https://img.shields.io/badge/AWS%20ECR-Container%20Registry-FF9900?logo=amazonaws&logoColor=white)

---

## What Is This Repository?

This is **not** an application repository.
This is the **CI side** of a two-repo GitOps architecture — it answers the question:

> *"Given application source code, how do you build, test, scan, and deliver a production-ready artifact in an automated and secure way?"*

Each pipeline inside this repo takes a real application (linked as a Git submodule) and runs it through a complete DevSecOps CI pipeline — the way real platform and DevSecOps teams operate.

### Two-Repo Architecture

```
┌─────────────────────────────────────┐     ┌──────────────────────────────────────────┐
│          devsecops-pipelines          │     │      platform-engineering-systems        │
│                                       │     │                                          │
│  CI Side                              │     │  CD + Infrastructure Side                │
│  ───────────────────────────────────  │     │  ────────────────────────────────────    │
│  • Checkout source code                │────▶│  • Deploy to EC2, ECS, EKS, Lambda, … │
│  • Build artifact (JAR / wheel / etc.) │     │  • Provision infra (Terraform)           │
│  • Run tests + SonarQube quality gate  │     │  • Orchestrate (Helm, ArgoCD)            │
│  • Trivy vulnerability scan            │     │  • GitOps sync (ArgoCD watches that)     │
│  • Push artifact to Nexus              │     │                                          │
│  • Build + push Docker image           │     │                                          │
│    (to ECR / Nexus / Docker Hub)       │     │                                          │
│  • Update image tag in CD repo ────────────────────────────────▶      (triggers CD)       │
└─────────────────────────────────────┘     └──────────────────────────────────────────┘
```

👉 CD + Infrastructure repo: [platform-engineering-systems](https://github.com/ibtisam-iq/platform-engineering-systems)

---

## CI Pipeline Stages

Every pipeline in this repo follows the same stage sequence, adapted per language and tool:

```
1. Checkout
   └─ Clone application source via Git submodule

2. Build
   └─ Compile and package the application
        Java   → mvn clean package
        Node   → npm install && npm run build
        Python → pip install -r requirements.txt

3. Test
   └─ Run unit + integration tests
        Java   → mvn test
        Node   → npm test
        Python → pytest

4. Code Quality
   └─ SonarQube static analysis
        Enforces quality gate before proceeding

5. Artifact Push (non-container)
   └─ Upload built artifact to Nexus Repository Manager
        Java   → JAR / WAR pushed to Nexus Maven repo
        Python → wheel pushed to Nexus PyPI repo
        Node   → tarball pushed to Nexus npm repo

6. Docker Build
   └─ Build Docker image from application Dockerfile

7. Image Scan
   └─ Trivy scans the image for CVEs before pushing
        Fails pipeline on CRITICAL vulnerabilities

8. Image Push
   └─ Push Docker image to registry
        Options: AWS ECR / Nexus Docker / Docker Hub

9. Tag Update (CI → CD Handoff)
   └─ Update image tag in platform-engineering-systems repo
        ArgoCD / CD pipeline detects the change and deploys
```

---

## Pipeline Matrix

The table below shows which CI tools are implemented per pipeline.

| Pipeline | Jenkins | GitHub Actions | Maven | npm | pip | SonarQube | Trivy | Nexus | ECR |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Java Monolith** | ✅ | ✅ | ✅ | — | — | ✅ | ✅ | ✅ | ✅ |
| **Node.js Monolith** | 🕒 | 🕒 | — | ✅ | — | 🕒 | 🕒 | 🕒 | 🕒 |
| **Python Monolith** | 🕒 | 🕒 | — | — | ✅ | 🕒 | 🕒 | 🕒 | 🕒 |

> ✅ Implemented    🕒 Planned
> Each pipeline is self-contained in its own subfolder and independently runnable.

---

## Repository Structure

```text
devsecops-pipelines/
│
├── pipelines/
│   ├── <pipeline-name>/            ← one folder per application
│   │   ├── app/                    ← Git submodule: application source code
│   │   ├── Jenkinsfile             ← Jenkins declarative pipeline
│   │   ├── .github/
│   │   │   └── workflows/
│   │   │       └── ci.yml          ← GitHub Actions workflow
│   │   ├── Dockerfile              ← application-specific Dockerfile
│   │   ├── sonar-project.properties← SonarQube project config
│   │   ├── trivy.yaml              ← Trivy scan config
│   │   └── README.md               ← pipeline-level walkthrough
│   │
│   └── <more-pipelines>/           ← added as new applications are onboarded
│
├── shared/                         ← reusable scripts, shared pipeline libraries
├── docs/                           ← cross-pipeline documentation
└── README.md
```

> Each pipeline is fully self-contained with its own Dockerfile, scanner config, and CI definitions.
> No pipeline shares configuration with another — changes in one cannot break others.

---

## Getting Started

### Prerequisites

```bash
git --version          # Git
docker --version       # Docker
aws --version          # AWS CLI v2 (for ECR push)
java --version         # Java 17+ (for Java pipelines)
node --version         # Node.js LTS (for Node pipelines)
python3 --version      # Python 3.10+ (for Python pipelines)
```

### Clone with Submodules

```bash
git clone --recurse-submodules https://github.com/ibtisam-iq/devsecops-pipelines.git
cd devsecops-pipelines
```

If you already cloned without submodules:

```bash
git submodule update --init --recursive
```

### Keeping Submodules Updated

To pull the latest commit from all submodule upstream repos:

```bash
git submodule update --remote --merge
git add .
git commit -m "chore: update all submodules to latest"
git push
```

### Run a Pipeline

Each pipeline folder is self-contained. Example — Java Monolith:

```bash
cd pipelines/java-monolith
# For Jenkins: point your Jenkins job at the Jenkinsfile here
# For GitHub Actions: the workflow triggers automatically on push
# See README.md inside for full setup instructions
```

---

## Engineering Principles

- **Pipeline isolation** — each pipeline is independently runnable; no shared state between pipelines
- **Security-first** — Trivy scans and SonarQube quality gates are non-negotiable stages
- **Registry-agnostic** — images can be pushed to ECR, Nexus, or Docker Hub depending on the setup
- **Submodule separation** — application source lives in its own repo; this repo owns CI operations
- **CI → CD handoff** — the pipeline's final step updates the image tag in the CD repo, closing the GitOps loop
- **Multi-tool** — each pipeline ships with both a Jenkinsfile and a GitHub Actions workflow

---

## Related Repositories

| Repository | Purpose |
|---|---|
| [devsecops-pipelines](https://github.com/ibtisam-iq/devsecops-pipelines) | This repo — CI pipelines |
| [platform-engineering-systems](https://github.com/ibtisam-iq/platform-engineering-systems) | CD + Infrastructure (deployment targets) |
| [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) | Spring Boot application source |

---

## Author

**Muhammad Ibtisam Iqbal**
DevOps Engineer · Cloud Infrastructure · Kubernetes
[GitHub](https://github.com/ibtisam-iq) · [LinkedIn](https://linkedin.com/in/ibtisam-iq)
