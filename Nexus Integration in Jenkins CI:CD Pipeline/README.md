```markdown
# Nexus Integration in Jenkins CI/CD Pipeline

After setting up Jenkins and successfully passing the SonarQube stage, the next crucial step in our CI/CD pipeline is to integrate with Nexus to upload our built artifacts. Nexus serves as a powerful repository manager, allowing us to store and retrieve artifacts, acting as a central hub for our project's dependencies and built artifacts.

## Jenkins Pipeline Script

Here is the complete Jenkins pipeline script, including the Nexus artifact upload stage:

```groovy
pipeline {
    agent any
    tools {
        maven "MAVEN3"
        jdk "OracleJDK11"
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
        stage("UploadArtifact") {
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
}
```

## Overview of Nexus Integration

Nexus OSS Sonatype is a software repository that stores and retrieves software artifacts. Here are the key points:

- **Artifact Storage**: Nexus allows us to store our built artifacts (e.g., WAR files). This is crucial for maintaining versions of our software that can be deployed later.
- **Dependency Management**: Nexus can also serve as a local repository for Maven or other package managers, helping to manage dependencies efficiently.
- **Repository Types**: Nexus supports various repository types, including Maven, npm, Docker, and more.

## Setting Up Nexus Repository

1. **Create a Repository**:
   - Go to Nexus and sign in.
   - Navigate to `Repositories` and create a new `Maven (hosted)` repository named `vprofile-repo`.

2. **Configure Jenkins Credentials**:
   - In Jenkins, go to `Manage Jenkins` > `Manage Credentials`.
   - Add a new global username and password credential for Nexus with ID `nexuslogin`.

## Integration with Jenkins

1. **Pipeline Configuration**:
   - The `UploadArtifact` stage in the Jenkins pipeline uses the `nexusArtifactUploader` plugin to upload the artifact to Nexus.
   - The artifact's version is dynamically generated using Jenkins environment variables (`BUILD_ID` and `BUILD_TIMESTAMP`).

2. **Artifact Versioning**:
   - Each build generates a unique version based on the build ID and timestamp, ensuring that artifacts are not overwritten.

3. **Running the Pipeline**:
   - After configuring the pipeline, run the build in Jenkins.
   - Verify that the artifact is successfully uploaded to Nexus by checking the repository.

This setup ensures that your artifacts are versioned and stored securely in Nexus, making them available for deployment and further testing by the operations team or automation scripts.
```
