# Java Monolith — CI Pipeline

This folder contains all CI pipeline definitions for the **Java Monolith** application — a Spring Boot 3.4.4 banking app running on Java 21 with MySQL. The application source lives in a separate repository ([ibtisam-iq/java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app)) and is linked here as a Git submodule at `app/`.

> This is the first application I onboarded into this repo. Everything built here — the pipeline structure, the folder conventions, the submodule pattern — became the template for all future applications.

---

## Application at a Glance

| Property | Value |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.4.4 |
| Build Tool | Maven |
| Database | MySQL 8.4 |
| Artifact | `bankapp-0.0.1-SNAPSHOT.jar` |
| Container Port | `8000` |
| Source Repo | [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) |

Before writing a single pipeline line, I read the application's `pom.xml`, `Dockerfile`, `application.properties`, and `compose.yml` to understand exactly what the build produces, what port it runs on, what environment variables it needs, and what the healthcheck endpoint looks like. That context is what makes the pipeline accurate rather than generic.

---

## Folder Structure

```text
pipelines/java-monolith/
├── app/                        ← Git submodule: github.com/ibtisam-iq/java-monolith-app
├── jenkins/
│   └── Jenkinsfile             ← Jenkins declarative pipeline (fully validated end-to-end)
├── github-actions/
│   └── ci.yml                  ← Reference copy only (see note below)
└── README.md                   ← This file
```

