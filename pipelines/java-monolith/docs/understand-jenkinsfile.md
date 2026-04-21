# Jenkinsfile — How and Why I Wrote It This Way

This document is my own reference for the `java-monolith` Jenkins pipeline. I wrote it so that when I come back to this file after a gap, I can immediately remember why I used a specific directive instead of the simpler version I used before.

I have two versions of a Jenkinsfile for this project:

1. **The older one** — which I wrote about a year ago when I was learning Jenkins basics. Simple, readable, uses direct `mvn` and `sonar-scanner` commands.
2. **The current one** — which I wrote now as a production-grade DevSecOps pipeline. More explicit, more structured, covers more systems.

The content in both is the same in terms of what it does. The difference is in **how explicitly** it handles things, and **why**.

---

## The Old Jenkinsfile (Reference)

<details>
<summary>Click to expand the old Jenkinsfile</summary>

```groovy
pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }


    stages {
        /*
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/ibtisam-iq/BankingApp-Java-MySQL.git'
            }
        }
        */

        // Continuous Integration

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Testing') {
            steps {
                sh "mvn test"
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs-report.html ."
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {                 // server name configured in jenkins
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                    '''
                        /*
                        -Dsonar.projectName=IbtisamIQbankapp \
                        -Dsonar.projectKey=IbtisamIQbankapp \
                        -Dsonar.java.binaries=target \
                        -Dsonar.branch.name=ibtisam
                        */
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Package') {
            steps {
                sh "mvn package"
            }
        }

        stage('Deploy Artifact To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'IbtisamIQ-nexus', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: false) {
                    sh "mvn deploy"
                }
            }
        }

        stage('Docker Image Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t mibtisam/bankingapp-java-mysql:$IMAGE_TAG ."
                    }
                }
            }
        }

        stage('Scan Image') {
            steps {
                sh "trivy image --format table -o image-report.html mibtisam/bankingapp-java-mysql:$IMAGE_TAG"
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push mibtisam/bankingapp-java-mysql:$IMAGE_TAG"
                    }
                }
            }
        }

        stage('Update Manifest File') {
            steps {
                script {
                    cleanWs()         // Clean workspace before starting
                    withCredentials([usernamePassword(credentialsId: 'git', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh '''
                            # Clone the Mega-Project-CD repository
                            # git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/mibtisam/Mega-Project-CD.git

                            # Update the image tag in the manifest.yaml file
                            cd Mega-Project-CD
                            sed -i "s|mibtisam/bankingapp-java-mysql:.*|mibtisam/bankingapp-java-mysql:${IMAGE_TAG}|" manifest/manifest.yaml

                            # Confirm changes
                            echo "Updated manifest file contents:"
                            cat manifest/manifest.yaml

                            # Commit and push the changes
                            git config user.name "Ibtisam"
                            git config user.email "loveyou@ibtisam.com"
                            git add manifest/manifest.yaml
                            git commit -m "Update image tag to ${IMAGE_TAG}"
                            git push origin main
                        '''
                    }
                }
            }
        }
    }
}
    // Post Actions

post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'tempmail@gmail.com',
                from: 'ibtisamiq@tempmail.com',
                replyTo: 'ibtisamiq@tempmail.com',
                mimeType: 'text/html',

            )
        }
    }
}
```

</details>

---

## Overall structure

Both are **Declarative Pipelines**. That means I must use the fixed skeleton:

```groovy
pipeline {
    agent any
    environment { ... }
    options { ... }
    stages { ... }
    post { ... }
}
```

`pipeline {}` is the root block — everything lives inside it. `agent any` means run on any available Jenkins node. Both the old and new files use `agent any` because I only have one node.

---

## Why I removed the `tools {}` block

The old file had:

```groovy
tools {
    maven 'maven3'
}
```

That told Jenkins: "resolve the Maven installation named `maven3` from Manage Jenkins → Tools, and put it on the PATH for this job."

