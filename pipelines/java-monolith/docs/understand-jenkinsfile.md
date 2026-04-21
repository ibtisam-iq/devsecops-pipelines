# Jenkinsfile — How and Why I Wrote It This Way

This document is my detailed reference for the `java-monolith` Jenkins pipeline. I wrote it so that when I revisit this project later, I can immediately remember not only **what each directive does**, but also **why I wrote it this way**, **what I changed compared to the old version**, and **what I had to troubleshoot and correct while building it**.

I have two versions of a Jenkinsfile for this project:

1. **The older one** — written when I was learning Jenkins fundamentals. It is simpler, shorter, and useful as a learning baseline.
2. **The current one** — written now as a production-oriented DevSecOps CI pipeline. It is more explicit, more defensive, and better aligned with how real CI systems behave in practice.

The business goal of both files is the same: build, test, scan, package, and publish the Java monolith. The difference is in **how much control, traceability, and operational correctness** the newer pipeline has.

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
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \\
                    '''
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
                    cleanWs()
                    withCredentials([usernamePassword(credentialsId: 'git', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh '''
                            cd Mega-Project-CD
                            sed -i "s|mibtisam/bankingapp-java-mysql:.*|mibtisam/bankingapp-java-mysql:${IMAGE_TAG}|" manifest/manifest.yaml
                            echo "Updated manifest file contents:"
                            cat manifest/manifest.yaml
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

## Overall Structure

Both are **Declarative Pipelines**. That means the Jenkinsfile must follow the fixed high-level skeleton:

```groovy
pipeline {
    agent any
    environment { ... }
    options { ... }
    stages { ... }
    post { ... }
}
```

`pipeline {}` is the root block. `agent any` means Jenkins can run this job on any available node. In my setup, I currently use a single Jenkins node, so `agent any` effectively means: run on my main Jenkins executor.

The important thing I learned while rewriting this file is that **Declarative Pipeline looks flexible at first, but its structure is actually strict**. Some directives can appear only at the top level, some only inside `stages`, and some only inside `steps`. That structural strictness became very important later when I had to troubleshoot coverage publishing.

---

## Why I Removed the `tools {}` Block

The old file had:

```groovy
tools {
    maven 'maven3'
}
```

That tells Jenkins to resolve a Maven installation named `maven3` from **Manage Jenkins → Tools** and inject it into the job's `PATH`.

I removed that from the current pipeline because my setup is different now. I install Maven, JDK 21, Docker, Trivy, kubectl, Helm, Terraform, Ansible, and AWS CLI **system-wide on the Jenkins host OS** using my own bootstrap process. Because they already live on the system `PATH`, Jenkins shell steps can use them directly. There is no need for Jenkins to resolve them from its UI-managed tool registry.

So the design decision here is simple:

- **Old approach** — Jenkins manages tool installation paths.
- **Current approach** — the operating system manages tool installation paths; Jenkins just executes them.

That makes the Jenkinsfile cleaner and keeps server configuration more consistent with how I manage the rest of my infrastructure.

---

## Why I Removed `SCANNER_HOME = tool 'sonar-scanner'`

The old file had:

```groovy
environment {
    SCANNER_HOME = tool 'sonar-scanner'
}
```

and then later:

```groovy
sh '$SCANNER_HOME/bin/sonar-scanner'
```

That pattern is correct only when the pipeline uses the **standalone SonarQube Scanner CLI binary**.

In the current Jenkinsfile, I changed the Sonar integration strategy entirely. Instead of using the standalone scanner binary, I switched to the **Maven Sonar plugin**:

```groovy
withSonarQubeEnv('sonar-server') {
    withMaven(globalMavenSettingsConfig: 'maven-settings') {
        sh '''
            mvn sonar:sonar ...
        '''
    }
}
```

This is a better fit because the project is already Maven-based. Maven already knows the project structure, target classes, test lifecycle, and plugin ecosystem. Letting Maven drive Sonar analysis keeps the pipeline more coherent.

So:

