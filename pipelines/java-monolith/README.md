# Java Monolith — DevSecOps Pipeline

This folder contains all CI/CD pipeline definitions and DevSecOps toolchain configurations for the **Java Monolith** application. The application source code lives in a separate repository ([ibtisam-iq/java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app)) and is linked here as a Git submodule under `app/`.

---

## Application Overview

| Property       | Details                                      |
|----------------|----------------------------------------------|
| **Language**   | Java                                         |
| **Type**       | Monolithic Web Application                   |
| **Build Tool** | Maven                                        |
| **Source Repo**| [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) |

---

## Repository Structure

```
java-monolith/
├── app/                        # Git submodule → java-monolith-app source code
├── jenkins/
│   ├── Jenkinsfile.ci          # CI pipeline: build, test, SonarQube analysis, Nexus publish
│   ├── Jenkinsfile.cd          # CD pipeline: deploy to EC2 / Docker
│   └── Jenkinsfile.full        # Full end-to-end CI/CD pipeline
├── github-actions/
│   ├── ci.yml                  # GitHub Actions CI workflow
│   └── cd.yml                  # GitHub Actions CD workflow
├── sonar/
│   └── sonar-project.properties  # SonarQube project configuration
├── docker/
│   └── docker-compose.yml      # Local pipeline testing setup
└── README.md
```

---

## CI/CD Toolchain

| Tool           | Purpose                              | Hosted At                              |
|----------------|--------------------------------------|----------------------------------------|
| **Jenkins**    | Primary CI/CD orchestration          | [jenkins.ibtisam-iq.com](https://jenkins.ibtisam-iq.com) |
| **SonarQube**  | Static code analysis & quality gates | [sonar.ibtisam-iq.com](https://sonar.ibtisam-iq.com)     |
| **Nexus**      | Artifact repository & storage        | [nexus.ibtisam-iq.com](https://nexus.ibtisam-iq.com)     |
| **Docker**     | Containerization & image build       | Docker Hub / Local Registry            |

---

## Pipeline Stages

### Jenkins CI Pipeline (`Jenkinsfile.ci`)

```
Checkout → Maven Build → Unit Tests → SonarQube Analysis → Quality Gate → Publish to Nexus
```

1. **Checkout** — Pull source code from submodule
2. **Build** — `mvn clean package` to compile and package the JAR/WAR
3. **Unit Tests** — `mvn test` with JUnit reports
4. **SonarQube Analysis** — Static code analysis with coverage report
5. **Quality Gate** — Fail pipeline if quality thresholds are not met
6. **Publish to Nexus** — Upload versioned artifact to Nexus repository

### Jenkins CD Pipeline (`Jenkinsfile.cd`)

```
Pull Artifact from Nexus → Build Docker Image → Push to Registry → Deploy to Target
```

---

## Getting Started

### 1. Clone with Submodules

```bash
git clone --recurse-submodules https://github.com/ibtisam-iq/devsecops-pipelines.git
cd devsecops-pipelines/pipelines/java-monolith
```

### 2. Update Submodule

```bash
git submodule update --remote --merge
git add pipelines/java-monolith/app
git commit -m "chore: update java-monolith submodule to latest"
git push
```

### 3. Run Jenkins Pipeline

- Open [Jenkins Dashboard](https://jenkins.ibtisam-iq.com)
- Create a new Pipeline job
- Point it to `pipelines/java-monolith/jenkins/Jenkinsfile.ci`
- Configure credentials: SonarQube token, Nexus credentials, Docker registry

---

## SonarQube Configuration

The `sonar/sonar-project.properties` file defines the project key, sources, and coverage paths. Ensure the SonarQube server URL and token are configured in Jenkins credentials as `SONAR_TOKEN`.

```properties
sonar.projectKey=java-monolith
sonar.projectName=Java Monolith App
sonar.sources=app/src/main/java
sonar.tests=app/src/test/java
sonar.java.binaries=app/target/classes
```

---

## Nexus Artifact Publishing

Artifacts are published to the hosted Maven repository on Nexus. The `pom.xml` in the source repo defines the `<distributionManagement>` block pointing to `https://nexus.ibtisam-iq.com`.

---

## Related Repositories

| Repository | Description |
|---|---|
| [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) | Application source code |
| [devsecops-pipelines](https://github.com/ibtisam-iq/devsecops-pipelines) | All pipeline definitions (this repo) |
| [platform-engineering-systems](https://github.com/ibtisam-iq/platform-engineering-systems) | Deployment infrastructure (EC2, EKS, Kubernetes) |

---

## What's Next

- [ ] Add `Jenkinsfile.cd` for EC2 deployment
- [ ] Add GitHub Actions CI workflow
- [ ] Add Kubernetes deployment manifests (in platform-engineering-systems)
- [ ] Integrate OWASP Dependency-Check for SecOps
- [ ] Add Trivy image scanning stage