I removed this in the new file because Maven, JDK 21, Docker, Trivy, kubectl, Helm, Terraform, Ansible, and AWS CLI are all installed system-wide on the Jenkins OS via `install-pipeline-tools`. They are already on the system `PATH`. Jenkins finds them automatically through shell — no UI registration needed.

The `tools {}` block is only useful when a tool is managed through Jenkins UI and Jenkins needs to resolve its installation path dynamically. Since I handle tool installation at the OS level, this block is unnecessary and I commented it out.

---

## Why I removed `SCANNER_HOME = tool 'sonar-scanner'`

The old file had:

```groovy
environment {
    SCANNER_HOME = tool 'sonar-scanner'
}
```

and then used it as:

```groovy
sh '$SCANNER_HOME/bin/sonar-scanner'
```

That pattern is for calling the **standalone SonarQube Scanner binary** directly. Jenkins needs `SCANNER_HOME` to know where the binary lives.

In the new file I switched to the **Maven Sonar plugin** approach:

```groovy
withSonarQubeEnv('sonar-server') {
    sh 'mvn sonar:sonar ...'
}
```

Here, Maven handles the analysis internally through its plugin. `withSonarQubeEnv('sonar-server')` injects `SONAR_HOST_URL` and the token automatically. There is no binary path to resolve, so `SCANNER_HOME` is not needed.

Both approaches work. I chose `mvn sonar:sonar` for the current pipeline because the project is Maven-based, and the plugin path is cleaner for that setup.

---

## Why the `environment {}` block grew

The old file only needed:

```groovy
environment {
    SCANNER_HOME = tool 'sonar-scanner'
    IMAGE_TAG    = "v${BUILD_NUMBER}"
}
```

The new pipeline interacts with more systems: source directory, Maven settings, Docker Hub, GHCR, Nexus Docker registry, and the CD repository. I centralised all reusable values here so I do not repeat hardcoded strings across stages:

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

If the Docker username, Nexus host, or app directory changes, I only update one place.

`IMAGE_TAG` is not here — it is set dynamically in the `Versioning` stage because it needs a Git SHA which is only available at runtime.

---

## Why I added an `options {}` block

The old file had no `options` block. The new one has:

```groovy
options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 45, unit: 'MINUTES')
    disableConcurrentBuilds()
    timestamps()
    ansiColor('xterm')
}
```

These are not build steps — they are operational controls for how the pipeline behaves on the Jenkins server.

- `buildDiscarder` — keeps only the last 10 builds so old logs and artifacts do not fill up disk.
- `timeout` — kills the pipeline if it hangs. Protects the agent from jobs that freeze due to network or registry issues.
- `disableConcurrentBuilds` — prevents two runs of the same job from clashing, especially since both would push the same image tags and update the same CD repo.
- `timestamps` — adds timestamps to console logs so I can see exactly when each step started.
- `ansiColor('xterm')` — lets colored output from Maven, Trivy, and other tools display properly in Jenkins logs.

---

## Why the Checkout stage looks so different

### Old style

```groovy
git branch: 'main', url: 'https://github.com/ibtisam-iq/BankingApp-Java-MySQL.git'
```

Simple and fine when source code lives directly in the root of one repo.

### New style

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

The repository structure changed. The source code now lives at `pipelines/java-monolith/app` inside this pipeline repository as a submodule. The short `git` step does not know how to initialise submodules. I need the full `checkout()` step with `SubmoduleOption` configured.

### Line by line