- **Old file** → standalone scanner binary → needs `SCANNER_HOME`
- **Current file** → Maven plugin execution → does **not** need `SCANNER_HOME`

I also rely on `withSonarQubeEnv('sonar-server')` to inject the SonarQube server URL and token automatically, so there is no reason to manually resolve a scanner installation path anymore.

---

## Why the `environment {}` Block Became Larger

The old file only needed:

```groovy
environment {
    SCANNER_HOME = tool 'sonar-scanner'
    IMAGE_TAG    = "v${BUILD_NUMBER}"
}
```

The current pipeline interacts with many more systems, so I centralised all reusable values in one place:

```groovy
environment {
    APP_NAME       = 'bankapp'
    APP_VERSION    = '0.0.1-SNAPSHOT'
    GROUP_ID       = 'com.ibtisamiq'
    DOCKER_USER    = 'mibtisam'
    IMAGE_NAME     = "${DOCKER_USER}/${APP_NAME}"
    GHCR_USER      = 'ibtisam-iq'
    GHCR_IMAGE     = "ghcr.io/${GHCR_USER}/${APP_NAME}"
    NEXUS_URL      = 'https://nexus.ibtisam-iq.com'
    NEXUS_DOCKER   = 'nexus.ibtisam-iq.com'
    APP_DIR        = 'pipelines/java-monolith/app'
}
```

This design has two advantages:

1. **Single source of truth** — reusable values are defined once.
2. **Less hardcoding inside stages** — stage logic stays focused on actions, not repeated strings.

For example, if the Docker namespace or Nexus hostname changes, I only update the value once instead of hunting through multiple stages.

Notice that `IMAGE_TAG` is not defined statically here. That is intentional. It depends on a Git SHA, and a Git SHA only becomes available after checkout at runtime.

---

## Why I Added an `options {}` Block

The old file had no `options` block. The current one adds:

```groovy
options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    timeout(time: 45, unit: 'MINUTES')
    disableConcurrentBuilds()
    timestamps()
    ansiColor('xterm')
}
```

These are not build steps. They are **pipeline behavior controls**.

### Why each one exists

- `buildDiscarder(logRotator(numToKeepStr: '10'))`
  - Keeps only the last 10 builds.
  - Prevents Jenkins disk usage from growing forever.

- `timeout(time: 45, unit: 'MINUTES')`
  - Kills the pipeline if it hangs.
  - Protects the Jenkins executor from indefinitely stuck jobs.

- `disableConcurrentBuilds()`
  - Prevents overlapping runs of the same pipeline.
  - Important because both runs would otherwise try to push the same tags and edit the same CD repo.

- `timestamps()`
  - Adds timestamps to console logs.
  - Very useful during troubleshooting when I need to know exactly when a stage started or where the delay happened.

- `ansiColor('xterm')`
  - Preserves colored console output.
  - Makes Trivy and other CLI output much easier to read inside Jenkins.

This block is one of the places where the current Jenkinsfile moves from “it works” to “it behaves well on a real Jenkins server.”

---

## Why the Checkout Stage Changed Completely

### Old style

```groovy
git branch: 'main', url: 'https://github.com/ibtisam-iq/BankingApp-Java-MySQL.git'
```

That was fine when the application repository itself was the direct source being built.

### Current style

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

The repository model changed. The source code for the app now lives inside this pipeline repo as a **Git submodule** under:

```text
pipelines/java-monolith/app/
```

The short `git` step does not properly handle submodule initialization. That is why I switched to the full `checkout()` call with `SubmoduleOption`.

### Why each important line exists

| Directive | Why it exists |
|---|---|
| `$class: 'GitSCM'` | Gives access to the full Jenkins Git plugin configuration model. |
| `branches: [[name: '*/main']]` | Explicitly checks out `main` using Jenkins Git plugin syntax. |
| `disableSubmodules: false` | Ensures submodules are not skipped. |
| `parentCredentials: true` | Reuses parent repo credentials for submodules if needed. |
| `recursiveSubmodules: true` | Supports nested submodules if they ever appear later. |
| `trackingSubmodules: false` | Checks out the exact pinned submodule commit, which is what I want for reproducible builds. |
| `credentialsId: 'github-creds'` | Uses the stored Jenkins GitHub credential rather than anonymous clone. |