> **GitHub Actions placement decision:** The `ci.yml` that actually runs lives in the application source repository at [java-monolith-app/.github/workflows/ci.yml](https://github.com/ibtisam-iq/java-monolith-app/blob/main/.github/workflows/ci.yml). GitHub Actions triggers on commits to the source repo, and the app code is already at the root — no submodule checkout needed. The copy here at `github-actions/ci.yml` is a **reference and documentation copy** kept in sync for comparison with the Jenkinsfile. For every future application onboarded, the same rule applies: `ci.yml` lives in the source repo; the Jenkinsfile lives here.

The `app/` directory is not application code I own — it is a pinned Git submodule pointing to the application source repository. When the Jenkins pipeline runs, it checks out this entire repo with submodules initialised, then uses `dir('pipelines/java-monolith/app')` to run all Maven and Docker commands in the correct subdirectory.

---

## CI/CD Stack

All pipelines in this folder run against my self-hosted stack. The full provisioning and configuration of this stack is documented in [`docs/cicd-stack-setup.md`](../../docs/cicd-stack-setup.md).

| Service | URL | Role in this pipeline |
|---|---|---|
| Jenkins | `https://jenkins.ibtisam-iq.com` | Runs the Jenkinsfile |
| SonarQube | `https://sonar.ibtisam-iq.com` | Static analysis + quality gate |
| Nexus | `https://nexus.ibtisam-iq.com` | Receives the published JAR artifact |
| Docker Hub | `hub.docker.com/u/mibtisam` | Receives the built image |
| GHCR | `ghcr.io/ibtisam-iq` | Secondary image registry |

Credential IDs used in the Jenkinsfile (configured in Jenkins → Credentials):

| ID | Type | Used for |
|---|---|---|
| `github-creds` | Username/Password | SCM checkout + CD repo update |
| `sonarqube-token` | Secret Text | SonarQube analysis authentication |
| `maven-settings` | Config File (XML) | Injects `settings.xml` with Nexus credentials |
| `docker-creds` | Username/Password | Docker Hub push |
| `ghcr-creds` | Username/Password | GHCR push |
| `nexus-creds` | Username/Password | Nexus Docker registry push |

GitHub Actions secrets (configured in java-monolith-app → Settings → Secrets):

| Secret | Used for |
|---|---|
| `DOCKER_USERNAME` / `DOCKER_PASSWORD` | Docker Hub push |
| `NEXUS_USERNAME` / `NEXUS_PASSWORD` | Nexus Docker push |
| `SONAR_TOKEN` / `SONAR_HOST_URL` | SonarQube analysis |
| `GIT_TOKEN` | CD repo update |
| `MAVEN_SETTINGS_XML` | Nexus Maven credentials |
| `GITHUB_TOKEN` | GHCR push (automatic — no setup needed) |

---

## How I Built the Jenkins Pipeline

### Step 1 — Read the application before writing anything

I opened `pom.xml` and noted: `groupId=com.example`, `artifactId=bankapp`, `version=0.0.1-SNAPSHOT`, Java 21, Spring Boot 3.4.4, JaCoCo plugin present. I opened the `Dockerfile` and noted: multi-stage build, runtime port `8000`, non-root user. This is what the pipeline is built around — not assumptions.

---

### Step 2 — Set up the `options {}` block

The first thing I added that was not in my older Jenkinsfile was a proper `options {}` block:

```groovy
options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 45, unit: 'MINUTES')
    disableConcurrentBuilds()
    timestamps()
    ansiColor('xterm')
}
```

These are operational controls — not build steps. They prevent disk fill-up from old builds, kill hung jobs automatically, stop two concurrent runs from clobbering each other's image tags, and make logs readable with timestamps and color.

---

### Step 3 — Centralise all values in `environment {}`

Instead of scattering hardcoded strings across stages, I put everything reusable at the top:

```groovy
environment {
    APP_NAME     = 'bankapp'
    APP_VERSION  = '0.0.1-SNAPSHOT'
    DOCKER_USER  = 'mibtisam'
    IMAGE_NAME   = "${DOCKER_USER}/${APP_NAME}"
    GHCR_USER    = 'ibtisam-iq'
    GHCR_IMAGE   = "ghcr.io/${GHCR_USER}/${APP_NAME}"
    NEXUS_URL    = 'https://nexus.ibtisam-iq.com'
    NEXUS_DOCKER = 'nexus.ibtisam-iq.com'
    APP_DIR      = 'pipelines/java-monolith/app'
}
```

`IMAGE_TAG` is intentionally absent here — it requires a Git SHA that is only available at runtime, so it is set dynamically in the Versioning stage.

---

### Step 4 — Checkout with submodule support

The simple `git` step does not initialise submodules. I need the full `checkout()` with `SubmoduleOption`:

```groovy
checkout([
    $class: 'GitSCM',
    branches: [[name: '*/main']],
    extensions: [
        [$class: 'SubmoduleOption',
            disableSubmodules: false,
            parentCredentials: true,
            recursiveSubmodules: true,
            trackingSubmodules: false]
    ],
    userRemoteConfigs: [[
        url: 'https://github.com/ibtisam-iq/devsecops-pipelines.git',
        credentialsId: 'github-creds'
    ]]
])
```

After this stage, the workspace is fully populated including the app source at `pipelines/java-monolith/app/`.

---

### Step 5 — Trivy filesystem scan first (fail-fast)

I moved the Trivy filesystem scan to run **before** compile and test. If there is a hardcoded secret or a critical CVE in `pom.xml`, I want to know immediately — not after wasting 3 minutes on a Maven build.

```groovy
dir(APP_DIR) {
    sh '''
        trivy fs \
            --scanners secret,vuln,config \
            --exit-code 1 --severity CRITICAL \
            --format json --output trivy-fs-report.json \
            . || true
        trivy fs \
            --scanners secret,vuln,config \
            --severity HIGH,MEDIUM \
            --format table .
    '''
}
```

The JSON report is archived as a Jenkins artifact. The second run prints a human-readable table for HIGH and MEDIUM findings in the console.

---

### Step 6 — Build an informative image tag

Old approach: `IMAGE_TAG = "v${BUILD_NUMBER}"` — tells me nothing about which commit is inside.

New approach:

```groovy
script {
    dir(APP_DIR) {
        def shortSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
        env.IMAGE_TAG = "${APP_VERSION}-${shortSha}-${BUILD_NUMBER}"
        // example: 0.0.1-SNAPSHOT-ab3f12c-42
    }
}
```

This tag encodes the Maven version, the exact Git commit, and the build number. When I see an image in any registry, I know immediately which code it contains.

---

### Step 7 — Build and test with `mvn clean verify`

Instead of separate `mvn compile` and `mvn test` stages (as in the old file), I use:

```groovy
dir(APP_DIR) {
    withMaven(globalMavenSettingsConfig: 'maven-settings') {
        sh 'mvn clean verify -B --no-transfer-progress'
    }
}
```

`mvn clean verify` runs the full lifecycle through the `verify` phase — compile, test, package, verify. `withMaven` injects the `settings.xml` stored in Jenkins Config File Provider so Maven can authenticate against Nexus for dependency resolution. The `post { always { ... } }` block inside this stage publishes JUnit results and JaCoCo coverage to Jenkins so the build UI shows parsed test counts and coverage trends.

---

### Step 8 — SonarQube analysis via Maven plugin

Old approach: `$SCANNER_HOME/bin/sonar-scanner` (standalone binary, needs `SCANNER_HOME` env var).

New approach:

```groovy
dir(APP_DIR) {
    withSonarQubeEnv('sonar-server') {
        sh '''
            mvn sonar:sonar -B \
                -Dsonar.projectKey=java-monolith \
                -Dsonar.projectName=java-monolith \
                -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
        '''
    }
}
```

`withSonarQubeEnv('sonar-server')` automatically injects `SONAR_HOST_URL` and the token. No `SCANNER_HOME` needed. The JaCoCo path wires up test coverage into the SonarQube analysis so the quality gate can include coverage metrics.

---

### Step 9 — Quality gate that actually fails

Old approach: `abortPipeline: false` with a 1-hour timeout — the pipeline would continue even if the gate failed.

New approach:

```groovy
timeout(time: 5, unit: 'MINUTES') {
    waitForQualityGate abortPipeline: true
}
```

`abortPipeline: true` means the pipeline hard-fails if the quality gate is not passed. I do not want to build and push a Docker image for code that failed static analysis. 5 minutes is the ceiling — the SonarQube webhook fires back to Jenkins within seconds of analysis completing.

---

### Step 10 — Publish JAR to Nexus

```groovy
dir(APP_DIR) {
    withMaven(globalMavenSettingsConfig: 'maven-settings') {
        sh 'mvn deploy -DskipTests -B --no-transfer-progress'
    }
}
```

`-DskipTests` skips tests — they already ran in Step 7. Running them again would be a redundant pass. The `maven-settings` config file provides the Nexus server credentials without hardcoding them in the Jenkinsfile or `pom.xml`.

---

### Step 11 — Docker build with OCI labels and multi-registry tags

```groovy
dir(APP_DIR) {
    sh """
        docker build \\
            --build-arg SERVER_PORT=8000 \\
            --label "org.opencontainers.image.version=${IMAGE_TAG}" \\
            --label "org.opencontainers.image.revision=\${GIT_COMMIT}" \\
            --label "org.opencontainers.image.created=\$(date -u +%Y-%m-%dT%H:%M:%SZ)" \\
            -t ${IMAGE_NAME}:${IMAGE_TAG} \\
            -t ${IMAGE_NAME}:latest \\
            -t ${GHCR_IMAGE}:${IMAGE_TAG} \\
            -t ${GHCR_IMAGE}:latest \\
            .
    """
}
```

I build once and apply all registry tags in that single build. The OCI labels mean I can `docker inspect` any image later and see exactly which commit and timestamp produced it.

---

### Step 12 — Trivy image scan before any push

```groovy
sh "trivy image --severity CRITICAL,HIGH --format table ${IMAGE_NAME}:${IMAGE_TAG}"
```

The image is scanned **after build but before any registry push**. A critically vulnerable image never reaches Docker Hub, GHCR, or Nexus.

---

### Step 13 — Push to multiple registries with explicit login/logout

I replaced `withDockerRegistry()` with explicit `docker login → push → docker logout` for each registry. `withDockerRegistry` works cleanly for Docker Hub but behaves inconsistently with GHCR and Nexus. Explicit login/logout is identical across all registries and makes the authentication flow visible in the code.

Registries pushed to:
- Docker Hub (`hub.docker.com`) — credential ID: `docker-creds`
- GHCR (`ghcr.io`) — credential ID: `ghcr-creds`
- Nexus Docker registry (`nexus.ibtisam-iq.com`) — credential ID: `nexus-creds`
- AWS ECR — written, commented out until ECR repository is provisioned

Each registry gets its own stage so Jenkins shows exactly which push failed if one does.

---

### Step 14 — Update CD repo with new image tag

```groovy
withCredentials([usernamePassword(credentialsId: 'github-creds', ...)]) {
    sh """
        rm -rf cd-repo
        git clone https://\${GIT_USER}:\${GIT_TOKEN}@github.com/ibtisam-iq/platform-engineering-systems.git cd-repo
        cd cd-repo
        git config user.email "ci@ibtisam-iq.com"
        git config user.name "Jenkins CI"
        echo "IMAGE_TAG=${IMAGE_TAG}" > java-monolith-image.env
        git add .
        git commit -m "ci: update bankapp image tag to ${IMAGE_TAG} [skip ci]" || echo "Nothing to commit"
        git push origin main
    """
}
```

`rm -rf cd-repo` cleans any leftover from a previous run. `[skip ci]` prevents the CD repo's own pipeline from triggering a recursive loop. `|| echo "Nothing to commit"` handles identical tags without failing the pipeline. ArgoCD watches this repo and detects the commit to trigger deployment.

---

### Step 15 — Post-build cleanup

```groovy
post {
    always {
        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        sh "docker rmi ${GHCR_IMAGE}:${IMAGE_TAG} || true"
        sh 'rm -rf cd-repo'
        cleanWs()
    }
}
```

Without cleanup, Docker images and cloned repos accumulate across builds and fill the agent disk. Every image built in this pipeline is removed after the run, and `cleanWs()` wipes the workspace entirely.

---

## Full Pipeline Stage Sequence

```
1.  Checkout              → Clone repo with submodules initialised
2.  Trivy FS Scan         → Scan source code and pom.xml for secrets, CVEs, misconfigs
3.  Versioning            → Build IMAGE_TAG from APP_VERSION + short Git SHA + BUILD_NUMBER
4.  Build & Test          → mvn clean verify (compile → test → package → verify)
                            JUnit results + JaCoCo coverage published to Jenkins
5.  SonarQube Analysis    → mvn sonar:sonar with coverage wired in
6.  Quality Gate          → waitForQualityGate — hard fail on gate failure
7.  Publish JAR           → mvn deploy -DskipTests → Nexus maven-releases / maven-snapshots
8.  Docker Build          → Build image with OCI labels, tag for all registries
9.  Trivy Image Scan      → Scan built image before any push
10. Push to Docker Hub    → mibtisam/bankapp:<tag> + :latest
11. Push to GHCR          → ghcr.io/ibtisam-iq/bankapp:<tag> + :latest
12. Push to Nexus         → nexus.ibtisam-iq.com/bankapp:<tag> + :latest
13. Push to AWS ECR       → (written, commented out — activates when ECR repo is ready)
14. Update CD Repo        → Write IMAGE_TAG to platform-engineering-systems; ArgoCD deploys
15. Cleanup               → Remove all built images + cd-repo clone + cleanWs()
```

---

## GitHub Actions Pipeline

The GitHub Actions `ci.yml` covers the same 14-stage sequence as the Jenkinsfile — same tools, same credential pattern, different runner model. Jenkins uses a persistent self-hosted server; GitHub Actions uses ephemeral cloud runners.

**Placement:** The workflow that actually runs lives in the source repository at [java-monolith-app/.github/workflows/ci.yml](https://github.com/ibtisam-iq/java-monolith-app/blob/main/.github/workflows/ci.yml). The copy here at `github-actions/ci.yml` is kept as a **reference and documentation copy** — useful for comparing the two pipeline implementations side by side. This is the established convention for every application in this repo: `ci.yml` belongs to the source repo, Jenkinsfile belongs here.

---

## Running the Jenkins Pipeline

**Prerequisites:** CI/CD stack is live (see [`docs/cicd-stack-setup.md`](../../docs/cicd-stack-setup.md)).

```bash
# Clone with submodules populated
git clone --recurse-submodules https://github.com/ibtisam-iq/devsecops-pipelines.git

# If already cloned without --recurse-submodules
git submodule update --init --recursive
```

**In Jenkins:**

1. New Item → Pipeline
2. Pipeline definition: **Pipeline script from SCM**
3. SCM: Git → `https://github.com/ibtisam-iq/devsecops-pipelines.git`
4. Credentials: `github-creds`
5. Script Path: `pipelines/java-monolith/jenkins/Jenkinsfile`
6. Save → Build Now

---

## Pipeline Documentation

| Document | What it covers |
|---|---|
| [`docs/understand-jenkinsfile.md`](../../docs/understand-jenkinsfile.md) | Line-by-line explanation of every directive in the Jenkinsfile — old vs new, and why each decision was made |
| [`docs/cicd-stack-setup.md`](../../docs/cicd-stack-setup.md) | Full stack provisioning: Jenkins, SonarQube, Nexus, credentials, webhooks, Maven settings |

---

## Related Repositories

| Repository | Role |
|---|---|
| [java-monolith-app](https://github.com/ibtisam-iq/java-monolith-app) | Application source — forked, modernized, and documented by me |
| [devsecops-pipelines](https://github.com/ibtisam-iq/devsecops-pipelines) | CI — this repo |
| [platform-engineering-systems](https://github.com/ibtisam-iq/platform-engineering-systems) | CD + Infrastructure — receives the image tag and deploys |
