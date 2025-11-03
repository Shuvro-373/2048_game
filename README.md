# 🎮 DevSecOps: Deploying the 2048 Game on Docker and Kubernetes with Jenkins CI/CD

---

## 📘 Project Overview

This project demonstrates the **end-to-end implementation of a secure CI/CD DevSecOps pipeline** for deploying the **2048 Game**, a web-based puzzle application, using **Jenkins**, **Docker**, and **Kubernetes** on **AWS EC2** instances.

The pipeline ensures **continuous integration, continuous delivery, automated security scanning**, and **real-time Kubernetes cluster monitoring** using **Prometheus** and **Grafana**.

---

## 🧩 Tech Stack

| Category | Tools & Technologies |
|-----------|----------------------|
| **Version Control** | Git & GitHub |
| **CI/CD Orchestration** | Jenkins |
| **Code Quality Analysis** | SonarQube |
| **Dependency Security** | OWASP Dependency-Check |
| **Container Security** | Trivy |
| **Containerization** | Docker |
| **Orchestration & Deployment** | Kubernetes (kubeadm setup) |
| **Monitoring & Visualization** | Prometheus + Grafana |
| **Cloud Infrastructure** | AWS EC2 (Ubuntu 24.04 LTS) |
| **Programming Language** | Python / Node.js |
| **Web Server** | NGINX |

---

## 🧱 Architecture Diagram

          ┌────────────────────────────┐
          │        GitHub Repo         │
          │     (2048 Game Source)     │
          └────────────┬───────────────┘
                       │
                       ▼
          ┌────────────────────────────┐
          │         Jenkins CI/CD       │
          │   Builds | Scans | Deploys  │
          └────────────┬───────────────┘
                       │
                       ▼
          ┌────────────────────────────┐
          │      Docker Hub Images      │
          └────────────┬───────────────┘
                       │
                       ▼
          ┌────────────────────────────┐
          │  Kubernetes Cluster (AWS)  │
          │ Master + Worker Nodes      │
          └────────────┬───────────────┘
                       │
                       ▼
          ┌────────────────────────────┐
          │ Prometheus + Grafana Stack │
          │ (Monitoring & Dashboards)  │
          └────────────────────────────┘

## 📜 Complete Jenkins Declarative Pipeline

pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'nodejs24'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git 'https://github.com/Shuvro-373/2048_game.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=Game \
                        -Dsonar.projectName=Game'''
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency-Check') {
            steps {
                dependencyCheck additionalArguments: '--scan . --format XML', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                archiveArtifacts artifacts: '**/dependency-check-report.*', onlyIfSuccessful: true
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs . \
                    --format table \
                    --severity HIGH,CRITICAL \
                    --no-progress > trivyfs.txt
                '''
                archiveArtifacts artifacts: 'trivyfs.txt', onlyIfSuccessful: true
            }
        }

        stage("Docker Build & Push") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub-creds') {
                        sh '''
                            docker build -t 2048 .
                            docker tag 2048 shuvro373/2048:latest
                            docker push shuvro373/2048:latest
                        '''
                    }
                }
            }
        }

        stage("Trivy Image Scan") {
            steps {
                sh '''
                    trivy image shuvro373/2048:latest \
                    --severity HIGH,CRITICAL \
                    --format table \
                    --no-progress > trivy.txt
                '''
                archiveArtifacts artifacts: 'trivy.txt', onlyIfSuccessful: true
            }
        }

                stage('Deploy to container') {
            steps {
                sh '''
                    docker rm -f 2048 || true
                    docker run -d --name 2048 -p 3000:3000 shuvro373/2048:latest
                '''
            }
        }

        stage('Deploy to kubernets'){
            steps{
                script{
                    withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                       sh 'kubectl apply -f deployment.yaml'
                  }
                }
            }
        }
    }
}

⚙️ Pipeline Workflow

1️⃣ Code Checkout
	•	Jenkins automatically clones the latest code from GitHub: git url: 'https://github.com/Shuvro-373/2048_game', branch: 'master'

⸻

2️⃣ SonarQube Analysis
	•	Scans the source code for bugs, code smells, and vulnerabilities.
	•	Ensures code quality using the SonarQube server integrated with Jenkins.

⸻

3️⃣ OWASP Dependency Check
	•	Detects insecure or outdated dependencies.
	•	Generates an HTML and XML report for dependency vulnerabilities.

⸻

4️⃣ Trivy Filesystem & Image Scan
	•	Scans the code directory for HIGH/CRITICAL vulnerabilities: trivy fs .
	•	Scans the Docker image after build: trivy image shuvro373/2048:latest

⸻

5️⃣ Docker Build & Push
	•	Jenkins builds and pushes the Docker image to Docker Hub: docker build -t shuvro373/2048:latest .
                                                              docker push shuvro373/2048:latest
⸻

6️⃣ Kubernetes Deployment
	•	Jenkins applies the manifests:kubectl apply -f deployment.yaml
                                  kubectl apply -f service.yaml
  •	The 2048 Game runs as a Kubernetes Deployment exposed via NodePort.



⸻

7️⃣ Prometheus & Grafana Monitoring

🟦 Prometheus:
	•	Installed on the same EC2 instance as the K8s Master.
	•	Collects metrics from:
	•	Node Exporter (for system metrics)
	•	Kube-State-Metrics (for pod/deployment state)
	•	Jenkins metrics endpoint
	•	Prometheus itself

🟩 Grafana:
	•	Connected to Prometheus as a data source.
	•	Displays dashboards for:
	•	Node resource usage
	•	Pod performance
	•	Jenkins health
	•	Kubernetes service uptime

⸻

🧠 Prometheus Configuration Example

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter_master'
    static_configs:
      - targets: ['3.84.40.40:9100']

  - job_name: 'node_exporter_worker'
    static_configs:
      - targets: ['3.84.24.56:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['13.219.84.38:8080']

  - job_name: 'kube-state-metrics'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['3.84.40.40:32555']

📊 Grafana Dashboards

You visualized:
	•	CPU & memory usage per node/pod
	•	Cluster-wide uptime
	•	Jenkins job performance
	•	2048 Game service response health

⸻

🧾 Project Achievements

✅ Fully automated CI/CD with Jenkins
✅ Code & image security scanning with SonarQube, OWASP, and Trivy
✅ Containerized 2048 game deployed to Kubernetes
✅ Real-time monitoring using Prometheus & Grafana
✅ Scalable, cloud-native infrastructure using AWS EC2
