# 🚀 Netflix Clone DevSecOps CI/CD Pipeline (Jenkins + SonarQube + Trivy + Docker + Prometheus + Grafana + ArgoCD)

This README provides the complete step-by-step guide to deploy **Netflix Clone** using **Jenkins**, **SonarQube**, **Trivy**, **Docker**, **Prometheus + Grafana**, and **ArgoCD**.

> ⚠️ **Note:** You can run this setup on **AWS EC2** or on a **Local Linux Machine**.  
> This guide assumes a local Linux machine for learning purposes.

---


# ✅ Prerequisites

- Ubuntu 22.04 (Local Linux machine or AWS EC2)
- GitHub repository: `https://github.com/hariharan-k21/Netflix-Clone.git`
- Ports to be open:
  - Jenkins: **8080**
  - SonarQube: **9000**
  - ArgoCD: **8888**
  - Prometheus: **9090**
  - Grafana: **3000**
  - Docker app: **8081**

---

# 🔥 Phase 1: Initial Setup and Deployment

## Step 1: Install Git & Clone Code

```bash
sudo apt update && sudo apt upgrade && sudo apt install git -y
git clone https://github.com/hariharan-k21/Netflix-Clone.git
cd Netflix-Clone

Step 2: Install Docker & Run App

sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock

Build and run:

docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest

    ⚠️ App will fail due to missing TMDB API key.

Step 3: Get TMDB API Key

    Go to TMDB website

    Login and create account

    Go to Settings → API

    Create API Key and copy it

Build again with API key:

docker build --build-arg TMDB_V3_API_KEY=<YOUR_API_KEY> -t netflix .

🛡️ Phase 2: Security Tools Setup
Install SonarQube

docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

Access:

http://localhost:9000

Login:

admin / admin

Install Trivy

sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

Scanning Image using Trivy:

Command: trivy image <docker-imageid>

🧪 Phase 3: CI/CD Setup (Jenkins)
Install Java

sudo apt update
sudo apt install fontconfig openjdk-17-jre -y
java -version

Install Jenkins

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins

Access Jenkins:

http://localhost:8080

Install Jenkins Plugins

Go to:

Manage Jenkins → Manage Plugins → Available

Install:

    Eclipse Temurin Installer

    SonarQube Scanner

    NodeJS Plugin

    OWASP Dependency-Check

    Docker Pipeline

    Prometheus Metrics Plugin

    Docker Build

    Docker API

Configure Tools

Go to:

Manage Jenkins → Global Tool Configuration

Install:

    JDK 17

    NodeJS 16

    Sonar Scanner

    Dependency-Check

Create SonarQube Project

Open SonarQube UI:

http://localhost:9000

Login: admin/admin

Create Project:

For this use Choose form Local in SonarQube, Select OS and Program Runtime.

    Project Name: Netflix

    Project Key: Netflix

Add SonarQube Token

    SonarQube → My Account

    Generate Token

    Copy token

Add in Jenkins:

Manage Jenkins → Credentials → Add Secret Text

ID: Sonar-token
DockerHub PAT (Important)

Go to DockerHub:

Account Settings → Security → New Access Token

Create token and copy.

Add in Jenkins:

Manage Jenkins → Credentials → Add Credentials

Type: Username with password

    Username: your-dockerhub-username

    Password: <PAT>

    ID: docker

Add SonarQube Webhook for Jenkins

Go to SonarQube:

Administration → Configuration → Webhooks

Add webhook:

    Name: Jenkins

    URL: http://localhost:8080/sonarqube-webhook/
    
From above URL, localhost won't work for setting up webhook in SonarQube, use the command hostname -I in your local machine, and put your machine's IP Address, you can verify by, it will return Jenkins Page.

🔍 Jenkinsfile Scripts
Jenkinsfile 1 (Phase 1)

pipeline {
    agent any

    tools {
        jdk 'jdk17' // Make sure JDK 17 is installed in Jenkins tools
        nodejs 'node16' // Ensure Node.js version 16 is installed in Jenkins tools
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner' // Set Sonar Scanner as an environment variable
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs() // Clean the workspace to avoid conflicts from previous builds
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/hariharan-k21/Netflix-Clone.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Wait for the SonarQube quality gate to pass
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                // Install project dependencies using npm
                sh "npm install"
            }
        }
    }

    post {
        // This will run regardless of the pipeline result
        always {
            echo 'Cleaning up after the build'
            cleanWs() // Always clean the workspace to ensure a fresh environment for the next build
        }
    }
}

Jenkinsfile 2 (Phase 2 + Docker + Trivy)

pipeline {
    agent any

    tools {
        // Make sure these tools are configured in Jenkins (Manage Jenkins → Global Tool Configuration)
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        // Sets the path for Sonar Scanner tool installed in Jenkins
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                // Deletes previous build files to ensure clean build
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                // Clones the project repository from GitHub
                git branch: 'main', url: 'https://github.com/hariharan-k21/Netflix-Clone.git'
            }
        }

        stage('Sonarqube Analysis') {
            steps {
                // Inject SonarQube env variables (server URL, token etc.)
                withSonarQubeEnv('sonar-server') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Wait for SonarQube quality gate result
                    def qg = waitForQualityGate()

                    // If Quality Gate fails, mark build as UNSTABLE
                    if (qg.status != 'OK') {
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                // Install node modules required for build
                sh "npm install"
            }
        }

        stage('OWASP FS SCAN') {
            steps {
                script {
                    // Run OWASP Dependency Check and store exit code
                    def rc = sh(
                        script: "dependencyCheck --scan ./ --disableYarnAudit --disableNodeAudit --format ALL --out dependency-check-report",
                        returnStatus: true
                    )

                    // If dependency-check is not installed, skip this stage
                    if (rc != 0) {
                        echo "Dependency-Check not installed. Skipping this stage."
                    } else {
                        // Publish the report to Jenkins
                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    }
                }
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                script {
                    // Run Trivy filesystem scan and store results
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        sh "trivy fs . > trivyfs.txt"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    // Login to Docker Hub using Jenkins credentials
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {

                        // Build Docker image with TMDB API key
                        sh "docker build --build-arg TMDB_V3_API_KEY=<USE_YOUR_OWN_API_KEY_FROM_TMDB> -t netflix ."

                        // Tag the image
                        sh "docker tag netflix hariharank21/netflix:latest"

                        // Push to Docker Hub
                        sh "docker push hariharank21/netflix:latest"
                    }
                }
            }
        }

        stage('TRIVY Image Scan') {
            steps {
                script {
                    // Scan Docker image for vulnerabilities
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        sh "trivy image hariharank21/netflix:latest > trivyimage.txt"
                    }
                }
            }
        }

        stage('Deploy to container') {
            steps {
                // Run Docker container from the pushed image
                sh "docker run -d --name netflix -p 8081:80 hariharank21/netflix:latest"
            }
        }
    }
}

🔍 Phase 4: Monitoring (Prometheus + Grafana)

Install Prometheus

sudo useradd --system --no-create-home --shell /bin/false prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
cd prometheus-2.47.1.linux-amd64/
sudo mkdir -p /data /etc/prometheus
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

Create systemd service:

sudo nano /etc/systemd/system/prometheus.service

Add:

[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online-target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target

Enable and start:

sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus

Access:

http://localhost:9090

Install Node Exporter

sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*

Create service:

sudo nano /etc/systemd/system/node_exporter.service

Add:

[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online-target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target

Enable and start:

sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter

Configure Prometheus to Scrape Metrics

Edit:

sudo nano /etc/prometheus/prometheus.yml

Add:

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['localhost:8080']

Check config:

promtool check config /etc/prometheus/prometheus.yml

Reload:

curl -X POST http://localhost:9090/-/reload

📊 Grafana Setup

sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get -y install grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server

Access:

http://localhost:3000

Login:

admin/admin

Change password.

Add Prometheus Data Source:

Configuration → Data Sources → Add → Prometheus
URL: http://localhost:9090

Import Dashboard:

Create → Import → Dashboard ID: <dashboard-id>, download the JSON and put the contents in the Grafana's JSON Space.

🧩 Phase 5: Kubernetes + ArgoCD (Manual Sync)

Install Minikube

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64

sudo usermod -aG docker $USER && newgrp docker

minikube start --driver=docker

To begin monitoring your Kubernetes cluster, you'll install the Prometheus Node Exporter. This component allows you to collect system-level metrics from your cluster nodes. In Minikube HELM is installed by default, if not install HELM by referring official documentation Here are the steps to install the Node Exporter using HELM:

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

kubectl create namespace prometheus-node-exporter

helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter

Update your Prometheus configuration (prometheus.yml) to add a new job for scraping metrics from localhost:9001/metrics. You can do this by adding the following configuration to your prometheus.yml file:

  - job_name: 'Netflix'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['localhost:9100']

Install ArgoCD

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.4/manifests/install.yaml
kubectl get all -n argocd

Access ArgoCD UI

kubectl port-forward svc/argocd-server -n argocd 8888:443

Open:

http://localhost:8888

Get password:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

Login:

admin / <password>

⚙️ Manual Sync (for learning purpose)

    Keep auto-sync OFF. You will manually sync apps to deploy updates.

🧹 Phase 6: Cleanup

Terminate or stop services if not needed.

