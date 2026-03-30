# DevOps Project: Automated CI/CD Pipeline for a 2-Tier Flask Application on Azure

**Author:** Manish Kumar Singh
**Date:** March 30, 2026

---

### **Table of Contents**

1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Step 1: Azure Virtual Machine Setup](#3-step-1-azure-virtual-machine-setup)
4. [Step 2: Install Dependencies on Azure VM](#4-step-2-install-dependencies-on-azure-vm)
5. [Step 3: Jenkins Installation and Setup](#5-step-3-jenkins-installation-and-setup)
6. [Step 4: GitHub Repository Configuration](#6-step-4-github-repository-configuration)
   * [Dockerfile](#dockerfile)
   * [docker-compose.yml](#docker-composeyml)
   * [Jenkinsfile](#jenkinsfile)
7. [Step 5: Jenkins Pipeline Creation and Execution](#7-step-5-jenkins-pipeline-creation-and-execution)
8. [Step 6: GitHub Webhook for Auto-Deploy](#8-step-6-github-webhook-for-auto-deploy)
9. [Conclusion](#9-conclusion)

---

### **1. Project Overview**

This project demonstrates how to deploy a 2-tier web application (Flask + MySQL) on a **Microsoft Azure Virtual Machine**. The application is containerized using Docker and Docker Compose. A full CI/CD pipeline is built using Jenkins to automatically build and deploy the application whenever new code is pushed to GitHub.

---

### **2. Architecture Diagram**

```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (on Azure VM)              |
+-----------------+      +----------------------+      |                             |
                                |                      | 1. Clones Repo              |
                                | Webhook              | 2. Builds Docker Image      |
                                | (auto-trigger)       | 3. Runs Docker Compose      |
                                v                      +--------------+--------------+
                         +------------+                               |
                         |  Webhook   |                               | Deploys
                         +------------+                               v
                                                       +-----------------------------+
                                                       |      Application Server     |
                                                       |      (Same Azure VM)        |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Docker Container: Flask | |
                                                       | +----------+--------------+ |
                                                       |            |                |
                                                       |            v                |
                                                       | +-------------------------+ |
                                                       | | Docker Container: MySQL | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```

---

### **3. Step 1: Azure Virtual Machine Setup**

#### **1. Create Azure Account**
- Go to https://azure.microsoft.com/free
- Sign up to get **$200 free credits for 30 days**
- Log in to https://portal.azure.com

#### **2. Launch Azure VM**
- Search **"Virtual Machines"** → click **Create → Azure Virtual Machine**
- Fill in the following details:

| Field | Value |
|-------|-------|
| Resource Group | `flask-devops-rg` (create new) |
| VM Name | `flask-devops-vm` |
| Region | Central India |
| Image | Ubuntu Server 22.04 LTS |
| Size | Standard_B2s (2 vCPUs, 4 GB RAM) |
| Authentication | SSH public key |
| Username | `azureuser` |
| Key pair name | `flask-devops-key` |

> ⚠️ Download the `.pem` file when prompted — it can only be downloaded once. Keep it safe.

#### **3. Configure Network Security Group (NSG)**

Add the following inbound port rules after VM creation:

| Port | Name | Purpose |
|------|------|---------|
| 22 | SSH | Remote access |
| 8080 | Allow-Jenkins | Jenkins dashboard |
| 5000 | Allow-Flask | Flask application |

> ⚠️ Do NOT expose port 3306 (MySQL) to the internet. Flask communicates with MySQL internally via Docker network.

#### **4. Connect to VM via SSH**

```bash
chmod 400 ~/Downloads/flask-devops-key.pem
ssh -i ~/Downloads/flask-devops-key.pem azureuser@<your-vm-public-ip>
```

---

### **4. Step 2: Install Dependencies on Azure VM**

```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install Git, Docker, and Docker Compose
sudo apt install git docker.io docker-compose-v2 -y

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add user to Docker group
sudo usermod -aG docker $USER
newgrp docker

# Install Java (required for Jenkins)
sudo apt install openjdk-17-jdk -y
```

Verify installations:
```bash
docker --version
docker compose version
java -version
```

---

### **5. Step 3: Jenkins Installation and Setup**

Jenkins is installed via the official WAR file method:

```bash
# Download Jenkins WAR file
sudo wget -O /opt/jenkins.war https://get.jenkins.io/war-stable/latest/jenkins.war

# Create Jenkins systemd service
sudo tee /etc/systemd/system/jenkins.service > /dev/null <<EOF
[Unit]
Description=Jenkins Service
After=network.target

[Service]
User=root
ExecStart=/usr/bin/java -jar /opt/jenkins.war --httpPort=8080
Restart=always

[Install]
WantedBy=multi-user.target
EOF

# Start and enable Jenkins
sudo systemctl daemon-reload
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Give Jenkins Docker access
sudo usermod -aG docker root
sudo systemctl restart jenkins
```

#### **Initial Jenkins Setup**

1. Get the initial admin password:
```bash
sudo cat /root/.jenkins/secrets/initialAdminPassword
```
2. Open browser → `http://<your-vm-public-ip>:8080`
3. Paste the password → click **Install Suggested Plugins** → create admin user

#### **Disable CSRF (required for GitHub Webhook)**

Go to **Manage Jenkins → Script Console** and run:
```groovy
import jenkins.model.Jenkins
import hudson.security.csrf.DefaultCrumbIssuer
def instance = Jenkins.getInstance()
instance.setCrumbIssuer(null)
instance.save()
println "CSRF disabled successfully"
```

---

### **6. Step 4: GitHub Repository Configuration**

Fork the repository and update the `Jenkinsfile` with your own GitHub repo URL.

#### **Dockerfile**

```dockerfile
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config && \
    rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 5000

CMD ["python", "app.py"]
```

#### **docker-compose.yml**

```yaml
version: "3.8"

services:
  mysql:
    container_name: mysql
    image: mysql
    environment:
      MYSQL_ROOT_PASSWORD: "root"
      MYSQL_DATABASE: "devops"
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - two-tier-nt
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot","-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  flask-app:
    container_name: two-tier-app
    build:
      context: .
    ports:
      - "5000:5000"
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_USER=root
      - MYSQL_PASSWORD=root
      - MYSQL_DB=devops
    networks:
      - two-tier-nt
    depends_on:
      mysql:
        condition: service_healthy
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

volumes:
  mysql_data:

networks:
  two-tier-nt:
```

#### **Jenkinsfile**

```groovy
pipeline {
    agent any
    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main', url: 'https://github.com/<your-username>/DevOps-Project-Two-Tier-Flask-App.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker Compose') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up -d --build'
            }
        }
    }
}
```

---

### **7. Step 5: Jenkins Pipeline Creation and Execution**

1. Jenkins dashboard → **New Item**
2. Enter name: `flask-two-tier` → select **Pipeline** → click **OK**
3. Scroll to **Pipeline** section and fill in:

| Field | Value |
|-------|-------|
| Definition | Pipeline script from SCM |
| SCM | Git |
| Repository URL | `https://github.com/<your-username>/DevOps-Project-Two-Tier-Flask-App.git` |
| Credentials | - none - (public repo) |
| Branch Specifier | `*/main` |
| Script Path | `Jenkinsfile` |

4. Click **Save** → **Build Now**
5. Monitor progress via **Console Output**

Verify containers are running:
```bash
docker ps
```

Access the application at:
```
http://<your-vm-public-ip>:5000
```

---

### **8. Step 6: GitHub Webhook for Auto-Deploy**

This completes the full CI/CD loop — every `git push` automatically triggers Jenkins without clicking "Build Now".

#### **Add Webhook in GitHub**

1. Go to your repo → **Settings → Webhooks → Add webhook**

| Field | Value |
|-------|-------|
| Payload URL | `http://<your-vm-public-ip>:8080/github-webhook/` |
| Content type | `application/json` |
| Secret | (leave empty) |
| Events | Just the push event |
| Active | ✅ checked |

2. Click **Add webhook**

#### **Configure Jenkins Job**

1. Jenkins → **flask-two-tier** → **Configure**
2. Under **Build Triggers** → check **"GitHub hook trigger for GITScm polling"** ✅
3. Click **Save**

#### **Test the Full CI/CD Loop**

1. Make any change in your GitHub repo (e.g., edit `README.md`)
2. Commit the change
3. Watch Jenkins → **Build History** — a new build triggers automatically ✅

---

### **9. Conclusion**

The CI/CD pipeline is now fully operational on Azure. Any `git push` to the `main` branch automatically triggers the Jenkins pipeline, which builds the Docker image and deploys the updated application — ensuring a seamless, hands-free deployment workflow from development to production.

**Tech Stack:**

| Layer | Technology |
|-------|-----------|
| Cloud | Microsoft Azure VM |
| OS | Ubuntu 22.04 LTS |
| CI/CD | Jenkins |
| Containerization | Docker & Docker Compose |
| Version Control | GitHub |
| Backend | Python Flask |
| Database | MySQL |

**Application URL:**
```
http://<your-vm-public-ip>:5000
```