This stage is not longer because Jenkins suddenly became more complicated. It is longer because the repository architecture is more advanced now.

---

## Why `dir(APP_DIR)` Appears Everywhere

The old pipeline assumed the application lived at the workspace root.

That is no longer true. The actual source is now inside:

```text
pipelines/java-monolith/app/
```

So any stage that needs `pom.xml`, `Dockerfile`, source code, or Maven targets must first change into that directory:

```groovy
dir(APP_DIR) {
    sh 'mvn clean verify ...'
}
```

Without `dir(APP_DIR)`, Jenkins would execute `mvn`, `docker build`, and Trivy scans from the wrong directory. Maven would fail because there would be no `pom.xml` in the workspace root.

This one change — introducing `APP_DIR` and wrapping build-related steps in `dir(APP_DIR)` — is what made the pipeline compatible with the submodule-based project structure.

---

## Why Trivy Filesystem Scan Runs Before Build

The old file ran:

```groovy
trivy fs --format table -o fs-report.html .
```

It was simple, but it was also loosely defined.

In the new file, I moved Trivy filesystem scanning **earlier**, before build and test, and made it more explicit:

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

Then I run a second human-readable table scan for HIGH and MEDIUM findings.

### Why this is better

- **Fail-fast principle** — if source files contain hardcoded secrets, critical vulnerabilities, or obvious misconfigurations, I want to know before spending time on compilation and analysis.
- **Explicit scanners** — `secret,vuln,config` documents clearly what is being scanned.
- **Archivable report** — JSON output becomes a Jenkins artifact.
- **Cleaner logs** — `--no-progress` removes unnecessary spinner noise.

The old scan was functional. The new scan is more operationally useful.

---

## Why I Added a Separate Versioning Stage

The old image tag was simple:

```groovy
IMAGE_TAG = "v${BUILD_NUMBER}"
```

That produces values like:

```text
v12
v13
v14
```

Those tags are valid, but they are not traceable. They tell me nothing about which Git commit produced the image.

The current pipeline generates the image tag dynamically in a dedicated stage:

```groovy
script {
    def shortSha = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
    env.IMAGE_TAG = "${APP_VERSION}-${shortSha}-${BUILD_NUMBER}"
}
```

Example:

```text
0.0.1-SNAPSHOT-ab3f12c-42
```

Now the image tag carries:

- the application version
- the exact source commit
- the Jenkins build number

That is much better for traceability, debugging, rollback analysis, and cross-checking what actually got deployed.

I placed this in a dedicated `Versioning` stage because it is an important pipeline event in its own right: **turning source state into a uniquely identifiable artifact version**.

---

## Why Build and Test Became `mvn clean verify`

The old file split compilation and tests into two stages:

```groovy
stage('Compile') {
    steps { sh "mvn compile" }
}

stage('Testing') {
    steps { sh "mvn test" }
}
```

The current file merges these into one stronger lifecycle command:

```groovy
withMaven(globalMavenSettingsConfig: 'maven-settings') {
    sh 'mvn clean verify -B --no-transfer-progress'
}
```

### Why I made this change

- `clean verify` covers a broader, more CI-appropriate lifecycle.
- It ensures the workspace is cleaned before build.
- It runs compile, tests, packaging-related lifecycle phases, and verification in one flow.
- `withMaven(...)` injects the managed `settings.xml`, which is better than relying on ad hoc Maven environment assumptions.
- `-B` keeps Maven non-interactive.
- `--no-transfer-progress` keeps logs readable.

This is also where I intended Jenkins to publish **JUnit test results** and **JaCoCo coverage** — which later turned into an important troubleshooting lesson.

---

## Why SonarQube Analysis Uses `mvn sonar:sonar`

The older Jenkinsfile used the standalone scanner path:

```groovy
$SCANNER_HOME/bin/sonar-scanner
```

The current file uses:

