DevSecOps: Deploying the 2048 Game on Docker & Kubernetes with Jenkins CI/CD

A production-style CI/CD pipeline for the classic 2048 web app.
This project integrates Jenkins, Docker, Kubernetes, SonarQube, OWASP Dependency-Check, Trivy, and Prometheus + Grafana for end-to-end build, test, security scanning, deployment, and monitoring.
	•	Instruction Blog: https://mohammadwithdevops.hashnode.dev/devsecops-deploying-the-2048-game-on-docker-and-kubernetes-with-jenkins-cicd
	•	Source Code: https://github.com/Shuvro-373/2048_game

⸻

📌 Highlights
	•	Automated CI/CD with Jenkins Pipeline (Groovy)
	•	Static code analysis via SonarQube
	•	Dependency vulnerabilities via OWASP Dependency-Check
	•	Image & filesystem scans via Trivy
	•	Container build & publish to Docker Hub
	•	Kubernetes deployment (apply manifests from pipeline)
	•	Observability with Prometheus and Grafana
	•	One-click local run with Docker

⸻

🗂️ Repository Structure

2048_game/
├─ Dockerfile
├─ README.md
├─ app.py
├─ deployment.yaml
├─ service.yaml
├─ nginx.conf
├─ static/
└─ templates/


⸻

🧭 System Architecture

flowchart LR
  Dev[Developer Commit] -->|Git Push| GitHub[(GitHub Repo)]
  GitHub -->|Webhook/Poll| Jenkins[Jenkins CI/CD]

  subgraph CI[CI Stages]
    A1[Clean Workspace]
    A2[Checkout]
    A3[SonarQube Analysis]
    A4[NPM Install]
    A5[OWASP Dependency-Check]
    A6[Trivy Filesystem Scan]
    A7[Docker Build & Push]
    A8[Trivy Image Scan]
  end

  Jenkins --> CI
  CI -->|Docker Image| DockerHub[(Docker Hub)]

  subgraph CD[CD Stages]
    B1[Deploy to Local Container]
    B2[Kubectl Apply Manifests]
  end

  CI --> CD
  CD -->|Pods/Service| K8s[(Kubernetes Cluster)]
  K8s -->|NodePort/Ingress| Users((End Users))

  subgraph Monitoring[Observability]
    P[Prometheus]
    G[Grafana]
  end

  K8s -->|/metrics| P
  P --> G


⸻

✅ Prerequisites
	•	Jenkins with the following tools configured:
	•	JDK 21, NodeJS 24 (Global Tool Configuration)
	•	SonarQube server (Manage Jenkins → System; name: sonar-server)
	•	Sonar Scanner tool (name: sonar-scanner)
	•	OWASP Dependency-Check (name: DP-Check)
	•	Docker on the Jenkins agent
	•	Trivy on the Jenkins agent
	•	DockerHub credentials (ID: dockerhub-creds)
	•	KubeConfig credentials (ID: k8s) for withKubeConfig{...}
	•	Kubernetes cluster accessible from Jenkins (kubeadm/minikube/managed)
	•	Docker Hub account (e.g., shuvro373/2048)

⸻

🚀 Quick Start (Local Docker)

# Build
docker build -t 2048 .

# Run
docker run -d --name 2048 -p 3000:3000 2048

# Test
curl -I http://localhost:3000


⸻

☸️ Kubernetes Deployment (Manifests)

# Create/update resources
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Check status
kubectl get pods,svc

# (If NodePort) Access via: http://<node-ip>:<nodeport>
# (If ClusterIP) Use port-forward for quick test:
kubectl port-forward svc/your-service-name 3000:3000


⸻

🔒 Security & Quality Gates
	•	SonarQube: Code quality & hotspots
	•	OWASP Dependency-Check: Third-party dependency CVEs
	•	Trivy fs: Filesystem scan before image build
	•	Trivy image: Built image vulnerabilities

Reports are archived as Jenkins artifacts.

⸻

📈 Monitoring (Prometheus + Grafana)
	•	Prometheus scrapes metrics from:
	•	Kubernetes (kube-state-metrics, node-exporter)
	•	Your application/exporters (if exposed)
	•	Grafana visualizes:
	•	Cluster health (CPU, memory, pod status)
	•	App-specific dashboards (if instrumented)

Tip: expose Prometheus/Grafana via NodePort during learning or use Ingress for production.

⸻

## 🧱 Architecture Diagram

```text
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

⸻

🧪 Full Jenkins Pipeline (Groovy)

Save this as Jenkinsfile in the repo root (or paste in a Pipeline job).

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

        stage('Deploy to kubernets') {
            steps {
                script {
                    withKubeConfig(
                        caCertificate: '',
                        clusterName: '',
                        contextName: '',
                        credentialsId: 'k8s',
                        namespace: '',
                        restrictKubeConfigAccess: false,
                        serverUrl: ''
                    ) {
                        sh 'kubectl apply -f deployment.yaml'
                    }
                }
            }
        }
    }
}


⸻

🔧 Useful Commands

# View service and open editor for quick patch (live edit)
kubectl edit svc <your-service-name>

# Check pod logs
kubectl logs -f deploy/<your-deployment-name>

# Restart a deployment
kubectl rollout restart deploy/<your-deployment-name>

# Rollback if needed
kubectl rollout undo deploy/<your-deployment-name>



⸻

📎 References
	•	Instruction Blog: DevSecOps: Deploying the 2048 Game on Docker and Kubernetes with Jenkins CI/CD
https://mohammadwithdevops.hashnode.dev/devsecops-deploying-the-2048-game-on-docker-and-kubernetes-with-jenkins-cicd
	•	Source Code: 2048 Game Repo
https://github.com/Shuvro-373/2048_game

⸻

👤 Author

Shuvro Deb Ray
DevSecOps | Cloud-Native | Kubernetes | Observability
