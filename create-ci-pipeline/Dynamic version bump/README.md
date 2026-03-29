## Demo Project - Dynamic Application Version Bump


### Topics of the Demo Project
### Dynamically increment application version in Jenkins pipeline

### Technologies Used

* Jenkins
* Docker 
* GitLab / GitHub 
* Git 
* Java 
* Maven

### Project Description
* Configure CI step: Increment patch version
* Configure CI step: Build Java application and clean old artifacts
* Configure CI step: Build Image with dynamic Docker Image Tag 
* Configure CI step: Push Image to private DockerHub repository 
* Configure CI step: Commit version update of Jenkins back to Git repository 
* Configure Jenkins pipeline to not trigger automatically on CI build commit to avoid commit loop

### Steps to Increment Patch Version
In the Jenkinsfile of the Java-Maven-App add a stage before the build stage, that increments the version (on the branches using a shared library, add the logic to the shared library):

    stages {
        stage("Increment Version") {
            steps {
                script {
                    echo 'incrementing the patch version of the application...'
                    sh 'mvn build-helper:parse-version versions:set \
                    -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.\\\${parsedVersion.nextIncrementalVersion} \
                    versions:commit'
                }
            }
        }
        stage("Build Application JAR") {
        ...
        }
        stage("Build and Publish Docker Image") {
        ...
        }
    }
build-helper:parse-version - Reads the current version from pom.xml and breaks it into parts (major.minor.patch)

If your version is 1.2.5, it parses: majorVersion=1, minorVersion=2, incrementalVersion=5
versions:set - Sets a new version in the pom.xml

-DnewVersion= tells it what the new version should be
${parsedVersion.majorVersion} = keeps major version (1)
${parsedVersion.minorVersion} = keeps minor version (2)
${parsedVersion.nextIncrementalVersion} = increments the patch/bugfix number (5 → 6)
Result: 1.2.5 becomes 1.2.6

versions:commit - Saves the change permanently to pom.xml


### Steps to Build Java Application and Clean Old Artifacts

Replace the command mvn package in the "Build Application JAR" stage with mvn clean package:

        stages {
            stage("Increment Version") {
            ...
            }
            stage("Build Application JAR") {
                steps {
                    script {
                        echo "building the application..."
                        sh 'mvn clean package'
                    }
                }
            }
            stage("Build and Publish Docker Image") {
            ...
            }

        }
This step is necessary to make sure we always have just one application jar version in the target folder, which makes it easier to select that jar in the Dockerfile to copy it into the image.

### Steps to Build Image with Dynamic Docker Image Tag and Push it to Private DockerHub Repository

Step 1: Set an environment variable holding the dynamic image tag

We can use the new application version for tagging the Docker image created later in the pipeline. To do this, we extend the script in the "Increment Version" stage as follows:
```
        stage("Increment Version") {
            steps {
                script {
                    echo 'incrementing the bugfix version of the application...'
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
This reads the pom.xml file and uses a regex pattern to find the version:

readFile('pom.xml') - Reads the entire pom.xml file as text
=~ '<version>(.+)</version>' - Regex that looks for <version>1.2.6</version> tags
(.+) - Captures whatever is between the tags (the actual version number)
matcher[0][1] - Gets the captured version string

[0] = first match found
[1] = the captured group (what's inside the parentheses in the regex)

So if the pom.xml has <version>1.2.6</version>, the variable version now contains "1.2.6"
We also append the current build number to the application version. $BUILD_NUMBER is an environment varibale provided by Jenkins.

Step2: Use image tag
In the "Build and Publish Docker Image" stage we replace the hardcoded image version with ${IMAGE_TAG}:

        stage("Build and Publish Docker Image") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                        echo "building the docker image..."
                        sh "docker build -t deepthisasi/demo-app:${IMAGE_TAG} ."
            
                        echo "publishing the docker image..."
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                        sh "docker push deepthisasi/demo-app:${IMAGE_TAG}"
                    }
                }
            }
        }
Step 3: Adjust Dockerfile

In the Dockerfile we use the hardcoded JAR version 1.0.0 when copying the JAR into the image and defining the command to start the application:

```
    COPY ./target/java-maven-app-1.0.0.jar /opt/bootcamp-java-maven-app
    CMD ["java", "-jar", "/opt/bootcamp-java-maven-app/java-maven-app-1.0.0.jar"]
```
This won't work anymore since Jenkins is incrementing the version in every build. Because we cleaned the target folder in every build, we can easily select the JAR file without knowing its version as follows:
```
    COPY ./target/java-maven-app-*.jar /opt/bootcamp-java-maven-app
    CMD java -jar /opt/bootcamp-java-maven-app/java-maven-app-*.jar
```
### Steps to Commit Version Update of Jenkins Back to Git Repository

Step 1: Add a new stage "Commit Version Update" to the Jenkinsfile

We add the new stage after the the "Build and Publish Docker Image" stage because we don't want to commit the version bump if something went wrong while building or publishing the image.

    stage("Build and Publish Docker Image") {
        ...
    }
    stage('Commit Version Update') {
        steps {
            script {
                withCredentials([usernamePassword(credentialsId: 'GitHub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "git remote set-url origin https://${USER}:${PASS}@github.com/deepthi-sasi/java-maven-app.git"
                    sh 'git add .'
                    sh 'git commit -m "jenkins: version bump'
                    sh 'git push origin HEAD:main'
                }
            }
        }
    }
We have to use git push origin HEAD:main (<src>:<dest>) instead of git push origin main or just git push because Jenkins does not check out a branch but a commit.

Step 2: Configure username and email for Git
To prevent Git from complaining (when doing a commit) that no author's email has been configured, we have to ssh into the Jenkins host server and execute the following commands:
```
docker exec -it <jenkins-container-id> bash
    git config --global user.email "jenkins@example.com"
    git config --global user.name "jenkins"
    exit
```

#### Steps to Configure Jenkins Pipeline to Not Trigger Automatically on CI build Commit
Step 1: Install a Jenkins plugin
Install the plugin called "Ignore Committer Strategy".

Step 2: Configure the pipeline
The installed plugin lets you configure an email address of a committer that will be ignored for triggering a build (jenkins@example.com in our case). Open the configuration page for the multibranch pipeline project and scroll down to the "Branch Sources" > "Git" section. Open the "Add" dropdown for "Build strategies", select "Ignore Committer Strategy" and enter the email address of the committer, whose commits are to be ignored: jenkins@example.com. Also make sure the "Allow builds when a changeset contains non-ignored author(s)" checkbox is selected.

Note: If you only use standard pipelines (no multibranch pipelines) it is not necessary to install this plugin. Just go to the pipeline configuration and scroll down to "Additional Behaviours" in the Git configuration. Click the "Add" dropdown and choose "Polling ignores commits from certain users" and enter the username of the committer to be ignored for triggering a build (jenkins in our case).