```groovy
withSonarQubeEnv('sonar-server') {
    withMaven(globalMavenSettingsConfig: 'maven-settings') {
        sh '''
            mvn sonar:sonar \
                -Dsonar.projectKey=IbtisamIQbankapp \
                -Dsonar.projectName=IbtisamIQbankapp \
                -Dsonar.java.binaries=target/classes \
                -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                -B --no-transfer-progress
        '''
    }
}
```

### Why this is better for this project

- The project is Maven-native, so Sonar analysis naturally belongs in Maven.
- Coverage data from JaCoCo can be passed in directly using Maven build outputs.
- `withSonarQubeEnv` injects server URL and auth automatically.
- The pipeline stays consistent: build system remains Maven, not Maven + external scanner binary.

This also made the file easier to explain later: **one ecosystem, one toolchain path**.

---

## Why the Quality Gate Changed

Old version:

```groovy
timeout(time: 1, unit: 'HOURS') {
    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
}
```

Current version:

```groovy
timeout(time: 5, unit: 'MINUTES') {
    waitForQualityGate abortPipeline: true
}
```

### Why I changed it

- `abortPipeline: true` means the build stops if quality fails.
- I do not want to continue into packaging, image build, and registry push if static analysis says the code quality gate failed.
- `credentialsId` is no longer needed there because the authentication context is already handled by `withSonarQubeEnv` during analysis.
- 5 minutes is a more realistic timeout than 1 hour for a webhook round-trip.

The old version was permissive. The current one enforces policy.

---

## Why There Is No Separate `Package` Stage Anymore

The old file had:

```groovy
stage('Package') {
    steps {
        sh "mvn package"
    }
}
```

I removed that stage because `mvn clean verify` already goes far enough in the lifecycle that running a separate package stage becomes redundant.

Then later, `mvn deploy -DskipTests` handles deployment to Nexus. So packaging is not missing — it is simply no longer duplicated.

---

## Why `mvn deploy` Uses `-DskipTests`

```groovy
sh 'mvn deploy -DskipTests -B --no-transfer-progress'
```

Tests already ran earlier in the Build & Test stage. Running them a second time during deploy would waste time without adding value.

This is a classic CI optimization: **run tests once in the correct validation stage, then reuse the validated build output for publishing**.

---

## Why Docker Build Became More Explicit

Old version:

```groovy
docker build -t mibtisam/bankingapp-java-mysql:$IMAGE_TAG .
```

Current version:

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

### Why this is better

- `--build-arg SERVER_PORT=8000` passes build-time configuration cleanly.
- OCI labels add artifact metadata for traceability.
- Multiple tags in one build avoid duplicate builds for different registries.
- The image becomes easier to inspect and audit later.

This is another example of the same design philosophy: not just "build an image" — build an image that is **traceable and multi-registry ready**.

---

## Why Registry Pushes Are Split into Separate Stages

The old Jenkinsfile only pushed to one registry. The current file breaks pushes into independent stages:

- Docker Hub
- GHCR
- Nexus Docker Registry
- AWS ECR (prepared but commented out)

That improves observability.

If Docker Hub succeeds but GHCR fails, Jenkins shows exactly where the problem occurred. If everything were collapsed into one shell script, failure visibility would be worse.

This is a small design choice, but it makes troubleshooting and demo explanation much better.

---

## Why I Use `withCredentials(...)` + Explicit `docker login`

The old file used:

```groovy
withDockerRegistry(credentialsId: 'docker-cred') {
    sh "docker push ..."
}
```

The current file uses explicit login/logout flows with `withCredentials(...)`.

### Why I changed it

- `withDockerRegistry` is convenient, but it is higher-level and less explicit.
- GHCR and Nexus often behave more predictably when login is handled explicitly.
- The authentication flow becomes visible to the reader.
- Explicit `docker logout` helps prevent credential/session mixing between registries.

So this was a tradeoff in favor of clarity and control over shorthand convenience.

---

## Why AWS ECR Is Already Written but Commented Out

I wrote the AWS ECR integration stage in advance but left it commented because the AWS-side prerequisites are not yet provisioned in the lab.

