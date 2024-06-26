# Jenkins Setup on EC2 for Java Application

I'll walk you through how I set up Jenkins on an EC2 instance running Ubuntu 22.04. This includes installing the necessary tools and plugins for my Java application project. I'll cover setting up multiple versions of JDK, configuring Jenkins plugins, and ensuring Jenkins is accessible and secure.

## Step 1: Launch an EC2 Instance

Launch an EC2 instance with the following specifications:
- **AMI**: Ubuntu Server 22.04 LTS
- **Instance type**: t2.micro (suitable for small projects and testing)
- **Security Group**: 
  - Allow SSH (port 22) from anywhere (0.0.0.0/0) for remote access.
  - Allow Custom TCP (port 8080) from anywhere (0.0.0.0/0) to access Jenkins.
  - Ensure Nexus runs on port 8081 and the security group allows traffic on this port.

## Step 2: Connect to Your EC2 Instance

Connect to your EC2 instance using SSH:

```sh
ssh -i /path/to/your-key-pair.pem ubuntu@your-ec2-public-ip
```

## Step 3: Update the System and Install Java Development Kits (JDKs)

Update the package list and install OpenJDK 11:

```sh
sudo apt update
sudo apt install openjdk-11-jdk -y
```

(Optional) Install OpenJDK 8 if needed:

```sh
sudo apt install openjdk-8-jdk -y
```

## Step 4: Install Maven, wget, and unzip

Install Maven, wget, and unzip which are essential for building Java applications:

```sh
sudo apt install maven wget unzip -y
```

## Step 5: Install Jenkins

Add Jenkins repository key:

```sh
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

Add Jenkins repository:

```sh
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
```

Update package list and install Jenkins:

```sh
sudo apt-get update
sudo apt-get install jenkins -y
```

## Step 6: Start and Enable Jenkins

Start the Jenkins service and enable it to run at boot:

```sh
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

## Step 7: Access Jenkins and Initial Setup

Retrieve the initial admin password:

```sh
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Open Jenkins in your web browser using `http://your-ec2-public-ip:8080`.

Enter the initial admin password retrieved earlier and proceed with the setup wizard. Install suggested plugins during the setup.

## Step 8: Configure Jenkins for My Java Application

Install the necessary Jenkins plugins:

1. Go to `Manage Jenkins` > `Manage Plugins` > `Available`.
2. Search and install plugins such as Git, Maven Integration, JDK Tool, and Build Timestamp Plugin (zentimestamp).

Configure JDK installations:

1. Go to `Manage Jenkins` > `Global Tool Configuration`.
2. Under JDK, configure both JDK 8 and JDK 11:
   - **Name**: JDK 8, **JAVA_HOME**: `/usr/lib/jvm/java-8-openjdk-amd64`
   - **Name**: JDK 11, **JAVA_HOME**: `/usr/lib/jvm/java-11-openjdk-amd64`

## Step 9: Create and Configure My First Jenkins Job

Create a new item (job) in Jenkins:

1. Go to the Jenkins dashboard.
2. Click on `New Item`.
3. Enter a name for my job, select `Freestyle project`, and click `OK`.

Configure the job:

- **Source Code Management**: Select Git and provide my repository URL.
- **Build Environment**: Set up any necessary build tools (e.g., Maven).
- **Build**: Define the build steps (e.g., Maven goals like `clean install`).

Build the job and ensure it completes successfully.

## Step 10: Secure Jenkins

Update security settings:

1. Go to `Manage Jenkins` > `Configure Global Security`.
2. Set up appropriate security measures, such as enabling Jenkins' own user database, allowing users to sign up, and assigning roles.

Restrict access to Jenkins:

- Consider restricting access to port 8080 to specific IP addresses in your security group settings to enhance security.

By following these steps, I now have a fully functional Jenkins setup on my EC2 instance, ready for continuous integration and deployment of my Java application.

