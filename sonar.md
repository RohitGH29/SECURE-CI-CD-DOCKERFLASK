# Installation and Configuration Guide

## Install and configure SonarQube (Master machine)

1. Start SonarQube Server:
   ```bash
   docker run -itd --name SonarQube-Server -p 9000:9000 sonarqube:lts-community
   ```

2. Open your browser and navigate to `http://ip:9000`
   - Initial credentials:
     - Username: admin
     - Password: admin

3. Generate a token on SonarQube website:
   - Click on Administration > Security > Users
   - Create a new token for Jenkins
     - Name: Jenkins
     - Generate and copy the token for later use
   - Configure webhooks:
     - Click on Administration > Configuration > Webhooks
     - Create a new webhook
       - Name: Jenkins
       - URL: http://ip:8080/sonarqube-webhook/

## Install Trivy (Jenkins Worker)

1. Download and install Trivy:
   ```bash
   wget https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.deb
   sudo dpkg -i trivy_0.18.3_Linux-64bit.deb
   ```

## Configure OWASP (Jenkins Worker)

1. Install Jenkins Plugins:
   - Navigate to Manage Jenkins > Plugins > Available plugins
   - Install the following plugins:
     - SonarQube Scanner
     - Sonar Quality Gates
     - OWASP Dependency-check
     - Docker

2. Configure SonarQube in Jenkins:
   - Navigate to Manage Jenkins > System Configuration
   - Under "SonarQube servers", add a new SonarQube installation:
     - Name:
     - URL: http://ip:9000
     - Add the previously generated token as credentials

3. Install SonarQube Scanner:
   - Navigate to Manage Jenkins > Global Tool Configuration > SonarQube Scanner
   - Add a SonarQube scanner installation:
     - Name: sonar
     - Install automatically

4. Install OWASP Dependency-Check:
   - Navigate to Manage Jenkins > Global Tool Configuration > Dependency-Check
   - Add a Dependency-Check installation:
     - Name: dc
     - Install automatically
     - Choose the installer from github.com

