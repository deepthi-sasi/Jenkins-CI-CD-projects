## Demo Project - Create CI Pipeline (Freestyle, Pipeline, Multibranch Pipeline)

### Topics of the Demo Project

### Create a CI pipeline using chained freestyle jobs, a pipeline and a multibranch pipeline.

### Technologies Used

* Jenkins  
* Docker
* Linux
* Git 
* Java 
* Maven

### Project Description

CI Pipeline for a Java Maven application to build and push to the repository.

* Install Build Tools (Maven, Node) in Jenkins 
* Make Docker available on Jenkins server 
* Create Jenkins credentials for a git repository 
* Create different Jenkins job types (Freestyle, Pipeline, Multibranch pipeline) for the Java Maven project with Jenkinsfile to:
    ```
    a. Connect to the application’s git repository
    b. Build Jar
    c. Build Docker Image
    d. Push to private DockerHub repository
    ```

### Steps to install Maven and Node in Jenkins
Step 1: Install Maven Plugin
For most of the usual build tools a related plugin is already installed. For Maven this is the case too. So we just have to configure the already installed plugin.
Go to the "Manage Jenkins" section and click on "Global Tool Configuration". Click on the "Add Maven" buttton, enter a name (e.g. maven-3.9) and click on "Save". Now you have the maven command available in all Jenkins jobs.

Step 2: Install npm and Node in the Jenkins container
Enter the docker container as root user (because we need the privilege to install tools):

```docker exec -u 0 -it <container-id> bash
Install curl:

apt update
apt install curl
```
Install node:
```
# get information about the current Linux distribution
cat /etc/issue
# => e.g. 'Debian GNU/Linux 11 \n \l'

# lookup the matching URL to download a node install script, e.g. under
# https://tecadmin.net/how-to-install-node-js-on-debian-11/
# execute the following curl command
curl -sL https://deb.nodesource.com/setup_18.x -o nodesource_setup.sh

# execute the install script and install node and npm
bash nodesource_setup.sh
apt-get install -y nodejs

# check the versions
node --version # v18.15.0
npm --version # 9.5.0
```
Steps to make Docker available on Jenkins server
Instead of installing Docker inside the Jenkins container, we mount the Docker runtime of the host system as a volume.

Step 1: Restart docker container with additional volumes
Stop the running Jenkins container and restart it with the following command:
```
docker run -p 8080:8080 -p 50000:50000 -d \
-v jenkins_home:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock \
-v $(which docker):/usr/bin/docker \
jenkins/jenkins:lts
```
Step 2: Provide jenkins user with missing file permission
The docker command is now available in the Jenkins container. However, the user jenkins (under which Jenkins is running) has no read/write permissions on the file /var/run/docker.sock. So we have to enter the Jenkins container as root user and provide the missing permissions to the user jenkins:
```
# enter the docker container as root user
docker exec -u 0 -it <container-id> bash

# provide missing permissions and exit
chmod 666 /var/run/docker.sock
exit
```
Check if jenkins user can execute docker commands

```docker exec -it <container-id> bash
docker pull hello-world
exit
```

If that doesn't work and you get errors like
```
docker: /lib/x86_64-linux-gnu/libc.so.6: version `GLIBC_2.32' not found (required by docker)
you can try using Ubuntu 18.04 LTS as host system or you have to install the docker runtime in the container:
```

stop the running Jenkins container and restart it with the following command:
```
docker run -p 8080:8080 -p 50000:50000 -d \
-v jenkins_home:/var/jenkins_home \
-v /var/run/docker.sock:/var/run/docker.sock \
jenkins/jenkins:lts
```

 enter this docker container as root user and install docker
```docker exec -u 0 -it <container-id> bash
apt update
apt install docker.io
exit

# check if jenkins user can execute docker commands
docker exec -it <container-id> bash
docker pull hello-world
exit
Now Jenkins can use the docker command in builds.

```
### Steps to create Jenkins credentials for the GitHub repository
### Step 1: Create a GitHub access token
Step 1: Create a GitHub Personal Access Token
GitHub no longer supports username/password authentication. To generate a token:

Go to GitHub → Profile → Settings → Developer settings → Personal access tokens → Tokens (classic)
Click Generate new token (classic)
Enter a token name (e.g. devops-bootcamp-jenkins), set an expiration date, and check the repo scope
Click Generate token and copy it immediately — it won't be shown again


Step 2: Add GitHub Credentials to Jenkins

Go to Manage Jenkins → Manage Credentials → Stores scoped to Jenkins → System → Global credentials (unrestricted)
Click Add Credentials
Set the kind to Username with password, scope to Global
Enter your GitHub username and the token as the password
Add an ID (e.g. GitHub) and click Create


This is ready to paste straight into your GitHub README. Want me to adjust the formatting or tone at all?
### Steps to create a Freestyle Job
Step 1: Create a New Freestyle Job

Go to Dashboard → New Item
Enter an item name (e.g. devops-bootcamp-freestyle), select Freestyle project and click OK


Step 2: Connect to the GitHub Repository

On the configuration page, go to Source Code Management and select Git
Enter the repository URL: https://github.com/deepthi-sasi/java-maven-app
Select the GitHub credentials configured earlier
Under Branches to build, set the branch specifier to */main and click Save


Step 3: Build the JAR

Go to Dashboard → Configure → Build Steps
From the Add build step dropdown, select Invoke top-level Maven targets
Select Maven version (e.g. maven-3.9), enter the goal package and click Save


Step 4: Build the Docker Image
Add the following Dockerfile to your project root, then commit and push it to GitHub:
```
FROM amazoncorretto:17-alpine-jdk

