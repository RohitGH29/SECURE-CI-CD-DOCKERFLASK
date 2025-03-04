# Deploy SecureFlask CI/CD Pipeline on EC2 using Jenkins

This guide walks you through deploying a SecureFlask CI/CD pipeline on an AWS EC2 instance using Jenkins, SonarQube, OWASP Dependency-Check, Trivy, and Docker.

## Prerequisites

Before you begin, ensure you have the following:
- An AWS EC2 instance (Ubuntu 20.04 recommended)
- A non-root user with sudo privileges
- Open ports: `22` (SSH), `8080` (Jenkins), `9000` (SonarQube), `5000` (Flask App)
- Docker Hub account
- GitHub repository containing your Flask application

## Step 1: Install Required Software

### Install Docker
1. Update the system and install Docker:
   ```bash
   sudo apt update
   sudo apt install docker.io -y
   sudo usermod -aG docker ubuntu && newgrp docker
   ```

2. Verify Docker installation:
   ```bash
   docker --version
   ```

### Install Python
1. Install Python and the `venv` module:
   ```bash
   sudo apt install python3-venv -y
   ```

### Install Jenkins
1. Update the system and install Jenkins:
   ```bash
   sudo apt update -y
   sudo apt install fontconfig openjdk-17-jre -y
   
   sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
   https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
   
   echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
   https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
   /etc/apt/sources.list.d/jenkins.list > /dev/null
   
   sudo apt-get update -y
   sudo apt-get install jenkins -y
   ```

2. Start Jenkins and check its status:
   ```bash
   sudo systemctl start jenkins
   sudo systemctl status jenkins
   ```

3. Access Jenkins:
   - Open your browser and navigate to `http://<EC2_PUBLIC_IP>:8080`.

4. Retrieve the initial admin password:
   ```bash
   sudo cat /var/lib/jenkins/secrets/initialAdminPassword
   ```

5. Add Jenkins to the Docker group:
   ```bash
   sudo usermod -aG docker jenkins
   sudo systemctl restart jenkins
   ```

## Step 2: Install and Configure SonarQube

1. Start SonarQube Server:
   ```bash
   docker run -itd --name SonarQube-Server -p 9000:9000 sonarqube:lts-community
   ```

2. Open your browser and navigate to `http://<EC2_PUBLIC_IP>:9000`
   - Initial credentials:
     - Username: admin
     - Password: admin

3. Generate a token on SonarQube:
   - Click on Administration > Security > Users
   - Create a new token for Jenkins
     - Name: Jenkins
     - Generate and copy the token for later use

   - Configure webhooks in SonarQube:
     - Click on Administration > Configuration > Webhooks
     - Create a new webhook with 
        - Name: Jenkins
        - URL: `http://<EC2_PUBLIC_IP>:8080/sonarqube-webhook/`

## Step 3: Install Trivy (Security Scanner)

1. Download and install Trivy:
   ```bash
   wget https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.deb
   sudo dpkg -i trivy_0.18.3_Linux-64bit.deb
   ```

## Step 4: Configure OWASP Dependency-Check in Jenkins

1. Install required plugins in Jenkins:
   - SonarQube Scanner
   - Sonar Quality Gates
   - OWASP Dependency-Check
   - Docker
   - Pipeline: Stage View

2. Configure SonarQube in Jenkins:
   - Manage Jenkins > System Configuration
   - Under "SonarQube servers", add a new SonarQube installation:
     - Name: Sonar
     - URL: http://ip:9000
     - Credentials  Click on Add 
        - kind: Secret text
        - secret: paste SonarQube token
        - ID: Sonar
        - Description: Sonar

3. Install SonarQube Scanner:
   - Manage Jenkins > Global Tool Configuration 
        - under "SonarQube Scanner installations"
             - Add SonarQube scanner 
                - Name: Sonar
                - Install automatically

4. Install OWASP Dependency-Check:
   - Manage Jenkins > Global Tool Configuration 
        - under "Dependency-Check installations"
            - Add Dependency-Check:
                - Name: dc
                - Install automatically
                - click on Add Installer
                     - Choose the installer from github.com

## Step 5: Configure Jenkins Pipeline

1. Add Docker Hub credentials in Jenkins:
   - Manage Jenkins > Credentials
   - Add Docker Hub credentials with:
     - **ID**: `docker-hub-creds`
     - **Username**: Your Docker Hub username.
     - **Password/Token**: Your Docker Hub password or access token.

2. Create a new Jenkins pipeline:
   - **Name**: `SecureFlaskPipeline`
   - **Type**: `Pipeline`

3. Use the following pipeline script:

   ```groovy
   pipeline {
       agent any
       environment {
           DOCKER_IMAGE = 'your-dockerhub-username/flask-app'
           SONAR_HOME = tool "Sonar"
       }
       stages {
           stage('Git: Code Checkout') {
               steps {
                   echo 'Checking out code...'
                   script {
                       checkout scm: [$class: 'GitSCM', branches: [[name: 'main']],
                           userRemoteConfigs: [[url: 'https://github.com/RohitGH29/SECURE-CI-CD-DOCKERFLASK.git']]]
                   }
               }
           }
           stage('Install Dependencies') {
               steps {
                   echo 'Installing dependencies...'
                   sh '''
                       python3 -m venv venv
                       . venv/bin/activate
                       pip install -r requirements.txt
                   '''
               }
           }
           stage('SonarQube Quality Analysis') {
            steps {
                echo 'SonarQube Quality Analysis...'
                withSonarQubeEnv("Sonar"){
                    sh "$SONAR_HOME/bin/sonar-scanner -Dsonar.projectName=SECURE-CI-CD-DOCKERFLASK -Dsonar.projectKey=SECURE-CI-CD-DOCKERFLASK"
                }
                
            }
        }
        stage('OWASP Dependency check') {
            steps {
                echo 'OWASP Dependency check...'
                dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'dc'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
               
            }
        }
        stage('SonarQube Quality Gate Scan') {
            steps {
                echo 'SonarQube Quality Gate Scan...'
                timeout(time: 2,unit: "MINUTES"){
                    waitForQualityGate abortPipeline: false
                }
               
            }
        }
        stage('Trivy file system Scan') {
            steps {
                echo 'Trivy file system Scan...'
                sh "trivy fs --format table -o trivy-fs-report.html ."
               
            }
        }
           stage('Build and Push Docker Image') {
               steps {
                   echo 'Building and pushing Docker image...'
                   sh 'docker build -t ${DOCKER_IMAGE} .'
                   withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                       sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                   }
                   sh 'docker push ${DOCKER_IMAGE}'
               }
           }
           stage('Deploy Application') {
               steps {
                   echo 'Deploying Flask application...'
                   sh '''
                       docker stop flask-app || true
                       docker rm flask-app || true
                       docker run -d -p 5000:5000 --name flask-app ${DOCKER_IMAGE}
                   '''
               }
           }
       }
   }
   ```

## Step 6: Verify Deployment

1. Open your browser and navigate to `http://<EC2_PUBLIC_IP>:5000`.
2. You should see the message:
   ```
   Welcome! You successfully deployed the Flask application on Docker using Jenkins pipeline.
   ```

## Conclusion

You have successfully set up a SecureFlask CI/CD pipeline on AWS EC2 using Jenkins, Docker, and security tools. 🎉