This was intentional. I wanted the Jenkinsfile to document the **next evolution path** of the pipeline while keeping the currently active version runnable.

That means the file is both:

- executable for the current stack
- instructional for the upcoming stack extension

---

## Why the CD Repo Update Stage Exists

The CI pipeline does not stop at producing an image. It must also hand off deployment state to the CD system.

The current stage clones `platform-engineering-systems`, writes the new image tag into `java-monolith-image.env`, commits, and pushes.

Compared to the old file, this version is cleaner because it:

- removes stale repo state before cloning
- sets Git identity explicitly
- handles “nothing to commit” safely
- adds `[skip ci]` to prevent recursive triggering
- prepares the path for future manifest patching with `sed`

This stage makes the CI → CD boundary explicit, which is important both architecturally and for explanation during interviews.

---

## The Troubleshooting I Had To Do While Writing the Current Jenkinsfile

This section is the most important update compared to the earlier version of this document.

When I first wrote the production-style Jenkinsfile, I correctly improved many things structurally, but I also introduced a **Jenkins DSL / plugin compatibility mistake** in the coverage publishing part.

That mistake taught me an important lesson:

> Writing a pipeline is not only about knowing shell commands and stages. It is also about understanding which Jenkins directives are valid in which scope, and which plugin owns which DSL.

---

## Problem 1 — I Put Coverage Publishing in the Wrong Scope

### What I originally wrote

Inside `stage('Build & Test')`, I added a stage-level post block like this:

```groovy
stage('Build & Test') {
    steps {
        dir(APP_DIR) {
            withMaven(globalMavenSettingsConfig: 'maven-settings') {
                sh 'mvn clean verify -B --no-transfer-progress'
            }
        }
    }
    post {
        always {
            junit testResults: "${APP_DIR}/target/surefire-reports/*.xml",
                  allowEmptyResults: true
            jacoco(
                execPattern:    "${APP_DIR}/target/jacoco.exec",
                classPattern:   "${APP_DIR}/target/classes",
                sourcePattern:  "${APP_DIR}/src/main/java",
                exclusionPattern: '**/*Test*'
            )
        }
    }
}
```

At first glance, this felt logical:

- tests run inside the Build & Test stage
- so publish test results and coverage inside that stage's `post`

That is a natural assumption, but it turned out to be structurally wrong.

### Why it was wrong

In Declarative Pipeline, a stage-level `post {}` block supports normal steps, but publisher-style integrations like coverage reporting are not always valid there in the same way they are at the top-level pipeline `post` block.

The important practical issue I discovered is this:

- `junit(...)` and especially coverage publishers behave as **post-build publishing actions**
- they belong in the **pipeline-level `post { always { ... } }`** area for safe, consistent execution
- placing them inside the stage-level `post` can lead to unsupported DSL behavior depending on the step/plugin combination

So although the intention was good, the location was wrong.

### How I corrected it

I removed the entire `post {}` block from inside `stage('Build & Test')` and moved test publishing + coverage publishing into the top-level `post { always { ... } }` section of the pipeline.

That keeps the stage focused on **producing outputs**, and the top-level `post` focused on **publishing results and cleaning up**.

---

## Problem 2 — I Used the Wrong Coverage DSL

### What I originally used

I used:

```groovy
jacoco(
    execPattern:    "${APP_DIR}/target/jacoco.exec",
    classPattern:   "${APP_DIR}/target/classes",
    sourcePattern:  "${APP_DIR}/src/main/java",
    exclusionPattern: '**/*Test*'
)
```

That syntax belongs to the older **JaCoCo Jenkins plugin**.

### Why that became a problem

My Jenkins setup uses the newer **Coverage Plugin**, whose DSL is different. The modern plugin expects:

```groovy
recordCoverage(...)
```

not `jacoco(...)`.

So this was not just a placement problem. It was also a **plugin ownership problem**.

I had effectively mixed:

- current Jenkins pipeline structure
- old coverage publishing syntax