| Line | What it means |
|---|---|
| `$class: 'GitSCM'` | Use the Git SCM plugin for this checkout. Unlocks advanced options. |
| `branches: [[name: '*/main']]` | Check out the `main` branch. The `*/` is the Jenkins Git plugin's branch pattern format. |
| `$class: 'SubmoduleOption'` | Configure how submodules are handled during checkout. |
| `disableSubmodules: false` | Do not skip submodules — initialise them. |
| `parentCredentials: true` | Use the same credentials I provided for the parent repo to access submodules too. Needed for private submodules. |
| `recursiveSubmodules: true` | Also fetch nested submodules if any submodule itself has submodules. |
| `trackingSubmodules: false` | Check out the exact commit pinned in the parent, not the latest commit of a branch. Better for reproducibility. |
| `url` | The pipeline repository URL to clone. |
| `credentialsId: 'github-creds'` | Use the GitHub credentials I stored in Jenkins under this ID. |

---

## Why Trivy filesystem scan runs before build

The old file ran Trivy filesystem scan after compile and test:

```groovy
stage('Trivy FS Scan') {
    steps {
        sh "trivy fs --format table -o fs-report.html ."
    }
}
```

I moved it to run **before** build in the new pipeline. The reason is fail-fast. If there is a hardcoded secret or a critical CVE in `pom.xml`, I want to know immediately — before wasting time on compile, test, and SonarQube.

The new scan is also more explicit:

```groovy
trivy fs \
    --scanners secret,vuln,config \
    --exit-code 1 \
    --severity CRITICAL \
    --no-progress \
    --format json \
    --output trivy-fs-report.json \
    . || true
```

- `--scanners secret,vuln,config` — scans for hardcoded secrets, dependency CVEs, and Dockerfile/config misconfigurations. The old scan did not specify scanners explicitly.
- `--format json --output trivy-fs-report.json` — produces a machine-readable report archived as a Jenkins artifact.
- The second Trivy run after it prints a human-readable table in console for HIGH and MEDIUM findings.
- `dir(APP_DIR)` — runs in the correct subdirectory since the app is not at workspace root.

---

## Why I created a separate Versioning stage

The old file used:

```groovy
IMAGE_TAG = "v${BUILD_NUMBER}"
```

Tags like `v15` are simple but tell me nothing about which code they contain.

The new file builds:

```groovy
env.IMAGE_TAG = "${APP_VERSION}-${shortSha}-${BUILD_NUMBER}"
// example: 0.0.1-SNAPSHOT-ab3f12c-42
```

This tag encodes:
- the Maven project version from `pom.xml`
- the Git commit short SHA
- the Jenkins build number

So when I see an image in a registry, I immediately know which commit it was built from.

`IMAGE_TAG` is set inside a `script {}` block because it involves a shell command (`git rev-parse --short HEAD`) and dynamic string building. Environment variables that need runtime computation cannot go in the static `environment {}` block at the top.

---

## Why Build and Test became `mvn clean verify`

The old file split these into separate stages:

```groovy
stage('Compile') {
    steps { sh "mvn compile" }
}

stage('Testing') {
    steps { sh "mvn test" }
}
```

The new file merges them into one:

```groovy
withMaven(globalMavenSettingsConfig: 'maven-settings') {
    sh 'mvn clean verify -B --no-transfer-progress'
}
```

- `mvn clean verify` runs a fuller Maven lifecycle: clean → compile → test → verify. It is more thorough than calling compile and test separately.
- `withMaven(globalMavenSettingsConfig: 'maven-settings')` injects the `settings.xml` I stored in Jenkins Config File Provider. This is needed so Maven can authenticate against Nexus for dependency resolution if any internal artifacts are required.
- `-B` is batch mode. Better for CI logs — no interactive prompts.
- `--no-transfer-progress` removes the noisy download progress lines from logs.

The `post { always { ... } }` block inside this stage publishes JUnit test results and JaCoCo coverage reports to Jenkins. The old file did not do this. It means Jenkins now shows parsed test counts and coverage trends in the build UI, not just a pass/fail result.

---

## Why SonarQube analysis uses `mvn sonar:sonar` instead of `$SCANNER_HOME/bin/sonar-scanner`

Already explained in the `SCANNER_HOME` section above. The key point is:

