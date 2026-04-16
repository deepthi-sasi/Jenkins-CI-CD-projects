# CI Pipeline — Freestyle, Pipeline & Multibranch Pipeline

> Build and publish a Java Maven application to a private DockerHub registry using three Jenkins job types — Freestyle, Pipeline, and Multibranch Pipeline.

![Jenkins](https://img.shields.io/badge/Jenkins-LTS-D24939?logo=jenkins&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![Java](https://img.shields.io/badge/Java-Maven-C71A36?logo=apachemaven&logoColor=white)
![Git](https://img.shields.io/badge/Git-GitHub-181717?logo=github&logoColor=white)

---

## Overview

This project demonstrates three different ways to create a CI pipeline in Jenkins for the same Java Maven application. Each job type builds the app, packages it as a Docker image, and pushes it to a private DockerHub registry.

**Technologies used:** Jenkins · Docker · Linux · Git · Java · Maven

### What this project covers

1. Install Maven and Node.js in Jenkins
2. Mount Docker from the host into the Jenkins container
3. Create GitHub credentials in Jenkins
4. Create a **Freestyle** job — configure build steps via the Jenkins UI
5. Create a **Pipeline** job — define the pipeline in a `Jenkinsfile`
6. Create a **Multibranch Pipeline** job — auto-detect branches and run per-branch pipelines


### Job type comparison

| | Freestyle | Pipeline | Multibranch Pipeline |
|---|---|---|---|
| Config location | Jenkins UI | `Jenkinsfile` in repo | `Jenkinsfile` in repo |
| Version controlled | No | Yes | Yes |
| Multi-branch support | No | No | Yes |
| Best for | Quick one-off jobs | Single-branch pipelines | Feature branch workflows |

---

## Prerequisites

- Jenkins running as a Docker container on a Linux host
- A GitHub account with a Java Maven application repository
- A private DockerHub repository

---

## Step 1 — Install Maven and Node.js in Jenkins

### Install Maven

Maven is supported via a built-in plugin — no extra install needed, just configure it:

1. Go to **Manage Jenkins** → **Global Tool Configuration**
2. Scroll to the **Maven** section and press **Add Maven**
3. Enter a name (e.g. `maven-3.9`) and press **Save**

Maven is now available by name in all Jenkins jobs.

### Install Node.js and npm

Node must be installed directly inside the Jenkins container:

```bash
# Enter the Jenkins container as root
docker exec -u 0 -it <container-id> bash

# Install curl
apt update
apt install curl

# Check your Linux distribution
cat /etc/issue
# e.g. Debian GNU/Linux 11

# Download and run the Node.js 18 setup script
curl -sL https://deb.nodesource.com/setup_18.x -o nodesource_setup.sh
bash nodesource_setup.sh

# Install Node.js and npm
apt-get install -y nodejs

# Verify
node --version   # v18.15.0
npm --version    # 9.5.0

exit
```

---

## Step 2 — Make Docker available on the Jenkins server

Rather than installing Docker inside the container, mount the host's Docker runtime as a volume.

### Restart Jenkins with Docker socket mounted

```bash
docker run -p 8080:8080 -p 50000:50000 -d \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts
```

### Fix Docker socket permissions

The `jenkins` user lacks write access to `/var/run/docker.sock` by default:

```bash
# Enter the container as root
docker exec -u 0 -it <container-id> bash

# Grant permissions
chmod 666 /var/run/docker.sock
exit

# Verify jenkins user can run Docker commands
docker exec -it <container-id> bash
docker pull hello-world
exit
```

### Troubleshooting: GLIBC version error

If you see an error like:
```
docker: /lib/x86_64-linux-gnu/libc.so.6: version 'GLIBC_2.32' not found
```

Install Docker inside the container instead of mounting the binary:

```bash
docker run -p 8080:8080 -p 50000:50000 -d \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

docker exec -u 0 -it <container-id> bash
apt update
apt install docker.io
exit

# Verify
docker exec -it <container-id> bash
docker pull hello-world
exit
```

---

## Step 3 — Create Jenkins credentials

### GitHub access token

GitHub no longer supports password authentication. Create a personal access token:

1. GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)** → **Generate new token (classic)**
2. Enter a name (e.g. `devops-bootcamp-jenkins`), set an expiry, select the **repo** scope
3. Press **Generate token** and copy it immediately

### Add GitHub credentials to Jenkins

1. **Manage Jenkins** → **Manage Credentials** → **System** → **Global credentials** → **Add credentials**
2. Configure:
  - **Kind:** `Username with password`
  - **Username:** your GitHub username
  - **Password:** the personal access token
  - **ID:** `GitHub`
3. Press **Create**

### Add DockerHub credentials to Jenkins

1. **Manage Jenkins** → **Manage Credentials** → **Global credentials** → **Add credentials**
2. Configure:
  - **Kind:** `Username with password`
  - **Username:** your DockerHub username
  - **Password:** your DockerHub password
  - **ID:** `DockerHub`
3. Press **Create**

---

## Freestyle Job

The Freestyle job configures all build steps through the Jenkins UI — no `Jenkinsfile` required.

### Create the job

**Dashboard** → **New Item** → enter name `devops-bootcamp-freestyle` → select **Freestyle project** → **OK**

### Connect to the Git repository

In **Source Code Management**, select **Git**, enter your repository URL and choose the `GitHub` credentials. Set branch to `*/main`.

### Build the JAR

In **Build Steps**, add **Invoke top-level Maven targets**, select `maven-3.9`, set goal to `package`.

### Add the Dockerfile

Add the following `Dockerfile` to your project root and push it to the repository:

```dockerfile
FROM amazoncorretto:17-alpine-jdk

EXPOSE 8080

COPY ./build/libs/java-app-1.0-SNAPSHOT.jar /usr/app
WORKDIR /usr/app

ENTRYPOINT ["java", "-jar", "java-app-1.0-SNAPSHOT.jar"]
```

### Build and push the Docker image

1. In **Build Environment**, enable **Use secret text(s) or file(s)**
2. Add a **Username and password (separated)** binding with variable names `USERNAME` and `PASSWORD`, linked to the `DockerHub` credential
3. In **Build Steps**, add an **Execute shell** step:

```bash
docker build -t deepthisasi/demo-app:1.1 .
echo $PASSWORD | docker login -u $USERNAME --password-stdin
docker push deepthisasi/demo-app:1.1
```

Press **Save** and run the build via **Build Now**. Verify the image appears in your DockerHub repository.

---

## Pipeline Job

The Pipeline job reads its build definition from a `Jenkinsfile` stored in the Git repository — fully version controlled.

### Create the job

**Dashboard** → **New Item** → enter name `devops-bootcamp-pipeline` → select **Pipeline** → **OK**

### Connect to the Git repository

In the **Pipeline** section, select **Pipeline script from SCM** → **Git**. Enter your repository URL, choose the `GitHub` credential, and set branch to `*/main`.

### Jenkinsfile

Add the following `Jenkinsfile` to your project root and push it to the repository:

```groovy
pipeline {
    agent any
    tools {
        maven 'maven-3.9'
    }
    stages {
        stage('Build Application JAR') {
            steps {
                script {
                    echo 'building the application...'
                    sh 'mvn package'
                }
            }
        }
        stage('Build and Publish Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'DockerHub',
                        usernameVariable: 'USERNAME',
                        passwordVariable: 'PASSWORD'
                    )]) {
                        echo 'building the docker image...'
                        sh 'docker build -t deepthisasi/demo-app:javamavenapp1.1 .'

                        echo 'publishing the docker image...'
                        sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
                        sh 'docker push deepthisasi/demo-app:javamavenapp1.1'
                    }
                }
            }
        }
    }
}
```

Run the build via **Build Now** and verify the image in DockerHub.

---

## Multibranch Pipeline Job

The Multibranch Pipeline automatically discovers all branches in the repository that contain a `Jenkinsfile` and creates a separate pipeline for each.

### Create the job

**Dashboard** → **New Item** → enter name `my-multibranch-pipeline` → select **Multibranch Pipeline** → **OK**

### Connect to the Git repository

1. In **Branch Sources**, press **Add source** → **Git**
2. Enter your repository URL and choose the `GitHub` credential
3. Under **Discover branches**, press **Add** → **Filter by name (with regular expression)**
4. Enter `.*` to match all branches
5. Press **Save**

Jenkins will immediately scan the repository, detect all branches containing a `Jenkinsfile`, create a pipeline per branch, and trigger the first builds automatically.

> No additional steps are needed — the `Dockerfile` and `Jenkinsfile` already added for the Pipeline job are reused here.

---

## Project structure

```
.
├── Jenkinsfile         # Pipeline definition (used by Pipeline and Multibranch jobs)
├── Dockerfile          # Docker image build instructions
├── pom.xml             # Maven build configuration
└── src/                # Java application source code
```

---

## Potential improvements

- **Parameterise the image tag** — replace hardcoded tags like `1.1` with a dynamic value derived from the Maven version or Git commit SHA
- **Add a test stage** — insert `mvn test` between the build and publish stages to catch failures before pushing an image
- **Use a shared library** — extract common pipeline logic (Docker login, build, push) into a Jenkins shared library to avoid duplication across jobs
- **Scan on push** — configure the multibranch pipeline to trigger on webhook events rather than polling, for faster feedback
- **Branch-specific behaviour** — use `when { branch 'main' }` conditions in the Jenkinsfile to restrict the publish step to the main branch only

---

## References

- [Jenkins Pipeline documentation](https://www.jenkins.io/doc/book/pipeline/)
- [Jenkins Multibranch Pipeline](https://www.jenkins.io/doc/book/pipeline/multibranch/)
- [Jenkins credentials plugin](https://www.jenkins.io/doc/book/using/using-credentials/)
- [GitHub personal access tokens](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)
- [Docker socket binding in Jenkins](https://www.jenkins.io/doc/book/installing/docker/)
