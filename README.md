# Zomato Clone — AWS Deployment with Terraform, Jenkins, SonarQube, Docker & Security Scanning

[![License](https://img.shields.io/badge/license-MIT-blue.svg)]()
[![Infra-as-Code](https://img.shields.io/badge/terraform-1.x+-623CE4.svg)]()
[![CI/CD](https://img.shields.io/badge/jenkins-pipelines-orange.svg)]()
[![Security](https://img.shields.io/badge/security-Aqua%20%7C%20Trivy%20%7C%20OWASP-red.svg)]()

Production-style Zomato clone deployed on AWS demonstrating:
- Infrastructure as Code (Terraform)
- CI/CD with Jenkins
- Code quality with SonarQube
- Containerization (Docker) and Node/npm-based services
- Image & dependency scanning (Trivy, Aqua) and DAST (OWASP ZAP)

IMPORTANT: Add the screenshots and architecture diagram to docs/images/ using the filenames listed in the "Screenshots" section so the README renders them correctly.

Table of Contents
- Project Overview
- Architecture (diagram)
- Screenshots (Jenkins, SonarQube, Plugins)
- Repo layout
- Getting started (local)
- Deploying to AWS (Terraform)
- CI/CD (Jenkins) — pipeline flow and Jenkinsfile
- Containerization (Docker) — Dockerfile example
- Security scanning (Trivy, Aqua, OWASP)
- Best practices & notes
- License & Contact

Project overview
----------------
This repository contains infrastructure and CI/CD automation for a Zomato-like food-ordering application. It shows how to provision AWS resources (EC2/ECR/RDS), build/publish Docker images, scan images and code, and run deployments through Jenkins pipelines using Terraform.

Architecture
------------
Place your architecture diagram at: docs/images/architecture.png

![Architecture Diagram (Image 1)](https://github.com/Naveen15github/Zomato-Clone-Deployment-using-AWS-Terraform-Jenkins-Docker/blob/8e61dfa2b898591893037621d4421dd22386499b/zomato%20clone.png)



  <img >
  <figcaption><strong>Figure —</strong> High-level CI/CD & infrastructure flow: developer → Terraform → Jenkins → SonarQube / npm / Trivy / Aqua / OWASP → Docker → AWS (ECR / ECS/EC2 / RDS).</figcaption>
</figure>

Screenshots (place images in docs/images/)
------------------------------------------
![(Image 1)](https://github.com/Naveen15github/Zomato-Clone-Deployment-using-AWS-Terraform-Jenkins-Docker/blob/8e61dfa2b898591893037621d4421dd22386499b/Screenshot%20(88).png)
![Architecture Diagram (Image 1)](https://github.com/Naveen15github/Zomato-Clone-Deployment-using-AWS-Terraform-Jenkins-Docker/blob/8e61dfa2b898591893037621d4421dd22386499b/Screenshot%20(89).png)
![Architecture Diagram (Image 1)](https://github.com/Naveen15github/Zomato-Clone-Deployment-using-AWS-Terraform-Jenkins-Docker/blob/8e61dfa2b898591893037621d4421dd22386499b/Screenshot%20(90).png)
![Architecture Diagram (Image 1)](https://github.com/Naveen15github/Zomato-Clone-Deployment-using-AWS-Terraform-Jenkins-Docker/blob/8e61dfa2b898591893037621d4421dd22386499b/Screenshot%20(91).png)
![Architecture Diagram (Image 1)](https://github.com/Naveen15github/Zomato-Clone-Deployment-using-AWS-Terraform-Jenkins-Docker/blob/8e61dfa2b898591893037621d4421dd22386499b/Screenshot%20(92).png)
![Architecture Diagram (Image 1)](https://github.com/Naveen15github/Zomato-Clone-Deployment-using-AWS-Terraform-Jenkins-Docker/blob/8e61dfa2b898591893037621d4421dd22386499b/Screenshot%20(93).png)


Repository layout (recommended)
- README.md
- terraform/            — IaC modules & environment configs (network, compute, rds, ecr, iam)
- jenkins/              — Jenkinsfiles, pipeline libraries, helper scripts
- services/
  - api/
  - web/
  - worker/
- docker/               — shared Docker base images & best-practices
- security/             — trivy/aqua/owasp configs & scan scripts
- docs/
  - images/             — screenshots & architecture diagram

Getting started (local)
-----------------------
Prerequisites:
- Node.js (LTS) + npm
- Docker
- Terraform v1.x
- AWS CLI configured with least-privilege IAM user
- Jenkins (for CI) — recommended to use an EC2 or containerized Jenkins master

Local dev quick steps:
1. Clone
   git clone https://github.com/<owner>/<repo>.git
2. Run API locally
   cd services/api
   npm ci
   npm run dev
3. Build Docker image for local testing
   docker build -t zomato-api:local -f services/api/Dockerfile .

Terraform (basic)
-----------------
1. Initialize:
   cd terraform
   terraform init
2. Plan:
   terraform plan -var-file=environments/staging.tfvars
3. Apply:
   terraform apply -var-file=environments/staging.tfvars

CI/CD (Jenkins) — pipeline flow and Jenkinsfile
-----------------------------------------------
This project uses a Jenkins declarative pipeline to run code quality checks, dependency scans, build, push and deploy. The Jenkinsfile is stored at `jenkins/Jenkinsfile` and is included below for convenience.

Key pipeline stages:
- clean workspace
- Checkout from Git
- Sonarqube Analysis
- Quality gate
- Install Dependencies
- OWASP Dependency-Check (FS scan)
- Trivy filesystem scan
- Docker Build & Push
- Trivy image scan
- Deploy to container (run)

Below is the Jenkinsfile (exact contents used by the pipeline — save it to `jenkins/Jenkinsfile`):

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/vijaygiduthuri/Zomato.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build -t zomato ."
                       sh "docker tag zomato vijaygiduthuri/zomato:latest "
                       sh "docker push vijaygiduthuri/zomato:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image vijaygiduthuri/zomato:latest > trivy.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name zomato -p 3000:3000 vijaygiduthuri/zomato:latest'
            }
        }
    }
}
```

Notes for Jenkins setup
- Install and configure tools with the same labels you reference in the file:
  - JDK tool named `jdk17`
  - NodeJS tool named `node16`
  - SonarQube Scanner tool named `sonar-scanner`
- Add credentials:
  - Jenkins credential id `Sonar-token` for Sonar token (Secret Text)
  - Jenkins credential id `docker` for DockerHub username/password (Username with password)
- Plugins required (examples):
  - Pipeline, NodeJS, SonarQube Scanner, OWASP Dependency-Check plugin, Trivy (invoked via shell), Docker Pipeline, Credentials Binding

Containerization (Docker)
-------------------------
Each service should include a small, reproducible Dockerfile and a .dockerignore file. Below is the Dockerfile you provided — include it in the service that serves the React app (for example: `services/web/Dockerfile`):

```dockerfile
# Use Node.js 16 slim as the base image
FROM node:16-slim

# Set the working directory
WORKDIR /app

# Copy package.json and package-lock.json to the working directory
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy the rest of the application code
COPY . .

# Build the React app
RUN npm run build

# Expose port 3000 (or the port your app is configured to listen on)
EXPOSE 3000

# Start your Node.js server (assuming it serves the React app)
CMD ["npm", "start"]
```

Recommended .dockerignore (create services/web/.dockerignore):
```
node_modules
dist
build
npm-debug.log
.DS_Store
.env
.git
.gitignore
```

Security scanning examples
-------------------------
- Trivy (image): trivy image --format json -o reports/trivy.json $IMAGE
- Aqua: run Aqua CLI or server-based scans and enforce policies
- OWASP ZAP (DAST): use zap-baseline.py or zap-full-scan.py against staging URL and archive reports
- OWASP Dependency-Check (SCA): run during build to find vulnerable dependencies

Best practices & important notes
--------------------------------
- Never commit secrets or tokens. Use Jenkins Credentials, AWS Secrets Manager, or SSM Parameter Store.
- Blur or redact any screenshots that expose account IDs, full public IPs, credentials, or tokens before committing.
- Use multi-stage builds and pin base images to minimize vulnerabilities.
- Treat CI quality/security gates as actionable signals; create an issue workflow for remediation.
- Keep images and reports archived in the build artifacts for traceability.


License
-------
MIT

Contact
-------
Maintainer: Naveen G (Naveen15github) — naveen6662005@gmail.com