- Old file → standalone scanner binary → needs `SCANNER_HOME` path
- New file → Maven Sonar plugin → `withSonarQubeEnv()` handles credentials and URL

The new invocation also passes JaCoCo coverage paths:

```groovy
-Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
```

This wires up coverage data from the test stage into the SonarQube analysis so the quality gate can include coverage metrics.

---

## Why the Quality Gate changed

Old file:

```groovy
timeout(time: 1, unit: 'HOURS') {
    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
}
```

New file:

```groovy
timeout(time: 5, unit: 'MINUTES') {
    waitForQualityGate abortPipeline: true
}
```

Two changes:

1. `abortPipeline: false` → `abortPipeline: true` — the new pipeline fails hard on gate failure. I do not want the pipeline to continue and push a bad image just because the gate said it was poor code.
2. 1 hour → 5 minutes — the SonarQube webhook fires back to Jenkins within seconds of analysis completing. 5 minutes is a reasonable ceiling. 1 hour was too forgiving and could mask a hanging step.
3. `credentialsId: 'sonar-token'` is removed because `withSonarQubeEnv('sonar-server')` already handles authentication upstream. The quality gate step reuses that context automatically.

---

## Why there is no separate `Package` stage

The old file had:

```groovy
stage('Package') {
    steps { sh "mvn package" }
}
```

I removed this because `mvn clean verify` already handles compile, test, and packaging in the Build & Test stage. Running `mvn package` separately would be a redundant Maven pass. The `Publish JAR to Nexus` stage uses `mvn deploy -DskipTests` which also handles packaging internally before deploying.

---

## Why `mvn deploy` uses `-DskipTests`

```groovy
sh 'mvn deploy -DskipTests -B --no-transfer-progress'
```

Tests already ran in the `Build & Test` stage. Running them again during deploy would double the test execution time for no benefit. `-DskipTests` skips them on the deploy pass.

---

## Why Docker Build tags for multiple registries at once

Old file:

```groovy
sh "docker build -t mibtisam/bankingapp-java-mysql:$IMAGE_TAG ."
```

New file:

```groovy
docker build \
    --build-arg SERVER_PORT=8000 \
    --label "org.opencontainers.image.version=${IMAGE_TAG}" \
    --label "org.opencontainers.image.revision=${GIT_COMMIT}" \
    --label "org.opencontainers.image.created=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
    -t ${IMAGE_NAME}:${IMAGE_TAG} \
    -t ${IMAGE_NAME}:latest \
    -t ${GHCR_IMAGE}:${IMAGE_TAG} \
    -t ${GHCR_IMAGE}:latest \
    .
```

- `--build-arg SERVER_PORT=8000` — passes a build argument into the Dockerfile at build time.
- `--label` lines — attach OCI-standard metadata labels to the image. When I inspect the image later, I can see exactly which version, commit, and timestamp it came from.
- Multiple `-t` flags — build once and apply all registry tags in that single build. This avoids rebuilding the image separately for Docker Hub and GHCR, which would be wasteful.

---

## Why registry push stages are separate

The old file had one push stage only for Docker Hub. The new file separates pushes into individual stages:

- Push to Docker Hub
- Push to GHCR
- Push to Nexus Registry
- Push to AWS ECR (commented out, ready for later)

This is cleaner for visibility. If Docker Hub push succeeds but GHCR fails, Jenkins shows exactly which stage failed. If everything is in one stage, a failure in the middle makes it unclear how far the push got.

---

## Why I use `withCredentials(...)` and explicit `docker login` instead of `withDockerRegistry(...)`

The old file used:

```groovy
withDockerRegistry(credentialsId: 'docker-cred') {
    sh "docker push ..."
}
```

The new file uses:

```groovy
withCredentials([usernamePassword(
    credentialsId: 'docker-creds',
    usernameVariable: 'DOCKER_USERNAME',
    passwordVariable: 'DOCKER_PASSWORD'
)]) {
    sh """
        echo "${DOCKER_PASSWORD}" | docker login -u "${DOCKER_USERNAME}" --password-stdin
        docker push ...
        docker logout
    """
}
```