That mismatch is exactly the kind of issue that happens when you understand the build toolchain well but still need to be careful about Jenkins plugin evolution.

### How I corrected it

I replaced the old `jacoco(...)` call with:

```groovy
recordCoverage(
    tools: [[
        parser: 'JACOCO',
        pattern: "${APP_DIR}/target/jacoco.exec"
    ]],
    sourceCodeRetention: 'EVERY_BUILD'
)
```

Now the coverage publishing step matches the plugin actually installed in Jenkins.

This is the correct mental model:

- **JaCoCo Maven plugin** generates the coverage data file
- **Jenkins Coverage Plugin** reads and publishes that file

Those are related, but they are not the same thing.

---

## Problem 3 — ECR Cleanup Could Become a Future Trap

This was not an active runtime bug yet, but it was a hidden future hazard I identified while reviewing the file.

### The situation

In the cleanup section, I had commented-out ECR image removal lines:

```groovy
# docker rmi ${ECR_IMAGE}:${IMAGE_TAG} || true
# docker rmi ${ECR_IMAGE}:latest || true
```

At the same time, the ECR variables in `environment {}` were also commented out.

### Why this matters

As long as those cleanup lines remain commented, there is no problem.

But if someone later uncommented only the cleanup lines without also uncommenting and defining the ECR environment variables, Jenkins would fail with a missing property error because `ECR_IMAGE` would not exist.

### How I corrected it

I kept those lines commented but added an explicit note in the file that they must only be uncommented **together with** the ECR environment variable block.

So this was a preventive fix — not a reactive one.

---

## What the Corrected Top-Level `post {}` Block Looks Like Now

After troubleshooting and correction, the logic now lives in the proper top-level `post { always { ... } }` block:

```groovy
post {
    always {
        junit testResults: "${APP_DIR}/target/surefire-reports/*.xml",
              allowEmptyResults: true

        recordCoverage(
            tools: [[
                parser: 'JACOCO',
                pattern: "${APP_DIR}/target/jacoco.exec"
            ]],
            sourceCodeRetention: 'EVERY_BUILD'
        )

        sh """
            docker rmi ${IMAGE_NAME}:${IMAGE_TAG}               || true
            docker rmi ${IMAGE_NAME}:latest                     || true
            docker rmi ${GHCR_IMAGE}:${IMAGE_TAG}               || true
            docker rmi ${GHCR_IMAGE}:latest                     || true
            docker rmi ${NEXUS_DOCKER}/${APP_NAME}:${IMAGE_TAG} || true
            docker rmi ${NEXUS_DOCKER}/${APP_NAME}:latest       || true
        """

        sh 'rm -rf cd-repo'
        cleanWs()
    }
}
```

This is structurally cleaner because the top-level `post` block now handles three categories of end-of-run behavior in one place:

1. **publish reports**
2. **clean up temporary/built artifacts**
3. **reset the workspace**

That separation is much more correct than trying to publish results inside one stage and clean up elsewhere.

---

## Why the `jacoco.exec` Path Is Correct

While troubleshooting coverage, I also had to verify whether this path was correct:

```groovy
"${APP_DIR}/target/jacoco.exec"
```

At first, this can look suspicious because Maven runs inside:

```groovy
dir(APP_DIR) {
    sh 'mvn clean verify ...'
}
```

So the question is: if Maven runs inside `APP_DIR`, does Jenkins still need the full path including `APP_DIR` when publishing coverage?

The answer is yes.

### Why

`recordCoverage(...)` runs from Jenkins pipeline context, not from inside the earlier `dir(APP_DIR)` shell context. So Jenkins resolves the coverage pattern relative to the **workspace root**, not relative to the shell's working directory used earlier.

That means this path is correct:

```text
$WORKSPACE/pipelines/java-monolith/app/target/jacoco.exec
```

So I did **not** need to change the pattern itself.

The bug was not the path. The bug was:

- wrong DSL (`jacoco` instead of `recordCoverage`)
- wrong placement (stage-level post instead of top-level post)

This distinction mattered during troubleshooting because it stopped me from “fixing” the wrong thing.

