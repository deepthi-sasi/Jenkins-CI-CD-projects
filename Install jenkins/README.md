## Demo Project - Install Jenkins

### Topics of the Demo Project
Install Jenkins on DigitalOcean

### Technologies Used
* Jenkins 
* Docker 
* DigitalOcean 
* Linux

### Project Description
Create an Ubuntu server on DigitalOcean  
Set up and run Jenkins as Docker container
Initialize Jenkins

### Steps to install Jenkins on a DigitalOcean Droplet
Step 1: Create Droplet
Login to your DigitalOcean account an create a new Droplet (4GB RAM). Jenkins needs at least 1GB RAM. Change the Droplet's name to something like 'jenkins-server' and attach a firewall rule-set to it opening port 22 for SSH from your machine's IP address and port 8080 (Type=Custom) for Jenkins from all IP addresses.
    
Step 2: Install Docker
SSH into the Droplet (ssh root@<droplet-ip>) and install Docker (since we want to run Jenkins in a Docker container):
    
    apt update
    apt install docker.io

Step 3: Start Jenkins in a Docker container
```
docker run -p 8080:8080 -p 50000:50000 -d -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts`
# Jenkins master and worker nodes communicate over port 50000
```


Step 4: Initialize Jenkins
```
Copy the initial administrator password:

# in the docker container
docker exec -it <container-id> bash

cat /var/jenkins_home/secrets/initialAdminPassword

# or directly on the host

docker volume inspect jenkins_home
# the path is displayed as "Mountpoint" -> /var/lib/docker/volumes/jenkins_home/_data
cat /var/lib/docker/volumes/jenkins_home/_data/secrets/initialAdminPassword


```

Open Jenkins in the browser under http://<droplet-ip>:8080, enter the initial administrator password and install the suggested plugins. After creating a first admin user, the initialization of Jenkins is done and the welcome screen is displayed.
