# DevOps Project Report: Automated CI/CD Pipeline for a 2-Tier Flask Application on AWS

**Author:** Parker

**Date:** August 23, 2025

---

### **Table of Contents**
1. [Project Overview](#1-project-overview)
2. [Architecture Diagram](#2-architecture-diagram)
3. [Step 1: AWS EC2 Instance Preparation](#3-step-1-aws-ec2-instance-preparation)
4. [Step 2: Install Dependencies on EC2](#4-step-2-install-dependencies-on-ec2)
5. [Step 3: Jenkins Installation and Setup](#5-step-3-jenkins-installation-and-setup)
6. [Step 4: GitHub Repository Configuration](#6-step-4-github-repository-configuration)
    * [Dockerfile](#dockerfile)
    * [docker-compose.yml](#docker-composeyml)
    * [Jenkinsfile](#jenkinsfile)
7. [Step 5: Jenkins Pipeline Creation and Execution](#7-step-5-jenkins-pipeline-creation-and-execution)
8. [Conclusion](#8-conclusion)
9. [Infrastructure Diagram](#9-infrastructure-diagram)
10. [Work flow Diagram](#10-work-flow-diagram)

---

### **1. Project Overview**
This report explains the complete procedure for deploying a 2-tier web application built with Flask and MySQL on an AWS EC2 instance. The application is containerized using Docker and managed through Docker Compose. A CI/CD pipeline is implemented using Jenkins to automate building and deployment whenever code changes are pushed to a GitHub repository.

---

### **2. Architecture Diagram**

```
+-----------------+      +----------------------+      +-----------------------------+
|   Developer     |----->|     GitHub Repo      |----->|        Jenkins Server       |
| (pushes code)   |      | (Source Code Mgmt)   |      |  (on AWS EC2)               |
+-----------------+      +----------------------+      |                             |
                                                       | 1. Clones Repo              |
                                                       | 2. Builds Docker Image      |
                                                       | 3. Runs Docker Compose      |
                                                       +--------------+--------------+
                                                                      |
                                                                      | Deploys
                                                                      v
                                                       +-----------------------------+
                                                       |      Application Server     |
                                                       |      (Same AWS EC2)         |
                                                       |                             |
                                                       | +-------------------------+ |
                                                       | | Docker Container: Flask | |
                                                       | +-------------------------+ |
                                                       |              |              |
                                                       |              v              |
                                                       | +-------------------------+ |
                                                       | | Docker Container: MySQL | |
                                                       | +-------------------------+ |
                                                       +-----------------------------+
```

---

### **3. Step 1: AWS EC2 Instance Preparation**

1.  **Launch EC2 Instance:**
    * Navigate to the AWS EC2 console.
    * Launch a new instance using the **Ubuntu 22.04 LTS** AMI.
    * Select the **t2.micro** instance type for free-tier eligibility.
    * Create and assign a new key pair for SSH access.



2.  **Configure Security Group:**
    * Create a security group with the following inbound rules:
        * **Type:** SSH, **Protocol:** TCP, **Port:** 22, **Source:** Your IP
        * **Type:** HTTP, **Protocol:** TCP, **Port:** 80, **Source:** Anywhere (0.0.0.0/0)
        * **Type:** Custom TCP, **Protocol:** TCP, **Port:** 5000 (for Flask), **Source:** Anywhere (0.0.0.0/0)
        * **Type:** Custom TCP, **Protocol:** TCP, **Port:** 8080 (for Jenkins), **Source:** Anywhere (0.0.0.0/0)



3.  **Connect to EC2 Instance:**
    * Use SSH to connect to the instance's public IP address.
    ```bash
    ssh -i /path/to/key.pem ubuntu@<ec2-public-ip>
    ```

---

### **4. Step 2: Install Dependencies on EC2**

1.  **Update System Packages:**
    ```bash
    sudo apt update && sudo apt upgrade -y
    ```

2.  **Install Git, Docker, and Docker Compose:**
    ```bash
    sudo apt install git docker.io docker-compose-v2 -y
    ```

3.  **Start and Enable Docker:**
    ```bash
    sudo systemctl start docker
    sudo systemctl enable docker
    ```

4.  **Add User to Docker Group (to run docker without sudo):**
    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    ```

---

### **5. Step 3: Jenkins Installation and Setup (Docker-Based Deployment)**

Instead of installing Jenkins directly on the EC2 host, Jenkins was deployed as a Docker container to ensure portability, isolation, and easier dependency management.

---

#### **1. Pull Jenkins Docker Image**

```bash
docker pull jenkins/jenkins:lts
```

This downloads the Long-Term Support (LTS) Jenkins image from Docker Hub.

---

#### **2. Create Jenkins Persistent Volume**

```bash
docker volume create jenkins_home
```

This volume stores:

• Jenkins jobs  
• Plugins  
• Pipeline configurations  
• Build history  

Ensuring data persists even if the container restarts.

---

#### **3. Run Jenkins Container**

```bash
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -u root \
  jenkins/jenkins:lts
```

**Explanation of Key Parameters**

```
-p 8080:8080     → Jenkins Web UI
-p 50000:50000   → Jenkins Agent Communication
-v jenkins_home  → Persistent Jenkins Data
-v docker.sock   → Host Docker Access
-u root          → Docker execution permissions
```

---

#### **4. Access Jenkins Dashboard**

Open browser and navigate to:

```
http://<ec2-public-ip>:8080
```

---

#### **5. Retrieve Initial Admin Password**

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Copy the password and paste it into the Jenkins setup screen.

---

#### **6. Install Suggested Plugins**

During initial setup:

• Selected **Install Suggested Plugins**  
• Installed pipeline, Git, Docker-related plugins  

This enabled CI/CD pipeline functionality.

---

#### **7. Create Admin User**

Configured:

• Username  
• Password  
• Email  
• Full name  

This completed Jenkins initialization.

---

#### **8. Install Docker CLI Inside Jenkins Container**

To allow Jenkins pipelines to execute Docker commands:

```bash
docker exec -u 0 -it jenkins bash
apt update
apt install docker.io -y
exit
```

---

#### **9. Install Docker Compose Plugin**

Required for multi-container deployment:

```bash
docker exec -u 0 -it jenkins bash
apt install docker-compose-plugin -y
exit
```

---

#### **10. Verify Docker Access Inside Jenkins**

```bash
docker exec -it jenkins docker ps
docker exec -it jenkins docker compose version
```

Successful output confirmed Jenkins could:

• Build images  
• Run containers  
• Execute Docker Compose deployments  

---

This completed the Jenkins setup required to run CI/CD pipelines for the 2-tier containerized application.


---

### **6. Step 4: GitHub Repository Configuration**

Ensure your GitHub repository contains the following three files.

#### **Dockerfile**
This file defines the environment for the Flask application container.
```dockerfile
# Use an official Python runtime as a parent image
FROM python:3.9-slim

# Set the working directory in the container
WORKDIR /app

# Install system dependencies required for mysqlclient
RUN apt-get update && apt-get install -y gcc default-libmysqlclient-dev pkg-config && \
    rm -rf /var/lib/apt/lists/*

# Copy the requirements file to leverage Docker cache
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the rest of the application code
COPY . .

# Expose the port the app runs on
EXPOSE 5000

# Command to run the application
CMD ["python", "app.py"]
```

#### **docker-compose.yml**
This file defines and orchestrates the multi-container application (Flask and MySQL).
```yaml
version: "3.8"

services:
  mysql:
    container_name: mysql
    image: mysql
    environment:
      MYSQL_DATABASE: "devops"
      MYSQL_ROOT_PASSWORD: "root"
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - two-tier
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-uroot", "-proot"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

  flask:
    build:
      context: .
    container_name: two-tier-app
    ports:
      - "5000:5000"
    environment:
      - MYSQL_HOST=mysql
      - MYSQL_USER=root
      - MYSQL_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DB=devops
    networks:
      - two-tier
    depends_on:
      - mysql
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 60s

volumes:
  mysql-data:

networks:
  two-tier:
```

#### **Jenkinsfile**
This file contains the pipeline-as-code definition for Jenkins.
```groovy
pipeline {
    agent any
    stages {
        stage('Clone Code') {
            steps {
                // Replace with your GitHub repository URL
                git branch: 'main', url: '[https://github.com/parkervijay/Devops-two-tier-application.git](https://github.com/parkervijay/Devops-two-tier-application.git)'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t flask-app:latest .'
            }
        }
        stage('Deploy with Docker Compose') {
            steps {
                // Stop existing containers if they are running
                sh 'docker compose down || true'
                // Start the application, rebuilding the flask image
                sh 'docker compose up -d --build'
            }
        }
    }
}
```

---

### **7. Step 5: Jenkins Pipeline Creation and Execution**

1.  **Create a New Pipeline Job in Jenkins:**
    * From the Jenkins dashboard, select **New Item**.
    * Name the project, choose **Pipeline**, and click **OK**.

2.  **Configure the Pipeline:**
    * In the project configuration, scroll to the **Pipeline** section.
    * Set **Definition** to **Pipeline script from SCM**.
    * Choose **Git** as the SCM.
    * Enter your GitHub repository URL.
    * Verify the **Script Path** is `Jenkinsfile`.
    * Save the configuration.



3.  **Run the Pipeline:**
    * Click **Build Now** to trigger the pipeline manually for the first time.
    * Monitor the execution through the **Stage View** or **Console Output**.



4.  **Verify Deployment:**
    * After a successful build, your Flask application will be accessible at `http://<your-ec2-public-ip>:5000`.
    * Confirm the containers are running on the EC2 instance with `docker ps`.

---
---

### **8. Challenges Faced & Solutions**

During the implementation of the automated CI/CD pipeline for the 2-Tier Flask and MySQL application, multiple real-time infrastructure, containerization, and orchestration challenges were encountered. This section documents the major issues faced, their root causes, and the solutions applied to successfully complete the deployment.

---

#### **Challenge 1: Jenkins Unable to Execute Docker Commands**

**Issue Observed**

Pipeline failed at the Docker stage:

```
docker: not found
```

**Root Cause**

Jenkins was running inside a Docker container without Docker CLI installed and without access to the host Docker daemon.

**Solution Implemented**

Mounted Docker socket:

```
-v /var/run/docker.sock:/var/run/docker.sock
```

Installed Docker CLI inside Jenkins container:

```bash
docker exec -u 0 -it jenkins bash
apt update
apt install docker.io -y
```

---

#### **Challenge 2: Docker Permission Denied Error**

**Issue Observed**

```
permission denied while trying to connect to docker.sock
```

**Root Cause**

Jenkins user lacked permission to access Docker daemon socket.

**Solution Implemented**

Ran Jenkins container with elevated privileges:

```bash
-u root
```

This granted Docker execution permissions.

---

#### **Challenge 3: Docker Compose Command Not Found**

**Issue Observed**

```
docker: 'compose' is not a docker command
```

**Root Cause**

Docker Compose plugin was not installed inside Jenkins container.

**Solution Implemented**

Installed Compose plugin:

```bash
apt install docker-compose-plugin -y
```

Deployment then executed successfully using:

```bash
docker compose up -d --build
```

---

#### **Challenge 4: EC2 Instance Freeze During Deployment**

**Issue Observed**

• SSH became unresponsive  
• Jenkins UI stopped loading  
• Pipeline stalled during MySQL startup  

**Root Cause**

t2.micro instance (1 GB RAM) could not handle:

```
Jenkins + Docker + MySQL + Flask + Build Processes
```

Memory exhaustion caused system freeze.

**Solution Implemented**

• Rebooted EC2 instance  
• Cleaned containers/images  
• Identified resource bottleneck  
• Recommended upgrade to t2.small  

---

#### **Challenge 5: Disk Space Exhaustion in Jenkins Home**

**Issue Observed**

Jenkins displayed disk space errors and stopped scheduling builds.

**Root Cause**

`jenkins_home` volume accumulated:

• Workspaces  
• Build logs  
• Git clones  
• Docker layers  

**Solution Implemented**

Performed cleanup:

```bash
docker system prune -a -f
rm -rf /var/lib/docker/volumes/jenkins_home/_data/workspace/*
rm -rf /var/lib/docker/volumes/jenkins_home/_data/jobs/*/builds/*
```

---

#### **Challenge 6: MySQL Container Restart Loop**

**Issue Observed**

```
Restarting (1)
```

**Root Cause**

MySQL logs showed InnoDB corruption:

```
Cannot create redo log files
Database not shut down cleanly
```

This occurred due to instance crash during DB initialization.

**Solution Implemented**

Reset MySQL volume:

```bash
docker compose down -v
docker compose up -d --build
```

Database reinitialized successfully.

---

#### **Challenge 7: Jenkins Executor Unavailable**

**Issue Observed**

```
Waiting for next available executor
```

**Root Cause**

Interrupted builds occupied executor slots after system freeze.

**Solution Implemented**

• Restarted Jenkins container  
• Terminated stuck builds  
• Verified executor configuration  

---

#### **Challenge 8: Docker Compose File Not Found (Manual Deployment)**

**Issue Observed**

```
Can't find docker-compose.yml
```

**Root Cause**

Compose command executed outside project directory.

**Solution Implemented**

Cloned repository locally:

```bash
git clone https://github.com/parkervijay/Devops-two-tier-application.git
cd Devops-two-tier-application
docker compose up -d --build
```

---

### **Key Technical Learnings**

```
• Jenkins in Docker requires socket binding.
• Docker CLI & Compose must exist inside Jenkins runtime.
• Resource sizing is critical for CI/CD + DB workloads.
• Disk monitoring is essential in build systems.
• Database volumes can corrupt after crashes.
• Health checks impact pipeline completion.
• Infrastructure failures can interrupt CI/CD.
```

---


### **9. Conclusion**
The CI/CD pipeline is now fully operational. Any `git push` to the `main` branch of the configured GitHub repository will automatically trigger the Jenkins pipeline, which will build the new Docker image and deploy the updated application, ensuring a seamless and automated workflow from development to production.