`withDockerRegistry` is a higher-level helper that works well for Docker Hub but does not always behave cleanly with GHCR and Nexus. Using explicit `docker login` → push → `docker logout` works identically for any registry and makes the authentication flow visible in the code. I also explicitly logout after each push to avoid session overlap between registry stages.

---

## Why AWS ECR is written but commented out

The ECR stage is fully written but commented out because the AWS credentials and ECR repository do not exist yet in my current lab environment. I wrote it now so the integration is documented, ready, and can be activated by simply uncommenting and setting three environment variables at the top.

---

## Why the CD Repo update stage exists

The old file tried to do this:

```groovy
cd Mega-Project-CD
sed -i "s|mibtisam/bankingapp-java-mysql:.*|...:${IMAGE_TAG}|" manifest/manifest.yaml
git commit -m "Update image tag to ${IMAGE_TAG}"
git push origin main
```

The new file does the same concept more cleanly:

```groovy
rm -rf cd-repo
git clone https://${GIT_USER}:${GIT_TOKEN}@github.com/ibtisam-iq/platform-engineering-systems.git cd-repo
cd cd-repo
echo "IMAGE_TAG=${IMAGE_TAG}" > java-monolith-image.env
git add .
git commit -m "ci: update bankapp image tag to ${IMAGE_TAG} [skip ci]" || echo "Nothing to commit"
git push origin main
```

Key improvements over the old version:

- `rm -rf cd-repo` cleans any leftover from a previous run before cloning fresh.
- `git config user.email / user.name` sets identity so commits are attributed properly.
- `|| echo "Nothing to commit"` handles the case where the same tag appears twice without failing the pipeline.
- `[skip ci]` in the commit message prevents the CD repo's own CI from triggering a recursive loop.
- The `sed` command is present as a comment, ready to activate once the Kubernetes manifests exist in that repo.

---

## Why `post {}` does cleanup

The old file's `post` block only sent an email notification. The new file does:

```groovy
post {
    always {
        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
        sh "docker rmi ${GHCR_IMAGE}:${IMAGE_TAG} || true"
        // ... more image removals
        sh 'rm -rf cd-repo'
        cleanWs()
    }
}
```

On a real Jenkins server, Docker images and cloned repos accumulate across builds. Without cleanup, the agent disk fills up over time. I remove every image I built and every temp directory I created, then call `cleanWs()` to wipe the workspace entirely.

---

## Why `dir(APP_DIR)` appears in most stages

The old pipeline assumed the application source was at workspace root. My current project stores the app at:

```
pipelines/java-monolith/app/
```

So any stage that needs to run Maven or Docker against the app source must first change directory:

```groovy
dir(APP_DIR) {
    sh 'mvn clean verify ...'
}
```

Without `dir(APP_DIR)`, Maven would look for `pom.xml` in the workspace root, not find it, and fail.

---

## Related Documentation

| Topic | File |
|---|---|
| Full CI/CD stack setup | [cicd-stack-setup.md](../../../docs/cicd-stack-setup.md) |
| Tool configuration rationale | [tool-configuration.md](https://github.com/ibtisam-iq/nectar/blob/main/delivery/jenkins/tool-configuration.md) |
| Credential types and injection | [credentials.md](https://github.com/ibtisam-iq/nectar/blob/main/delivery/jenkins/credentials.md) |
| SonarQube ↔ Jenkins integration | [sonar-jenkins.md](https://github.com/ibtisam-iq/nectar/blob/main/delivery/sonarqube/sonar-jenkins.md) |
| Nexus ↔ Jenkins integration | [nexus-jenkins.md](https://github.com/ibtisam-iq/nectar/blob/main/delivery/nexus/nexus-jenkins.md) |
