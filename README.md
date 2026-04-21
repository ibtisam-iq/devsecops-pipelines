# DevSecOps Pipelines

**The CI layer of my three-repo GitOps architecture — where application code gets built, secured, packaged, and handed off to the deployment layer.**

![Jenkins](https://img.shields.io/badge/Jenkins-CI%2FCD-D24939?logo=jenkins&logoColor=white)
![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-CI%2FCD-2088FF?logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Containers-2496ED?logo=docker&logoColor=white)
![SonarQube](https://img.shields.io/badge/SonarQube-Code%20Quality-4E9BCD?logo=sonarqube&logoColor=white)
![Trivy](https://img.shields.io/badge/Trivy-Security%20Scan-1904DA?logo=aquasecurity&logoColor=white)
![Nexus](https://img.shields.io/badge/Nexus-Artifact%20Registry-1B9640?logo=sonatype&logoColor=white)
![AWS ECR](https://img.shields.io/badge/AWS%20ECR-Container%20Registry-FF9900?logo=amazonaws&logoColor=white)

---

## What I Built Here

I built this repository to be the automated CI backbone for multiple applications I own and maintain.

Each application I work on lives in its own dedicated repository — some forked from upstream projects and heavily modernized by me, others built from scratch. Once an application is containerized and validated locally, I onboard it here as a Git submodule and build a full DevSecOps pipeline around it.

The goal of this repo is to demonstrate something specific: **I can take any application — Java, Node.js, Python, or otherwise — and build a production-grade, security-hardened CI pipeline for it using any CI tool.** Not configure a template once. Actually build, run, debug, and validate it end to end.

> I built and ran every pipeline in this repository against my own self-hosted CI/CD stack — Jenkins at `https://jenkins.ibtisam-iq.com`, SonarQube at `https://sonar.ibtisam-iq.com`, and Nexus at `https://nexus.ibtisam-iq.com`. How I provisioned and configured that stack is documented in [`docs/cicd-stack-setup.md`](docs/cicd-stack-setup.md).

---

## Where This Repo Fits

This is **Repo 2** in a deliberate three-repo separation of concerns:

```
Repo 1 — Application Source           Repo 2 — CI (this repo)                  Repo 3 — Deployment
──────────────────────────            ─────────────────────────────────         ───────────────────────────
java-monolith-app              ──▶    devsecops-pipelines                ──▶   platform-engineering-systems
node-monolith-app                      Checkout → Build → Test                   EC2 · ECS · EKS · GitOps
python-monolith-app                    SonarQube quality gate                    Helm · Terraform · ArgoCD
(each is my own repo)                  Trivy vulnerability scan
                                       Push JAR/wheel/tarball → Nexus
                                       Build & scan Docker image
                                       Push image → registry
                                       Update image tag in Repo 3  ────────────────────────────────────────▶
                                                                                 ArgoCD detects & deploys
```

Every application in Repo 1 is a repository I own. Some I forked and modernized — updating `pom.xml`, adding Dockerfiles, `compose.yml`, environment variable standardization, and health probes — so they could actually run in containers and survive a Kubernetes liveness check. That modernization work is documented inside each application's own README.

---

## How I Onboard Each Application

For every application I add to this repo, I follow the same sequence:

**Step 1 — Fork and modernize the application source repo**

I fork the upstream repo into my own GitHub account, then did everything needed to make it production-ready: dependency upgrades, environment variable refactoring, multi-stage Dockerfile, `compose.yml`, `.dockerignore`, `.gitignore`, and health endpoint configuration. The application lives in its own repo — independently versioned and documented.

**Step 2 — Add it here as a Git submodule**

```bash
git submodule add https://github.com/ibtisam-iq/<app-repo>.git pipelines/<app-name>/app
git submodule update --init --recursive
```

The submodule pins to a specific commit. When I update the application source — fix a build issue, add a new config, adjust a dependency — I pull the latest into this repo:

```bash
git submodule update --remote --merge
git add .
git commit -m "chore: update <app-name> submodule to latest"
git push
```

**Step 3 — Read the application before writing a single pipeline line**

I read `pom.xml` (or `package.json`, or `requirements.txt`), the `Dockerfile`, and `application.properties` before writing anything. The pipeline has to know: what is the build command, what is the artifact filename, what port does the app expose, what environment variables does it need, what does the healthcheck endpoint look like.

**Step 4 — Write the Jenkinsfile**

A declarative Jenkins pipeline covering all DevSecOps stages. I ran this against my live Jenkins instance, debugged real errors, and validated each stage produced the expected output — artifact in Nexus, image in the registry, quality gate passed.

**Step 5 — Write the GitHub Actions workflow**

The same pipeline logic re-implemented as a GitHub Actions workflow — same stages, different syntax, different runner model. Both pipelines are independently runnable and cover the full CI sequence.

---

## CI Pipeline Stage Sequence

Every pipeline I wrote in this repo follows this stage sequence, adapted to the language and build tool of the specific application:

```
1.  Checkout          → Clone this repo with submodules; app source populates pipelines/<app>/app/
2.  Versioning        → Extract version from pom.xml / package.json / setup.py
3.  Build             → Compile and package
                          Java   → mvn clean package
                          Node   → npm ci && npm run build
                          Python → pip install && python -m build
4.  Test              → Run unit + integration tests with coverage
                          Java   → mvn test (JaCoCo coverage report)
                          Node   → npm test
                          Python → pytest --cov
5.  SonarQube Scan    → Static analysis via sonar-scanner
                        Configured server: sonar-server (https://sonar.ibtisam-iq.com)
                        Configured credential ID: sonarqube-token
6.  Quality Gate      → waitForQualityGate() — pipeline fails here if gate is not passed
7.  Artifact Push     → Upload built artifact to Nexus
                          Java   → JAR to maven-releases / maven-snapshots
                                   via settings.xml (Config File ID: maven-settings)
                          Node   → tarball to Nexus npm repo
                          Python → wheel to Nexus PyPI repo
8.  Docker Build      → Build image from application Dockerfile
9.  Trivy Scan        → Scan image for CVEs before it ever reaches a registry
                        Fails pipeline on CRITICAL severity
10. Image Push        → Push to Docker Hub (credential ID: docker-creds)
                        or AWS ECR / Nexus Docker registry depending on setup
11. Tag Update        → Update image tag in platform-engineering-systems (Repo 3)
                        ArgoCD detects the commit and triggers deployment
```

---

## Applications Onboarded

| Application | Source Repo | Language | Jenkins | GitHub Actions |
|---|---|---|:---:|:---:|
| **java-monolith** | [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) | Java 21 / Spring Boot | ✅ | ✅ |
| **node-monolith** | *(planned)* | Node.js | 🕒 | 🕒 |
| **python-monolith** | *(planned)* | Python 3.12 | 🕒 | 🕒 |

> ✅ Implemented and validated    🕒 Planned — submodule slot reserved

Each application is in its own subfolder under `pipelines/` and is fully self-contained. Changing one pipeline cannot affect another.

---

## Repository Structure

```text
devsecops-pipelines/
│
├── pipelines/
│   ├── java-monolith/                  ← First application onboarded
│   │   ├── app/                        ← Git submodule: github.com/ibtisam-iq/java-monolith-app
│   │   ├── jenkins/
│   │   │   └── Jenkinsfile             ← Jenkins declarative pipeline (fully validated)
│   │   ├── github-actions/
│   │   │   └── ci.yml                  ← GitHub Actions workflow (same stages, different syntax)
│   │   └── README.md                   ← Pipeline walkthrough for this specific app
│   │
│   └── <next-app>/                     ← Added as each new application is onboarded
│       ├── app/                        ← Git submodule: next application source
│       ├── jenkins/
│       └── github-actions/
│
├── docs/
│   ├── cicd-stack-setup.md             ← How I built and configured my Jenkins/SonarQube/Nexus stack
│   └── architecture.md                 ← Three-repo architecture overview
│
└── README.md
```

---

## My CI/CD Stack

Before running any pipeline in this repo, I built and configured the full CI/CD infrastructure from scratch. The complete setup — provisioning, Jenkins post-setup scripts, plugin installation, credentials, SonarQube integration, Nexus Maven settings — is documented step by step in [`docs/cicd-stack-setup.md`](docs/cicd-stack-setup.md).

| Service | URL | Configured As |
|---|---|---|
| Jenkins | `https://jenkins.ibtisam-iq.com` | CI orchestrator; 2 executors on built-in node |
| SonarQube | `https://sonar.ibtisam-iq.com` | Static analysis server; webhook → Jenkins |
| Nexus | `https://nexus.ibtisam-iq.com` | Maven releases, snapshots, and Docker registry |

Credentials configured in Jenkins (IDs used in Jenkinsfiles):

| Credential ID | Type | Purpose |
|---|---|---|
| `sonarqube-token` | Secret Text | SonarQube user token for `jenkins-ci` user |
| `github-creds` | Username/Password | GitHub PAT for SCM checkout and tag updates |
| `docker-creds` | Username/Password | Docker Hub access token for image push |
| `nexus-creds` | Username/Password | Nexus `jenkins-ci` user for artifact push |
| `maven-settings` | Config File (XML) | Global `settings.xml` with Nexus server credentials |

Tools installed system-wide on the Jenkins OS (no Jenkins UI tool config needed):

`mvn 3.9.15` · `node 22 LTS` · `docker 29.x` · `trivy 0.69.3` · `aws-cli v2` · `kubectl 1.35` · `helm 4.1.4` · `terraform 1.14.x`

The only tool configured via `Manage Jenkins → Tools` is the SonarQube Scanner (`sonar-scanner`) — because it cannot be installed as a plain binary and must be managed through Jenkins.

---

## Clone and Run

```bash
# Clone with all submodules populated
git clone --recurse-submodules https://github.com/ibtisam-iq/devsecops-pipelines.git
cd devsecops-pipelines

# If already cloned without submodules
git submodule update --init --recursive

# Update all submodules to their latest upstream commit
git submodule update --remote --merge
git add . && git commit -m "chore: update submodules" && git push
```

To run the Java Monolith pipeline:
- **Jenkins:** Create a Pipeline job, point it at `pipelines/<app-name>/jenkins/Jenkinsfile`
- **GitHub Actions:** Workflow triggers automatically on push; see `pipelines/<app-name>/github-actions/ci.yml`

Full per-pipeline setup instructions are in each pipeline's own `README.md`.

---

## Key Idea

> Application code is the input. This repo is where it gets built, tested, scanned, and packaged. Repo 3 is where it runs.

| Repository | Role |
|---|---|
| **[java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app)** (and others) | Application source — forked, modernized, and maintained by me |
| **[devsecops-pipelines](https://github.com/ibtisam-iq/devsecops-pipelines)** | CI — builds, tests, scans, and delivers artifacts |
| **[platform-engineering-systems](https://github.com/ibtisam-iq/platform-engineering-systems)** | CD + Infrastructure — deploys and operates the artifacts |

---

## Author

**Muhammad Ibtisam Iqbal**
DevOps Engineer · Platform Engineering · Cloud Infrastructure
[GitHub](https://github.com/ibtisam-iq) · [LinkedIn](https://linkedin.com/in/ibtisam-iq) · [Portfolio](https://ibtisam-iq.com)
