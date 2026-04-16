# Dynamic Application Version Bump in Jenkins Pipeline

> Automatically increment the application version on every CI build, tag the Docker image with the new version, and commit the version bump back to Git — without triggering an infinite build loop.

![Jenkins](https://img.shields.io/badge/Jenkins-LTS-D24939?logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Java](https://img.shields.io/badge/Java-Maven-C71A36?logo=apachemaven&logoColor=white)
![Git](https://img.shields.io/badge/Git-GitHub-181717?logo=github&logoColor=white)

---

## Overview

This project extends the CI pipeline for a Java Maven application to include fully automated version management. On every build, Jenkins increments the patch version in `pom.xml`, uses the new version to tag the Docker image, pushes the image to DockerHub, and commits the version bump back to the Git repository — all without triggering an unintended second build.

**Technologies used:** Jenkins · Docker · GitHub · Git · Java · Maven

### Pipeline stages

| Stage | Description |
|---|---|
| Increment version | Bump the patch version in `pom.xml` and set `IMAGE_TAG` |
| Build application JAR | Clean and compile the Maven artifact |
| Build & push Docker image | Build image with dynamic tag and push to DockerHub |
| Commit version update | Push the `pom.xml` version bump back to Git |

### How dynamic versioning works

```
pom.xml version: 1.2.5
         │
         ▼
mvn build-helper:parse-version versions:set
         │
         ├── majorVersion    = 1  (unchanged)
         ├── minorVersion    = 2  (unchanged)
         └── nextIncrementalVersion = 6  (patch + 1)
         │
         ▼
pom.xml version: 1.2.6
IMAGE_TAG = 1.2.6-<BUILD_NUMBER>
Docker image: deepthisasi/demo-app:1.2.6-42
```

---

## Prerequisites

- Jenkins running with a configured multibranch or standard pipeline
- Maven configured in Jenkins as `maven-3.9` (see the [CI pipeline project](./README-jenkins-ci-pipeline.md))
- Docker available in the Jenkins container
- Jenkins credentials:
    - `DockerHub` — username + password for DockerHub
    - `GitHub` — username + personal access token for GitHub

---

## Step 1 — Increment the patch version

Add an `Increment Version` stage at the top of your pipeline stages in the `Jenkinsfile`. This stage uses the Maven `build-helper` and `versions` plugins to read the current version, increment the patch number, and write it back to `pom.xml`.

It also reads the new version back from `pom.xml` and sets it as an environment variable for use in later stages:

```groovy
stage('Increment Version') {
    steps {
        script {
            echo 'incrementing the patch version of the application...'
            sh 'mvn build-helper:parse-version versions:set \
                -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                versions:commit'

            def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
            def version = matcher[0][1]
            env.IMAGE_TAG = "$version-$BUILD_NUMBER"
        }
    }
}
```

### What each Maven goal does

| Goal | Description |
|---|---|
| `build-helper:parse-version` | Reads the current version from `pom.xml` and splits it into `majorVersion`, `minorVersion`, `incrementalVersion` |
| `versions:set -DnewVersion=...` | Writes the new version back to `pom.xml` |
| `versions:commit` | Saves the change permanently (removes the `pom.xml.versionsBackup` file) |

### How `IMAGE_TAG` is set

```groovy
def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
def version = matcher[0][1]
env.IMAGE_TAG = "$version-$BUILD_NUMBER"
```

- `readFile('pom.xml')` — reads the full file content as a string
- `=~ '<version>(.+)</version>'` — applies a regex to find the version tag
- `matcher[0][1]` — `[0]` is the first match, `[1]` is the captured group (the version string)
- `$BUILD_NUMBER` — a built-in Jenkins environment variable; appending it makes every image tag unique even if the version hasn't changed

---

## Step 2 — Build the application JAR

Replace `mvn package` with `mvn clean package` to ensure only one JAR exists in the `target/` folder at a time. This is required because the Dockerfile uses a wildcard to select the JAR, and multiple versions would cause ambiguity:

```groovy
stage('Build Application JAR') {
    steps {
        script {
            echo 'building the application...'
            sh 'mvn clean package'
        }
    }
}
```

---

## Step 3 — Build and push the Docker image with a dynamic tag

Use `${IMAGE_TAG}` (set in Step 1) instead of a hardcoded version:

```groovy
stage('Build and Publish Docker Image') {
    steps {
        script {
            withCredentials([usernamePassword(
                credentialsId: 'DockerHub',
                usernameVariable: 'USER',
                passwordVariable: 'PASS'
            )]) {
                echo 'building the docker image...'
                sh "docker build -t deepthisasi/demo-app:${IMAGE_TAG} ."

                echo 'publishing the docker image...'
                sh 'echo $PASS | docker login -u $USER --password-stdin'
                sh "docker push deepthisasi/demo-app:${IMAGE_TAG}"
            }
        }
    }
}
```

### Update the Dockerfile

Because the JAR filename now changes with every version, update the `Dockerfile` to use a wildcard instead of a hardcoded version:

```dockerfile
FROM amazoncorretto:17-alpine-jdk

EXPOSE 8080

COPY ./target/java-maven-app-*.jar /opt/bootcamp-java-maven-app/
CMD java -jar /opt/bootcamp-java-maven-app/java-maven-app-*.jar
```

> Using `mvn clean package` (Step 2) guarantees only one JAR exists in `target/`, so the wildcard always resolves to exactly one file.

---

## Step 4 — Commit the version update back to Git

Add a `Commit Version Update` stage after the image push. Placing it last ensures the commit only happens if the build and publish both succeeded:

```groovy
stage('Commit Version Update') {
    steps {
        script {
            withCredentials([usernamePassword(
                credentialsId: 'GitHub',
                usernameVariable: 'USER',
                passwordVariable: 'PASS'
            )]) {
                sh "git remote set-url origin https://${USER}:${PASS}@github.com/deepthi-sasi/java-maven-app.git"
                sh 'git add .'
                sh 'git commit -m "jenkins: version bump"'
                sh 'git push origin HEAD:main'
            }
        }
    }
}
```

> **Why `git push origin HEAD:main`?** Jenkins checks out a specific commit (detached HEAD), not a branch. Using `HEAD:main` explicitly tells Git to push the current commit to the `main` branch, which avoids the `error: src refspec main does not match any` error you'd get with `git push origin main`.

### Configure Git user identity in Jenkins

Git requires a user identity to make commits. Set it once inside the Jenkins container:

```bash
docker exec -it <container-id> bash
git config --global user.email "jenkins@example.com"
git config --global user.name "jenkins"
exit
```

---

## Step 5 — Prevent an infinite build loop

The version bump commit from Step 4 will trigger another pipeline build, which would trigger another commit, and so on. Prevent this by ignoring commits made by the Jenkins user.

### For Multibranch Pipeline jobs

1. Install the **Ignore Committer Strategy** plugin: **Manage Jenkins** → **Manage Plugins** → search for `Ignore Committer Strategy` → **Install without restart**
2. Open the multibranch pipeline configuration → **Branch Sources** → **Git** → under **Build strategies**, press **Add** → **Ignore Committer Strategy**
3. Enter `jenkins@example.com` (the email configured in Step 4)
4. Ensure **Allow builds when a changeset contains non-ignored author(s)** is checked

### For standard Pipeline jobs

No plugin needed:

1. Open the pipeline configuration → scroll to **Additional Behaviours** in the Git section
2. Press **Add** → **Polling ignores commits from certain users**
3. Enter `jenkins` (the username configured in Step 4)

---

## Complete Jenkinsfile

```groovy
#!/usr/bin/env groovy

pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    stages {
        stage('Increment Version') {
            steps {
                script {
                    echo 'incrementing the patch version of the application...'
                    sh 'mvn build-helper:parse-version versions:set \
                        -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                        versions:commit'

                    def matcher = readFile('pom.xml') =~ '<version>(.+)</version>'
                    def version = matcher[0][1]
                    env.IMAGE_TAG = "$version-$BUILD_NUMBER"
                }
            }
        }
        stage('Build Application JAR') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn clean package'
                }
            }
        }
        stage('Build and Publish Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'DockerHub',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )]) {
                        echo 'building the docker image...'
                        sh "docker build -t deepthisasi/demo-app:${IMAGE_TAG} ."
                        echo 'publishing the docker image...'
                        sh 'echo $PASS | docker login -u $USER --password-stdin'
                        sh "docker push deepthisasi/demo-app:${IMAGE_TAG}"
                    }
                }
            }
        }
        stage('Commit Version Update') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'GitHub',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASS'
                    )]) {
                        sh "git remote set-url origin https://${USER}:${PASS}@github.com/deepthi-sasi/java-maven-app.git"
                        sh 'git add .'
                        sh 'git commit -m "jenkins: version bump"'
                        sh 'git push origin HEAD:main'
                    }
                }
            }
        }
    }
}
```

---

## Potential improvements

- **Skip version increment on hotfix branches** — use a `when` condition to only run the increment stage on `main` or `release/*` branches
- **Use semantic versioning more granularly** — expose `major` and `minor` bumps as pipeline parameters for more control
- **Tag the Git commit** — add `git tag v${IMAGE_TAG}` and `git push origin --tags` alongside the version commit for full release traceability
- **Store `IMAGE_TAG` as a build artifact** — write it to a file and archive it so downstream jobs can reference the exact image that was built
- **Use a shared library** — extract the version increment logic into a Jenkins shared library function so it can be reused across multiple pipelines

---

## References

- [Maven Versions Plugin](https://www.mojohaus.org/versions/versions-maven-plugin/)
- [Maven Build Helper Plugin](https://www.mojohaus.org/build-helper-maven-plugin/)
- [Ignore Committer Strategy Jenkins plugin](https://plugins.jenkins.io/ignore-committer-strategy/)
- [Jenkins built-in environment variables](https://www.jenkins.io/doc/book/pipeline/jenkinsfile/#using-environment-variables)
- [CI Pipeline project](./README-jenkins-ci-pipeline.md) — prerequisite Jenkins setup