---

## What I Actually Troubleshooted

The real troubleshooting sequence was not random. It followed a clear logic:

### Step 1 — Review the intent

I first checked what I was trying to achieve:

- run Maven tests
- generate JaCoCo coverage
- publish test reports in Jenkins
- publish coverage in Jenkins

That part was conceptually valid.

### Step 2 — Review plugin ownership

Then I checked which plugin owns which DSL:

- JaCoCo data generation happens in Maven
- Jenkins coverage visualization depends on the Jenkins plugin installed
- my Jenkins instance uses the Coverage Plugin

That immediately made `jacoco(...)` suspicious.

### Step 3 — Review Declarative Pipeline scope rules

Then I checked where publishers should live in Declarative Pipeline.

That exposed the second issue: even if the syntax were correct, the stage-level `post` placement was not the safest or most structurally correct place for coverage publishing.

### Step 4 — Verify the file path assumption

Before changing the path, I verified what `${APP_DIR}/target/jacoco.exec` resolves to from the Jenkins workspace. That turned out to be correct, so I left it unchanged.

### Step 5 — Apply the smallest correct fix

The final correction was:

- delete the stage-level `post`
- move `junit(...)` to top-level `post { always }`
- replace `jacoco(...)` with `recordCoverage(...)`
- keep ECR cleanup commented with an explanatory note

That was the minimal correction that fixed the real problem without introducing unnecessary changes.

---

## What This Taught Me

This troubleshooting exercise reinforced several practical lessons:

### 1. Jenkins knowledge is not just shell knowledge

A pipeline can have perfectly valid shell commands and still be structurally wrong from Jenkins' point of view. Jenkins has its own DSL rules, scope rules, and plugin-owned steps.

### 2. Plugins matter as much as syntax

Two steps may look related — `jacoco(...)` and `recordCoverage(...)` — but belong to different plugin generations. Knowing which plugin is actually installed matters.

### 3. Scope matters in Declarative Pipeline

Where I place a step is just as important as what the step does. A valid-looking directive in the wrong scope can still be incorrect.

### 4. Do not “fix” the path unless the path is actually wrong

It would have been easy to start changing `${APP_DIR}/target/jacoco.exec`, but that would have been solving the wrong problem. The path was fine. The DSL and placement were not.

### 5. Production-grade pipelines are built iteratively

A strong Jenkinsfile is usually not written perfectly in one attempt. It improves as I validate syntax, understand plugin behavior better, and tighten structure based on real troubleshooting.

---

## Why the Current Jenkinsfile Is Better Than the Old One

The current file is better not because it is longer, but because it is more deliberate.

It now has:

- correct submodule-aware checkout
- centralised environment values
- operational pipeline controls via `options {}`
- fail-fast Trivy scanning
- traceable image tagging
- Maven-native SonarQube analysis
- hard quality gate enforcement
- multi-registry image publishing
- CI → CD handoff through a separate repo
- proper top-level result publishing and cleanup
- corrected coverage DSL and corrected publisher placement

In other words, it does not just “run a build.” It behaves like a real CI pipeline that I can explain, debug, maintain, and extend.

---

## Related Documentation

| Topic | File |
|---|---|
| Full CI/CD stack setup | [cicd-stack-setup.md](../../../docs/cicd-stack-setup.md) |
| Main pipeline walkthrough | [../README.md](../README.md) |
| Tool configuration rationale | [tool-configuration.md](https://github.com/ibtisam-iq/nectar/blob/main/delivery/jenkins/tool-configuration.md) |
| Credential types and injection | [credentials.md](https://github.com/ibtisam-iq/nectar/blob/main/delivery/jenkins/credentials.md) |
| SonarQube ↔ Jenkins integration | [sonar-jenkins.md](https://github.com/ibtisam-iq/nectar/blob/main/delivery/sonarqube/sonar-jenkins.md) |
| Nexus ↔ Jenkins integration | [nexus-jenkins.md](https://github.com/ibtisam-iq/nectar/blob/main/delivery/nexus/nexus-jenkins.md) |
