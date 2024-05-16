
### Overview

This guide covers the setup and automation of a complete CI/CD pipeline using Jenkins, Maven, SonarQube, and Slack notifications. It includes detailed steps for integrating Nexus for artifact management, setting up Jenkins on EC2, configuring SonarQube for code analysis, and automating notifications to Slack.

### Table of Contents

1. [Setting Up Jenkins on EC2 Ubuntu 22.04](#setting-up-jenkins-on-ec2-ubuntu-2204)
2. [Nexus Integration in Jenkins CI/CD Pipeline](#nexus-integration-in-jenkins-cicd-pipeline)
3. [SonarQube Integration with Jenkins Pipeline](#sonarqube-integration-with-jenkins-pipeline)
4. [Continuous Integration Pipeline Automation - Slack Notification](#continuous-integration-pipeline-automation---slack-notification)


### Setting Up Jenkins on EC2 Ubuntu 22.04

**Steps:**

1. **Launch an EC2 Instance**:
    - Choose Ubuntu 22.04 as the AMI.
    - Configure security groups to allow HTTP, HTTPS, and custom TCP for Jenkins (port 8080).

2. **Install Java**:
    ```sh
    sudo apt update
    sudo apt install openjdk-11-jdk -y
    ```

3. **Install Jenkins**:
    ```sh
    wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
    sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
    sudo apt update
    sudo apt install jenkins -y
    sudo systemctl start jenkins
    sudo systemctl enable jenkins
    ```

4. **Access Jenkins**:
    - Navigate to `http://<your-ec2-public-ip>:8080`
    - Follow the on-screen instructions to complete the setup.

### Nexus Integration in Jenkins CI/CD Pipeline

**Steps:**

1. **Install Nexus**:
    - Follow the official [Sonatype Nexus Repository Manager installation guide](https://help.sonatype.com/repomanager3/installation) to set up Nexus on your server.
    - USERDATA
      ### Nexus Installation on CentOS 7

**Steps:**

1. **Create a Bash Script for Nexus Installation**:

```bash
#!/bin/bash

# Install Java
yum install java-1.8.0-openjdk.x86_64 wget -y   

# Create necessary directories
mkdir -p /opt/nexus/   
mkdir -p /tmp/nexus/                           
cd /tmp/nexus/

# Download Nexus
NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
wget $NEXUSURL -O nexus.tar.gz
sleep 10

# Extract Nexus
EXTOUT=`tar xzvf nexus.tar.gz`
NEXUSDIR=`echo $EXTOUT | cut -d '/' -f1`
sleep 5
rm -rf /tmp/nexus/nexus.tar.gz
cp -r /tmp/nexus/* /opt/nexus/
sleep 5

# Create nexus user and set permissions
useradd nexus
chown -R nexus.nexus /opt/nexus 

# Create systemd service for Nexus
cat <<EOT>> /etc/systemd/system/nexus.service
[Unit]                                                                          
Description=Nexus service                                                       
After=network.target                                                            
                                                                  
[Service]                                                                       
Type=forking                                                                    
LimitNOFILE=65536                                                               
ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start                                  
ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop                                    
User=nexus                                                                      
Restart=on-abort                                                                
                                                                  
[Install]                                                                       
WantedBy=multi-user.target                                                      
EOT

```

2. **Create a Maven Repository**:
    - Navigate to Nexus UI.
    - Create a new hosted repository for Maven.

3. **Configure Jenkins for Nexus**:
    - Install the Nexus Artifact Uploader plugin in Jenkins.
    - Configure the Nexus repository details in Jenkins global configuration.

### SonarQube Integration with Jenkins Pipeline

**Steps:**

1. **Install SonarQube**:
    - Follow the steps in the [SonarQube Installation on Ubuntu 22.04](#sonarqube-installation-on-ubuntu-2204) section to set up SonarQube on your server.
    - USERDATA
      Here's the complete script for installing SonarQube on Ubuntu 22.04 along with the configurations and services setup:

```bash
#!/bin/bash

# Backup and update system configuration
cp /etc/sysctl.conf /root/sysctl.conf_backup
cat <<EOT > /etc/sysctl.conf
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
EOT

cp /etc/security/limits.conf /root/sec_limit.conf_backup
cat <<EOT > /etc/security/limits.conf
sonarqube   -   nofile   65536
sonarqube   -   nproc    409
EOT

# Update and install Java
sudo apt-get update -y
sudo apt-get install openjdk-11-jdk -y
sudo update-alternatives --config java

# Verify Java installation
java -version

# Update and install PostgreSQL
sudo apt update
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
sudo apt-get update
sudo apt-get install postgresql postgresql-contrib -y

# Enable and start PostgreSQL service
sudo systemctl enable postgresql.service
sudo systemctl start postgresql.service

# Configure PostgreSQL for SonarQube
sudo echo "postgres:admin123" | chpasswd
runuser -l postgres -c "createuser sonar"
sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
systemctl restart postgresql

# Verify PostgreSQL is running
netstat -tulpena | grep postgres

# Download and install SonarQube
sudo mkdir -p /sonarqube/
cd /sonarqube/
sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
sudo apt-get install zip -y
sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube

# Create SonarQube user and set permissions
sudo groupadd sonar
sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
sudo chown sonar:sonar /opt/sonarqube/ -R

# Backup and configure sonar.properties
cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
cat <<EOT > /opt/sonarqube/conf/sonar.properties
sonar.jdbc.username=sonar
sonar.jdbc.password=admin123
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.web.javaAdditionalOpts=-server
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
sonar.log.level=INFO
sonar.path.logs=logs
EOT

# Create systemd service for SonarQube
cat <<EOT > /etc/systemd/system/sonarqube.service
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
EOT

# Reload systemd daemon and enable SonarQube service
systemctl daemon-reload
systemctl enable sonarqube.service

# Install and configure Nginx
apt-get install nginx -y
rm -rf /etc/nginx/sites-enabled/default
rm -rf /etc/nginx/sites-available/default

cat <<EOT > /etc/nginx/sites-available/sonarqube
server {
    listen 80;
    server_name sonarqube.groophy.in;

    access_log /var/log/nginx/sonar.access.log;
    error_log /var/log/nginx/sonar.error.log;

    proxy_buffers 16 64k;
    proxy_buffer_size 128k;

    location / {
        proxy_pass http://127.0.0.1:9000;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;

        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto http;
    }
}
EOT

ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
systemctl enable nginx.service

# Allow necessary ports through the firewall
sudo ufw allow 80,9000,9001/tcp

# Reboot the system
echo "System reboot in 30 sec"
sleep 30
reboot
```
2. **Configure SonarQube in Jenkins**:
    - Install the SonarQube Scanner plugin in Jenkins.
    - 
    - Add SonarQube server details in Jenkins global configuration.

3. **Update Jenkins Pipeline**:
    - Add SonarQube analysis stage in your Jenkins pipeline script.

### Continuous Integration Pipeline Automation - Slack Notification

**Overview**

In this section, we will set up Slack notifications for our Jenkins CI/CD pipeline.

**Step-by-Step Guide**

1. **Log into Slack**:
    - Create or log into your Slack account.
    - Create a workspace (e.g., `vprofilecicd`).
    - Create a channel within the workspace (e.g., `jenkinscicd`).

2. **Add Jenkins App to Slack**:
    - Search for Jenkins CI in Slack apps and add it to your workspace.
    - Choose the channel you created (`jenkinscicd`).
    - Copy the integration token provided by Slack.

3. **Configure Jenkins**:
    - Go to Jenkins and install the Slack Notification plugin.
    - Navigate to `Manage Jenkins` -> `Configure System` and find the Slack section.
    - Enter your workspace name (`vprofilecicd`) and add the integration token.

4. **Update the Jenkins Pipeline**:
    - Add a post-action step to send notifications.

**Jenkins Pipeline Script**

Here is the complete pipeline script with Slack notifications configured:

```groovy
def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]

pipeline {
    agent any
    tools {
        maven "MAVEN3"
        jdk "OracleJDK11"
    }
    environment {
        BUILD_TIMESTAMP = sh(script: 'date +%Y%m%d%H%M%S', returnStdout: true).trim()
    }
    stages {
        stage('Fetch code') {
            steps {
                git branch: 'vp-rem', url: 'https://github.com/DAMMYTJ/javaapprepo.git'
            }  
        }

        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo "Now Archiving."
                    archiveArtifacts artifacts: '**/*.war'
                }
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
        }

        stage('Sonar Analysis') {
            environment {
                scannerHome = tool 'sonar4.7'
            }
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                }
            }
        }

        stage("Upload Artifact") {
            steps {
                nexusArtifactUploader(
                    nexusVersion: 'nexus3',
                    protocol: 'http',
                    nexusUrl: '172.31.48.20:8081',
                    groupId: 'QA',
                    version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                    repository: 'vprofile-repo',
                    credentialsId: 'nexuslogin',
                    artifacts: [
                        [artifactId: 'vproapp',
                         classifier: '',
                         file: 'target/vprofile-v2.war',
                         type: 'war']
                    ]
                )
            }
        }
    }
    post {
        always {
            echo 'Slack Notifications.'
            slackSend channel: '#jenkinscicd',
                color: COLOR_MAP[currentBuild.currentResult],
                message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
        }
    }
}
```

<img width="1792" alt="Screenshot 2024-05-16 at 10 05 01 PM" src="https://github.com/DAMMYTJ/Complete-CI-CD-Pipeline-Guide/assets/111458165/294e12ab-a8d2-49af-b4ad-dd412e1f85c1">
<img width="1792" alt="Screenshot 2024-05-16 at 10 05 05 PM" src="https://github.com/DAMMYTJ/Complete-CI-CD-Pipeline-Guide/assets/111458165/727e4d67-8087-4b0b-961a-232ba1a3f8b9">

<img width="1792" alt="Screenshot 2024-05-16 at 10 05 18 PM" src="https://github.com/DAMMYTJ/Complete-CI-CD-Pipeline-Guide/assets/111458165/198ef6e3-e203-4e45-b641-6dcbfa74033f">

<img width="1792" alt="Screenshot 2024-05-16 at 10 05 28 PM" src="https://github.com/DAMMYTJ/Complete-CI-CD-Pipeline-Guide/assets/111458165/42469ba9-c114-4f26-9e20-7ba4ed2dd5a5">
<img width="1792" alt="Screenshot 2024-05-16 at 10 07 24 PM" src="https://github.com/DAMMYTJ/Complete-CI-CD-Pipeline-Guide/assets/111458165/2a034002-42f3-4514-bdbe-7f8d90e6c591">

<img width="1792" alt="Screenshot 2024-05-16 at 10 07 31 PM" src="https://github.com/DAMMYTJ/Complete-CI-CD-Pipeline-Guide/assets/111458165/3d4c14ee-b492-4939-884f-e1231253a5bf">
<img width="1792" alt="Screenshot 2024-05-16 at 10 08 02 PM" src="https://github.com/DAMMYTJ/Complete-CI-CD-Pipeline-Guide/assets/111458165/f294be0a-e70a-4aff-b639-315d15c390cb">










