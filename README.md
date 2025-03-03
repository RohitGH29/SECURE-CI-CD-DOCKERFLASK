# Flask Application Deployment Using Jenkins CI/CD Pipeline and Docker

This project demonstrates setting up a CI/CD pipeline using Jenkins to deploy a Flask application on Docker. The pipeline includes stages for code checkout, installing dependencies, building a Docker image, pushing it to Docker Hub, and deploying it on Docker. The purpose is to provide a step-by-step guide to deploying Flask applications effectively.

---

## Prerequisites

Before proceeding, ensure you have the following:

1. **AWS EC2 Instance**:
   - Launch an **Ubuntu** EC2 instance (Type t3.medium).
   - Open the following ports in the security group:
     - **8080**: For Jenkins access.
     - **5000**: For Flask application access.

2. **Required Tools**:
   - Docker
   - Python 3.x
   - Jenkins

---

## Step 1: Install Required Tools on Your EC2 Instance

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

---

## Step 2: Configure Jenkins

### Install Required Plugins
1. Login to Jenkins and navigate to **Manage Jenkins > Manage Plugins**.
2. Install the following plugins:
   - **Docker Pipeline**
   - **Pipeline: Stage View**

### Add Credentials
1. Navigate to **Manage Jenkins > Credentials**.
2. Add Docker Hub credentials with:
   - **ID**: `docker-hub-creds`
   - **Username**: Your Docker Hub username.
   - **Password/Token**: Your Docker Hub password or access token.

### Create a New Pipeline Job
1. Go to the Jenkins dashboard and create a new item:
   - **Name**: `FlaskAppPipeline`
   - **Type**: `Pipeline`

2. Add the following pipeline script in the **Pipeline Definition**:

   ```groovy
   pipeline {
       agent any
       environment {
           DOCKER_IMAGE = 'your-dockerhub-username/flask-app'
       }
       stages {
           stage('Git: Code Checkout') {
               steps {
                   echo 'Checking out code...'
                   script {
                       checkout scm: [$class: 'GitSCM', branches: [[name: 'main']],
                           userRemoteConfigs: [[url: 'https://github.com/RohitGH29/CI-CD-DockerFlask.git']]]
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
           stage('Build Docker Image') {
               steps {
                   echo 'Building Docker image...'
                   sh 'docker build -t ${DOCKER_IMAGE} .'
               }
           }
           stage('Push to Docker Hub') {
               steps {
                   echo 'Pushing Docker image to Docker Hub...'
                   withCredentials([usernamePassword(credentialsId: 'docker-hub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                       sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                   }
                   sh 'docker push ${DOCKER_IMAGE}'
               }
           }
           stage('Deploy on Docker') {
               steps {
                   echo 'Deploying application...'
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

---

## Step 3: Run the Pipeline

1. Trigger the pipeline from Jenkins.
2. Monitor the pipeline stages:
   - **Code Checkout**: Fetches the Flask app code from GitHub.
   - **Install Dependencies**: Installs Python dependencies.
   - **Build Docker Image**: Builds a Docker image for the Flask app.
   - **Push to Docker Hub**: Uploads the image to Docker Hub.
   - **Deploy on Docker**: Runs the Flask app as a Docker container.

---

## Step 4: Verify Deployment

1. Open your browser and navigate to `http://<EC2_PUBLIC_IP>:5000`.
2. You should see the message:
   ```
   Welcome! You successfully deployed the Flask application on Docker using Jenkins pipeline.
   ```

---

## Additional Notes

- Ensure your `requirements.txt` file is updated with all necessary dependencies for the Flask app.
- Replace `your-dockerhub-username` in the pipeline script with your actual Docker Hub username.
- For security, restrict Jenkins credentials visibility to relevant jobs only.

---
