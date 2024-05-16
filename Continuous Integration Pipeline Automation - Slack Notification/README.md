## Continuous Integration Pipeline Automation

So, you might think now we automated the continuous integration pipeline, but there are two things still that are manual:

1. We click on the `Build Now` button to start the pipeline.
   - It should automatically detect changes in the source code and run the pipeline.
   
2. The checks to see if our jobs are passing or failing.
   - We should not have to manually check Jenkins for job statuses.
   - We should be able to send automatic notifications for job statuses, whether they pass or fail.

### Setting Up Notifications with Jenkins

Jenkins has many plugins that allow for notifications. Let's set up Slack notifications, which is one of the most common collaboration tools used today.

#### Step-by-Step Guide

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
                git branch: 'vp-rem', url: 'https://github.com/devopshydclub/vprofile-repo.git'
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

---

By following these steps, you can ensure that your Jenkins pipeline is fully automated, with notifications being sent to your Slack channel whenever a build completes, whether it passes or fails. This enhances the efficiency of your CI/CD process by automating the monitoring of your builds.
