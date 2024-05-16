SonarQube Integration with Jenkins Pipeline

Overview
Hey everyone, I want to share how we can leverage our pipeline-as-code knowledge to set up a complete continuous integration (CI) pipeline with code analysis using SonarQube. We'll integrate SonarQube into our Jenkins pipeline to ensure our code adheres to best practices and is free from vulnerabilities.

What is Code Analysis?
Code analysis is a critical step in ensuring code quality. It involves running various tests on your code to catch issues early. This isn't about software testing but about analyzing the code itself for potential problems. It helps identify vulnerabilities, improve adherence to coding standards, and generally enhances the robustness and security of our code.

Tools i Use
Here's a quick rundown of the tools we’ll be using:

Jenkins: Our CI server.
SonarQube: For detailed code analysis.
Checkstyle: To check our Java code against coding standards.
Maven: For building the project and running tests.
Steps to Integrate SonarQube with Jenkins
Set Up SonarQube Server
First, make sure your SonarQube server is up and running. Log in to both SonarQube and Jenkins.

Install SonarQube Scanner in Jenkins

Go to Manage Jenkins > Global Tool Configuration.
Add a new SonarQube scanner, name it sonar4.7.
Save the configuration.
Configure SonarQube Server in Jenkins

Navigate to Manage Jenkins > Configure System.
Scroll down to the SonarQube Servers section and add a new server.
Provide a name (e.g., sonar) and the server URL.
Generate an authentication token from SonarQube and add it to Jenkins.
Pipeline Configuration
Here’s the pipeline script we’ll use in our Jenkins job:

groovy
Copy code
pipeline {
    agent any
    tools {
        maven "MAVEN3"
        jdk "OracleJDK11"
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
    }
}
Pipeline Stages Explained
Fetch code: This stage clones the repository from GitHub.
Build: Compiles the project using Maven, with tests skipped to speed up the process.
Test: Runs unit tests using Maven.
Checkstyle Analysis: Uses Checkstyle to analyze the code for adherence to coding standards.
Sonar Analysis: This is where we integrate SonarQube Scanner to analyze the code and upload the results to our SonarQube server.
Quality Gates
SonarQube uses quality gates to determine if the code meets certain standards. By default, it uses the "Sonar way" quality gate. We can monitor our project’s compliance with these gates through the SonarQube dashboard, looking out for bugs, vulnerabilities, and code smells.

Conclusion
By setting up this CI pipeline, we ensure our code is not just built and tested but also thoroughly analyzed for quality. Integrating SonarQube with Jenkins helps us maintain high standards, catch issues early, and improve the overall quality and security of our codebase. Let’s dive in and make our code the best it can be!