EXPOSE 8080

COPY ./build/libs/java-app-1.0-SNAPSHOT.jar /usr/app
WORKDIR /usr/app

ENTRYPOINT ["java", "-jar", "java-app-1.0-SNAPSHOT.jar"]

```
In the Jenkins job, add an Execute shell build step with:

```
docker build -t java-maven-app:1.0.0 .

```

Step 5: Configure credentials for the private DockerHub repository
1. Go to Dashboard → Manage Jenkins → Manage Credentials → Stores scoped to Jenkins → Jenkins → Global credentials
2. Click Add Credentials
3. Enter your DockerHub username, password, and an ID (e.g. DockerHub)


Step 6: Push to private DockerHub repository
1. Open your Jenkins build configuration and go to the Build Environment section 
2. Check Use secret text(s) or file(s)
3. Add a Username and password (separated) binding and define environment variable names (e.g. USERNAME and PASSWORD), then select the DockerHub credentials
4. In the Build section (Execute shell), update the image tag to match your private repo (e.g. `docker build -t <your-repo>/<image-name>:<tag> .`) and add commands to log in and push:

```
docker build -t deepthisasi/demo-app:1.1 .
echo $PASSWORD | docker login -u $USERNAME --password-stdin
docker push deepthisasi/demo-app:1.1
```
Save the build configuration and run the build ("Dashboard" > "devops-bootcamp-freestyle" > "Build Now"). Then go to your private repository on DockerHub and check if the new image got pushed.

### Steps to create a Pipeline Job

Step 1: Create a New Pipeline Job

Go to Dashboard → New Item
Enter an item name (e.g. devops-bootcamp-pipeline), select Pipeline and click OK

Step 2: Connect to the GitHub Repository

On the configuration page, go to the Pipeline section and select Pipeline script from SCM
Select Git from the SCM dropdown
Enter the repository URL: https://github.com/deepthi-sasi/java-maven-app
Select the GitHub credentials configured earlier
Under Branches to build, set the branch specifier to */main and click Save

Step 3: Build the JAR via Jenkinsfile
Add a Jenkinsfile to your project root to define the pipeline. The file should include a stage to build the application using Maven:

    pipeline {
        agent any
        tools {
            maven 'maven-3.9'
        }
        stages {
            stage("Build Application JAR") {
                steps {
                    script {
                        echo "building the application..."
                        sh 'mvn package'
                    }       
                }
            }
        }   
    }
Step 4: Build Docker Image
Add the same Dockerfile to your project sources as in step 4 of the freestyle job above. But instead of adding an "Execute shell" step in the build configuration, add the following stage to the Jenkinsfile:

    
    stage("Build Docker Image") {
        steps {
          script {
              echo "building the docker image..."
              sh 'docker build -t java-maven-app:1.0.0 .'
          }
        }
    }
  
Step 5: Configure credentials for the private DockerHub repository
See step 5 of the freestyle job above.

Step 6: Push to private DockerHub repository
Replace the "Build Docker Image" stage with the following stage to build, login and push the image to the private repository:
```
stage("Build and Publish Docker Image") {
    steps {
      script {
        withCredentials([usernamePassword(credentialsId: 'DockerHub', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
          echo "building the docker image..."
          sh 'docker build -t deepthisasi/demo-app:javamavenapp1.1 .'
          echo "publishing the docker image..."
          sh "echo $PASSWORD | docker login -u $USERNAME --password-stdin"
          sh 'docker push deepthisasi/demo-app:javamavenapp1.1'
        }
      }
    }
}
```

Go to "Dashboard" > "devops-bootcamp-pipeline" > "Build Now"). Then go to your private repository on DockerHub and check if the new image got pushed.

### Steps to create a Multibranch Pipeline Job

Step 1: Create a new Multibranch Pipeline Job
Go to "Dashboard" > "New Item", enter an item name (e.g. my-multibranch-pipeline), select the "Multibranch Pipeline" area and press the "Ok" button.

Step 2: Connect to the application’s git repository
On the configuration page, go to the "Branch Sources" section and select "Git" from the "Add source" dropdown. Enter the URL of the GitHub repository holding the Java Maven application (https://github.com/deepthi-sasi/java-maven-app). Choose the credentials created in step 1 of the Freestyle project above. Open the "Add" dropdown under "Discover branches" and select "Filter by name (with regular expression)". Enter .* to select all branches.

As soon as you press the "Save" button, Jenkins scans the Git repository for all branches containing a Jenkinsfile, creates related pipelines and starts the builds.

Because we already added a Dockerfile and Jenkinsfile to the project sources, the final build workflow is already set up and the build is executed. There's nothing left to do for steps 3 to 6 (see Pipeline job above).
